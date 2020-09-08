<template>
  <ul class="el-select-group__wrap" v-show="visible">
    <li class="el-select-group__title">{{ label }}</li>
    <li>
      <ul class="el-select-group">
        <slot></slot>
      </ul>
    </li>
  </ul>
</template>

<script type="text/babel">
import { useEmitter } from 'element-ui/src/use/emitter'
import { getCurrentInstance } from 'vue'
export default {
  setup() {
    const { on, broadcast } = useEmitter()
    return {
      on,
      broadcast
    }
  },
  name: 'ElOptionGroup',

  componentName: 'ElOptionGroup',

  props: {
    label: String,
    disabled: {
      type: Boolean,
      default: false
    }
  },

  data() {
    return {
      visible: true
    }
  },

  watch: {
    disabled(val) {
      this.broadcast('ElOption', 'handleGroupDisabled', val)
    }
  },

  methods: {
    queryChange() {
      const { subTree } = getCurrentInstance()
      this.visible =
        subTree.children &&
        Array.isArray(subTree.children) &&
        subTree.children.some((option) => option.visible === true)
    }
  },

  created() {
    this.on('queryChange', this.queryChange)
  },

  mounted() {
    if (this.disabled) {
      this.broadcast('ElOption', 'handleGroupDisabled', this.disabled)
    }
  }
}
</script>
