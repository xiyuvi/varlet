<template>
  <div :class="classes(n(), [vertical, n('--column'), n('--row')])" :style="rootStyles">
    <slot />
  </div>
</template>

<script lang="ts">
import { defineComponent, computed, type StyleValue } from 'vue'
import { props } from './props'
import { createNamespace } from '../utils/components'
import { toSizeUnit } from '../utils/elements'

const { name, n, classes } = createNamespace('avatar-group')

export default defineComponent({
  name,
  props,
  setup(props) {
    const rootStyles = computed<StyleValue>(() => {
      if (props.offset == null) {
        return {}
      }

      return {
        '--avatar-group-offset': toSizeUnit(props.offset),
      } as StyleValue
    })

    return {
      rootStyles,
      n,
      classes,
      toSizeUnit,
    }
  },
})
</script>

<style lang="less">
@import '../styles/common';
@import './avatarGroup';
</style>
