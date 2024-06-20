<script setup lang="ts">
import OptionsProfitCalculator from './components/OptionsProfitCalculator.vue'
import { Chart as ChartJS, Title, Tooltip, Legend, LineElement, CategoryScale, LinearScale, PointElement } from 'chart.js'
import Annotation from 'chartjs-plugin-annotation';

import { sampleData } from './data/options-data';

ChartJS.register(Title, Tooltip, Legend, Annotation, LineElement, CategoryScale, LinearScale, PointElement)
</script>

<template>
  <OptionsProfitCalculator :optionsData="sampleData" />
</template>

<style scoped>

</style>
