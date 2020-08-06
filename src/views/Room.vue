<template>
  <div class="room">
    <video id="js-local-stream" width="400" autoplay playsinline></video>
    <p>{{ state.peerID }}</p>
    <div class="my-4">
      <input
        v-model="state.roomID"
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none"
        id="room-id"
        type="text"
        placeholder="Room Name"
      />
    </div>
    <button
      class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none"
      type="button"
      @click="onJoin"
    >
      Join
    </button>
    <button
      class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none"
      type="button"
      @click="onLeave"
    >
      Leave
    </button>

    <div class="remote-streams" id="js-remote-streams"></div>
  </div>
</template>

<script lang="ts">
/* eslint-disable @typescript-eslint/no-explicit-any */
import { defineComponent, onMounted, reactive } from 'vue'
import Peer, { MeshRoom } from 'skyway-js'

type State = {
  peerID: string
  roomID: string
}

const peer = new Peer({
  key: process.env.VUE_APP_AKYWAY_APIKEY
    ? process.env.VUE_APP_AKYWAY_APIKEY
    : '',
  debug: 3,
})

let localStream: MediaStream
let room: MeshRoom
let localVideo
let remoteVideos: HTMLElement

export default defineComponent({
  name: 'home',
  setup() {
    const state: State = reactive({
      peerID: '',
      roomID: '',
    })

    const onJoin = () => {
      if (!peer.open) {
        return
      }
      room = peer.joinRoom(state.roomID, {
        mode: 'mesh',
        stream: localStream,
      })

      room.on('stream', async (stream) => {
        const newVideo = document.createElement('video') as any
        newVideo.srcObject = stream
        newVideo.playsInline = true
        newVideo.setAttribute('data-peer-id', stream.peerId)
        remoteVideos.append(newVideo)
        await newVideo.play().catch(console.error)
      })

      room.on('peerLeave', (peerId) => {
        const remoteVideo = remoteVideos.querySelector(
          `[data-peer-id="${peerId}"]`
        ) as any
        remoteVideo.srcObject.getTracks().forEach((track: any) => track.stop())
        remoteVideo.srcObject = null
        remoteVideo.remove()
      })

      room.once('close', () => {
        Array.from(remoteVideos.children).forEach((remoteVideo: any) => {
          remoteVideo.srcObject
            .getTracks()
            .forEach((track: any) => track.stop())
          remoteVideo.srcObject = null
          remoteVideo.remove()
        })
      })
    }

    const onLeave = () => {
      room.close()
    }

    onMounted(async () => {
      localVideo = document.getElementById('js-local-stream') as any
      remoteVideos = document.getElementById('js-remote-streams') as any
      const localStream = await navigator.mediaDevices
        .getUserMedia({
          audio: true,
          video: true,
        })
        .catch(console.error)

      localVideo.muted = true
      localVideo.srcObject = localStream
      localVideo.playsInline = true
      await localVideo.play().catch(console.error)
    })

    return { state, onJoin, onLeave }
  },
})
</script>