<script>
import { get, values, pick } from 'lodash'
import { mapState, mapGetters } from 'vuex'
import { date } from 'quasar'

import EventBus from 'src/utils/event-bus'
import { extractLocationDataFromPlace } from 'src/utils/places'
import logger from 'src/utils/logger'

import HeroTwoColLayout from 'src/layouts/HeroTwoColLayout'

import AppGalleryUploader from 'src/components/AppGalleryUploader'
import CustomAttributesEditor from 'src/components/CustomAttributesEditor'
import DateRangePicker from 'src/components/DateRangePicker'
import PlacesAutocomplete from 'src/components/PlacesAutocomplete'
import SelectAssetType from 'src/components/SelectAssetType'
import SelectCategories from 'src/components/SelectCategories'

import PageComponentMixin from 'src/mixins/pageComponent'

export default {
  components: {
    HeroTwoColLayout,
    AppGalleryUploader,
    CustomAttributesEditor,
    DateRangePicker,
    PlacesAutocomplete,
    SelectAssetType,
    SelectCategories,
  },
  mixins: [
    PageComponentMixin,
  ],
  data () {
    return {
      name: '',
      nameMaxLength: 70,
      description: '',
      descriptionMaxLength: 2000,
      price: null,
      startDate: '',
      endDate: '',
      quantity: 1,
      locations: [],
      options: ['option1'],
      selectedCategory: null,
      editingAssetType: null,
      visibleStep: 1,
      requestAuthentication: false,
      editingCustomAttributes: {},
      assetImages: [],
      newUserImages: [], // to add to user images for reuse
      uploaderFiles: [],
      creatingAsset: false,
    }
  },
  computed: {
    defaultAssetType () {
      const assetTypesConfig = get(this.common.config, 'stelace.instant.assetTypes')
      if (!assetTypesConfig) return this.assetTypes[0]

      let defaultType

      const assetTypesIds = Object.keys(assetTypesConfig)
      assetTypesIds.forEach(assetTypeId => {
        if (defaultType) return

        const assetTypeConfig = assetTypesConfig[assetTypeId]
        if (assetTypeConfig.isDefault) {
          defaultType = this.assetTypes.find(assetType => assetType.id === assetTypeId)
        }
      })

      if (defaultType) return defaultType
      else return this.assetTypes[0]
    },
    showSelectAssetType () {
      if (this.filteredAssetTypesIds) return this.filteredAssetTypesIds.length > 1
      else return this.assetTypes.length > 1
    },
    filteredAssetTypesIds () {
      const config = this.common.config
      const assetTypesConfig = get(config, 'stelace.instant.assetTypes')

      if (!assetTypesConfig) return null

      return Object.keys(assetTypesConfig)
    },
    isAssetTypeReadonly () {
      return !!this.asset.asset.id
    },
    selectedAssetType () {
      if (!this.asset.asset.id) {
        if (this.editingAssetType) return this.editingAssetType
        else {
          if (this.assetTypes.length === 1) return this.assetTypes[0]
          else return this.assetTypes.find(assetType => assetType.id === this.defaultAssetType.id) || null
        }
      } else {
        return this.common.assetTypesById[this.asset.asset.assetTypeId] || {}
      }
    },
    editableCustomAttributeNames () {
      if (!this.selectedAssetType) return []

      const config = this.common.config
      const assetTypesConfig = get(config, 'stelace.instant.assetTypes')

      if (!assetTypesConfig) return []

      const assetTypeConfig = assetTypesConfig[this.selectedAssetType.id]
      if (!assetTypeConfig) return []

      return assetTypeConfig.customAttributes || []
    },
    customAttributes () {
      const attrs = this.common.customAttributesById
      const activeNames = this.editableCustomAttributeNames

      return values(attrs).filter((ca) => activeNames.includes(ca.name))
    },
    assetTypes () {
      return values(this.common.assetTypesById)
    },
    showAvailabilityDates () {
      if (!this.selectedAssetType) return false

      return this.selectedAssetType.timeBased && !this.selectedAssetType.infiniteStock
    },
    locationName () {
      const locations = this.locations
      return get(locations, '[0].shortDisplayName', '')
    },
    reusableImages () {
      return this.currentUser.id ? this.currentUser.images : []
    },
    ...mapState([
      'asset',
      'common',
      'content',
      'style',
    ]),
    ...mapGetters([
      'currentUser',
      'canPublishAsset',
    ]),
  },
  async preFetch ({ store }) {
    await store.dispatch('initEditAssetPage')
  },
  async mounted () {
    EventBus.$on('authStatusChanged', (status) => this.onAuthChange(status))
  },
  beforeDestroy () {
    EventBus.$off('authStatusChanged', (status) => this.onAuthChange(status))
  },
  methods: {
    afterAuth () {
      if (this.currentUser.id && this.currentUser.locations.length) {
        this.locations = [this.currentUser.locations[0]]
      }
    },
    changeCustomAttributes (customAttributes) {
      this.editingCustomAttributes = customAttributes
    },
    selectAssetType (assetType) {
      this.editingAssetType = assetType
    },
    selectCategory (category) {
      this.selectedCategory = category
    },
    onAuthChange (status) {
      if (status === 'success' && this.requestAuthentication) {
        this.requestAuthentication = false

        this.createAsset()
      } else if (status === 'closed') {
        this.requestAuthentication = false
      }
    },
    uploaderFilesChanged (files) {
      this.uploaderFiles = files
      this.assetImages = files
    },
    removeImage (removed) {
      this.newUserImages = this.newUserImages.filter(img => img.name !== removed.name)
    },
    uploadCompleted ({ transformedUploadedFiles, uploadedOrReused }) {
      this.assetImages = uploadedOrReused
      this.newUserImages = transformedUploadedFiles

      if (this.creatingAsset) this.createAsset()
    },
    async createAsset () {
      if (!this.currentUser.id) {
        this.requestAuthentication = true
        this.openAuthDialog({ action: 'createAsset' })
        return
      }

      // prevents unauthorized users from creating asset after authentication
      if (!this.checkAccessOrRedirect('createAsset')) return

      if (this.step > 3) {
        try {
          const uploadPending = this.uploaderFiles.length && this.assetImages.length < this.uploaderFiles.length
          this.creatingAsset = true

          // if upload is processing, do not perform the asset creation logic right now
          // the logic will be triggered at the end of the upload from afterUploadCompleted
          if (uploadPending) return

          const images = this.assetImages.map(img => {
            delete img.reused
            return img
          })

          let assetQuantity = this.quantity

          const shouldCreateAvailability = this.showAvailabilityDates && this.startDate

          const availabilityAttrs = {}

          if (shouldCreateAvailability) {
            const isAnUnavailability = !this.endDate

            if (!isAnUnavailability) {
              availabilityAttrs.startDate = this.startDate
              availabilityAttrs.endDate = this.endDate
              availabilityAttrs.quantity = this.quantity

              // we want the asset to be available only during the availability period
              assetQuantity = 0
            } else {
              availabilityAttrs.startDate = date.addToDate(new Date(), { year: -1 }).toISOString()
              availabilityAttrs.endDate = this.startDate
              availabilityAttrs.quantity = 0
            }
          }

          const attrs = {
            // autogrow on name QInput makes it a textarea, with possible line returns
            name: this.name.replace('\n', ''),
            assetTypeId: this.selectedAssetType.id,
            description: this.description,
            price: this.price,
            quantity: this.selectedAssetType.infiniteStock ? 1 : assetQuantity,
            locations: this.locations,
            categoryId: this.selectedCategory ? this.selectedCategory.id : null,
            customAttributes: pick(this.editingCustomAttributes, this.editableCustomAttributeNames),
            active: true,
            validated: this.canPublishAsset,
            metadata: {
              images,
              // Save dates to create custom availabilities with Workflows
              startDate: this.startDate,
              endDate: this.endDate
            }
          }

          if (this.content.currency) {
            attrs.currency = this.content.currency
          }

          const asset = await this.$store.dispatch('createAsset', { attrs })

          if (shouldCreateAvailability) {
            availabilityAttrs.assetId = asset.id
            await this.$store.dispatch('createAvailability', { attrs: availabilityAttrs })
          }

          this.notifySuccess('notification.saved')
          // this.resetForm() // useful when not keeping the user on the current page

          const updateUserImages = !!this.newUserImages.length
          const updateUserLocations = !!(!this.currentUser.locations.length && asset.locations.length)
          const needUpdateUser = updateUserImages || updateUserLocations

          if (needUpdateUser) {
            const attrs = {}

            if (updateUserImages) {
              const currentUserImages = this.currentUser.images.map(img => img.name)
              // Ensuring we create no duplicate if user has logged in after uploading existing images
              const dedup = this.newUserImages.filter(img => !currentUserImages.includes(img.name))

              attrs.images = this.currentUser.images.concat(dedup)
            }
            if (updateUserLocations) {
              attrs.locations = [asset.locations[0]]
            }

            try { // not awaiting this since it’s not critical
              this.$store.dispatch('updateUser', {
                userId: this.currentUser.id,
                attrs
              })
            } catch (e) {
              logger(e)
            }
          }

          this.creatingAsset = false

          // Show that the asset is ready…
          this.$router.push({ name: 'asset', params: { id: asset.id } })

          // …and prompt to validate account if needed
          if (!this.canPublishAsset) {
            this.$q.dialog({
              title: this.$t({ id: 'user.account.validation_header' }),
              message: this.$t({ id: 'user.account.validation_required_message' }),
              ok: {
                label: this.$t({ id: 'prompt.validate' }),
                color: 'positive',
                class: 'q-ma-sm'
              }
            }).onOk(() => {
              this.$router.push({ name: 'publicProfile', params: { id: this.currentUser.id } })
            })
          }
        } catch (err) {
          this.creatingAsset = false

          this.notifyWarning('error.unknown_happened_header')
        }
      }
    },
    /* resetForm () {
      this.name = ''
      this.description = ''
      this.price = null
      this.startDate = ''
      this.endDate = ''
      this.quantity = 1
      this.locations = []
      this.selectedCategory = null
      this.editingCustomAttributes = {}

      this.resetUploader()
    }, */
    selectStartDate (startDate) {
      this.startDate = startDate
    },
    selectEndDate (endDate) {
      this.endDate = endDate
    },
    forceEndDateAfterStartDate (date) {
      return new Date(date).toISOString() >= this.startDate
    },
    selectPlace (place) {
      this.locations = place ? [extractLocationDataFromPlace(place)] : []
    },
  }
}
</script>

<template>
  <HeroTwoColLayout>
    <template v-slot:heroContent>
      <q-list
        dense
        number
        padding
        class="text-left"
      >
        <q-item
          class="q-pa-md text-h5"
        >
          <q-item-section>
            {{ $t({ id: 'pages.recruiters.itemOne' }) }}
          </q-item-section>
        </q-item>
        <q-item
          class="q-pa-md text-h5"
        >
          <q-item-section>
            {{ $t({ id: 'pages.recruiters.itemTwo' }) }}
          </q-item-section>
        </q-item>
        <q-item
          class="q-pa-md text-h5"
        >
          <q-item-section>
            {{ $t({ id: 'pages.recruiters.itemThree' }) }}
          </q-item-section>
        </q-item>
      </q-list>
    </template>

    <section class="q-pa-sm">
      <form
        class="text-center stl-content-container stl-content-container--large margin-h-center q-mb-xl"
        @submit.prevent="createAsset"
      >
        <div class="step-1 q-py-lg">
          <div class="q-mt-md row justify-center">
            <SelectAssetType
              v-if="showSelectAssetType"
              :filtered-ids="filteredAssetTypesIds"
              :initial-asset-type="selectedAssetType"
              :label="$t({ id: 'asset.asset_type_label' })"
              :show-search-icon="false"
              :rules="[
                selectedAssetType => !!selectedAssetType ||
                  $t({ id: 'form.error.missing_field' })
              ]"
              :readonly="isAssetTypeReadonly"
              square
              outlined
              @change="selectAssetType"
            />
          </div>
          <div class="row justify-center">
            <QInput
              v-model="name"
              class="name-input"
              :label="$t({ id: 'asset.name_label' })"
              :counter="name.length > nameMaxLength / 2"
              :maxlength="nameMaxLength"
              :rules="[
                name => !!name ||
                  $t({ id: 'form.error.missing_title' })
              ]"
              debounce="1000"
              autogrow
              square
              outlined
              required
            />
          </div>
        </div>

        <transition enter-active-class="animated fadeInUp">
          <div
            class="step-2 q-py-lg"
          >
            <div class="row justify-between">
              <div class="flex-item--grow-shrink-auto q-pr-lg">
                <SelectCategories
                  :initial-category="selectedCategory"
                  :label="$t({ id: 'asset.category_label' })"
                  :show-search-icon="false"
                  :rules="[
                    selectedCategory => !!selectedCategory ||
                      $t({ id: 'form.error.missing_field' })
                  ]"
                  square
                  outlined
                  @change="selectCategory"
                />
              </div>
              <div style="flex: 1 2 auto">
                <QInput
                  v-model="price"
                  type="number"
                  :label="$t({ id: 'pricing.price_label' })"
                  :rules="[
                    price => Number.isFinite(parseFloat(price)) ||
                      $t({ id: 'form.error.missing_price' })
                  ]"
                  required
                  square
                  outlined
                />
              </div>
            </div>
          </div>
        </transition>

        <transition enter-active-class="animated fadeInUp">
          <div
            class="step-3 q-py-lg"
          >
            <DateRangePicker
              v-show="showAvailabilityDates"
              class="q-mb-xl"
              :start-date="startDate"
              :end-date="endDate"
              :missing-end-date-meaning="$t({ id: 'time.missing_end_date_meaning' })"
              start-date-required
              square
              outlined
              @changeStartDate="selectStartDate"
              @changeEndDate="selectEndDate"
            />

            <div class="row justify-between">
              <div class="col-sm-5">
                <PlacesAutocomplete
                  :label="$t({ id: 'places.address_placeholder' })"
                  :initial-query="locationName"
                  square
                  outlined
                  @selectPlace="selectPlace"
                />
              </div>
              <div class="col-sm-5">
                <QInput
                  v-show="!selectedAssetType.infiniteStock"
                  v-model="quantity"
                  :label="$t({ id: 'asset.quantity_label' })"
                  required
                  type="number"
                  min="0"
                  :rules="[
                    quantity => parseFloat(quantity) > 0 ||
                      $t({ id: 'form.error.missing_field' })
                  ]"
                  square
                  outlined
                />
              </div>
            </div>
          </div>
        </transition>

        <transition enter-active-class="animated fadeInUp">
          <div
            class="step-4 q-py-lg"
          >
            <div class="row justify-between">
              <div class="col-12 col-md-7">
                <QInput
                  v-model="description"
                  class="q-mb-md"
                  :label="$t({ id: 'asset.description_label' })"
                  :maxlength="descriptionMaxLength"
                  :counter="description.length > (descriptionMaxLength / 2)"
                  :rules="[
                    description => !!description ||
                      $t({ id: 'form.error.missing_description' })
                  ]"
                  type="textarea"
                  square
                  outlined
                  required
                />
              </div>
              <div class="col-12 col-sm-6 col-md-5 q-pl-md">
                <CustomAttributesEditor
                  :definitions="customAttributes"
                  :values="editingCustomAttributes"
                  @change="changeCustomAttributes"
                />
              </div>
            </div>

            <div class="step-asset-picture q-py-lg">
              <AppContent
                class="text-h5"
                tag="h3"
                entry="pages"
                field="new_asset.picture_incentive"
              />
              <AppGalleryUploader
                :reused-images="reusableImages"
                @uploader-files-changed="uploaderFilesChanged"
                @upload-completed="uploadCompleted"
                @remove="removeImage"
              />
            </div>

            <QBtn
              class="q-my-md text-weight-bold"
              :loading="creatingAsset"
              :label="$t( { id: 'prompt.ad_button' })"
              :rounded="style.roundedTheme"
              :disabled="!uploaderFiles.length"
              color="secondary"
              size="lg"
              type="submit"
            />
            <AppContent
              v-show="!uploaderFiles.length"
              tag="div"
              class="text-body2 text-center text-grey q-pa-sm"
              entry="form"
              field="error.missing_image"
            />
          </div>
        </transition>
      </form>
    </section>
  </HeroTwoColLayout>
</template>

<style lang="stylus" scoped>
.name-input
  // Using flex-basis instead of max-width for IE11
  // https://github.com/philipwalton/flexbugs#flexbug-17
  flex: 1 0 30rem
  min-width: 0
</style>
