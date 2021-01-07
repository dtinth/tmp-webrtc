<template>
  <h1>tmp-webrtc</h1>
  <Configurator v-if="configuring" :config="config" />
  <template v-else>
    <FileSharing :p2pt="p2pt" />
    <p><button @click="configuring = true">Configure</button></p>
  </template>
</template>

<script lang="ts">
import { computed, defineComponent, reactive, ref } from 'vue'
import Configurator from './Configurator.vue'
import FileSharing from './FileSharing.vue'
import { Config } from './types'

/// <reference path="./p2pkit.d.ts" />

function useConfig(onSave: () => void) {
  const save = (nextConfig: Config) => {
    Object.assign(config, nextConfig)
    localStorage.TMP_WEBRTC_CHANNEL = config.channel
    onSave()
  }
  const config: Config & { save: typeof save } = reactive({
    channel: localStorage.TMP_WEBRTC_CHANNEL || '',
    save,
  })
  return config
}

const announceURLs = [
  'wss://tracker.openwebtorrent.com',
  'wss://tracker.sloppyta.co:443/announce',
  'wss://tracker.novage.com.ua:443/announce',
  'wss://tracker.btorrent.xyz:443/announce',
]

export default defineComponent({
  components: {
    Configurator,
    FileSharing,
  },
  setup() {
    const config = useConfig(() => {
      configuring.value = false
    })
    const configuring = ref(!config.channel)
    const p2pt = computed(() => {
      return new p2pkit.P2PT(announceURLs, 'tmp-webrtc$' + config.channel)
    })
    return { config, configuring, p2pt }
  },
})
</script>
