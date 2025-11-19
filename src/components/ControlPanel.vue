<!-- ControlPanel.vue -->
<template>
  <div class="bg-white rounded-lg shadow-2xl border border-gray-200 p-4 z-40">
    
    <!-- Header - Always visible -->
    <div 
      @click="toggleExpanded"
      class="flex items-center"
      :class = "isExpanded ? 'mb-4' : 'mb-0'"
    >
      <h3 class="text-base lg:text-lg font-semibold text-gray-800">Uitleg en Instellingen</h3>
      <div 
        class="ml-auto flex items-center justify-center bg-gray-100 hover:bg-gray-200 cursor-pointer w-8 h-8 rounded-full text-gray-300 text-sm transition-transform duration-200"
        :class="{ 'rotate-180': isExpanded }"
      >
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="3" stroke="currentColor" class="w-5 h-5 text-gray-500">
          <path stroke-linecap="round" stroke-linejoin="round" d="m19.5 8.25-7.5 7.5-7.5-7.5" />
        </svg>
      </div>
    </div>

    <!-- Collapsible Content -->
    <div v-show="isExpanded">
      <!-- Peilindexatie Controls -->
      <p class = "text-xs lg:text-sm font-normal text-gray-600 text-justify mb-6">
        Deze visualisatie ondersteunt de peilafweging. Rijnland werkt toe naar droogleggingen van rond de 40cm. Hier ziet u de drooglegging van peilvakken in 2025. Met een keuze hieronder voor een percentage peilindexatie (‘meedalen’), kunt u als gebruiker visualiseren hoe de droogleggingen zich de komende decennia ontwikkelen.
      </p>
      <div class="mb-6 space-y-3">
        <h4 class="text-sm lg:text-base font-medium text-gray-700">Peilindexatie</h4>
        <div class="flex gap-2">
          <button
            v-for="preset in presetValues"
            :key="preset"
            @click="setPeilindexatie(preset)"
            :class="[
              'group relative flex-1 px-3 py-2 rounded-md text-xs lg:text-sm border font-medium transition-colors cursor-pointer',
              selectedPeilindexatie === preset
                ? 'bg-[#0083F1] text-white border-transparent'
                : 'bg-gray-100 text-gray-700 border-gray-300 hover:bg-gray-200',
              preset === 75 ? 'ring-2 ring-gray-300 ring-offset-3' : ''
            ]"
          >
            {{ preset }}%
            <span v-if="preset === 75" class="absolute hidden group-hover:block bg-white/75 border border-gray-300 shadow-lg px-3 py-2 rounded text-sm font-normal text-gray-600 whitespace-nowrap -top-11 left-1/2 -translate-x-1/2 z-10">
              huidig beleid
            </span>
          </button>
        </div>
      </div>

      <!-- Year Controls -->
      <div class="mb-6 space-y-3">
        <!-- Year Slider -->
        <div class="space-y-2">
          <div class="flex justify-between text-xs lg:text-sm text-gray-600">
            <span>2025</span>
            <span class="font-medium text-base lg:text-lg">{{ currentYear }}</span>
            <span>2075</span>
          </div>
          <input 
            v-model.number="currentYear"
            type="range" 
            min="2025" 
            max="2075" 
            step="1"
            class="w-full h-3 bg-gray-200 rounded-lg appearance-none cursor-pointer slider"
            @input="onYearChange"
          >
        </div>
      </div>

      <!-- Play Button -->
      <div class="flex items-center justify-center">
        <button 
          @click="togglePlay"
          class="px-4 py-2 bg-[#0083F1] hover:bg-blue-600 text-white rounded text-sm font-medium transition-colors cursor-pointer flex flex-row items-center"
        >
          <div class="mr-2">
            <svg v-if="!isPlaying" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="size-4">
              <path d="M3 3.732a1.5 1.5 0 0 1 2.305-1.265l6.706 4.267a1.5 1.5 0 0 1 0 2.531l-6.706 4.268A1.5 1.5 0 0 1 3 12.267V3.732Z" />
            </svg>
            <svg v-else xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="size-4">
              <path d="M4.5 2a.5.5 0 0 0-.5.5v11a.5.5 0 0 0 .5.5h1a.5.5 0 0 0 .5-.5v-11a.5.5 0 0 0-.5-.5h-1ZM10.5 2a.5.5 0 0 0-.5.5v11a.5.5 0 0 0 .5.5h1a.5.5 0 0 0 .5-.5v-11a.5.5 0 0 0-.5-.5h-1Z" />
            </svg>
          </div>
          {{ isPlaying ? 'Pauzeren' : 'Afspelen' }}
        </button>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'

// Define props
const props = defineProps({
  selectedPolder: {
    type: Object,
    default: null
  }
})

// Define emits for parent communication
const emit = defineEmits(['yearChange', 'peilindexatieChange'])

// Expanded state
const isExpanded = ref(true)

// Watch for selectedPolder changes
watch(() => props.selectedPolder, (newValue) => {
  if (newValue !== null && window.innerWidth < 1536) {
    // Collapse on small screens when a polder is selected
    isExpanded.value = false
  }
})

// Toggle function
const toggleExpanded = () => {
  isExpanded.value = !isExpanded.value
}

// Animation state
const currentYear = ref(2025)
const isPlaying = ref(false)
const animationInterval = ref(null)

// Peilindexatie state
const presetValues = [0, 25, 50, 75]
const selectedPeilindexatie = ref(75)

// Constants
const startYear = 2025
const endYear = 2075

// Animation controls
const togglePlay = () => {
  isPlaying.value = !isPlaying.value
  if (isPlaying.value) {
    startAnimation()
  } else {
    stopAnimation()
  }
}

const startAnimation = () => {
  animationInterval.value = setInterval(() => {
    if (currentYear.value >= endYear) {
      currentYear.value = startYear
    } else {
      currentYear.value += 1
    }
    
    // Emit year change
    emit('yearChange', {
      year: currentYear.value,
      peilindexatie: selectedPeilindexatie.value
    })
  }, 200)
}

const stopAnimation = () => {
  isPlaying.value = false
  if (animationInterval.value) {
    clearInterval(animationInterval.value)
    animationInterval.value = null
  }
}

const onYearChange = () => {
  if (isPlaying.value) {
    stopAnimation()
  }
  
  // Emit manual year change
  emit('yearChange', {
    year: currentYear.value,
    peilindexatie: selectedPeilindexatie.value
  })
}

// Peilindexatie controls
const setPeilindexatie = (value) => {
  selectedPeilindexatie.value = value
  
  // Reset to 2025 when peilindexatie changes
  currentYear.value = startYear
  stopAnimation()
  
  // Emit peilindexatie change
  emit('peilindexatieChange', {
    peilindexatie: value,
    currentYear: currentYear.value
  })
  
  // Also emit year change since we reset to 2025
  emit('yearChange', {
    year: currentYear.value,
    peilindexatie: value
  })
}

// Cleanup on unmount
onUnmounted(() => {
  stopAnimation()
})

// Auto-start animation after mount
onMounted(() => {
  // Emit initial state
  emit('yearChange', {
    year: currentYear.value,
    peilindexatie: selectedPeilindexatie.value
  })
})
</script>

<style scoped>
.slider::-webkit-slider-thumb {
  appearance: none;
  height: 20px;
  width: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
}

.slider::-moz-range-thumb {
  height: 20px;
  width: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: none;
}
</style>