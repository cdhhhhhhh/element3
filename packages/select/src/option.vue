<template>
    <li
            @mouseenter="hoverItem"
            @click.stop="selectOptionClick"
            class="el-select-dropdown__item"
            v-show="visible"
            :class="{
      selected: itemSelected,
      'is-disabled': disabled || groupDisabled || limitReached,
      hover: hover
    }"
    >
        <slot>
            <span>{{ currentLabel }}</span>
        </slot>
    </li>
</template>

<script type="text/babel">
  import { useEmitter } from 'element-ui/src/use/emitter'
  import { getValueByPath, escapeRegexpString } from 'element-ui/src/utils/util'
  import {
    computed,
    getCurrentInstance,
    inject,
    provide,
    reactive,
    toRefs,
    watch,
    watchEffect,
    nextTick,
    onMounted,
    onDeactivated
  } from 'vue'

  export default {

    name: 'ElOption',

    componentName: 'ElOption',

    setup(props) {
      const { on, dispatch } = useEmitter()
      const {
        value,
        label,
        created,
        disabled
      } = toRefs(props)

      const state = reactive({
        index: -1,
        groupDisabled: false,
        visible: true,
        hitState: false,
        hover: false
      })

      const select = inject('select').ctx

      function queryChange(query) {
        state.visible =
          new RegExp(escapeRegexpString(query), 'i').test(currentLabel.value) ||
          created.value
        if (!state.visible) {
          select.filteredOptionsCount--
        }
      }

      function handleGroupDisabled(val) {
        state.groupDisabled = val
      }

      function hoverItem() {
        if (!disabled.value && !state.groupDisabled) {
          select.hoverIndex = select.options.indexOf(getCurrentInstance())
        }
      }

      function selectOptionClick() {
        if (disabled.value !== true && state.groupDisabled !== true) {
          dispatch('ElSelect', 'handleOptionClick', [getCurrentInstance(), true])
        }
      }

      function onCreated() {
        select.options.push(getCurrentInstance())
        select.cachedOptions.push(getCurrentInstance())
        select.optionsCount++
        select.filteredOptionsCount++

        on('queryChange', queryChange)
        on('handleGroupDisabled', handleGroupDisabled)
      }

      function isEqual(a, b) {
        if (!isObject) {
          return a === b
        } else {
          const valueKey = select.valueKey
          return getValueByPath(a, valueKey) === getValueByPath(b, valueKey)
        }
      }

      function contains(arr = [], target) {
        if (!isObject) {
          return arr && arr.indexOf(target) > -1
        } else {
          const valueKey = select.valueKey
          return (
            arr &&
            arr.some((item) => {
              return (
                getValueByPath(item, valueKey) ===
                getValueByPath(target, valueKey)
              )
            })
          )
        }
      }

      const isObject = computed(() => {
        return (
          Object.prototype.toString.call(value.value).toLowerCase() ===
          '[object object]'
        )
      })
      const currentLabel = computed(() => {

        return label?.value || (isObject.value ? '' : value.value)
      })

      // const currentValue = computed(() => {
      //   return value.value || label.value || ''
      //
      // })

      const itemSelected = computed(() => {
        if (!select.multiple) {
          return isEqual(value.value, select.value)
        } else {
          return contains(select.value, value.value)
        }
      })

      const limitReached = computed(() => {
        if (select.multiple) {
          return (
            !itemSelected.value &&
            (select.value || []).length >= select.multipleLimit &&
            select.multipleLimit > 0
          )
        } else {
          return false
        }
      })

      onCreated()
      onDeactivated(() => {
        const { selected, multiple } = select
        const selectedOptions = multiple ? selected : [selected]
        //TODO
        const index = select.cachedOptions.indexOf(getCurrentInstance())
        const selectedIndex = selectedOptions.indexOf(getCurrentInstance())

        // if option is not selected, remove it from cache
        if (index > -1 && selectedIndex < 0) {
          select.cachedOptions.splice(index, 1)
        }
        select.onOptionDestroy(select.options.indexOf(getCurrentInstance()))
      })

      watch(value, (val, oldVal) => {
        const { remote, valueKey } = select
        if (!created.value && !remote) {
          if (
            valueKey &&
            typeof val === 'object' &&
            typeof oldVal === 'object' &&
            val[valueKey] === oldVal[valueKey]
          ) {
            return
          }
          dispatch('ElSelect', 'setSelected')
        }
      })

      watch(currentLabel, () => {
        if (!created.value && !select.remote)
          dispatch('ElSelect', 'setSelected')
      })

      return {
        ...toRefs(state),
        hoverItem,
        selectOptionClick,
        itemSelected,
        limitReached,
      }
    },
    props: {
      value: {
        required: true
      },
      label: [String, Number],
      created: Boolean,
      disabled: {
        type: Boolean,
        default: false
      }
    }
  }
</script>
