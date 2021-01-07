<template>
  <div>
    <h2>{{ receiving ? 'Receiving' : 'Sending' }} files</h2>
    <div v-for="(entry, index) of incoming" :key="index">
      Receiving <strong>{{ entry.name }}</strong>
      <div>
        <progress :value="entry.percent" max="100"></progress>
        {{ entry.status }}
      </div>
    </div>
    <div v-for="(entry, index) of outgoing" :key="index">
      Sending <strong>{{ entry.name }}</strong>
      <div>
        <progress :value="entry.percent" max="100"></progress>
        {{ entry.status }}
      </div>
    </div>
    <p>
      <strong>Trackers:</strong> {{ trackerStatus }}<br />
      <strong>Peers:</strong> {{ peersStatus }}
    </p>
  </div>
</template>

<script lang="ts">
import { defineComponent, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import { v4 as uuid } from 'uuid'
const spf = new p2pkit.SimplePeerFiles.default()

export default defineComponent({
  props: {
    p2pt: {},
  },
  setup(props) {
    const p2pt: any = props.p2pt
    const sending = new URLSearchParams(location.search).has('send')
    const receiving = !sending
    const trackerStatus = ref('Pending...')
    const peersStatus = ref('Pending...')

    type OutgoingFileTransfer = {
      name: string
      status: string
      fileTransferInitiated: boolean
      percent: number
    }
    const offerings = reactive(
      new Map<
        string,
        { name: string; blob: Blob; transfers: Map<any, OutgoingFileTransfer> }
      >()
    )
    const activePeers = new Set()

    type IncomingFileTransfer = {
      status: string
      peer: any
      name: string
      percent: number
    }
    const acceptedFiles = new Map<string, IncomingFileTransfer>()

    const incoming: IncomingFileTransfer[] = reactive([])
    const outgoing: OutgoingFileTransfer[] = reactive([])

    const sendFile = (name: string, blob: Blob) => {
      offerings.set(uuid(), { name, blob, transfers: new Map() })
      matchOfferings()
    }

    const matchOfferings = () => {
      for (const [id, offering] of offerings) {
        for (const peer of activePeers) {
          if (!offering.transfers.has(peer)) {
            const outgoingTransfer: OutgoingFileTransfer = reactive({
              status: 'Offering file...',
              name: offering.name,
              fileTransferInitiated: false,
              percent: 0,
            })
            offering.transfers.set(peer, outgoingTransfer)
            outgoing.push(outgoingTransfer)
            console.log('Offering file', offering.name, 'to a connected peer')
            p2pt.send(peer, { offer: { id, name: offering.name } })
          }
        }
      }
    }

    onMounted(() => {
      p2pt.start()

      const updatePeerStats = () => {
        peersStatus.value = `Connected to ${activePeers.size}`
      }
      p2pt.on('msg', async (peer, msg) => {
        if (typeof msg !== 'object') return
        console.log('Received', msg)
        try {
          if (msg.offer) {
            if (!receiving) return
            if (acceptedFiles.has(msg.offer.id)) return
            const incomingTransfer: IncomingFileTransfer = reactive({
              status: 'Accepting file...',
              peer,
              name: msg.offer.name,
              percent: 0,
            })
            acceptedFiles.set(msg.offer.id, incomingTransfer)
            incoming.push(incomingTransfer)

            spf.receive(peer, msg.offer.id).then((fileTransfer) => {
              fileTransfer.on('progress', (percentage, bytes) => {
                incomingTransfer.percent = percentage
                incomingTransfer.status = 'Received ' + bytes + ' bytes'
              })
              fileTransfer.on('done', (file) => {
                incomingTransfer.status = 'Receiving completed'
              })
            })
            console.log('Accepting file', msg.offer.name)
            p2pt.send(peer, { accept: { id: msg.offer.id } })
            return
          }

          if (msg.accept) {
            const offering = offerings.get(msg.accept.id)
            if (!offering) return
            const outgoingTransfer = offering.transfers.get(peer)
            if (!outgoingTransfer) return
            if (outgoingTransfer.fileTransferInitiated) return
            outgoingTransfer.fileTransferInitiated = true
            outgoingTransfer.status = 'Initiating file transfer'
            const fileTransfer = await spf.send(
              peer,
              msg.accept.id,
              new File([offering.blob], offering.name, {
                type: offering.blob.type,
              })
            )
            fileTransfer.on('progress', (percentage, bytes) => {
              outgoingTransfer.percent = percentage
              outgoingTransfer.status = 'Sent ' + bytes + ' bytes'
            })
            fileTransfer.on('done', () => {
              outgoingTransfer.status = 'Sending completed'
            })
            outgoingTransfer.status = 'Transferring file'
            fileTransfer.start()
            return
          }
        } catch (error) {
          console.log('Error', error)
        }
      })
      p2pt.on('peerconnect', (peer) => {
        activePeers.add(peer)
        updatePeerStats()
        matchOfferings()
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

    onMounted(() => {
      const startTmpSessionWith = (targetWindow, sessionId) => {
        window.addEventListener('message', async (e) => {
          if (e.data.id === 'getfile') {
            sendFile(e.data.result.file.name, e.data.result.blob)
          }
        })
        targetWindow.postMessage(
          {
            jsonrpc: '2.0',
            method: 'tmp/getOpenedFile',
            params: { sessionId },
            id: 'getfile',
          },
          '*'
        )
      }
      const match = location.hash.match(/tmpsessionid=([\w-]+)/)
      if (window.opener && match) {
        startTmpSessionWith(window.opener, match[1])
      }
    })

    return { trackerStatus, peersStatus, incoming, outgoing, receiving }
  },
})
</script>
