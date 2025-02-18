<script lang="ts">
import { defineComponent } from '@vue/composition-api'
import {
  BridgeEvents,
  isPlainObject,
  sortByKey,
  openInEditor,
  copyToClipboard,
} from '@vue-devtools/shared-utils'
import DataFieldEdit from '@front/mixins/data-field-edit'
import { formattedValue, valueType, valueDetails } from '@front/util/format'
import { getBridge } from '../bridge'

function subFieldCount (value) {
  if (Array.isArray(value)) {
    return value.length
  } else if (value && typeof value === 'object') {
    return Object.keys(value).length
  } else {
    return 0
  }
}

export default defineComponent({
  name: 'DataField',

  mixins: [
    DataFieldEdit,
  ],

  props: {
    field: {
      type: Object,
      required: true,
    },
    depth: {
      type: Number,
      required: true,
    },
    path: {
      type: String,
      required: true,
    },
    forceCollapse: {
      type: String,
      default: null,
    },
    isStateField: {
      type: Boolean,
      default: false,
    },
  },

  data () {
    return {
      contextMenuOpen: false,
      limit: 20,
      expanded: false,
    }
  },

  computed: {
    depthMargin (): number {
      return (this.depth + 1) * 14 + 10
    },

    valueType (): string {
      return valueType(this.field.value)
    },

    valueDetails (): string {
      return valueDetails(this.field.value)
    },

    rawValueType (): string {
      return typeof this.field.value
    },

    isExpandableType (): boolean {
      let value = this.field.value
      if (this.valueType === 'custom') {
        value = value._custom.value
      }
      const closed = this.fieldOptions.closed
      const closedDefined = typeof closed !== 'undefined'
      return (!closedDefined &&
        (
          Array.isArray(value) ||
          isPlainObject(value)
        )) ||
        (
          closedDefined &&
          !closed
        )
    },

    formattedValue (): string {
      const value = this.field.value
      const objectType = value?._custom?.objectType || this.field.objectType
      if (objectType === 'Reactive') {
        return 'Reactive'
      } else if (this.fieldOptions.abstract) {
        return ''
      } else {
        let result = `<span class="value-formatted-ouput">${formattedValue(value)}</span>`
        if (objectType) {
          result += ` <span class="text-gray-500">(${objectType})</span>`
        }
        return result
      }
    },

    rawValue (): any {
      let value = this.field.value

      // CustomValue API
      const isCustom = this.valueType === 'custom'
      let inherit = {}
      if (isCustom) {
        inherit = value._custom.fields || {}
        value = value._custom.value
      }

      if (value && value._isArray) {
        value = value.items
      }
      return { value, inherit }
    },

    formattedSubFields (): any[] {
      let { value, inherit } = this.rawValue

      if (Array.isArray(value)) {
        return value.slice(0, this.limit).map((item, i) => ({
          key: i,
          value: item,
          ...inherit,
        }))
      } else if (typeof value === 'object') {
        value = Object.keys(value).map(key => ({
          key,
          value: value[key],
          ...inherit,
        }))
        if (this.valueType !== 'custom') {
          value = sortByKey(value)
        }
      }

      return value.slice(0, this.limit)
    },

    subFieldCount (): number {
      const { value } = this.rawValue
      return subFieldCount(value)
    },

    valueTooltip (): string {
      const type = this.valueType
      if (this.field.raw) {
        return `<span class="font-mono">${this.field.raw}</span>`
      } else if (type === 'custom') {
        return this.field.value._custom.tooltip
      } else if (type.indexOf('native ') === 0) {
        return type.substring('native '.length)
      } else {
        return null
      }
    },

    fieldOptions (): any {
      if (this.valueType === 'custom') {
        return Object.assign({}, this.field, this.field.value._custom)
      } else {
        return this.field
      }
    },

    editErrorMessage (): string {
      if (!this.valueValid) {
        return 'Invalid value (must be valid JSON)'
      } else if (!this.keyValid) {
        if (this.duplicateKey) {
          return 'Duplicate key'
        } else {
          return 'Invalid key'
        }
      }
      return ''
    },

    valueClass (): string[] {
      const cssClass = [this.valueType, `raw-${this.rawValueType}`]
      if (this.valueType === 'custom') {
        const value = this.field.value
        value._custom.type && cssClass.push(`type-${value._custom.type}`)
        value._custom.class && cssClass.push(value._custom.class)
      }
      return cssClass
    },

    displayedKey (): string {
      let key = this.field.key
      if (typeof key === 'string') {
        key = key.replace('__vue__', '')
      }
      return key
    },

    customActions (): { icon: string, tooltip?: string }[] {
      return this.field.value?._custom?.actions ?? []
    },
  },

  watch: {
    forceCollapse: {
      handler (value) {
        if (value === 'expand' && this.depth < 4) {
          this.expanded = true
        } else if (value === 'collapse') {
          this.expanded = false
        }
      },
      immediate: true,
    },
  },

  created () {
    const value = this.field.value && this.field.value._custom ? this.field.value._custom.value : this.field.value
    this.expanded = this.depth === 0 && this.field.key !== '$route' && (subFieldCount(value) < 12)
  },

  methods: {
    copyValue () {
      copyToClipboard(this.field.value)
    },

    copyPath () {
      copyToClipboard(this.path)
    },

    onClick (event) {
      // Cancel if target is interactive
      if (event.target.tagName === 'INPUT' || event.target.className.includes('button')) {
        return
      }

      // CustomValue API `file`
      if (this.valueType === 'custom' && this.fieldOptions.file) {
        return openInEditor(this.fieldOptions.file)
      }
      if (this.valueType === 'custom' && this.fieldOptions.type === '$refs') {
        if (this.$isChrome) {
          const evl = `inspect(window.__VUE_DEVTOOLS_INSTANCE_MAP__.get("${this.fieldOptions.uid}").$refs["${this.fieldOptions.key}"])`
          chrome.devtools.inspectedWindow.eval(evl)
        } else {
          window.alert('DOM inspection is not supported in this shell.')
        }
      }

      // Default action
      this.toggle()
    },

    toggle () {
      if (this.isExpandableType) {
        this.expanded = !this.expanded

        // @ts-ignore
        !this.expanded && this.cancelCurrentEdition()
      }
    },

    hyphen: v => v.replace(/\s/g, '-'),

    onContextMenuMouseEnter () {
      // @ts-ignore
      clearTimeout(this.$_contextMenuTimer)
    },

    onContextMenuMouseLeave () {
      // @ts-ignore
      clearTimeout(this.$_contextMenuTimer)
      this.$_contextMenuTimer = setTimeout(() => {
        this.contextMenuOpen = false
      }, 4000)
    },

    showMoreSubfields () {
      this.limit += 20
    },

    logToConsole (level = 'log') {
      getBridge().send(BridgeEvents.TO_BACK_LOG, {
        level,
        value: this.field.value,
        revive: true,
      })
    },

    executeCustomAction (index: number) {
      getBridge().send(BridgeEvents.TO_BACK_CUSTOM_STATE_ACTION, {
        value: this.field.value,
        actionIndex: index,
      })
    },
  },

  renderError: null,
})
</script>

<template>
  <div class="data-field">
    <VTooltip
      :style="{ marginLeft: depth * 14 + 'px' }"
      :disabled="!field.meta"
      :class="{
        'force-toolbar z-10': contextMenuOpen || editing,
      }"
      class="self"
      placement="left"
      :offset="[0, 24]"
      @click.native="onClick"
      @mouseenter.native="onContextMenuMouseEnter"
      @mouseleave.native="onContextMenuMouseLeave"
    >
      <span
        v-show="isExpandableType"
        :class="{ rotated: expanded }"
        class="arrow right"
      />
      <span
        v-if="editing && renamable"
      >
        <input
          ref="keyInput"
          v-model="editedKey"
          class="edit-input key-input"
          :class="{ error: !keyValid }"
          @keydown.esc.capture.stop.prevent="cancelEdit()"
          @keydown.enter="submitEdit()"
        >
      </span>
      <span
        v-else
        v-tooltip="fieldOptions.abstract && {
          content: valueTooltip,
          html: true
        }"
        :class="{ abstract: fieldOptions.abstract }"
        class="key text-purple-700 dark:text-purple-300"
      >{{ displayedKey }}</span><span
        v-if="!fieldOptions.abstract"
        class="colon"
      >:</span>

      <span
        v-if="editing"
        class="edit-overlay"
      >
        <input
          ref="editInput"
          v-model="editedValue"
          class="edit-input value-input text-black"
          :class="{ error: !valueValid }"
          list="special-tokens"
          @keydown.esc.capture.stop.prevent="cancelEdit()"
          @keydown.enter="submitEdit()"
        >
        <span class="actions">
          <VueIcon
            v-if="!editValid"
            v-tooltip="editErrorMessage"
            class="small-icon warning"
            icon="warning"
          />
          <template v-else>
            <VueButton
              v-tooltip="{
                content: $t('DataField.edit.cancel.tooltip'),
                html: true
              }"
              class="icon-button flat"
              icon-left="cancel"
              @click="cancelEdit()"
            />
            <VueButton
              v-tooltip="{
                content: $t('DataField.edit.submit.tooltip'),
                html: true
              }"
              class="icon-button flat"
              icon-left="save"
              @click="submitEdit()"
            />
          </template>
        </span>
      </span>
      <template v-else>
        <!-- eslint-disable vue/no-v-html -->
        <span
          v-tooltip="{
            content: valueTooltip,
            html: true
          }"
          :class="valueClass"
          class="value"
          @dblclick="openEdit()"
          v-html="formattedValue"
        />
        <!-- eslint-enable vue/no-v-html -->
        <span class="actions">
          <VueDropdown
            v-if="valueDetails"
          >
            <template #trigger>
              <VueButton
                v-tooltip="'More details'"
                class="edit-value icon-button flat"
                icon-left="info"
              />
            </template>

            <template #default>
              <div class="px-4 py-2 flex flex-col max-h-48 overflow-auto relative">
                <div class="flex items-center fixed top-0 right-0">
                  <VueButton
                    v-close-popper
                    class="edit-value icon-button flat"
                    icon-left="close"
                  />
                </div>
                <div class="whitespace-pre-wrap max-w-lg break-words text-xs font-mono">{{ valueDetails }}</div>
              </div>
            </template>
          </VueDropdown>
          <VueButton
            v-for="(action, index) of customActions"
            :key="index"
            v-tooltip="action.tooltip"
            class="icon-button flat"
            :icon-left="action.icon"
            @click="executeCustomAction(index)"
          />
          <VueButton
            v-if="valueType === 'native Error'"
            v-tooltip="'Log error to console'"
            class=" icon-button flat"
            icon-left="input"
            @click="logToConsole('error')"
          />
          <VueButton
            v-if="isValueEditable"
            v-tooltip="'Edit value'"
            class="edit-value icon-button flat"
            icon-left="edit"
            @click="openEdit()"
          />
          <template v-if="quickEdits">
            <VueButton
              v-for="(info, index) of quickEdits"
              :key="index"
              v-tooltip="{
                content: info.title || 'Quick edit',
                html: true,
              }"
              :class="info.class"
              :icon-left="info.icon"
              class="quick-edit icon-button flat"
              @click="quickEdit(info, $event)"
            />
          </template>
          <VueButton
            v-if="isSubfieldsEditable && !addingValue"
            v-tooltip="'Add new value'"
            class="add-value icon-button flat"
            icon-left="add_circle"
            @click="addNewValue()"
          />
          <VueButton
            v-if="removable"
            v-tooltip="'Remove value'"
            class="remove-field icon-button flat"
            icon-left="delete"
            @click="removeField()"
          />

          <!-- Context menu -->
          <VueDropdown
            :open.sync="contextMenuOpen"
          >
            <VueButton
              slot="trigger"
              icon-left="more_vert"
              class="icon-button flat"
            />

            <div
              class="context-menu-dropdown"
              @mouseenter="onContextMenuMouseEnter"
              @mouseleave="onContextMenuMouseLeave"
            >
              <VueDropdownButton
                icon-left="flip_to_front"
                @click="copyValue"
              >
                {{ $t('DataField.contextMenu.copyValue') }}
              </VueDropdownButton>
              <VueDropdownButton
                icon-left="flip_to_front"
                @click="copyPath"
              >
                {{ $t('DataField.contextMenu.copyPath') }}
              </VueDropdownButton>
            </div>
          </VueDropdown>
        </span>
      </template>

      <template #popper>
        <div
          v-if="field.meta"
          class="meta"
        >
          <div
            v-for="(val, key) in field.meta"
            :key="key"
            class="meta-field"
          >
            <span class="key">{{ key }}</span>
            <span class="value">{{ val }}</span>
          </div>
        </div>
      </template>
    </VTooltip>
    <div
      v-if="expanded && isExpandableType"
      class="children"
    >
      <DataField
        v-for="subField in formattedSubFields"
        :key="subField.key"
        :field="subField"
        :parent-field="field"
        :depth="depth + 1"
        :path="`${path}.${subField.key}`"
        :editable="isEditable"
        :removable="isSubfieldsEditable"
        :renamable="editable && valueType === 'plain-object'"
        :force-collapse="forceCollapse"
        :is-state-field="isStateField"
        @edit-state="(path, payload) => $emit('edit-state', path, payload)"
      />
      <VueButton
        v-if="subFieldCount > limit"
        v-tooltip="'Show more'"
        :style="{ marginLeft: depthMargin + 'px' }"
        icon-left="more_horiz"
        class="icon-button flat more"
        @click="showMoreSubfields()"
      />
      <DataField
        v-if="isSubfieldsEditable && addingValue"
        ref="newField"
        :field="newField"
        :depth="depth + 1"
        :path="`${path}.${newField.key}`"
        :renamable="valueType === 'plain-object'"
        :force-collapse="forceCollapse"
        editable
        removable
        :is-state-field="isStateField"
        @cancel-edit="addingValue = false"
        @submit-edit="addingValue = false"
        @edit-state="(path, payload) => $emit('edit-state', path, payload)"
      />
    </div>
  </div>
</template>

<style lang="stylus" scoped>
.data-field
  user-select text
  font-size 12px
  @apply font-mono;
  cursor pointer

.self
  height 20px
  line-height 20px
  position relative
  white-space nowrap
  padding-left 14px
  .high-density &
    height 14px
    line-height 14px
  span, div
    display inline-block
    vertical-align middle
  .arrow
    position absolute
    top 7px
    left 0px
    transition transform .1s ease
    &.rotated
      transform rotate(90deg)
  .actions
    display inline-flex
    align-items center
    position relative
    top -1px
    > *
      visibility hidden
      &:first-child
        margin-left 6px
      &:not(:last-child)
        margin-right 6px
      &.v-popper--shown
        visibility visible
    .icon-button
      user-select none
      width 20px
      height @width
    .icon-button >>> .vue-ui-icon,
    .small-icon
      width 16px
      height @width
    .warning >>> svg
      fill $orange
  &:hover,
  &.force-toolbar
    .actions > *
      visibility visible
  .colon
    margin-right .5em
    position relative

  .type
    color $background-color
    padding 3px 6px
    font-size 10px
    line-height 10px
    height 16px
    border-radius 3px
    margin 2px 6px
    position relative
    background-color #eee
    &.prop
      background-color #96afdd
    &.computed
      background-color #af90d5
    &.vuex-getter
      background-color #5dd5d5
    &.firebase-binding
      background-color #ffcc00
    &.observable
      background-color #ff9999
    .vue-ui-dark-mode &
      color: #242424

  .edit-overlay
    display inline-flex
    align-items center

.key
  &.abstract
    color $blueishGrey
    .vue-ui-dark-mode &
      color lighten($blueishGrey, 20%)
.value
  display inline-block
  color #444
  &.string, &.native
    color $red
  &.string
    >>> span
      color $black
      .vue-ui-dark-mode &
        color $red
  &.null
    color #999
  &.literal
    color $vividBlue
  &.raw-boolean >>> .value-formatted-ouput
    width 36px
    display inline-block
  &.native.Error
    background $red
    color $white !important
    padding 0 4px
    border-radius $br
    &::before
      content 'Error: '
      opacity .75
  &.custom
    &.type-component
      color $green
      &::before,
      &::after
        color $darkGrey
      &::before
        content '<'
      &::after
        content '>'
    &.type-function
      font-style italic
      >>> span
        color $vividBlue
        font-family dejavu sans mono, monospace
        .platform-mac &
          font-family Menlo, monospace
        .platform-windows &
          font-family Consolas, Lucida Console, Courier New, monospace
        .vue-ui-dark-mode &
          color $purple
    &.type-component-definition
      color $green
      >>> span
        color $darkerGrey
    &.type-reference
        opacity 0.5
      >>> .attr-title
        color #800080
        .vue-ui-dark-mode &
          color #e36eec

  .vue-ui-dark-mode &
    color #bdc6cf
    &.string, &.native
      color #e33e3a
    &.null
      color #999
    &.literal
      color $purple

.meta
  font-size 12px
  font-family Menlo, Consolas, monospace
  min-width 150px
  .key
    display inline-block
    width 80px
    color lighten(#881391, 60%)
    .vue-ui-dark-mode &
      color #881391
  .value
    color white
    .vue-ui-dark-mode &
      color black
.meta-field
  &:not(:last-child)
    margin-bottom 4px

.edit-input
  font-family Menlo, Consolas, monospace
  border solid 1px $green
  border-radius 3px
  padding 2px
  outline none
  &.error
    border-color $orange
.value-input
  width 180px
.key-input
  width 90px
  color #881391

.remove-field
  margin-left 10px

.context-menu-dropdown
  .vue-ui-button
    display block
    width 100%

.more
  width 20px
  height @width
  >>> .vue-ui-icon
    width 16px
    height @width
</style>
