<script setup>
import LoadingBar from '..'
import VarButton from '../../button'
import VarSpace from '../../space'
import dark from '../../themes/dark'
import { ref } from 'vue'
import { watchLang, AppType, watchDarkMode } from '@varlet/cli/client'
import { use, pack } from './locale'

const hasCustomStyle = ref(false)

function setStyle() {
  if (hasCustomStyle.value) {
    LoadingBar.resetDefaultOptions()
  } else {
    LoadingBar.setDefaultOptions({
      errorColor: '#ff8800',
      color: '#10afef',
      height: '5px',
    })
  }

  hasCustomStyle.value = !hasCustomStyle.value
}

watchDarkMode(dark)
watchLang(use)

LoadingBar.setDefaultOptions({
  top: '14.5vmin',
})
</script>

<template>
  <app-type>{{ pack.basicUsage }}</app-type>
  <var-space direction="column" :size="['3vmin', '4vmin']">
    <var-button type="primary" block @click="LoadingBar.start()">{{ pack.start }}</var-button>
    <var-button type="primary" block @click="LoadingBar.finish()">{{ pack.finish }}</var-button>
    <var-button type="primary" block @click="LoadingBar.error()">{{ pack.error }}</var-button>
    <var-button type="primary" block @click="setStyle">{{ hasCustomStyle ? pack.clear : pack.custom }}</var-button>
  </var-space>
</template>
