<template>
  <div>
    <h2>Send files</h2>
    <p>
      <strong>Trackers:</strong> {{ trackerStatus }}<br />
      <strong>Peers:</strong> {{ peersStatus }}
    </p>
  </div>
</template>

<script lang="ts">
import { defineComponent, onBeforeUnmount, onMounted, ref } from 'vue'

export default defineComponent({
  props: {
    p2pt: {},
  },
  setup(props) {
    const p2pt: any = props.p2pt
    const trackerStatus = ref('Pending...')
    const peersStatus = ref('Pending...')
    onMounted(() => {
      const activePeers = new Set()
      p2pt.start()

      const updatePeerStats = () => {
        peersStatus.value = `Connected to ${activePeers.size}`
      }
      p2pt.on('peerconnect', (peer) => {
        activePeers.add(peer)
        updatePeerStats()
      })
      p2pt.on('peerclose', (peer) => {
        activePeers.delete(peer)
        updatePeerStats()
      })

      const updateTrackerStats = (stats) => {
        trackerStatus.value = `Connected to ${stats.connected} / ${stats.total}`
      }
      p2pt.on('trackerwarning', (error, stats) => {
        updateTrackerStats(stats)
      })
      p2pt.on('trackerconnect', (tracker, stats) => {
        updateTrackerStats(stats)
      })
    })
    onBeforeUnmount(() => {
      p2pt.destroy()
    })
    return { trackerStatus, peersStatus }
  },
})
</script>
