<script>
import { mapState, mapGetters } from 'vuex'
import * as mutationTypes from 'src/store/mutation-types'

import AccessComponent from 'src/components/AccessComponent'
import AppLogo from 'src/components/AppLogo'
import AppMiniLogo from 'src/components/AppMiniLogo'
import SearchToolbar from 'src/components/SearchToolbar'
import SelectCategories from 'src/components/SelectCategories'
import PlacesAutocomplete from 'src/components/PlacesAutocomplete'

import AuthDialogMixin from 'src/mixins/authDialog'

import { isProvider, populateUser } from 'src/utils/user'

export default {
  components: {
    AccessComponent,
    AppLogo,
    AppMiniLogo,
    PlacesAutocomplete,
    SearchToolbar,
    SelectCategories,
  },
  mixins: [
    AuthDialogMixin,
  ],
  data () {
    return {
      priceInputTouched: false,
      searchByCategory: process.env.VUE_APP_SEARCH_BY_CATEGORY === 'true',
      creatingOrganization: false,
    }
  },
  computed: {
    isMenuOpened () {
      return this.layout.isMenuOpened
    },
    isHome () {
      return this.route.name === 'home'
    },
    isSearch () {
      return this.route.name === 'search'
    },
    isRecruiters () {
      return this.route.name === 'recruiters'
    },
    selectedCategory () {
      const selectedCategoryId = this.search.searchFilters.filters.categoryId
      if (!selectedCategoryId) return null

      const category = this.common.categoriesById[selectedCategoryId]
      return category
    },
    maximumPrice () {
      if (!this.priceInputTouched) return ''

      return this.search.priceRange.max
    },
    userConversations () {
      return this.conversations.filter(c => !c.isEmpty)
    },
    hasConversations () {
      return this.userConversations.length > 0
    },
    nbUnreadConversations () {
      return this.userConversations.reduce((count, c) => count + !c.read, 0)
    },
    accountName () {
      return this.currentUser.firstName // TODO: i18n for name/familyName order
        ? `${this.currentUser.firstName} ${this.currentUser.lastName || ''}` : this.currentUser.displayName
    },
    showAccountAvatar () {
      return this.$q.screen.gt.xs
    },
    isCurrentUserProvider () {
      return isProvider(this.currentUser)
    },
    ...mapState([
      'common',
      'content',
      'layout',
      'search',
      'style',
      'route',
      'auth',
    ]),
    ...mapGetters([
      'conversations',
      'currentUser',
      'isPremium',
      'currentOrganizations',
      'defaultSearchMode',
      'canCreateOrganization'
    ]),
  },
  watch: {
    async currentUser (current, previous) {
      if (current.id !== previous.id) {
        this.$store.dispatch('selectSearchMode', { searchMode: this.defaultSearchMode })

        if (isProvider(current) && current.locations.length) {
          const loc = current.locations[0]

          this.$store.commit({
            type: mutationTypes.SET_SEARCH_LOCATION,
            queryLocation: loc.shortDisplayName,
            latitude: loc.latitude,
            longitude: loc.longitude
          })
        }

        if (this.isSearch) this.searchAssets()
        if (current.id) await this.$store.dispatch('fetchMessages')
      }
    },
  },
  methods: {
    toggleMenu (visible = !this.isMenuOpened) {
      this.$store.commit(mutationTypes.LAYOUT__TOGGLE_MENU, { visible })
    },
    logout () {
      this.$store.dispatch('logout')

      if (this.$route.meta.mustBeLogged) {
        this.$router.push({ path: '/' })
      }
    },
    async searchAssets () {
      if (this.isSearch) {
        await this.$store.dispatch('searchAssets')
      } else {
        this.$router.push({ name: 'search' })
      }
    },
    updateQuery (query) {
      this.$store.commit(mutationTypes.SET_SEARCH_QUERY, { query })

      this.searchAssets()
    },
    selectCategory (category) {
      this.$store.commit({
        type: mutationTypes.SEARCH__SET_SEARCH_FILTERS,
        filters: {
          categoryId: category && category.id
        }
      })

      this.searchAssets()
    },
    selectPlace (place) {
      if (place) {
        this.$store.commit({
          type: mutationTypes.SET_SEARCH_LOCATION,
          queryLocation: place.shortDisplayName,
          latitude: place.latitude,
          longitude: place.longitude
        })
      } else {
        this.$store.commit({
          type: mutationTypes.UNSET_SEARCH_LOCATION
        })
      }

      this.$store.commit({
        type: mutationTypes.SEARCH__SET_MAP_OPTIONS,
        useMapCenter: false,
        latitude: null,
        longitude: null
      })

      this.searchAssets()
    },
    updateMaxPrice (maximumPrice) {
      if (isNaN(maximumPrice)) return

      this.priceInputTouched = true

      let maxPrice
      if (maximumPrice === '') {
        maxPrice = this.search.priceDefaultMax
        this.priceInputTouched = false
      } else {
        maxPrice = maximumPrice
      }

      this.$store.commit(mutationTypes.SET_PRICE_RANGE, {
        min: this.search.priceRange.min,
        max: maxPrice
      })

      this.searchAssets()
    },
    resetMaxPrice () {
      this.priceInputTouched = false

      this.$store.commit(mutationTypes.SET_PRICE_RANGE, {
        min: this.search.priceRange.min,
        max: this.search.priceDefaultMax
      })

      this.searchAssets()
    },
    async createOrganization () {
      this.creatingOrganization = true

      const mainOrganization = this.currentOrganizations[0]
      populateUser(mainOrganization, { isCurrentUser: true })

      const newOrgNameWithSuffix = this.$t({ id: 'user.account.org_name_with_number_suffix' }, {
        name: mainOrganization.displayName,
        number: this.currentOrganizations.length + 1
      })

      try {
        await this.$store.dispatch('createOrganization', {
          attrs: {
            displayName: newOrgNameWithSuffix,
            email: mainOrganization.email,
            organizations: {
              [mainOrganization.id]: {}
            },
            metadata: {
              _private: {
                phone: mainOrganization.phone,
                taxId: mainOrganization.taxId,
              }
            }
          }
        }, { stelaceOrganizationId: mainOrganization.id })

        this.$refs.accountMenu.hide()

        this.$router.push({ name: 'publicProfile', params: { id: this.currentUser.id } })
      } catch (err) {
        this.notifyWarning('error.unknown_happened_header')
      } finally {
        this.creatingOrganization = false
      }
    },
    async selectOrganization (org) {
      await this.$store.dispatch('selectOrganization', { organizationId: org.id })
      this.$router.push({ name: 'publicProfile', params: { id: this.currentUser.id } })
    }
  }
}
</script>

<template>
  <QHeader
    reveal
    :bordered="!isHome && !isMenuOpened"
    :reveal-offset="100"
    :class="[
      isHome ? 'q-pa-md transparent-header' : 'bg-white',
      isMenuOpened ? 'header--raise-above-menu-dialog' : ''
    ]"
  >
    <QToolbar>
      <AppLink
        v-if="showAccountAvatar"
        :class="[isHome ? '' : 'text-primary', 'logo-container anchor-text--reset cursor-pointer q-mr-sm']"
        :to="{ name: 'home' }"
        :aria-label="$t({ id: 'navigation.home' })"
        flat
      >
        <AppLogo class="company-logo gt-xs q-mr-sm" />
      </AppLink>

      <QBtn
        v-else
        :class="[isHome ? '' : 'text-primary', 'logo-container q-mr-sm']"
        :aria-label="$t({ id: 'navigation.menu' })"
        flat
        @click="toggleMenu"
      >
        <AppMiniLogo class="company-mini-logo current-color xs" />
      </QBtn>

      <div
        v-show="!isHome && !isMenuOpened && !isRecruiters"
        class="header__search-bar row no-wrap shadow-2 q-px-sm"
      >
        <QInput
          v-if="!searchByCategory"
          :value="search.query"
          :input-class="search.query ? 'text-right' : ''"
          :label="$t({ id: 'form.search.query_placeholder' })"
          :debounce="500"
          dense
          @input="updateQuery"
        >
          <template v-slot:prepend>
            <QBtn
              :aria-label="$t({ id: 'form.search.query_placeholder' })"
              color="primary"
              icon="search"
              flat
              dense
              rounded
              @click="searchAssets"
            />
          </template>
          <template v-slot:append>
            <QIcon
              v-show="search.query.length"
              class="cursor-pointer"
              name="close"
              @click="updateQuery('')"
            />
          </template>
        </QInput>
        <SelectCategories
          v-if="searchByCategory"
          :initial-category="selectedCategory"
          :hide-input-on-select="true"
          :label="$t({ id: 'form.search.query_placeholder' })"
          :icon-button-action="searchAssets"
          dense
          icon-color="primary"
          search-icon-position="left"
          @change="selectCategory"
        />
        <PlacesAutocomplete
          class="gt-sm"
          :label="$t({ id: 'form.search.near_location_placeholder' })"
          :hide-input-on-select="true"
          icon-color="grey-4"
          search-icon-position="left"
          read-search-store
          dense
          @selectPlace="selectPlace"
        />

        <QInput
          :value="maximumPrice"
          :label="$t({ id: 'form.search.maximum_price' })"
          class="gt-md"
          input-class="text-right"
          type="number"
          min="0"
          dense
          @input="updateMaxPrice"
        >
          <template v-slot:prepend>
            <QIcon
              :name="content.currency === 'EUR' ? 'euro_symbol' : 'attach_money'"
              color="grey-4"
            />
          </template>
          <template v-slot:append>
            <QIcon
              :class="['cursor-pointer', maximumPrice ? '' : 'hidden']"
              name="close"
              @click="resetMaxPrice"
            />
          </template>
        </QInput>
      </div>

      <div class="q-mx-md text-weight-medium">
        <div
          v-show="isRecruiters"
          class="text-default-color"
        >
          {{ $t({ id: 'pages.recruiters.tagline' }) }}
        </div>
      </div>

      <QSpace />

      <QBtn
        v-if="currentUser.id"
        v-show="hasConversations"
        :to="{ name: 'inbox' }"
        :class="['q-mx-md header__inbox', isMenuOpened ? 'invisible' : '']"
        :aria-label="$t({ id: 'navigation.inbox' })"
        :color="isHome ? 'white' : ( style.colorfulTheme ? 'primary' : 'default-color')"
        flat
        round
        icon="mail"
      >
        <QBadge
          v-show="nbUnreadConversations"
          class="inbox-badge"
          color="red"
        >
          {{ nbUnreadConversations }}
        </QBadge>
      </QBtn>

      <QBtn
        v-show="!currentUser.id && !isRecruiters"
        flat
        no-caps
        :class="[
          'flex-item--auto q-mx-md text-weight-medium gt-xs',
          isHome ? 'text-white' : 'text-default-color',
          isMenuOpened ? 'invisible' : ''
        ]"
        @click="openAuthDialog({ redirectAfterSignup: true })"
      >
        {{ $t({ id: 'authentication.log_in_button' }) }}
      </QBtn>

      <!-- Hidden right drawer -->
      <!-- <QBtn dense flat round icon="menu" @click="right = !right"/> -->

      <AccessComponent action="viewCreateAssetCta">
        <QBtn
          v-show="!isRecruiters"
          class="create-assset-button q-px-md flex-item--auto"
          :to="{ name: 'newAsset' }"
          :loading="content.fetchingContentStatus"
          :rounded="style.roundedTheme"
          :label="$t({ id: 'navigation.new_listing' })"
          icon="add_box"
          color="secondary"
          align="between"
          dense
        />
      </AccessComponent>

      <QBtn
        v-if="currentUser.id && showAccountAvatar"
        class="q-ml-sm"
        flat
      >
        <AppAvatar
          :user="currentUser"
          size="2rem"
        />
        <QMenu ref="accountMenu" content-class="header__account-menu">
          <div class="q-pa-md">
            <div class="text-weight-medium q-mb-md">
              <AppContent
                class="text-h6"
                entry="navigation"
                field="account"
              />
              <AppContent
                v-if="isPremium"
                class="text-uppercase non-selectable q-ml-md"
                tag="QChip"
                entry="user"
                field="account.premium_label"
                square
                color="secondary"
                text-color="white"
              />
            </div>
            <div
              v-if="!isCurrentUserProvider || !canCreateOrganization"
              class="text-center text-body1 text-weight-medium q-mb-md"
            >
              {{ accountName }}
            </div>
            <div
              v-else
              class="text-center q-mb-md"
            >
              <QSelect
                :value="currentUser"
                :options="currentOrganizations"
                :option-value="org => org && org.id"
                :option-label="org => org && org.displayName"
                class="organizations-select q-mb-sm"
                @input="org => selectOrganization(org)"
              />
              <QBtn
                :loading="creatingOrganization"
                flat
                @click="createOrganization"
              >
                <AppContent
                  entry="user"
                  field="account.create_organization_button"
                />
              </QBtn>
            </div>
            <div class="column items-stretch">
              <!-- TODO: switch organizations / create new new ones as provider -->
              <AppContent
                v-close-popup
                tag="QBtn"
                entry="user"
                field="account.update_profile_button"
                :to="{ name: 'publicProfile', params: { id: currentUser.id } }"
                :rounded="style.roundedTheme"
                class="q-mb-md self-center"
                color="primary"
              />
              <QBtn
                class="q-mb-sm"
                align="left"
                flat
                @click="openAuthDialog({ formType: 'changePassword' })"
              >
                <QIcon
                  name="lock"
                  :left="true"
                />
                <AppContent
                  entry="user"
                  field="account.new_password_label"
                />
              </QBtn>
              <QBtn
                v-close-popup
                align="left"
                flat
                @click="logout"
              >
                <QIcon
                  name="power_settings_new"
                  :left="true"
                />
                <AppContent
                  entry="authentication"
                  field="log_out_button"
                />
              </QBtn>
            </div>
          </div>
        </QMenu>
      </QBtn>
    </QToolbar>

    <QDialog
      :value="isMenuOpened"
      maximized
      transition-show="slide-down"
      transition-hide="slide-up"
      @input="toggleMenu"
    >
      <div class="bg-white navigation-menu">
        <QList>
          <QItem
            v-close-popup
            :to="{ name: 'home' }"
            exact-active-class="text-weight-medium"
            clickable
          >
            <QItemSection>{{ $t({ id: 'navigation.home' }) }}</QItemSection>
          </QItem>

          <QItem
            v-close-popup
            :to="{ name: 'search' }"
            exact-active-class="text-weight-medium"
            clickable
          >
            <QItemSection>{{ $t({ id: 'navigation.search' }) }}</QItemSection>
          </QItem>

          <div v-if="!showAccountAvatar">
            <QSeparator />

            <QItem
              v-show="!currentUser.id"
              v-close-popup
              clickable
              @click="openAuthDialog({ redirectAfterSignup: true })"
            >
              <QItemSection>{{ $t({ id: 'authentication.log_in_button' }) }}</QItemSection>
            </QItem>

            <QItem
              v-show="currentUser.id"
              v-close-popup
              :to="{ name: 'publicProfile', params: { id: currentUser.id }}"
              exact-active-class="text-weight-medium"
              clickable
            >
              <QItemSection>{{ $t({ id: 'navigation.profile' }) }}</QItemSection>
            </QItem>
            <QItem
              v-show="currentUser.id"
              v-close-popup
              clickable
              @click="logout"
            >
              <QItemSection>{{ $t({ id: 'authentication.log_out_button' }) }}</QItemSection>
            </QItem>
          </div>
        </QList>
      </div>
    </QDialog>

    <SearchToolbar />
  </QHeader>
</template>

<style lang="stylus" scoped>
$header-min-breakpoint = 359px

.transparent-header
  background: linear-gradient(180deg, rgba(0,0,0,0.8) -80%, rgba(0,0,0,0) 100%), \
              linear-gradient(170deg, rgba(0,0,0,0.8) -80%, rgba(0,0,0,0) 18%)

.header--raise-above-menu-dialog
  z-index: $z-fullscreen + 1

.logo-container svg
  max-height: $toolbar-min-height
.company-logo
  width: 9rem
.company-mini-logo
  height: 1.8rem

// Form
.header__search-bar
  border-radius: $generic-border-radius
  // Handle long input
  max-width: calc(100% - 4rem) // Make place for logo
  @media (min-width $header-min-breakpoint)
    max-width: calc(100% - 8rem) // and inbox icon
  @media (min-width $breakpoint-sm-min)
    max-width: 50%
  > .q-field
    min-width: 10rem // ensure visible placeholder when input is hidden after select in PlacesAutocomplete
    max-width: 20rem
    @media (min-width $breakpoint-md-min) and (max-width $breakpoint-md-max)
      max-width: 15rem

.header__inbox
  @media (max-width: $header-min-breakpoint)
    display: none
.inbox-badge
  position: absolute
  top: 0
  right: 0
  pointer-events: none
  border-radius: $badge-rounded-border-radius

.create-assset-button
  @media (max-width: 850px)
    display: none

// Menu Dialog
.navigation-menu
  padding-top: 1.5 * $toolbar-min-height
</style>

<style lang="stylus">
.filter-dialog .fixed-full
  top: 2 * $toolbar-min-height

// Local quasar override
.header__search-bar
  .q-field__label
    color: $font-color
    opacity: inherit
    font-weight: 500

// Ensuring button text does not wrap
// Don’t set it too high though due to Quasar automatic sizing
.header__account-menu
  min-width: 260px

.organizations-select
  max-height: 8rem
</style>
