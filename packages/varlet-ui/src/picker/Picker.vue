<template>
  <component
    :is="dynamic ? n('$-popup') : Transition"
    v-bind="
      dynamic
        ? {
            onOpen,
            onOpened,
            onClose,
            onClosed,
            onClickOverlay,
            onRouteChange,
            closeOnClickOverlay,
            teleport,
            show,
            safeArea,
            'onUpdate:show': handlePopupUpdateShow,
            position: 'bottom',
            class: n('popup'),
          }
        : null
    "
    var-picker-cover
  >
    <div :class="n()" v-bind="$attrs">
      <div :class="n('toolbar')" v-if="toolbar">
        <slot name="cancel">
          <var-button
            :class="n('cancel-button')"
            var-picker-cover
            text
            :text-color="cancelButtonTextColor"
            @click="cancel"
          >
            {{ cancelButtonText ?? pack.pickerCancelButtonText }}
          </var-button>
        </slot>
        <slot name="title">
          <div :class="n('title')">{{ title ?? pack.pickerTitle }}</div>
        </slot>
        <slot name="confirm">
          <var-button
            :class="n('confirm-button')"
            text
            var-picker-cover
            :text-color="confirmButtonTextColor"
            @click="confirm"
          >
            {{ confirmButtonText ?? pack.pickerConfirmButtonText }}
          </var-button>
        </slot>
      </div>
      <div :class="n('columns')" :style="{ height: `${columnHeight}px` }">
        <div
          :class="n('column')"
          v-for="c in scrollColumns"
          :key="c.id"
          @touchstart.passive="handleTouchstart($event, c)"
          @touchmove.prevent="handleTouchmove($event, c)"
          @touchend="handleTouchend(c)"
        >
          <div
            :class="n('scroller')"
            :ref="(el) => setScrollEl(el, c)"
            :style="{
              transform: `translateY(${c.translate}px)`,
              transitionDuration: `${c.duration}ms`,
              transitionProperty: c.duration ? 'transform' : 'none',
            }"
            @transitionend="handleTransitionend(c)"
          >
            <div
              :class="n('option')"
              :style="{ height: `${optionHeight}px` }"
              v-for="(t, i) in c.column.texts"
              :key="t"
              @click="handleClick(c, i)"
            >
              <div :class="n('text')">{{ textFormatter(t, c.columnIndex) }}</div>
            </div>
          </div>
        </div>
        <div
          :class="n('picked')"
          :style="{
            top: `${center}px`,
            height: `${optionHeight}px`,
          }"
        ></div>
        <div :class="n('mask')" :style="{ backgroundSize: `100% ${(columnHeight - optionHeight) / 2}px` }"></div>
      </div>
    </div>
  </component>
</template>

<script lang="ts">
import VarButton from '../button'
import VarPopup from '../popup'
import { defineComponent, watch, ref, computed, Transition, toRaw, type ComponentPublicInstance } from 'vue'
import { props, type CascadeColumn, type NormalColumn } from './props'
import { useTouch } from '@varlet/use'
import { clamp, clampArrayRange, isArray } from '@varlet/shared'
import { toPxNum, getTranslateY } from '../utils/elements'
import { pack } from '../locale'
import { createNamespace, call } from '../utils/components'
import { type Texts } from './index'

export interface ScrollColumn {
  id: number
  touching: boolean
  index: number
  columnIndex: number
  prevY: number
  momentumPrevY: number
  momentumTime: number
  translate: number
  duration: number
  scrolling: boolean
  column: NormalColumn
  columns?: CascadeColumn[]
  scrollEl: HTMLElement | null
}

const { name, n, classes } = createNamespace('picker')

const MOMENTUM_RECORD_TIME = 300
const MOMENTUM_ALLOW_DISTANCE = 15
const TRANSITION_DURATION = 200
const MOMENTUM_TRANSITION_DURATION = 1000
let sid = 0

export default defineComponent({
  name,
  components: {
    VarButton,
    VarPopup,
  },
  inheritAttrs: false,
  props,
  setup(props) {
    const scrollColumns = ref<ScrollColumn[]>([])
    const optionHeight = computed(() => toPxNum(props.optionHeight))
    const optionCount = computed(() => toPxNum(props.optionCount))
    const center = computed(() => (optionCount.value * optionHeight.value) / 2 - optionHeight.value / 2)
    const columnHeight = computed(() => optionCount.value * optionHeight.value)
    const { prevY, moveY, dragging, startTouch, moveTouch, endTouch } = useTouch()

    let prevIndexes: number[] = []

    watch(
      () => props.columns,
      (newValue: any) => {
        scrollColumns.value = props.cascade
          ? normalizeCascadeColumns(toRaw(newValue) as CascadeColumn[])
          : normalizeNormalColumns(toRaw(newValue) as NormalColumn[] | Texts)

        const { indexes } = getPicked()
        prevIndexes = [...indexes]
      },
      {
        immediate: true,
        deep: true,
      }
    )

    function setScrollEl(el: Element | ComponentPublicInstance | null, scrollColumn: ScrollColumn) {
      scrollColumn.scrollEl = el as HTMLElement
    }

    function handlePopupUpdateShow(value: boolean) {
      call(props['onUpdate:show'], value)
    }

    function clampTranslate(scrollColumn: ScrollColumn) {
      const minTranslate = center.value - scrollColumn.column.texts.length * optionHeight.value
      const maxTranslate = optionHeight.value + center.value
      scrollColumn.translate = clamp(scrollColumn.translate, minTranslate, maxTranslate)
    }

    function getTargetIndex(scrollColumn: ScrollColumn, viewTranslate: number) {
      const index = Math.round((center.value - viewTranslate) / optionHeight.value)

      return clampArrayRange(index, scrollColumn.column.texts)
    }

    function updateTranslate(scrollColumn: ScrollColumn) {
      scrollColumn.translate = center.value - scrollColumn.index * optionHeight.value
      return scrollColumn.translate
    }

    function getPicked() {
      const texts = scrollColumns.value.map((scrollColumn) => scrollColumn.column.texts[scrollColumn.index])
      const indexes = scrollColumns.value.map((scrollColumn) => scrollColumn.index)

      return {
        texts,
        indexes,
      }
    }

    function scrollTo(scrollColumn: ScrollColumn, duration = 0) {
      updateTranslate(scrollColumn)
      scrollColumn.duration = duration
    }

    function momentum(scrollColumn: ScrollColumn, distance: number, duration: number) {
      scrollColumn.translate += (Math.abs(distance / duration) / 0.003) * (distance < 0 ? -1 : 1)
    }

    function handleClick(scrollColumn: ScrollColumn, index: number) {
      if (dragging.value) {
        return
      }

      scrollColumn.index = index
      scrollColumn.scrolling = true
      scrollTo(scrollColumn, TRANSITION_DURATION)
    }

    function handleTouchstart(event: TouchEvent, scrollColumn: ScrollColumn) {
      scrollColumn.touching = true
      scrollColumn.translate = getTranslateY(scrollColumn.scrollEl as HTMLElement)
      startTouch(event)
    }

    function handleTouchmove(event: TouchEvent, scrollColumn: ScrollColumn) {
      if (!scrollColumn.touching) {
        return
      }

      moveTouch(event)
      scrollColumn.scrolling = false
      scrollColumn.duration = 0
      scrollColumn.prevY = prevY.value
      scrollColumn.translate += moveY.value

      clampTranslate(scrollColumn)

      const now = performance.now()
      if (now - scrollColumn.momentumTime > MOMENTUM_RECORD_TIME) {
        scrollColumn.momentumTime = now
        scrollColumn.momentumPrevY = scrollColumn.translate
      }
    }

    function handleTouchend(scrollColumn: ScrollColumn) {
      endTouch()

      scrollColumn.touching = false
      scrollColumn.prevY = 0
      const distance = scrollColumn.translate - scrollColumn.momentumPrevY
      const duration = performance.now() - scrollColumn.momentumTime
      const shouldMomentum = Math.abs(distance) >= MOMENTUM_ALLOW_DISTANCE && duration <= MOMENTUM_RECORD_TIME

      if (shouldMomentum) {
        momentum(scrollColumn, distance, duration)
      }

      scrollColumn.index = getTargetIndex(scrollColumn, scrollColumn.translate)
      const oldTranslate = scrollColumn.translate
      const newTranslate = updateTranslate(scrollColumn)
      scrollColumn.scrolling = newTranslate !== oldTranslate
      scrollTo(scrollColumn, shouldMomentum ? MOMENTUM_TRANSITION_DURATION : TRANSITION_DURATION)

      // Can't trigger transition end when not scrolling, change needs to be triggered manually.
      if (!scrollColumn.scrolling) {
        change(scrollColumn)
      }
    }

    function handleTransitionend(scrollColumn: ScrollColumn) {
      scrollColumn.scrolling = false
      change(scrollColumn)
    }

    function normalizeNormalColumns(normalColumns: NormalColumn[]) {
      return normalColumns.map((column: NormalColumn | any[], columnIndex: number) => {
        const normalColumn = (isArray(column) ? { texts: column } : column) as NormalColumn
        const scrollColumn: ScrollColumn = {
          id: sid++,
          prevY: 0,
          momentumPrevY: 0,
          touching: false,
          translate: center.value,
          index: normalColumn.initialIndex ?? 0,
          columnIndex,
          duration: 0,
          momentumTime: 0,
          column: normalColumn,
          scrollEl: null,
          scrolling: false,
        }
        scrollTo(scrollColumn)
        return scrollColumn
      })
    }

    function normalizeCascadeColumns(cascadeColumns: CascadeColumn[]) {
      const scrollColumns: ScrollColumn[] = []

      createChildren(scrollColumns, cascadeColumns, 0, true)

      return scrollColumns
    }

    function createChildren(
      scrollColumns: ScrollColumn[],
      children: CascadeColumn[],
      columnIndex: number,
      initial = false
    ) {
      if (isArray(children) && children.length) {
        const index = initial ? props.cascadeInitialIndexes[scrollColumns.length] ?? 0 : 0

        const scrollColumn: ScrollColumn = {
          id: sid++,
          prevY: 0,
          momentumPrevY: 0,
          touching: false,
          translate: center.value,
          index,
          columnIndex,
          duration: 0,
          momentumTime: 0,
          column: {
            texts: children.map((cascadeColumn) => cascadeColumn[props.textKey]),
          },
          columns: children,
          scrollEl: null,
          scrolling: false,
        }

        scrollColumns.push(scrollColumn)
        scrollTo(scrollColumn)
        createChildren(
          scrollColumns,
          (scrollColumn.columns as CascadeColumn[])[scrollColumn.index].children,
          columnIndex + 1,
          initial
        )
      }
    }

    function rebuildChildren(scrollColumn: ScrollColumn) {
      scrollColumns.value.splice(scrollColumns.value.indexOf(scrollColumn) + 1)
      createChildren(
        scrollColumns.value,
        (scrollColumn.columns as CascadeColumn[])[scrollColumn.index].children,
        scrollColumn.columnIndex + 1
      )
    }

    function isSamePicked() {
      const { indexes } = getPicked()
      return indexes.every((index, idx) => index === prevIndexes[idx])
    }

    function change(scrollColumn: ScrollColumn) {
      const { cascade, onChange } = props

      if (isSamePicked()) {
        return
      }

      if (cascade) {
        rebuildChildren(scrollColumn)
      }

      const hasScrolling = scrollColumns.value.some((scrollColumn) => scrollColumn.scrolling)

      if (hasScrolling) {
        return
      }

      // rebuild will update the value of picked, so need to get the latest value again.
      const { texts, indexes } = getPicked()
      prevIndexes = [...indexes]
      call(onChange, texts, indexes)
    }

    function stopScroll() {
      if (props.cascade) {
        const currentScrollColumn = scrollColumns.value.find((scrollColumn) => scrollColumn.scrolling)

        if (currentScrollColumn) {
          currentScrollColumn.index = getTargetIndex(currentScrollColumn, getTranslateY(currentScrollColumn.scrollEl!))
          currentScrollColumn.scrolling = false
          scrollTo(currentScrollColumn)
          rebuildChildren(currentScrollColumn)
        }
      } else {
        scrollColumns.value.forEach((scrollColumn) => {
          scrollColumn.index = getTargetIndex(scrollColumn, getTranslateY(scrollColumn.scrollEl!))
          scrollTo(scrollColumn)
        })
      }
    }

    // expose
    function confirm() {
      stopScroll()

      const { texts, indexes } = getPicked()
      prevIndexes = [...indexes]
      call(props.onConfirm, texts, indexes)
    }

    // expose
    function cancel() {
      stopScroll()

      const { texts, indexes } = getPicked()
      prevIndexes = [...indexes]
      call(props.onCancel, texts, indexes)
    }

    return {
      pack,
      optionHeight,
      optionCount,
      scrollColumns,
      columnHeight,
      center,
      Transition,
      n,
      classes,
      setScrollEl,
      handlePopupUpdateShow,
      handleTouchstart,
      handleTouchmove,
      handleTouchend,
      handleTransitionend,
      confirm,
      cancel,
      handleClick,
    }
  },
})
</script>

<style lang="less">
@import '../styles/common';
@import '../button/button';
@import '../popup/popup';
@import './picker';
</style>
