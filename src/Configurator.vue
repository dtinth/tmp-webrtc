<template>
  <div>
    <h2>Configuration</h2>
    <form @submit.prevent="submit">
      <p>
        Please set a channel name. Files can be sent to peers on the same
        channel, so make sure it is unique.
      </p>
      <p>
        <label>
          <strong>Channel Name</strong>
          <input type="text" v-model="channel" />
        </label>
      </p>
      <input type="submit" value="Save config" />
    </form>
  </div>
</template>

<script>
import { defineComponent, ref } from 'vue'

export default defineComponent({
  props: {
    config: {
      type: Object,
    },
  },
  setup(props) {
    const channel = ref(props.config.channel || '')
    const submit = () => {
      props.config.save({ channel: channel.value })
    }
    return { channel, submit }
  },
})
</script>
