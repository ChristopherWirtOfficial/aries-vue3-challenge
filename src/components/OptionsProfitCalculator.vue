<script setup lang="ts">
import { Scatter } from 'vue-chartjs';
import { sampleData } from '../data/options-data';
import { ref, computed } from 'vue';

const chartOptions = {
    responsive: true,
    plugins: {
            annotation: {
                annotations: {
                    lineAtZero: {
                        drawTime: 'beforeDatasetsDraw',
                        type: 'line',
                        yMin: 0,
                        yMax: 0,
                        borderColor: 'white',
                        borderWidth: 2,
                        borderDash: [8, 3], // Optional: for a dashed line,
                        
                    }
                }
            }
        }
};

const options = ref([
  { id: 0, enabled: true, color: '#f87979' },
  { id: 1, enabled: false, color: '#4499EE'},
  { id: 2, enabled: false, color: '#99EE21'},
  { id: 3, enabled: false, color: '#11AAAA'},
]);

const selectedOptions = computed(() => options.value.filter(option => option.enabled));

const N_STEPS = 10;
// For every selected option, we compute several potential expiry price steps
//   from -2.5*premium to 2.5*premium on either side of the strike price
//   and calculate the profit/loss at each step

type OptionsChartData = {
    id: number;
    potentialPrices: number[];
    potentialProfitOrLoss: number[];
    strikePrice: number;
    premium: number;
};

const potentialExpiryPriceSteps = computed<OptionsChartData[]>(() => {
    const ret = selectedOptions.value.map(({ id }) => {
        const option = sampleData[id];
        const premium = option.long_short === 'long' ? option.ask : option.bid;

        // From -2.5*premium to 2.5*premium, which we'll then calculate potentialPrices from based on strike_price + potentialPriceDeviation
        const potentialPriceDeviations = Array.from({ length: N_STEPS }, (_, i) => -2.5 + (i / (N_STEPS)) * 5);
        const potentialPrices = potentialPriceDeviations
            .map(deviation => option.strike_price + deviation * premium)
            .map(price => Math.round(price * 100) / 100);

        // We can then calculate the potential profits (or losses) at each step based on the option type, strike price, and premium
        const potentialProfitOrLoss = potentialPrices.map(price => price - option.strike_price - premium);

        return {
            id,
            potentialPrices,
            potentialProfitOrLoss,
            strikePrice: option.strike_price,
            premium,
        };
    });

    return ret;
});

const chartData = computed(() => ({
    labels: ['January', 'February', 'March', 'April', 'May', 'June', 'July'],
    datasets: potentialExpiryPriceSteps.value.map(option => ({
        label: `Option ${option.id}`,
        data: option.potentialPrices.map((price, i) => ({ x: price, y: option.potentialProfitOrLoss[i] })),
        backgroundColor: options.value[option.id].color,
    })),
}));

console.log(potentialExpiryPriceSteps.value);
</script>

<template>
    <h1>Options Profit Calculator</h1>
    <Scatter
        id="options-profit-line-chart"
        :options="chartOptions"
        :data="chartData"
    />
    <div class="options-list">
        <div v-for="(option, index) in options" :key="index">
        <input type="checkbox" v-model="option.enabled" />
        <label>Option {{ option.id }}</label>
        </div>
    </div>
    <div>
        <h2>Selected Options</h2>
        <ul>
            <li v-for="option in selectedOptions" :key="option.id">
                Option {{ option.id }}
            </li>
        </ul>
    </div>
</template>

<style scoped>
.options-list {
    justify-content: center;
    display: flex;
    gap: 10px;
}
</style>
