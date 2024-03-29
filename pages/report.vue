<template>
  <v-container>
    <v-row justify="center">
      <v-col cols="12" sm="6">
        <v-date-picker
          v-model="datePickerInterval"
          range
          :first-day-of-week="1"
          full-width
          color="primary"
          :show-week="true"
        />
      </v-col>
    </v-row>
    <v-row justify="center">
      <v-data-table :headers="headers" :items="report" />
    </v-row>
  </v-container>
</template>
<script lang="ts">
import { computed, defineComponent, onBeforeUnmount, onMounted, ref, watch } from '@nuxtjs/composition-api'
import { Interval, TimeAggregatorResult } from 'electron/src/timeAggregator'
import { Messages } from '~/electron/src/messages'
import { getDateOffset, useDate } from '~/composition/useDate'
import { sendMessage } from '~/composition/useMessage'
import { useMessageListener } from '~/composition/useMessageListener'
import { useReport } from '~/composition/useReport'

export default defineComponent({
  setup() {
    const toISOStr = (date: number) => {
      return new Date(date - getDateOffset(date)).toISOString().substring(0, 10)
    }

    const fromISOStr = (str: string): number => {
      const [year, month, day] = str.split('-')
      return new Date(Number(year), Number(month) - 1, Number(day)).getTime()
    }

    const date = useDate(ref(Date.now()))
    const result = ref<TimeAggregatorResult[]>([])

    useMessageListener(Messages.NewReport, (_, message: TimeAggregatorResult[]) => {
      result.value = message
    })

    const currentDate = Date.now()
    const interval = ref<Interval>({ from: currentDate, to: currentDate })

    const generateReport = () => {
      console.log(`Generating report for value: ${JSON.stringify(interval.value)}`)
      sendMessage(Messages.GenerateReport, interval.value)
    }

    const updateInterval = (newInterval?: Interval) => {
      if (newInterval) {
        interval.value.from = newInterval.from
        interval.value.to = newInterval.to
        generateReport()
      }
    }

    updateInterval(date.firstAndLastDayOfWeek.value)
    watch(date.firstAndLastDayOfWeek, updateInterval)

    let reportRefresher: ReturnType<typeof setInterval> | null
    onMounted(() => {
      generateReport()
      reportRefresher = setInterval(generateReport, 10000)
    })

    onBeforeUnmount(() => {
      if (reportRefresher) {
        clearInterval(reportRefresher)
        reportRefresher = null
      }
    })

    type DatePickerInterval = string[]

    const tmpDatePickerInterval = ref<DatePickerInterval>([
      toISOStr(interval.value.from),
      toISOStr(interval.value.to),
    ])

    const datePickerInterval = computed<DatePickerInterval>({
      get() {
        return tmpDatePickerInterval.value
      },
      set([from, to]: DatePickerInterval) {
        tmpDatePickerInterval.value = [from, to].filter(v => v !== undefined).sort()
        const v = tmpDatePickerInterval.value
        if (v.length === 2) {
          // to-date has to be a day after in order to include the whole day in the report
          console.log(v)
          const toDate = new Date(fromISOStr(v[1]) + 24 * 3600 * 1000 - 1)
          updateInterval({ from: fromISOStr(v[0]), to: toDate.getTime() })
        }
      },
    })

    return {
      datePickerInterval,
      currentDate: toISOStr(currentDate),
      dates: ref([]),
      ...useReport({
        result,
        interval,
      }),
    }
  },
})
</script>
