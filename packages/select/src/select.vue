<template>
  <div
    class="el-select"
    :class="[selectSize ? 'el-select--' + selectSize : '']"
    @click.stop="toggleMenu"
    v-clickoutside="handleClose"
  >
    <div
      class="el-select__tags"
      v-if="multiple"
      ref="tags"
      :style="{ 'max-width': inputWidth - 32 + 'px', width: '100%' }"
    >
      <span v-if="collapseTags && selected.length">
        <el-tag
          :closable="!selectDisabled"
          :size="collapseTagSize"
          :hit="selected[0].hitState"
          type="info"
          @close="deleteTag($event, selected[0])"
          disable-transitions
        >
          <span class="el-select__tags-text">{{
            selected[0].currentLabel
          }}</span>
        </el-tag>
        <el-tag
          v-if="selected.length > 1"
          :key="selected.length - 1"
          :closable="false"
          :size="collapseTagSize"
          type="info"
          disable-transitions
        >
          <span class="el-select__tags-text">+ {{ selected.length - 1 }}</span>
        </el-tag>
      </span>
      <transition-group @after-leave="resetInputHeight" v-if="!collapseTags">
        <div key="transition-group-key">
          <el-tag
            v-for="item in selected"
            :key="getValueKey(item)"
            :closable="!selectDisabled"
            :size="collapseTagSize"
            :hit="item.hitState"
            type="info"
            @close="deleteTag($event, item)"
            disable-transitions
          >
            <span class="el-select__tags-text">{{ item.currentLabel }}</span>
          </el-tag>
        </div>
      </transition-group>

      <input
        type="text"
        class="el-select__input"
        :class="[selectSize ? `is-${selectSize}` : '']"
        :disabled="selectDisabled"
        :autocomplete="autocomplete"
        @focus="handleFocus"
        @blur="softFocus = false"
        @keyup="managePlaceholder"
        @keydown="resetInputState"
        @keydown.down.prevent="navigateOptions('next')"
        @keydown.up.prevent="navigateOptions('prev')"
        @keydown.enter.prevent="selectOption"
        @keydown.esc.stop.prevent="visible = false"
        @keydown.delete="deletePrevTag"
        @keydown.tab="visible = false"
        @compositionstart="handleComposition"
        @compositionupdate="handleComposition"
        @compositionend="handleComposition"
        v-model="query"
        @input="debouncedQueryChange"
        v-if="filterable"
        :style="{
          'flex-grow': '1',
          width: inputLength / (inputWidth - 32) + '%',
          'max-width': inputWidth - 42 + 'px'
        }"
        ref="input"
      />
    </div>
    <el-input
      ref="reference"
      v-model="selectedLabel"
      type="text"
      :placeholder="currentPlaceholder"
      :name="name"
      :id="id"
      :autocomplete="autocomplete"
      :size="selectSize"
      :disabled="selectDisabled"
      :readonly="readonly"
      :validate-event="false"
      :class="{ 'is-focus': visible }"
      :tabindex="multiple && filterable ? '-1' : null"
      @focus="handleFocus"
      @blur="handleBlur"
      @keyup="debouncedOnInputChange"
      @keydown.down.stop.prevent="navigateOptions('next')"
      @keydown.up.stop.prevent="navigateOptions('prev')"
      @keydown.enter.prevent="selectOption"
      @keydown.esc.stop.prevent="visible = false"
      @keydown.tab="visible = false"
      @paste="debouncedOnInputChange"
      @mouseenter.prevent="inputHovering = true"
      @mouseleave="inputHovering = false"
    >
      <template v-slot:prefix v-if="$slots.prefix">
        <slot name="prefix"></slot>
      </template>
      <template v-slot:suffix>
        <i
          v-show="!showClose"
          :class="[
            'el-select__caret',
            'el-input__icon',
            'el-icon-' + iconClass
          ]"
        ></i>
        <i
          v-if="showClose"
          class="el-select__caret el-input__icon el-icon-circle-close"
          @click="handleClearClick"
        ></i>
      </template>
    </el-input>
    <transition
      name="el-zoom-in-top"
      @before-enter="handleMenuEnter"
      @after-leave="doDestroy"
    >
      <el-select-menu
        ref="popper"
        :append-to-body="popperAppendToBody"
        v-show="visible && emptyText !== false"
      >
        <el-scrollbar
          tag="ul"
          wrap-class="el-select-dropdown__wrap"
          view-class="el-select-dropdown__list"
          ref="scrollbar"
          :class="{
            'is-empty': !allowCreate && query && filteredOptionsCount === 0
          }"
          v-show="options.length > 0 && !loading"
        >
          <el-option :value="query" created v-if="showNewOption"></el-option>
          <slot></slot>
        </el-scrollbar>
        <template
          v-if="
            emptyText &&
            (!allowCreate || loading || (allowCreate && options.length === 0))
          "
        >
          <slot name="empty" v-if="$slots.empty"></slot>
          <p class="el-select-dropdown__empty" v-else>
            {{ emptyText }}
          </p>
        </template>
      </el-select-menu>
    </transition>
  </div>
</template>

<script type="text/babel">
import ElInput from 'element-ui/packages/input'
import ElSelectMenu from './select-dropdown.vue'
import useFocus from 'element-ui/src/use/focus.js'
import ElOption from './option.vue'
import ElTag from 'element-ui/packages/tag'
import ElScrollbar from 'element-ui/packages/scrollbar'
import debounceFun from 'throttle-debounce/debounce'
import Clickoutside from 'element-ui/src/utils/clickoutside'
import {
  addResizeListener,
  removeResizeListener
} from 'element-ui/src/utils/resize-event'
import { t } from 'element-ui/src/locale'
import scrollIntoView from 'element-ui/src/utils/scroll-into-view'
import {
  getValueByPath,
  isEdge,
  isIE,
  valueEquals
} from 'element-ui/src/utils/util'
import NavigationMixin from './navigation-mixin'
import { isKorean } from 'element-ui/src/utils/shared'
import { useEmitter } from 'element-ui/src/use/emitter'

import {
  computed,
  getCurrentInstance,
  inject,
  provide,
  reactive,
  toRefs,
  watch,
  nextTick,
  onMounted,
  onUnmounted,
  ref
} from 'vue'

export default {
  mixins: [NavigationMixin],
  emits: [
    'visible-change',
    'update:modelValue',
    'change',
    'clear',
    'remove-tag',
    'focus',
    'blur',
    'handleOptionClick',
    'setSelected'
  ],
  setup(props, { emit }) {
    const { ctx } = getCurrentInstance()
    provide('select', getCurrentInstance())
    const focus = useFocus('reference')

    const { broadcast, dispatch, on } = useEmitter()

    const {
      name,
      id,
      modelValue,
      autocomplete,
      automaticDropdown,
      size,
      disabled,
      clearable,
      filterable,
      allowCreate,
      loading,
      popperClass,
      remote,
      loadingText,
      noMatchText,
      noDataText,
      multiple,
      multipleLimit,
      placeholder,
      defaultFirstOption,
      reserveKeyword,
      valueKey,
      collapseTags,
      popperAppendToBody,
      remoteMethod,
      filterMethod
    } = toRefs(props)

    const state = reactive({
      options: [],
      cachedOptions: [],
      createdLabel: null,
      createdSelected: false,
      selected: multiple.value ? [] : {},
      inputLength: 20,
      inputWidth: 0,
      initialInputHeight: 0,
      cachedPlaceHolder: '',
      optionsCount: 0,
      filteredOptionsCount: 0,
      visible: false,
      softFocus: false,
      selectedLabel: '',
      hoverIndex: -1,
      query: '',
      previousQuery: null,
      inputHovering: false,
      currentPlaceholder: '',
      menuVisibleOnFocus: false,
      isOnComposition: false,
      isSilentBlur: false
    })

    const selectSize = computed(() => {
      const elFormItem = inject('elFormItem')
      const _elFormItemSize = (elFormItem || {}).elFormItemSize
      return size?.value || _elFormItemSize || (ctx.$ELEMENT || {}).size
    })

    const collapseTagSize = computed(() => {
      return ['small', 'mini'].indexOf(selectSize.value) > -1 ? 'mini' : 'small'
    })

    const selectDisabled = computed(() => {
      const elForm = inject('elForm')
      return disabled.value || (elForm || {}).disabled
    })

    const readonly = computed(() => {
      return (
        !filterable.value ||
        multiple.value ||
        (!isIE() && !isEdge() && !state.visible)
      )
    })

    const showClose = computed(() => {
      const hasValue = multiple.value
        ? Array.isArray(modelValue.value) && modelValue.value.length > 0
        : modelValue.value !== undefined &&
          modelValue.value !== null &&
          modelValue.value !== ''

      return (
        clearable.value &&
        !selectDisabled.value &&
        state.inputHovering &&
        hasValue
      )
    })

    const iconClass = computed(() => {
      return remote.value && filterable.value
        ? ''
        : state.visible
        ? 'arrow-up is-reverse'
        : 'arrow-up'
    })

    const debounce = computed(() => {
      return remote.value ? 300 : 0
    })

    const emptyText = computed((ctx) => {
      if (loading.value) {
        return loadingText?.value || t('el.select.loading')
      } else {
        if (remote.value && state.query === '' && state.options.length === 0)
          return false
        if (
          filterable.value &&
          state.query &&
          state.options.length > 0 &&
          state.filteredOptionsCount === 0
        ) {
          return noMatchText?.value || t('el.select.noMatch')
        }
        if (state.options.length === 0) {
          return noDataText?.value || t('el.select.noData')
        }
      }
      return null
    })

    const showNewOption = computed(() => {
      const hasExistingOption = state.options
        .filter((option) => !option.created)
        .some((option) => option.currentLabel === state.query)
      return (
        filterable.value &&
        allowCreate?.value &&
        state.query !== '' &&
        !hasExistingOption
      )
    })
    function checkDefaultFirstOption() {
      state.hoverIndex = -1
      // highlight the created option
      let hasCreated = false
      for (let i = state.options.length - 1; i >= 0; i--) {
        if (state.options[i].created) {
          hasCreated = true
          state.hoverIndex = i
          break
        }
      }
      if (hasCreated) return
      for (let i = 0; i !== state.options.length; ++i) {
        const option = state.options[i]
        if (state.query) {
          // highlight first options that passes the filter
          if (!option.disabled && !option.groupDisabled && option.visible) {
            state.hoverIndex = i
            break
          }
        } else {
          // highlight currently selected option
          if (option.itemSelected) {
            state.hoverIndex = i
            break
          }
        }
      }
    }

    function handleQueryChange(val) {
      if (state.previousQuery === val || state.isOnComposition) return
      if (
        state.previousQuery === null &&
        (typeof filterMethod?.value === 'function' ||
          typeof remoteMethod?.value === 'function')
      ) {
        state.previousQuery = val
        return
      }
      state.previousQuery = val
      nextTick(() => {
        if (state.visible) broadcast('ElSelectDropdown', 'updatePopper')
      })
      state.hoverIndex = -1
      if (multiple.value && filterable.value) {
        nextTick(() => {
          const length = ctx.$refs.input.value.length * 15 + 20
          state.inputLength = collapseTags.value ? Math.min(50, length) : length
          managePlaceholder()
          resetInputHeight()
        })
      }
      if (remote.value && typeof remoteMethod?.value === 'function') {
        state.hoverIndex = -1
        remoteMethod.value(val)
      } else if (typeof filterMethod?.value === 'function') {
        filterMethod?.value(val)
        broadcast('ElOptionGroup', 'queryChange')
      } else {
        state.filteredOptionsCount = state.optionsCount
        broadcast('ElOption', 'queryChange', val)
        broadcast('ElOptionGroup', 'queryChange')
      }
      if (
        defaultFirstOption.value &&
        (filterable.value || remote.value) &&
        state.filteredOptionsCount
      ) {
        checkDefaultFirstOption()
      }
    }

    function getOption(value) {
      let option
      const isObject =
        Object.prototype.toString.call(value).toLowerCase() ===
        '[object object]'
      const isNull =
        Object.prototype.toString.call(value).toLowerCase() === '[object null]'
      const isUndefined =
        Object.prototype.toString.call(value).toLowerCase() ===
        '[object undefined]'

      for (let i = state.cachedOptions.length - 1; i >= 0; i--) {
        const cachedOption = state.cachedOptions[i]
        const isEqual = isObject
          ? getValueByPath(cachedOption.value, valueKey.value) ===
            getValueByPath(value, valueKey.value)
          : cachedOption.value === value
        if (isEqual) {
          option = cachedOption
          break
        }
      }
      if (option) return option
      const label = !isObject && !isNull && !isUndefined ? value : ''
      const newOption = {
        value: value,
        currentLabel: label
      }
      if (multiple.value) {
        newOption.hitState = false
      }
      return newOption
    }

    function resetHoverIndex() {
      setTimeout(() => {
        if (!multiple.value) {
          state.hoverIndex = state.options.indexOf(state.selected)
        } else {
          if (state.selected.length > 0) {
            state.hoverIndex = Math.min.apply(
              null,
              state.selected.map((item) => state.options.indexOf(item))
            )
          } else {
            state.hoverIndex = -1
          }
        }
      }, 300)
    }

    function resetInputHeight() {
      if (collapseTags.value && !filterable.value) return
      nextTick(() => {
        if (!ctx.$refs.reference) return
        const inputChildNodes = ctx.$refs.reference.$el.childNodes
        const input = [].filter.call(
          inputChildNodes,
          (item) => item.tagName === 'INPUT'
        )[0]
        const tags = ctx.$refs.tags
        const sizeInMap = state.initialInputHeight || 40
        input.style.height =
          state.selected.length === 0
            ? sizeInMap + 'px'
            : Math.max(
                tags
                  ? tags.clientHeight + (tags.clientHeight > sizeInMap ? 6 : 0)
                  : 0,
                sizeInMap
              ) + 'px'
        if (state.visible && emptyText.value !== false) {
          broadcast('ElSelectDropdown', 'updatePopper')
        }
      })
    }

    function emitChange(val) {
      if (!valueEquals(modelValue.value, val)) {
        emit('change', val)
      }
    }

    function setSoftFocus() {
      state.softFocus = true
      const input = ctx.$refs.input || ctx.$refs.reference

      if (input) {
        input.focus()
      }
    }

    function getValueIndex(arr = [], value) {
      const isObject =
        Object.prototype.toString.call(value).toLowerCase() ===
        '[object object]'
      if (!isObject) {
        return arr.indexOf(value)
      } else {
        let index = -1
        arr.some((item, i) => {
          if (
            getValueByPath(item, valueKey.value) ===
            getValueByPath(value, valueKey.value)
          ) {
            index = i
            return true
          }
          return false
        })
        return index
      }
    }

    function handleMenuEnter() {
      nextTick(() => scrollToOption(state.selected))
    }

    function scrollToOption(option) {
      const target =
        Array.isArray(option) && option[0] ? option[0].$el : option.$el
      if (ctx.$refs.popper && target) {
        const menu = ctx.$refs.popper.$el.querySelector(
          '.el-select-dropdown__wrap'
        )
        scrollIntoView(menu, target)
      }
      ctx.$refs.scrollbar && ctx.$refs.scrollbar.handleScroll()
    }

    function handleOptionSelect(option, byClick) {
      if (multiple.value) {
        const value = (modelValue.value || []).slice()
        const optionIndex = getValueIndex(value, option.value)
        if (optionIndex > -1) {
          value.splice(optionIndex, 1)
        } else if (
          multipleLimit.value <= 0 ||
          value.length < multipleLimit.value
        ) {
          value.push(option.value)
        }
        emit('update:modelValue', value)
        emitChange(value)
        if (option.created) {
          state.query = ''
          handleQueryChange('')
          state.inputLength = 20
        }
        if (filterable.value) ctx.$refs.input.focus()
      } else {
        emit('update:modelValue', option.value)
        emitChange(option.value)
        state.visible = false
      }
      state.isSilentBlur = byClick
      setSoftFocus()
      if (state.visible) return
      nextTick(() => {
        scrollToOption(option)
      })
    }
    function handleComposition(event) {
      const text = event.target.value
      if (event.type === 'compositionend') {
        state.isOnComposition = false
        nextTick(() => handleQueryChange(text))
      } else {
        const lastCharacter = text[text.length - 1] || ''
        state.isOnComposition = !isKorean(lastCharacter)
      }
    }
    watch(selectDisabled, () => {
      nextTick(() => {
        resetInputHeight()
      })
    })
    watch(placeholder, (val) => {
      state.cachedPlaceHolder = state.currentPlaceholder = val
    })

    watch(modelValue, (val, oldVal) => {
      if (multiple.value) {
        resetInputHeight()
        if (
          (val && val.length > 0) ||
          (ctx.$refs.input && state.query !== '')
        ) {
          state.currentPlaceholder = ''
        } else {
          state.currentPlaceholder = state.cachedPlaceHolder
        }
        if (filterable.value && !reserveKeyword.value) {
          state.query = ''
          handleQueryChange(state.query)
        }
      }
      setSelected()
      if (filterable.value && !multiple.value) {
        state.inputLength = 20
      }
      if (!valueEquals(val, oldVal)) {
        dispatch('ElFormItem', 'el.form.change', val)
      }
    })

    watch(
      () => state.visible,
      (val) => {
        if (!val) {
          broadcast('ElSelectDropdown', 'destroyPopper')
          if (ctx.$refs.input) {
            ctx.$refs.input.blur()
          }
          state.query = ''
          state.previousQuery = null
          state.selectedLabel = ''
          state.inputLength = 20
          state.menuVisibleOnFocus = false
          resetHoverIndex()
          nextTick(() => {
            if (
              ctx.$refs.input &&
              ctx.$refs.input.value === '' &&
              state.selected.length === 0
            ) {
              state.currentPlaceholder = state.cachedPlaceHolder
            }
          })
          if (!multiple.value) {
            if (state.selected) {
              if (
                filterable.value &&
                allowCreate?.value &&
                state.createdSelected &&
                state.createdLabel
              ) {
                state.selectedLabel = state.createdLabel
              } else {
                state.selectedLabel = state.selected.currentLabel
              }
              if (filterable.value) state.query = state.selectedLabel
            }

            if (filterable.value) {
              state.currentPlaceholder = state.cachedPlaceHolder
            }
          }
        } else {
          broadcast('ElSelectDropdown', 'updatePopper')
          if (filterable.value) {
            state.query = remote.value ? '' : state.selectedLabel
            handleQueryChange(state.query)
            if (multiple.value) {
              ctx.$refs.input.focus()
            } else {
              if (!remote.value) {
                broadcast('ElOption', 'queryChange', '')
                broadcast('ElOptionGroup', 'queryChange')
              }

              if (state.selectedLabel) {
                state.currentPlaceholder = state.selectedLabel
                state.selectedLabel = ''
              }
            }
          }
        }
        emit('visible-change', val)
      }
    )

    watch(
      () => state.options,
      () => {
        // check ssr
        if (window === undefined) return

        nextTick(() => {
          broadcast('ElSelectDropdown', 'updatePopper')
        })
        if (multiple.value) {
          resetInputHeight()
        }
        const inputs = ctx.$el.querySelectorAll('input')
        if ([].indexOf.call(inputs, document.activeElement) === -1) {
          setSelected()
        }
        if (
          defaultFirstOption.value &&
          (filterable.value || remote.value) &&
          state.filteredOptionsCount
        ) {
          checkDefaultFirstOption()
        }
      },
      {
        deep: true
      }
    )

    function onInputChange() {
      if (filterable.value && state.query !== state.selectedLabel) {
        state.query = state.selectedLabel
        handleQueryChange(state.query)
      }
    }

    function onCreate() {
      state.cachedPlaceHolder = state.currentPlaceholder = placeholder.value
      if (multiple.value && !Array.isArray(modelValue.value)) {
        emit('update:modelValue', [])
      }
      if (!multiple.value && Array.isArray(modelValue.value)) {
        emit('update:modelValue', '')
      }
      on('handleOptionClick', handleOptionSelect)
      on('setSelected', setSelected)
    }

    const debouncedOnInputChange = debounceFun(debounce.value, () => {
      onInputChange()
    })

    function resetInputWidth() {
      if (!ctx.$refs.reference) return
      state.inputWidth = ctx.$refs.reference.$el.getBoundingClientRect().width
    }

    function handleResize() {
      resetInputWidth()
      if (multiple.value) resetInputHeight()
    }

    const debouncedQueryChange = debounceFun(debounce.value, (e) => {
      handleQueryChange(e.target.value)
    })
    onCreate()

    onMounted(() => {
      if (
        multiple.value &&
        Array.isArray(modelValue.value) &&
        modelValue.value.length > 0
      ) {
        state.currentPlaceholder = ''
      }
      addResizeListener(ctx.$el, handleResize)

      const reference = ctx.$refs.reference
      if (reference && reference.$el) {
        const sizeMap = {
          medium: 36,
          small: 32,
          mini: 28
        }
        const input = reference.$el.querySelector('input')
        state.initialInputHeight =
          input.getBoundingClientRect().height || sizeMap[selectSize.value]
      }
      if (remote.value && multiple.value) {
        resetInputHeight()
      }
      nextTick(() => {
        if (reference && reference.$el) {
          state.inputWidth = reference.$el.getBoundingClientRect().width
        }
      })
      setSelected()
    })
    onUnmounted(() => {
      if (ctx.$el && handleResize) removeResizeListener(ctx.$el, handleResize)
    })

    function toggleMenu() {
      if (!selectDisabled.value) {
        if (state.menuVisibleOnFocus) {
          state.menuVisibleOnFocus = false
        } else {
          state.visible = !state.visible
        }
        if (state.visible) {
          ;(ctx.$refs.input || ctx.$refs.reference).focus()
        }
      }
    }
    function deleteTag(event, tag) {
      const index = state.selected.indexOf(tag)
      if (index > -1 && !selectDisabled.value) {
        const value = modelValue.value.slice()
        value.splice(index, 1)
        emit('update:modelValue', value)
        emitChange(value)
        emit('remove-tag', tag.value)
      }
      event.stopPropagation()
    }
    function selectOption() {
      if (!state.visible) {
        toggleMenu()
      } else {
        if (state.options[state.hoverIndex]) {
          handleOptionSelect(state.options[state.hoverIndex])
        }
      }
    }

    function deleteSelected(event) {
      event.stopPropagation()
      const value = multiple.value ? [] : ''
      emit('update:modelValue', value)
      emitChange(value)
      state.visible = false
      emit('clear')
    }

    function getValueKey(item) {
      if (
        Object.prototype.toString.call(item.value).toLowerCase() !==
        '[object object]'
      ) {
        return item.value
      } else {
        return getValueByPath(item.value, valueKey.value)
      }
    }
    // option组件
    function onOptionDestroy(index) {
      if (index > -1) {
        state.optionsCount--
        state.filteredOptionsCount--
        state.options.splice(index, 1)
      }
    }
    function doDestroy() {
      ctx.$refs.popper && ctx.$refs.popper.doDestroy()
    }

    function handleClose() {
      state.visible = false
    }

    function setSelected() {
      if (!multiple.value) {
        const option = getOption(modelValue.value)
        if (option.created) {
          state.createdLabel = option.currentLabel
          state.createdSelected = true
        } else {
          state.createdSelected = false
        }
        state.selectedLabel = option.currentLabel

        state.selected = option
        if (filterable.value) state.query = state.selectedLabel
        return
      }
      const result = []
      if (Array.isArray(modelValue.value)) {
        modelValue.value.forEach((value) => {
          result.push(getOption(value))
        })
      }
      state.selected = result
      nextTick(() => {
        resetInputHeight()
      })
    }

    function handleFocus(event) {
      if (!state.softFocus) {
        if (automaticDropdown.value || filterable.value) {
          state.visible = true
          if (filterable.value) {
            state.menuVisibleOnFocus = true
          }
        }
        emit('focus', event)
      } else {
        state.softFocus = false
      }
    }

    function blur() {
      state.visible = false
      ctx.$refs.reference.blur()
    }

    function handleBlur(event) {
      setTimeout(() => {
        if (state.isSilentBlur) {
          state.isSilentBlur = false
        } else {
          emit('blur', event)
        }
      }, 50)
      state.softFocus = false
    }

    function handleClearClick(event) {
      deleteSelected(event)
    }
    function toggleLastOptionHitState(hit) {
      if (!Array.isArray(state.selected)) return
      const option = state.selected[state.selected.length - 1]
      if (!option) return

      if (hit === true || hit === false) {
        option.hitState = hit
        return hit
      }

      option.hitState = !option.hitState
      return option.hitState
    }

    function deletePrevTag(e) {
      if (e.target.value.length <= 0 && !toggleLastOptionHitState()) {
        const value = modelValue.value.slice()
        value.pop()
        emit('update:modelValue', value)
        emitChange(value)
      }
    }

    function managePlaceholder() {
      if (state.currentPlaceholder !== '') {
        state.currentPlaceholder = ctx.$refs.input.value
          ? ''
          : state.cachedPlaceHolder
      }
    }

    function resetInputState(e) {
      if (e.keyCode !== 8) toggleLastOptionHitState(false)
      state.inputLength = ctx.$refs.input.value.length * 15 + 20
      resetInputHeight()
    }

    return {
      ...toRefs(state),
      selectSize,
      multiple,
      collapseTagSize,
      selectDisabled,
      readonly,
      showClose,
      iconClass,
      debounce,
      emptyText,
      name,
      id,
      autocomplete,
      showNewOption,
      popperAppendToBody,
      // method
      debouncedOnInputChange,
      debouncedQueryChange,
      toggleMenu,
      selectOption,
      getValueKey,
      onOptionDestroy,
      deleteTag,
      handleMenuEnter,
      handleComposition,
      handleClose,
      doDestroy,
      handleClearClick,
      handleBlur,
      blur,
      handleFocus,
      setSelected,
      deletePrevTag,
      managePlaceholder,
      resetInputState,
      resetInputHeight,
      popperClass,
      handleOptionClick: handleOptionSelect,
      focus
    }
  },
  name: 'ElSelect',

  componentName: 'ElSelect',

  components: {
    ElInput,
    ElSelectMenu,
    ElOption,
    ElTag,
    ElScrollbar
  },

  directives: { Clickoutside: Clickoutside },

  props: {
    name: String,
    id: String,
    modelValue: {
      required: true
    },
    autocomplete: {
      type: String,
      default: 'off'
    },
    automaticDropdown: Boolean,
    size: String,
    disabled: Boolean,
    clearable: Boolean,
    filterable: Boolean,
    allowCreate: Boolean,
    loading: Boolean,
    popperClass: String,
    remote: Boolean,
    loadingText: String,
    noMatchText: String,
    noDataText: String,
    remoteMethod: Function,
    filterMethod: Function,
    multiple: Boolean,
    multipleLimit: {
      type: Number,
      default: 0
    },
    placeholder: {
      type: String,
      default() {
        return t('el.select.placeholder')
      }
    },
    defaultFirstOption: Boolean,
    reserveKeyword: Boolean,
    valueKey: {
      type: String,
      default: 'value'
    },
    collapseTags: Boolean,
    popperAppendToBody: {
      type: Boolean,
      default: true
    }
  }
}
</script>
