<template>
  <q-page class="q-pa-sm">
    <DxScheduler
      ref="scheduler"
      time-zone="Europe/Oslo"
      :data-source="dataSource"
      :current-date="currentDate"
      :views="views"
      :height="600"
      :start-day-hour="7"
      current-view="day"
    >
      <DxResource
        :data-source="types"
        field-expr="typeId"
        label="Type"
      />
    </DxScheduler>
    <q-input class="q-mt-sm" outlined style="width: 50%" v-model="durationInMinutes" type="text" label="Booking duration in minutes" />
    <q-btn no-caps class="q-my-sm q-mr-lg" color="primary" icon="mdi-text-search" label="Find First Available appointment per photographer" @click="onClick" />
 <!--    <q-btn class="q-my-sm q-ml-sm" color="green-5" icon="mdi-close-circle-multiple-outline" label="Clear available appointments" @click="onClear" />
   <q-checkbox v-model="firstbreak" label="Stop on first free space" />  --> 
    <q-input outlined style="width: 50%" v-model="result" type="textarea" label="Result" />
  </q-page>
</template>

<script>
import * as DevExtreme from 'devextreme-vue'
import { DxScheduler, DxResource } from 'devextreme-vue/scheduler'
import moment from 'moment'

const data = {
  "photographers": [
    {
      "id": "1",
      "name": "Otto Crawford",
      "availabilities": [
        {
          "starts": "2020-11-25T08:00:00.000Z",
          "ends": "2020-11-25T16:00:00.000Z"
        }
      ],
      "bookings": [
        {
          "id": "1",
          "starts": "2020-11-25T08:30:00.000Z",
          "ends": "2020-11-25T10:25:00.000Z"
        },
        {
          "id": "2",
          "starts": "2020-11-25T11:30:00.000Z",
          "ends": "2020-11-25T12:35:00.000Z"
        }
      ]
    },
    {
      "id": "2",
      "name": "Jens Mills",
      "availabilities": [
        {
          "starts": "2020-11-25T08:00:00.000Z",
          "ends": "2020-11-25T09:00:00.000Z"
        },
        {
          "starts": "2020-11-25T13:00:00.000Z",
          "ends": "2020-11-25T16:00:00.000Z"
        }
      ],
      "bookings": [
        {
          "id": "11",
          "starts": "2020-11-25T13:00:00.000Z",
          "ends": "2020-11-25T13:45:00.000Z"
        },
        {
          "id": "10",
          "starts": "2020-11-25T15:00:00.000Z",
          "ends": "2020-11-25T16:00:00.000Z"
        }
      ]
    }
  ]
}

export default {
  data () {
    return {
      firstbreak: true,
      durationInMinutes: 30,
      result: '',
      views: ['day', 'week', 'month'],
      currentDate: new Date(2020, 10, 25),
      dataSource: [],
      slots: [],
      types: [
        {
          text: 'Available',
          id: 1,
          color: '#1e90ff'
        }, {
          text: 'Booked',
          id: 2,
          color: 'red'
        },{
          text: 'Slots',
          id: 3,
          color: '#56ca85'
        }
      ],
      timeSlots: [],
      appointments: []
    }
  },
  methods: {
    getMinutes (d1, d2) {
      const start = moment(d1)
      const end = moment(d2)
      var duration = moment.duration(end.diff(start))
      var minutes = duration.asMinutes()      
      return minutes
    },
    addHours (date, hours) {
      const ndate = new Date(date)
      ndate.setHours(ndate.getHours() + hours)
      return ndate
    },
    fixDates (arr) {
      arr.forEach(itm => {
        const dstart = new Date(itm.starts)
        dstart.setHours(dstart.getHours() - 1)
        const dend = new Date(itm.ends)
        dend.setHours(dend.getHours() - 1)
        itm.start = dstart
        itm.end = dend
        itm.duration = this.getMinutes(itm.start, itm.end)
      })
    },
    addAppointment (appointment) {
      this.$refs.scheduler.instance.addAppointment(appointment)
    },
    isSlotInBooking (slot, bookings) {
      let result = 0
      for(const booking of bookings) {
        if ((booking.start >= slot.start && booking.start < slot.end)/* || (booking.end > slot.start)*/) {
          result = booking.id
          break
        }
      }
      return result
    },
    findFreeSlots (photographer, avail, duration, bookings) {
      let start = avail.start
      while(start <= avail.end) {
        const slot = { start: start, end: moment(start).add(duration, 'minutes').toDate() }
        if (slot.end > avail.end) {
          break
        }
        const exists = this.timeSlots.find(f => f.photographer.id === photographer.id)
        const bookingId = this.isSlotInBooking(slot, bookings)
        if (!bookingId && !exists) {
          this.timeSlots.push({
            photographer: {
              id: photographer.id,
              name: photographer.name
            },
            timeSlot: {
              starts: slot.start,
              ends: slot.endDate
            }
          })
          const appointment = {
            id: photographer.id,
            text: photographer.name + ' free slot',
            startDate: slot.start,
            endDate: slot.end,
            typeId: 3
          }
          this.addAppointment(appointment)
          this.appointments.push(appointment)
          if (this.firstbreak) {
            break
          }
        }        
        if (bookingId) {
          const booking = bookings.find(f => f.id === bookingId)
          start = booking.end
        } else {
          start = moment(start).add(duration, 'minutes').toDate()
        }
      }
    },
    availableTimeSlotsForBooking (durationInMinutes) {
      this.timeSlots = []
      data.photographers.forEach(photographer => {
        for(const avail of photographer.availabilities) {
          if (avail.duration >= durationInMinutes) {
            this.findFreeSlots(photographer, avail, durationInMinutes, photographer.bookings)
          }
        }
      })
      return this.timeSlots
    },
    initDataSource () {
      this.dataSource = []
      for(const photographer of data.photographers) {
        for(const availability of photographer.availabilities) {
          this.dataSource.push({
            id: photographer.id + Date.now(),
            text: photographer.name + ' - availabale',
            startDate: availability.start,
            endDate: availability.end,
            typeId: 1
          })
        }
        for(const booking of photographer.bookings) {
          this.dataSource.push({
            id: photographer.id + Date.now(),
            text: photographer.name + ' - booked',
            startDate: booking.start,
            endDate: booking.end,
            typeId: 2
          })
        }
      }
    },
    onClear () {
      if (this.appointments.length) {
        this.appointments.forEach(s => {
          this.$refs.scheduler.instance.deleteAppointment(s)
        })
      }
      this.timeSlot = []
      this.result = ''
    },
    onClick () {
      this.onClear()
      const slots = this.availableTimeSlotsForBooking(this.durationInMinutes * 1)
      this.result = JSON.stringify(slots, null, 2)
    }
  },
  mounted () {
    data.photographers.forEach(photographer => {
      this.fixDates(photographer.availabilities)
      this.fixDates(photographer.bookings)
    })
    this.initDataSource()
  },
  components: {
    DxScheduler, DxResource
  }
}
</script>
