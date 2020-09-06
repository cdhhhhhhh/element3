<template>
  <div
    class="el-select-dropdown el-popper"
    :class="[{ 'is-multiple': $parent.multiple }, popperClass]"
    :style="{ minWidth: minWidth }"
  >
    <slot></slot>
  </div>
</template>

<script type="text/babel">
import Popper from 'element-ui/src/utils/vue-popper'
import { useEmitter } from 'element-ui/src/use/emitter'
import {
  ref,
  computed,
  watch,
  onMounted,
  getCurrentInstance,
  inject
} from 'vue'
export default {
  name: 'ElSelectDropdown',

  componentName: 'ElSelectDropdown',

  mixins: [Popper],
  setup() {
    const { on } = useEmitter()

    const select = inject('select')
    const minWidth = ref('')
    let popperElm, referenceElm

    const { refs, ctx } = getCurrentInstance()

    const popperClass = computed(() => {
      return select.refs.popperClass
    })

    watch(
      () => select.refs.inputWidth,
      () => {
        minWidth.value = select.$el.getBoundingClientRect().width + 'px'
      }
    )
    onMounted(() => {
      referenceElm = select.refs.reference.$el
      popperElm = select.refs.popperElm = select.$el
      on('updatePopper', () => {
        if (select.visible) ctx.updatePopper()
      })
      on('destroyPopper', ctx.destroyPopper)
    })
    return {
      minWidth,
      popperElm,
      referenceElm,
      popperClass
    }
  },
  props: {
    placement: {
      default: 'bottom-start'
    },

    boundariesPadding: {
      default: 0
    },

    popperOptions: {
      default() {
        return {
          gpuAcceleration: false
        }
      }
    },

    visibleArrow: {
      default: true
    },

    appendToBody: {
      type: Boolean,
      default: true
    }
  }
}
</script>
