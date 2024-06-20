<script setup lang="ts">
import { OptionsData } from '../data/options-data';
import { defineProps, defineEmits, computed } from 'vue';

const props = defineProps<{
  option: OptionsData;
  selected: boolean;
}>();

const emits = defineEmits(['update:selected']);

const toggleSelected = () => {
  emits('update:selected', !props.selected);
};

const cardClasses = computed(() => ({
  'card': true,
  'selected': props.selected,
}));
</script>

<template>
  <div :class="cardClasses" @click="toggleSelected">
    <div class="card-details">
      <span>Type: {{ props.option.long_short }} {{ props.option.type }}</span>
      <span>Strike Price: ${{ props.option.strike_price }}</span>
      <span>Premium: ${{ props.option.long_short === 'long' ? props.option.ask : props.option.bid }}</span>
    </div>
    <div class="checkbox-container">
      <input type="checkbox" v-model="props.selected" @click.stop>
    </div>
  </div>
</template>

<style scoped>
.card {
  border: 1px solid #777;
  border-radius: 4px;
  padding: 10px;
  margin: 5px;
  cursor: pointer;
  user-select: none;
}

.card span {
  text-transform: capitalize;
}

.selected {
  border: 1px solid #CCC;
  background-color: #444;
}

.card-details {
  display: flex;
  flex-direction: column;
  align-items: start;
}
.checkbox-container {
  display: none;
}
</style>
