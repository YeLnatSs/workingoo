<script>
import { mapGetters } from 'vuex'

export default {
  props: {
    entry: {
      type: String,
      required: true
    },
    field: {
      type: String,
      required: true
    },
    defaultMessage: {
      type: String,
      default: ''
    },
    tag: { // forced to 'div' when rendering HTML
      type: String,
      default: 'span'
    },
    options: {
      type: Object,
      default: () => ({})
    }
  },
  data () {
    return {}
  },
  computed: {
    contentKey () {
      return `${this.entry}.${this.field}`
    },
    value () {
      const messageDescriptor = { id: this.contentKey }
      if (this.defaultMessage) messageDescriptor.defaultMessage = this.defaultMessage

      return this.$t(messageDescriptor, this.options)
    },
    renderAsHTML () {
      return this.isTransformedContent(this.contentKey) === 'markdown'
    },
    ...mapGetters([
      'isTransformedContent'
    ])
  },
}
</script>

<template>
  <!--
    SECURITY WARNING: XSS attacks
    Please note v-html below is only safe to use with trusted content:
    - value is NOT user-content but content authored by your team or trusted translators
    - value was sanitized or rendered in an inherently safe way as it is in this template,
      since we keep markdown-it `html` option disabled and only render markdown syntax,
      so that existing HTML is escaped anyway.
      https://github.com/markdown-it/markdown-it/blob/master/docs/security.md
  -->
  <!-- eslint-disable vue/no-v-html -->
  <div
    is="div"
    v-if="renderAsHTML"
    class="stl-content-entry"
    :stl-content-entry="entry"
    :stl-content-field="field"
    v-bind="$attrs"
    v-on="$listeners"
    v-html="value"
  />
  <!-- eslint-enable vue/no-v-html -->
  <!--
    TODO: insert stl-content-x attributes only when needed to avoid bloating template
          like when receiving an instruction from dashboard editor (postMessage)
  -->
  <component
    :is="tag"
    v-else
    class="stl-content-entry"
    :stl-content-entry="entry"
    :stl-content-field="field"
    v-bind="$attrs"
    v-on="$listeners"
  >
    {{ value }}
  </component>
</template>

<style lang="stylus">
// rendered markdown is wrapped in p tag
.stl-content-entry p
  margin-bottom: 0

.stl-content-entry a
  color: inherit
  text-decoration: none
  &:focus, &:hover
    text-decoration: underline
</style>
