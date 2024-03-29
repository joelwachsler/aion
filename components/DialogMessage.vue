<template>
  <v-row justify="center">
    <v-dialog
      :value="dialog"
      persistent
      max-width="500"
    >
      <v-card>
        <v-card-title class="text-h5">
          You've been away for {{ awayCounter }}
        </v-card-title>
        <v-card-text>Would you like this time to be tracked?</v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn
            color="error darken-1"
            text
            @click="doNotTrack"
          >
            No
          </v-btn>
          <v-btn
            color="green darken-1"
            text
            @click="track"
          >
            Yes
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>
<script lang="ts">
import { computed, defineComponent, onBeforeUnmount, onMounted, ref, watch } from '@nuxtjs/composition-api'
import { sendMessage } from '~/composition/useMessage'
import { useMessageListener } from '~/composition/useMessageListener'
import { UpdateEventArgs } from '~/electron/src/messageListener/impl/updateEvent'
import { Messages } from '~/electron/src/messages'
import { TimeEvent } from '~/electron/src/timeAggregator'

export default defineComponent({
  setup() {
    const eventToHandle = ref<TimeEvent | undefined>()
    const dialog = computed(() => eventToHandle.value !== undefined)

    watch(dialog, shouldDisplay => {
      if (shouldDisplay) {
        // eslint-disable-next-line no-new
        // new Notification({ title: 'Action required' })
        sendMessage(Messages.SendNotification, { title: 'Action required' })
      }
    })

    const awayTimestamp = computed(() => eventToHandle.value?.timestamp ?? 0)
    const currentTimestamp = ref(Date.now())

    const awayCounter = computed(() => {
      const seconds = Math.round((currentTimestamp.value - awayTimestamp.value) / 1000)
      return new Date(seconds * 1000).toISOString().substr(11, 8)
    })

    let intervalRef: ReturnType<typeof setInterval> | undefined
    onMounted(() => {
      intervalRef = setInterval(() => {
        currentTimestamp.value = Date.now()
      }, 1000)

      sendMessage(Messages.UiReady)
    })

    onBeforeUnmount(() => {
      if (intervalRef) {
        clearInterval(intervalRef)
        intervalRef = undefined
      }
    })

    useMessageListener(Messages.ManualEventHandling, (_, event: TimeEvent) => {
      eventToHandle.value = event
    })

    return {
      track() {
        const msg: UpdateEventArgs = {
          eventId: eventToHandle.value?.id ?? '-1',
          fieldsToUpdate: {
            track: true,
          },
        }
        sendMessage(Messages.UpdateEvent, msg)
        eventToHandle.value = undefined
        sendMessage(Messages.GetSecondsTrackedForDay, Date.now())
      },
      doNotTrack() {
        const lastEvent = eventToHandle.value
        if (lastEvent) {
          const msg: UpdateEventArgs = {
            eventId: lastEvent.id,
            fieldsToUpdate: {
              track: false,
            },
          }
          sendMessage(Messages.UpdateEvent, msg)
          eventToHandle.value = undefined

          // track the current event
          sendMessage(Messages.NewEvent, lastEvent.name)
          sendMessage(Messages.GetSecondsTrackedForDay, Date.now())
        }
      },
      awayCounter,
      dialog,
    }
  },
})
</script>
