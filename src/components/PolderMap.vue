<!-- PolderMap.vue -->
<template>
  <div class="h-screen w-screen relative overflow-hidden">
    <!-- Full Screen Map -->
    <div id="map" class="w-full h-full z-10"></div>
    
    <!-- Control Panel -->
    <ControlPanel 
      @yearChange="onYearChange"
      @peilindexatieChange="onPeilindexatieChange"
    />
    
    <!-- Animation Panel (positioned under ControlPanel when polder is selected) -->
    <div 
      v-if="selectedPolder != null" 
      class="fixed top-80 bottom-4 right-4 w-80 flex justify-center items-center z-50"
    >
      <div class="w-full h-full">
        <AnimationPanel 
          :polderData="selectedPolder"
          :currentYear="currentYear"
          :peilindexatie="currentPeilindexatie"
          @close="closeAnimation"
        />
      </div>
    </div>
  </div>
</template>

<style>
/* Remove focus outline from map elements */
.leaflet-container {
  outline: none !important;
}

.leaflet-container:focus {
  outline: none !important;
}

/* Remove any default selection styling */
.leaflet-interactive {
  outline: none !important;
}

.leaflet-interactive:focus {
  outline: none !important;
}
</style>

<script setup>
import { ref, onMounted, computed, toRaw } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import AnimationPanel from './AnimationPanel.vue'
import ControlPanel from './ControlPanel.vue'

const map = ref(null)
const polderLayer = ref(null)
const selectedPolder = ref(null)
const selectedLayer = ref(null) // Track the currently selected layer

// Control panel state
const currentYear = ref(2025)
const currentPeilindexatie = ref(50)

// Store the original polder data for calculations
const originalPolderData = ref([])

const initMap = () => {
  // Initialize map centered on Netherlands
  map.value = L.map('map').setView([52.1326, 5.2913], 8)
  
  // Add OpenStreetMap tiles
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap contributors'
  }).addTo(map.value)
}

// Function to calculate all yearly data for a polder
const calculateYearlyData = (initialDrooglegging, peilindexatiePercent) => {
  const startYear = 2025
  const endYear = 2075
  const subsidencePerYear = 0.7
  const peilindexatieDecimal = peilindexatiePercent / 100
  const minDrooglegging = 40 // Minimum drooglegging to maintain
  
  const yearlyData = {}
  
  // Initial values for 2025
  yearlyData[startYear] = {
    peatThickness: 200, // cm
    waterHeight: 60, // cm  
    drooglegging: initialDrooglegging,
    effectivePeilindexatie: peilindexatieDecimal
  }
  
  // Calculate each subsequent year
  for (let year = startYear + 1; year <= endYear; year++) {
    const prevYear = yearlyData[year - 1]
    
    // Peat thickness always decreases by subsidence (if peat remains)
    const newPeatThickness = Math.max(0, prevYear.peatThickness - subsidencePerYear)
    
    // Calculate what drooglegging would be with current peilindexatie
    let newDrooglegging = prevYear.drooglegging - subsidencePerYear + (subsidencePerYear * peilindexatieDecimal)
    
    let newWaterHeight
    let effectivePeilindexatie
    
    // Check if we've reached the minimum drooglegging threshold
    if (newDrooglegging <= minDrooglegging) {
      // Lock drooglegging at minimum
      newDrooglegging = minDrooglegging
      
      // Adjust water height to maintain the minimum drooglegging
      // Water must drop by the full subsidence amount to keep drooglegging constant
      newWaterHeight = Math.max(0, prevYear.waterHeight - subsidencePerYear)
      
      // Effective peilindexatie is now 100% (fully compensating)
      effectivePeilindexatie = 1.0
    } else {
      // Normal operation: apply peilindexatie
      effectivePeilindexatie = peilindexatieDecimal
      newWaterHeight = Math.max(0, prevYear.waterHeight - (subsidencePerYear * effectivePeilindexatie))
    }
    
    yearlyData[year] = {
      peatThickness: newPeatThickness,
      waterHeight: newWaterHeight,
      drooglegging: Math.max(0, newDrooglegging),
      effectivePeilindexatie: effectivePeilindexatie
    }
  }
  
  return yearlyData
}

// Function to get drooglegging for current year and peilindexatie
const getDroogleggingForYear = (polderData, year, peilindexatie) => {
  if (!polderData.yearlyData || !polderData.yearlyData[peilindexatie]) {
    // Calculate if not cached
    polderData.yearlyData = polderData.yearlyData || {}
    polderData.yearlyData[peilindexatie] = calculateYearlyData(polderData.originalDrooglegging, peilindexatie)
  }
  
  return polderData.yearlyData[peilindexatie][year]?.drooglegging || polderData.originalDrooglegging
}
// Function to determine color based on drooglegging value (in cm)
const getDroogleggingColor = (drooglegging) => {
  if (drooglegging <= 40) {
    return '#006400'  // Dark green
  } else if (drooglegging < 50) {
    return '#FFFF00'  // Yellow
  } else if (drooglegging < 60) {
    return '#FFA500'  // Light orange
  } else if (drooglegging < 70) {
    return '#FF4500'  // Dark orange
  } else {
    return '#FF0000'  // Red
  }
}

// Function to update all polder colors based on current year and peilindexatie
const updatePolderColors = () => {
  if (!polderLayer.value) return
  
  polderLayer.value.eachLayer((layer) => {
    const polderData = layer.feature.properties
    const currentDrooglegging = getDroogleggingForYear(polderData, currentYear.value, currentPeilindexatie.value)
    
    // Update the feature properties with current values
    layer.feature.properties.drooglegging = currentDrooglegging
    
    // Apply new style
    const isSelected = selectedLayer.value === layer
    layer.setStyle(getPolderStyle(currentDrooglegging, isSelected))
  })
}

// Function to get the base style for a polder based on its drooglegging
const getPolderStyle = (drooglegging, isSelected = false) => {
  const baseColor = getDroogleggingColor(drooglegging)
  
  return {
    color: baseColor,
    weight: isSelected ? 4 : 2,
    fillOpacity: isSelected ? 0.8 : 0.4,
    fillColor: baseColor
  }
}

const resetLayerStyle = (layer) => {
  const drooglegging = layer.feature.properties.drooglegging
  layer.setStyle(getPolderStyle(drooglegging, false))
}

const setSelectedStyle = (layer) => {
  const drooglegging = layer.feature.properties.drooglegging
  layer.setStyle(getPolderStyle(drooglegging, true))
}

const loadPolders = async () => {
  try {
    console.log('Fetching polders.json...')
    const response = await fetch(`${import.meta.env.BASE_URL}polders.json`)
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const poldersData = await response.json()
    console.log('Loaded polders data:', poldersData)
    console.log('Number of polders:', poldersData.length)
    
    // Convert your array format to proper GeoJSON
    const geojsonData = {
      type: "FeatureCollection",
      features: poldersData.map(polder => ({
        type: "Feature",
        properties: {
          id: polder.id,
          name: polder.name,
          drooglegging: polder.drooglegging_2025,
          originalDrooglegging: polder.drooglegging_2025, // Store original value
        },
        geometry: polder.geometry
      }))
    }
    
    // Store original data for calculations
    originalPolderData.value = poldersData
    
    console.log('Converted to GeoJSON')
    
    // Create GeoJSON layer
    polderLayer.value = L.geoJSON(geojsonData, {
      style: (feature) => {
        const drooglegging = feature.properties.drooglegging
        console.log(`Setting color for ${feature.properties.name}: drooglegging=${drooglegging}cm, color=${getDroogleggingColor(drooglegging)}`)
        return getPolderStyle(drooglegging, false)
      },
      onEachFeature: (feature, layer) => {
        console.log(`Adding ${feature.properties.name}`)
        
        // Hover effects
        layer.on('mouseover', (e) => {
          // Show current drooglegging value in tooltip
          const polderData = layer.feature.properties
          const currentDrooglegging = getDroogleggingForYear(polderData, currentYear.value, currentPeilindexatie.value)
          const tooltipContent = `${feature.properties.name}<br/>Drooglegging: ${currentDrooglegging.toFixed(1)}cm (${currentYear.value})`
          layer.bindTooltip(tooltipContent, {
            permanent: false,
            direction: 'top',
            className: 'bg-white px-2 py-1 rounded shadow-lg border'
          }).openTooltip()
          
          // Only apply hover style if not selected
          if (selectedLayer.value !== layer) {
            layer.setStyle({
              fillOpacity: 0.8,
              weight: 4
            })
          }
        })
        
        layer.on('mouseout', () => {
          layer.closeTooltip()
          
          // Only reset style if this layer is NOT the selected one
          // use toRaw to ignore proxy
          if (toRaw(selectedLayer.value) !== layer) {
            resetLayerStyle(layer)
          }
          // If it IS the selected layer, keep the selected style
        })
        
        // Click event
        layer.on('click', () => {
          console.log('Clicked polder:', feature.properties.id, feature.properties.name)
          
          // Reset previous selection
          if (selectedLayer.value) {
            resetLayerStyle(selectedLayer.value)
          }
          
          // Set new selection
          selectedLayer.value = layer
          setSelectedStyle(layer)
          selectedPolder.value = feature.properties

          // Zoom to the clicked polder
          const bounds = layer.getBounds()
          map.value.fitBounds(bounds, { 
            padding: [100, 100],
            maxZoom: 13,
            animate: true,
            duration: 0.5
          })
        })
      }
    }).addTo(map.value)
    
    console.log('GeoJSON layer created and added to map')
    
    // Check bounds and fit map
    const bounds = polderLayer.value.getBounds()
    console.log('Layer bounds:', bounds)
    console.log('Bounds valid?', bounds.isValid())
    
    if (bounds.isValid()) {
      console.log('Southwest:', bounds.getSouthWest())
      console.log('Northeast:', bounds.getNorthEast())
      map.value.fitBounds(bounds, { padding: [20, 20] })
      console.log('Map fitted to bounds with padding')
    } else {
      console.log('Invalid bounds, using default Netherlands view')
      map.value.setView([52.1326, 5.2913], 8)
    }
    
    // Force a redraw
    setTimeout(() => {
      map.value.invalidateSize()
      console.log('Map size invalidated')
    }, 100)
    
  } catch (error) {
    console.error('Error loading polders:', error)
    alert(`Error loading polders: ${error.message}`)
  }
}

// Control panel event handlers
const onYearChange = (data) => {
  currentYear.value = data.year
  currentPeilindexatie.value = data.peilindexatie
  updatePolderColors()
  
  // Update selected polder data if one is selected
  if (selectedPolder.value && selectedLayer.value) {
    const polderData = selectedLayer.value.feature.properties
    const currentData = getDroogleggingForYear(polderData, currentYear.value, currentPeilindexatie.value)
    selectedPolder.value = {
      ...selectedPolder.value,
      drooglegging: currentData,
      yearlyData: polderData.yearlyData // Pass the full yearly data for animation
    }
  }
}

const onPeilindexatieChange = (data) => {
  currentPeilindexatie.value = data.peilindexatie
  
  // Clear cached yearly data for all polders when peilindexatie changes
  if (polderLayer.value) {
    polderLayer.value.eachLayer((layer) => {
      layer.feature.properties.yearlyData = {}
    })
  }
  
  updatePolderColors()
  
  // Update selected polder data if one is selected
  if (selectedPolder.value && selectedLayer.value) {
    const polderData = selectedLayer.value.feature.properties
    const currentData = getDroogleggingForYear(polderData, currentYear.value, currentPeilindexatie.value)
    selectedPolder.value = {
      ...selectedPolder.value,
      drooglegging: currentData,
      yearlyData: polderData.yearlyData // Pass the updated yearly data
    }
  }
}

const closeAnimation = () => {
  // Reset selected layer style when closing
  if (selectedLayer.value) {
    console.log('Closing animation - resetting style for:', selectedLayer.value.feature.properties.name)
    resetLayerStyle(selectedLayer.value)
    selectedLayer.value = null
  }
  selectedPolder.value = null
}

onMounted(() => {
  initMap()
  loadPolders()
})
</script>