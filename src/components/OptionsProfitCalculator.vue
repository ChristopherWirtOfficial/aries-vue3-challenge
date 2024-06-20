<script setup lang="ts">
import { Scatter } from 'vue-chartjs';
import { sampleData } from '../data/options-data';
import { ref, computed } from 'vue';

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
  { id: 0, enabled: true, color: '#f87979' },
  { id: 1, enabled: false, color: '#4499EE'},
  { id: 2, enabled: false, color: '#99EE21'},
  { id: 3, enabled: false, color: '#11AAAA'},
]);

const selectedOptions = computed(() => options.value.filter(option => option.enabled));

const N_STEPS = 20;

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
        const potentialPriceDeviations = Array.from({ length: N_STEPS }, (_, i) => -2.5 + (i / (N_STEPS )) * 10);
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
    datasets: potentialExpiryPriceSteps.value.map(option => ({
        label: `Option ${option.id}`,
        data: option.potentialPrices.map((price, i) => ({ x: price, y: option.potentialProfitOrLoss[i] })),
        backgroundColor: options.value[option.id].color,
    })),
}));


/*
    TODO: Changes to make
    - Go back to a (dyanmic) fixed set of prices. Based on the min/max price, have N steps between the global min/max price
    - Specifically insert (unless present) the break-even price of each option into our set of X values
    - ONLY THEN after constructing the X values do we calculate the profits exercising any given option at every X value
    - Then we add the profit (or loss, if negative) at each X value to get our actual Y value for total proift/loss at each price

    All of this is because these options are for the same instrument. You want to maximize the profit over all of the selected options
       as if they were all going to be executed by the same person, not individually. 

    We'll then be able to find which options you should select as a function of the expected exercise price

    TODO: Stretch Goals
    - Create small cards for each option, and place them vertically on the side of the graph. Both display and selection controls.
*/

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
</template>

<style scoped>
.options-list {
    justify-content: center;
    display: flex;
    gap: 10px;
}
</style>
