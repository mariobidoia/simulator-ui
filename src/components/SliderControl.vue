<template>
  <div>
    <div class="flex justify-between text-sm mb-1">
      <label class="text-slate-300">{{ label }}</label>
      <span class="font-mono text-cyan-300">{{ modelValue }}</span>
    </div>
    <input 
      type="range" 
      :value="modelValue" 
      @input="$emit('update:modelValue', Number($event.target.value)); $emit('update')" 
      :min="min" :max="max" :step="step" 
      class="w-full accent-cyan-500 bg-slate-700 rounded-lg appearance-none h-2 cursor-pointer"
    />
  </div>
</template>

<script setup>
defineProps({
  modelValue: Number,
  label: String,
  min: [String, Number],
  max: [String, Number],
  step: [String, Number]
});

defineEmits(['update:modelValue', 'update']);
</script>