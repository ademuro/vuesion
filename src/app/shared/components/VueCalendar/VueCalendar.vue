<template>
  <div :class="$style.calendar">
    <div :class="$style.header">
      <vue-headline
        level="4"
        :native="false"
        @click="setSelecting('year')"
        @keypress.enter.space.stop.prevent="setSelecting('year')"
        role="button"
        tabindex="0"
        :aria-label="selectedYear"
      >
        {{ selectedYear }}
      </vue-headline>
      <vue-headline
        level="5"
        :native="false"
        @click="setSelecting('date')"
        @keypress.enter.space.stop.prevent="setSelecting('date')"
        role="button"
        tabindex="0"
        :aria-label="$d(calculatedDate, 'calendarHeader')"
      >
        {{ $d(calculatedDate, 'calendarHeader') }}
      </vue-headline>
    </div>

    <div :class="$style.body" v-if="selecting === 'date'">
      <div :class="$style.date">
        <div
          :class="$style.arrow"
          @click="setByMonth(currentMonth - 1)"
          @keypress.enter.space.stop.prevent="setByMonth(currentMonth - 1)"
          role="button"
          :aria-label="$t('components.calendar.previousMonth' /* previous month */)"
          tabindex="0"
        ></div>

        <div :class="$style.currentDate">{{ $d(new Date(currentYear, currentMonth, 1), 'calendarNav') }}</div>

        <div
          :class="$style.arrow"
          @click="setByMonth(currentMonth + 1)"
          @keypress.enter.space.stop.prevent="setByMonth(currentMonth + 1)"
          role="button"
          :aria-label="$t('components.calendar.nextMonth' /* next month */)"
          tabindex="0"
        ></div>
      </div>

      <table>
        <thead>
          <tr>
            <td v-for="(weekday, idx) in weekdays" :key="`weekday-${idx}`" :class="$style.disabledDay">
              <span>{{ weekday }}</span>
            </td>
          </tr>
        </thead>

        <tbody>
          <tr v-for="(days, daysIdx) in calendar" :key="`days-${daysIdx}`">
            <td
              :class="[
                day.currentDay ? $style.currentDay : '',
                day.disabled ? $style.disabledDay : '',
                day.selected ? $style.selectedDay : '',
              ]"
              v-for="(day, dayIdx) in days"
              :key="`day-${dayIdx}`"
              :tabindex="day.day ? 0 : null"
              :aria-label="day.day ? $d(new Date(currentYear, currentMonth, day.day), 'calendarLabel') : null"
              @keydown.enter.stop.prevent="setByDay(day)"
              @keydown.space.stop.prevent="setByDay(day)"
              @click="setByDay(day)"
            >
              <span>{{ day.day }}</span>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div :class="$style.year" v-if="selecting === 'year'">
      <div
        :class="[year.selected ? $style.selected : '']"
        :id="`${year.year}-calendar-year`"
        v-for="year in years"
        :key="year.year"
        @click="setByYear(year.year)"
        @keypress.enter.space.stop.prevent="setByYear(year.year)"
        :aria-label="year.year"
        tabindex="0"
      >
        {{ year.year }}
      </div>
    </div>

    <div :class="$style.footer">
      <vue-button @click.stop.prevent="onClose" ghost>{{ $t('common.cancel' /* Cancel */) }}</vue-button>
      <vue-button @click.stop.prevent="onChange" color="primary">{{ $t('common.ok' /* Ok */) }}</vue-button>
    </div>
  </div>
</template>

<script lang="ts">
import chunk from 'lodash/chunk';
import VueButton from '../VueButton/VueButton.vue';
import VueHeadline from '../VueHeadline/VueHeadline.vue';

interface IData {
  selecting: string;
  currentMonth: number;
  currentYear: number;
  selectedDayOfWeek: number;
  selectedDay: number;
  selectedMonth: number;
  selectedYear: number;
}

interface IDay {
  day: number;
  currentDay: boolean;
  selected: boolean;
  disabled: boolean;
}

interface IYear {
  year: number;
  selected: boolean;
}

export default {
  name: 'VueCalendar',
  components: {
    VueHeadline,
    VueButton,
  },
  props: {
    today: {
      type: Date,
      default: () => new Date(),
    },
    minDate: {
      type: Date,
    },
    maxDate: {
      type: Date,
    },
    firstDayOfWeek: {
      type: Number,
      default: 0,
    },
    startDate: {
      type: Date,
    },
    endDate: {
      type: Date,
    },
    selectedDate: {
      type: Date,
    },
  },
  computed: {
    calculatedDate(): Date {
      return new Date(this.selectedYear, this.selectedMonth, this.selectedDay);
    },
    calendar(): IDay[][] {
      let days: number[] = [];

      const paddingLeft = new Date(this.currentYear, this.currentMonth, 1).getDay() - this.firstDayOfWeek;
      const daysInMonth = 32 - new Date(this.currentYear, this.currentMonth, 32).getDate();

      if (paddingLeft >= 0) {
        days = days.concat(Array(paddingLeft).fill(null));
      }

      for (let i = 0; i < daysInMonth; i++) {
        days.push(i + 1);
      }

      const paddingRight = Math.ceil(days.length / 7) * 7 - days.length;

      days = days.concat(Array(paddingRight).fill(null));

      const dayObjects: IDay[] = days.map(
        (day: number): IDay => {
          const date: Date = day ? new Date(this.currentYear, this.currentMonth, day) : new Date(0, 0, 0);
          const currentDay: boolean =
            date.getTime() ===
            new Date(this.today.getFullYear(), this.today.getMonth(), this.today.getDate()).getTime();
          let disabled: boolean = this.dayDisabled(day, date);
          let selected: boolean = date.getTime() === this.calculatedDate.getTime();

          if (this.startDate) {
            disabled = date.getTime() < this.startDate.getTime();
            selected = date.getTime() >= this.startDate.getTime() && date.getTime() <= this.calculatedDate.getTime();
          }

          if (this.endDate) {
            selected = date.getTime() <= this.endDate.getTime() && date.getTime() >= this.calculatedDate.getTime();
          }

          if (day === null) {
            selected = false;
          }

          return { day, currentDay, selected, disabled } as IDay;
        },
      );

      return chunk(dayObjects, 7);
    },
    years(): IYear[] {
      let firstYear;

      if (this.minDate) {
        firstYear = this.minDate.getFullYear();
      } else {
        firstYear = new Date().getFullYear();
      }

      const through = this.maxDate ? this.maxDate.getFullYear() + 1 - firstYear : 101;
      const years = [];

      for (let i = firstYear, len = firstYear + through; i < len; i++) {
        years.push(i);
      }

      return years.map((year) => {
        return { year, selected: year === this.selectedYear } as IYear;
      });
    },
    weekdays(): string[] {
      const weekdays: string[] = [
        this.$t('components.calendar.sunday.short' /* S */),
        this.$t('components.calendar.monday.short' /* M */),
        this.$t('components.calendar.tuesday.short' /* T */),
        this.$t('components.calendar.wednesday.short' /* W */),
        this.$t('components.calendar.thursday.short' /* T */),
        this.$t('components.calendar.friday.short' /* F */),
        this.$t('components.calendar.saturday.short' /* S */),
      ];
      const orderedDays: string[] = [];
      let startDay: number = this.firstDayOfWeek;

      for (let i = 0; i < 7; i++) {
        orderedDays.push(weekdays[startDay]);

        startDay++;

        if (startDay === 7) {
          startDay = 0;
        }
      }

      return orderedDays;
    },
  },
  created() {
    this.setDate();
  },
  data(): IData {
    return {
      selecting: 'date',
      currentMonth: null,
      currentYear: null,
      selectedDayOfWeek: null,
      selectedDay: null,
      selectedMonth: null,
      selectedYear: null,
    };
  },
  methods: {
    setSelecting(value: string): void {
      this.selecting = value;
    },
    setByDay(day: IDay): void {
      if (day.disabled) {
        return;
      }

      this.selectedYear = this.currentYear;
      this.selectedDay = day.day;
      this.selectedMonth = this.currentMonth;
      this.selectedDayOfWeek = new Date(this.selectedYear, this.selectedMonth, day.day).getDay();
    },
    setByMonth(month: number): void {
      if (month === 12) {
        this.currentYear = this.currentYear + 1;
        this.currentMonth = 0;

        return;
      }

      if (month === -1) {
        this.currentYear = this.currentYear - 1;
        this.currentMonth = 11;

        return;
      }

      this.currentMonth = month;
    },
    setByYear(year: number): void {
      this.selectedYear = year;
      this.currentYear = year;
      this.selecting = 'date';
    },
    setDate(): void {
      let date: Date = this.today;

      this.selectedDay = date.getDate();

      if (this.minDate) {
        date = this.minDate;
        this.selectedDay = this.minDate.getDate() + 1;
      }

      if (this.startDate) {
        date = this.startDate;
        this.selectedDay = this.startDate.getDate();
      }

      if (this.selectedDate) {
        date = this.selectedDate;
        this.selectedDay = this.selectedDate.getDate();
      }

      this.selectedDayOfWeek = date.getDay();
      this.selectedMonth = date.getMonth();
      this.currentMonth = date.getMonth();
      this.selectedYear = date.getFullYear();
      this.currentYear = date.getFullYear();
    },
    dayDisabled(day: number, date: Date): boolean {
      let disabled: boolean = false;

      if (!day) {
        disabled = true;
      } else if (this.minDate && date.getTime() < this.minDate.getTime()) {
        disabled = true;
      } else if (this.maxDate && date.getTime() > this.maxDate.getTime()) {
        disabled = true;
      }

      return disabled;
    },
    onChange(): void {
      this.$emit('change', this.calculatedDate);
      this.onClose();
    },
    onClose(): void {
      this.$emit('close');
    },
  },
};
</script>

<style lang="scss" module>
@import '~@/app/shared/design-system';

.calendar {
  width: 100%;
  max-width: $calendar-max-width;
  position: relative;
  box-shadow: $calendar-shadow;
  background: $calendar-bg;
  margin: $calendar-margin;
  display: inline-block;
}

.header {
  color: $calendar-header-color;
  padding: $calendar-header-padding;
  background: $calendar-header-bg;
  text-shadow: $calendar-header-text-shadow;

  cursor: pointer;

  * {
    margin: 0;
    font-weight: $calendar-header-font-weight;
  }
}

.body {
  border: $calendar-body-border;
  padding: $calendar-body-padding;

  table {
    display: block;

    thead,
    tbody {
      display: block;
    }

    tr {
      display: flex;
      flex-wrap: wrap;

      td {
        &:before {
          content: '';
          float: left;
          padding-top: 100%;
        }
        border-radius: $calendar-day-border-radius;
        flex: 1;
        text-align: center;

        &:hover {
          background: $calendar-day-hover;
          cursor: pointer;
        }

        span {
          position: relative;
          top: 20%;
        }
      }
    }
  }
}

.date {
  display: flex;
  height: $space-32;
  flex-direction: row;
}

.arrow {
  flex: 1 0 15%;
  cursor: pointer;
  background: $calendar-arrow-bg;
  position: relative;

  &:hover {
    background: $calendar-arrow-hover-bg;
  }

  &:before,
  &:after {
    content: '';
    transition: all 0.25s ease-in-out;
    position: absolute;
    background-color: $calendar-arrow-color;
    width: $space-2;
    height: $space-12;
    left: 50%;
    margin-left: -($space-8 - $space-2);
    top: $space-4;
  }

  &:before {
    transform: translate($space-4, 0) rotate(45deg);
  }

  &:after {
    transform: translate($space-4, $space-8) rotate(135deg);
  }

  &:last-child {
    &:before,
    &:after {
    }

    &:before {
      transform: translate($space-4, 0) rotate(-45deg);
    }

    &:after {
      transform: translate($space-4, $space-8) rotate(-135deg);
    }
  }
}

.currentDate {
  cursor: default;
  text-align: center;
  flex: 1 0 70%;
  font-weight: $font-weight-regular;
}

.currentDay {
  font-weight: $calendar-current-day-font-weight;
  background: $calendar-current-day-bg;
  color: $calendar-current-day-color;

  &:hover {
    background-color: $calendar-current-day-bg !important;
  }
}

.selectedDay {
  font-weight: $calendar-selected-day-font-weight;
  background-color: $calendar-selected-day-bg;
  color: $calendar-selected-day-color;

  &:hover {
    background-color: $calendar-selected-day-bg !important;
  }
}

.disabledDay {
  opacity: 0.4;
  user-select: none;
  outline: transparent;
}

.year {
  box-shadow: inset 0 -1px 1px rgba(0, 0, 0, 0.075);
  max-height: 312px;
  overflow-y: scroll;
  text-align: center;

  div {
    cursor: pointer;
    padding: $space-8 0;
    transition: background-color 0.15s;

    &:hover {
      background-color: $brand-bg-color;
    }
  }

  .selected {
    font-size: $font-size-h4;
  }
}

.footer {
  padding: $space-12;
  display: flex;
  justify-content: flex-end;
  border: $calendar-body-border;
  border-top: none;

  button {
    margin-bottom: 0;

    &:last-child {
      margin-right: 0;
    }
  }
}
</style>
