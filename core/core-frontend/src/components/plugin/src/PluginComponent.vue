<script lang="ts" setup>
import noLic from './nolic.vue'
import { ref, useAttrs, onMounted } from 'vue'
import { execute, randomKey, formatArray } from './convert'
import { loadPluginApi, loadDistributed, xpackModelApi } from '@/api/plugin'
import { useCache } from '@/hooks/web/useCache'
import { i18n } from '@/plugins/vue-i18n'
import * as Vue from 'vue'
import axios from 'axios'
import * as Pinia from 'pinia'
import * as vueRouter from 'vue-router'
import { useEmitt } from '@/hooks/web/useEmitt'

const { wsCache } = useCache()

const plugin = ref()

const loading = ref(false)

const attrs = useAttrs()

const showNolic = () => {
  plugin.value = noLic
  loading.value = false
}
const generateRamStr = (len: number) => {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  let randomStr = ''
  for (var i = 0; i < len; i++) {
    randomStr += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return randomStr
}

const importProxy = (bytesArray: any[]) => {
  /* const promise = import(
    `../../../../../../../${formatArray(bytesArray[7])}/${formatArray(bytesArray[8])}/${formatArray(
      bytesArray[9]
    )}/${formatArray(bytesArray[10])}/${formatArray(bytesArray[11])}.vue`
  ) */
  const promise = import(
    `../../../../../../../extensions-view-3dpie/${formatArray(bytesArray[8])}/${formatArray(
      bytesArray[9]
    )}/${formatArray(bytesArray[10])}/${formatArray(bytesArray[11])}.vue`
  )
  promise
    .then((res: any) => {
      plugin.value = res.default
    })
    .catch(e => {
      console.error(e)
      showNolic()
    })
}

const loadComponent = () => {
  loading.value = true
  const byteArray = wsCache.get(`de-plugin-proxy-plugin`)
  if (byteArray) {
    importProxy(JSON.parse(byteArray))
    loading.value = false
    return
  }
  const key = generateRamStr(randomKey())
  loadPluginApi(key)
    .then(response => {
      let code = response.data
      const byteArray = execute(code, key)
      storeCacheProxy(byteArray)
      importProxy(byteArray)
    })
    .catch(() => {
      emits('loadFail')
      showNolic()
    })
    .finally(() => {
      loading.value = false
    })
}
const storeCacheProxy = byteArray => {
  const result = []
  byteArray.forEach(item => {
    result.push([...item])
  })
  wsCache.set(`de-plugin-proxy-plugin`, JSON.stringify(result))
}
const pluginProxy = ref(null)
const invokeMethod = param => {
  if (pluginProxy.value['invokeMethod']) {
    pluginProxy.value['invokeMethod'](param)
  } else {
    pluginProxy.value[param.methodName](param.args)
  }
}

onMounted(async () => {
  const key = 'xpack-model-distributed'
  let distributed = false
  if (wsCache.get(key) === null) {
    const res = await xpackModelApi()
    wsCache.set('xpack-model-distributed', res.data)
    distributed = res.data
  } else {
    distributed = wsCache.get(key)
  }
  if (distributed) {
    if (window['DEXPack']) {
      const xpack = await window['DEXPack'].mapping[attrs.jsname]
      plugin.value = xpack.default
    } else {
      window['Vue'] = Vue
      window['Axios'] = axios
      window['Pinia'] = Pinia
      window['vueRouter'] = vueRouter
      window['MittAll'] = useEmitt().emitter.all
      window['I18n'] = i18n
      loadDistributed().then(async res => {
        new Function(res.data)()
        const xpack = await window['DEXPack'].mapping[attrs.jsname]
        plugin.value = xpack.default
      })
    }
  } else {
    loadComponent()
  }
})

const emits = defineEmits(['loadFail'])
defineExpose({
  invokeMethod
})
</script>

<template>
  <component
    :key="attrs.jsname"
    ref="pluginProxy"
    :is="plugin"
    v-loading="loading"
    v-bind="attrs"
  ></component>
</template>

<style lang="less" scoped></style>
