<script setup lang="ts">
import { Scatter } from 'vue-chartjs';
import { OptionsData } from '../data/options-data';
import { ref, computed } from 'vue';

import OptionCard from './OptionCard.vue';

const props = defineProps<{
    optionsData: OptionsData[],
}>();

const currencyFormatter = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD',
});

const chartOptions = {
    responsive: true,
    plugins: {
            annotation: {
                annotations: {
                    breakevenLine: {
                        drawTime: 'beforeDatasetsDraw', // Ensures the data draws over the break-even line
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
  { id: 0, enabled: true },
  { id: 1, enabled: false },
  { id: 2, enabled: false },
  { id: 3, enabled: false },
]);

const selectedOptions = computed(() => options.value.filter(option => option.enabled));

const N_STEPS = 20;

// Calculate our domain based on the selected options
const domain = computed(() => {
    const minMax = selectedOptions.value.reduce((acc, { id }) => {
        const option = props.optionsData[id];
        const premium = option.long_short === 'long' ? option.ask : option.bid;

        return {
            min: Math.min(acc.min, option.strike_price - 2.5 * premium),
            max: Math.max(acc.max, option.strike_price + 2.5 * premium),
        };
    }, { min: Infinity, max: -Infinity });

    return minMax;
});


const xValues = computed(() => {
    const { min, max } = domain.value;
    const iterativeSteps = Array.from({ length: N_STEPS }, (_, i) => min + (i / (N_STEPS - 1)) * (max - min));

    return iterativeSteps;
});



// Construct our y values according to each x value as the price of the underlying instrument
//   if all selected options are executed for that price
const profitLossValues = computed(() => {
    return xValues.value.map(price => {
        return selectedOptions.value.reduce((acc, { id }) => {
            const option = props.optionsData[id];
            const premium = option.long_short === 'long' ? option.ask : option.bid;
            let profitLoss = 0;

            if (option.type === 'Call') {
                if (option.long_short === 'long') {
                    // Long Call
                    profitLoss = price > option.strike_price ? price - option.strike_price - premium : -premium;
                } else {
                    // Short Call
                    profitLoss = price > option.strike_price ? premium - (price - option.strike_price) : premium;
                }
            } else if (option.type === 'Put') {
                if (option.long_short === 'long') {
                    // Long Put
                    profitLoss = price < option.strike_price ? option.strike_price - price - premium : -premium;
                } else {
                    // Short Put
                    profitLoss = price < option.strike_price ? premium - (option.strike_price - price) : premium;
                }
            }

            return acc + profitLoss;
        }, 0);
    });
});


const breakEvenPoints = computed(() => {
    let bePoints: number[] = [];
    const prices = xValues.value;
    const profits = profitLossValues.value;
    
    // Iterate over the profit/loss array to find where it crosses zero
    for (let i = 1; i < profits.length; i++) {
        if (profits[i] === 0) {
            bePoints.push(prices[i]);
        } else if (profits[i] * profits[i - 1] < 0) { // Change in sign indicates crossing zero
            // Linear interpolation to find the more accurate break-even point
            const x1 = prices[i - 1], y1 = profits[i - 1];
            const x2 = prices[i], y2 = profits[i];
            const beX = x1 - y1 * (x2 - x1) / (y2 - y1); // Interpolation formula
            bePoints.push(beX);
        }
    }
    return bePoints;
});

const maxProfit = computed(() => {
    return Math.max(...profitLossValues.value);
});

const maxLoss = computed(() => {
    return Math.min(...profitLossValues.value);
});


const chartData = computed(() => ({
    datasets: [{
        label: 'Total Profit/Loss',
        data: xValues.value.map((price, index) => ({
            x: price,
            y: profitLossValues.value[index],
        })),
        backgroundColor: '#f87979',
    },
    {
        label: 'Break-even',
        data: breakEvenPoints.value.map(point => ({
            x: point,
            y: 0,
        })),
        backgroundColor: '#A98989',
        border: '3px',
        borderColor: 'white',
        pointRadius: 5
    },
],
}));

</script>

<template>
    <h1>Options Profit Calculator</h1>
    <div style="display: flex; gap: 1rem;">
        <div style="max-width: 100vw">
            <Scatter
                width="800px"
                id="options-profit-line-chart"
                :options="chartOptions"
                :data="chartData"
            />
        </div>
        <div class="options-list">
            <div v-for="option in options">
                <OptionCard
                    v-if="!!props.optionsData?.[option.id]"
                    :key="option.id"
                    :option="props.optionsData[option.id]"
                    v-model:selected="option.enabled"
                />
            </div>
        </div>
    </div>
    <div class='max-profit-loss'>
        <p>Maximum Profit: {{ currencyFormatter.format(maxProfit) }}</p>
        <p>Maximum Loss: {{ currencyFormatter.format(maxLoss) }}</p>
    </div>
</template>

<style scoped>
.options-list {
    justify-content: center;
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.max-profit-loss {
    width: fit-content;
}
</style>
