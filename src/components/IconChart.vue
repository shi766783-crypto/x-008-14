<template>
  <svg :viewBox="viewBox" class="icon-chart" :style="{ width: size + 'px', height: size + 'px' }" aria-hidden="true">
    <path v-if="name" :d="pathData" :fill-rule="evenOddNames[name] ? 'evenodd' : 'nonzero'" />
  </svg>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  name: { type: String, default: '' },
  size: { type: [Number, String], default: 16 }
})

const paths = {
  home: 'M12 3l8 7v11h-6v-6h-4v6H4V10z',
  chart: 'M4 20V10h4v10zm6 0V4h4v16zm6 0v-7h4v7z',
  add: 'M11 5h2v6h6v2h-6v6h-2v-6H5v-2h6z',
  account: 'M12 12a4 4 0 100-8 4 4 0 000 8zm0 2c-4 0-8 2-8 6v1h16v-1c0-4-4-6-8-6z',
  list: 'M4 6h16v2H4zm0 5h16v2H4zm0 5h16v2H4z',
  target: 'M12 2a10 10 0 100 20 10 10 0 000-20zm0 4a6 6 0 110 12 6 6 0 010-12zm0 4a2 2 0 110 4 2 2 0 010-4z',
  trophy: 'M6 3h12v2a6 6 0 01-12 0zM8 7v1a4 4 0 008 0V7zm-2 2H3a1 1 0 001 1h3zm12 0h3a1 1 0 01-1 1h-3zM7 21h10v2H7zm1-2h8l-1-5h-6z',
  user: 'M12 12a4 4 0 100-8 4 4 0 000 8zm0 2c-5 0-8 2.5-8 5v3h16v-3c0-2.5-3-5-8-5z',
  calendar: 'M5 3h14a2 2 0 012 2v14a2 2 0 01-2 2H5a2 2 0 01-2-2V5a2 2 0 012-2zM5 9h14v11a1 1 0 01-1 1H6a1 1 0 01-1-1V9zM7 1a1 1 0 011 1v1a1 1 0 11-2 0V2a1 1 0 011-1zm10 0a1 1 0 011 1v1a1 1 0 11-2 0V2a1 1 0 011-1zM7.5 10.5a1 1 0 100 2 1 1 0 000-2zm4.5 0a1 1 0 100 2 1 1 0 000-2zm4.5 0a1 1 0 100 2 1 1 0 000-2zM7.5 14.5a1 1 0 100 2 1 1 0 000-2zm4.5 0a1 1 0 100 2 1 1 0 000-2zm4.5 0a1 1 0 100 2 1 1 0 000-2z'
}

const viewBox = computed(() => '0 0 24 24')
const pathData = computed(() => paths[props.name] || '')
const evenOddNames = { calendar: true }
</script>

<style scoped>
.icon-chart {
  fill: currentColor;
  vertical-align: middle;
  flex-shrink: 0;
}
</style>
