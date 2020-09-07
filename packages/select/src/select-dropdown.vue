<template>
  <div
    class="el-select-dropdown el-popper"
    :class="[{ 'is-multiple': multiple }, popperClass]"
    :style="{ minWidth: minWidth }"
  >
    <slot></slot>
  </div>
</template>

<script type="text/babel">
import Popper from 'element-ui/src/utils/vue-popper'
import { useEmitter } from 'element-ui/src/use/emitter'
import { ref, watch, onMounted, getCurrentInstance, inject } from 'vue'
export default {
  name: 'ElSelectDropdown',

  componentName: 'ElSelectDropdown',
  emits:['destroyPopper','updatePopper'],
  mixins: [Popper],
  setup() {
    const { on } = useEmitter()

    const select = inject('select')
    const minWidth = ref('')
    let popperElm, referenceElm

    const { ctx } = getCurrentInstance()

    // const popperClass = computed(() => {
    //   return select.refs.popperClass
    // })
    //
    //
    watch(
      () => select.ctx.inputWidth,
      () => {
        minWidth.value = select.ctx.$el.getBoundingClientRect().width + 'px'
      }
    )

    onMounted(() => {
      referenceElm = select.refs.reference.$el
      popperElm = select.refs.popperElm = ctx.$el
      on('updatePopper', () => {
        if (select.ctx.visible) ctx.updatePopper()
      })
      on('destroyPopper', ctx.destroyPopper)
    })
    return {
      minWidth,
      popperElm,
      referenceElm,
      popperClass: select.ctx.popperClass,
      multiple: select.ctx.multiple
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
