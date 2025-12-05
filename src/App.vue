<script setup lang="ts">
import { onMounted, ref, computed } from 'vue'
import L from 'leaflet'

// Leaflet CSS
import 'leaflet/dist/leaflet.css'
import 'leaflet.markercluster/dist/MarkerCluster.css'
import 'leaflet.markercluster/dist/MarkerCluster.Default.css'
import 'leaflet.markercluster'

// Data Sources
import stopsDataRaw from './data/stops.json'
import stopsMetaRaw from './data/stops_metadata.json'
import shapesDataRaw from './data/shapes_lite.json'

// Declaration des types
interface Stop {
  stop_id: string
  stop_name: string
  stop_lat: number
  stop_lon: number
}

interface StopMeta {
  is_tram: boolean
  lines: string[]
  wheelchair?: boolean
}

interface Shape {
  route_name: string
  color: string
  points: number[][]
}

// Cast raw JSON to typed variables for better IDE support
const stopsData = stopsDataRaw as Stop[]
const stopsMeta = stopsMetaRaw as Record<string, StopMeta>
const shapesData = shapesDataRaw as Record<string, Shape>

// --- 3. APPLICATION STATE --------------------------------
// References to DOM elements and Leaflet instances
const mapContainer = ref<HTMLElement | null>(null)
const map = ref<L.Map | null>(null)
const markersLayer = ref<L.MarkerClusterGroup | null>(null) // Handles thousands of dots
const shapesLayer = ref<L.LayerGroup | null>(null) // Handles colored lines

// UI / Filter State
const showStops = ref(true)
const showBus = ref(true)
const showTram = ref(true)
const showAllLines = ref(false) // Toggle for expanding the sidebar list
const searchQuery = ref('') // Text input for filtering lines
const selectedLines = ref<Set<string>>(new Set()) // Stores "Line 1", "C2", etc.

// --- 4. ICONS & ASSETS -----------------------------------
// SVG Strings for Map Markers and Sidebar Icons
const busSVG = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 640" width="16" height="16" fill="currentColor"><path d="M192 64C139 64 96 107 96 160L96 448C96 477.8 116.4 502.9 144 510L144 544C144 561.7 158.3 576 176 576L192 576C209.7 576 224 561.7 224 544L224 512L416 512L416 544C416 561.7 430.3 576 448 576L464 576C481.7 576 496 561.7 496 544L496 510C523.6 502.9 544 477.8 544 448L544 160C544 107 501 64 448 64L192 64zM160 240C160 222.3 174.3 208 192 208L296 208L296 320L192 320C174.3 320 160 305.7 160 288L160 240zM344 320L344 208L448 208C465.7 208 480 222.3 480 240L480 288C480 305.7 465.7 320 448 320L344 320zM192 384C209.7 384 224 398.3 224 416C224 433.7 209.7 448 192 448C174.3 448 160 433.7 160 416C160 398.3 174.3 384 192 384zM448 384C465.7 384 480 398.3 480 416C480 433.7 465.7 448 448 448C430.3 448 416 433.7 416 416C416 398.3 430.3 384 448 384zM248 136C248 122.7 258.7 112 272 112L368 112C381.3 112 392 122.7 392 136C392 149.3 381.3 160 368 160L272 160C258.7 160 248 149.3 248 136z"/></svg>`
const tramSVG = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 640" width="16" height="16" fill="currentColor"><path d="M128 72C128 58.7 138.7 48 152 48L488 48C501.3 48 512 58.7 512 72L512 104C512 117.3 501.3 128 488 128C474.7 128 464 117.3 464 104L464 96L344 96L344 160L384 160C437 160 480 203 480 256L480 416C480 447.2 465.1 475 442 492.5L506.3 568.5C514.9 578.6 513.6 593.8 503.5 602.3C493.4 610.8 478.2 609.6 469.7 599.5L395.1 511.4C391.5 511.8 387.8 512 384 512L256 512C252.2 512 248.5 511.8 244.9 511.4L170.3 599.5C161.7 609.6 146.6 610.9 136.5 602.3C126.4 593.7 125.1 578.6 133.7 568.5L198 492.5C174.9 475 160 447.2 160 416L160 256C160 203 203 160 256 160L296 160L296 96L176 96L176 104C176 117.3 165.3 128 152 128C138.7 128 128 117.3 128 104L128 72zM256 224C238.3 224 224 238.3 224 256L224 288C224 305.7 238.3 320 256 320L384 320C401.7 320 416 305.7 416 288L416 256C416 238.3 401.7 224 384 224L256 224zM288 416C288 398.3 273.7 384 256 384C238.3 384 224 398.3 224 416C224 433.7 238.3 448 256 448C273.7 448 288 433.7 288 416zM384 448C401.7 448 416 433.7 416 416C416 398.3 401.7 384 384 384C366.3 384 352 398.3 352 416C352 433.7 366.3 448 384 448z"/></svg>`

// --- 5. HELPERS ------------------------------------------
// Create a quick lookup to get color by line name (e.g. "C1" -> "#FF0000")
const lineColors = computed(() => {
  const colors: Record<string, string> = {}
  Object.values(shapesData).forEach((shape) => {
    colors[shape.route_name] = shape.color
  })
  return colors
})

// --- 6. COMPUTED PROPERTIES (Data Filtering) -------------
// Step 1: Filter by Bus vs Tram toggle
const stopsFilteredByType = computed(() => {
  return stopsData.filter((stop) => {
    const meta = stopsMeta[stop.stop_id]
    const isTram = meta ? meta.is_tram : false

    if (isTram && !showTram.value) return false
    if (!isTram && !showBus.value) return false
    return true
  })
})

// Step 2: Filter by Specific Selected Lines (if any are selected)
const visibleStops = computed(() => {
  if (!showStops.value) return []
  const baseStops = stopsFilteredByType.value
  if (selectedLines.value.size === 0) return baseStops // Show all if none selected

  return baseStops.filter((stop) => {
    const meta = stopsMeta[stop.stop_id]
    if (!meta) return false
    // Check if this stop belongs to at least one selected line
    return meta.lines.some((line) => selectedLines.value.has(line))
  })
})

// Step 3: Calculate Statistics for Sidebar (Counts & List of Lines)
const stats = computed(() => {
  let busCount = 0
  let tramCount = 0
  const linesMap = new Map<string, number>()

  // Aggregate counts from the filtered data
  stopsFilteredByType.value.forEach((stop) => {
    const meta = stopsMeta[stop.stop_id]
    if (meta) {
      if (meta.is_tram) tramCount++
      else busCount++

      meta.lines.forEach((line) => {
        linesMap.set(line, (linesMap.get(line) || 0) + 1)
      })
    }
  })

  // Convert Map to Array and Sort by popularity
  let sortedLines = Array.from(linesMap.entries())
    .sort((a, b) => b[1] - a[1])
    .map(([name, count]) => ({ name, count }))

  // Apply Search Filter
  if (searchQuery.value.trim() !== '') {
    const query = searchQuery.value.toLowerCase()
    sortedLines = sortedLines.filter((l) => l.name.toLowerCase().includes(query))
  }

  // Limit initial display to 6 lines unless "Show All" is clicked
  const displayedLines =
    showAllLines.value || searchQuery.value !== '' ? sortedLines : sortedLines.slice(0, 6)

  return { busCount, tramCount, displayedLines, totalLines: sortedLines.length }
})

// --- 7. MAP INITIALIZATION -------------------------------
onMounted(() => {
  if (mapContainer.value) {
    initMap()
    drawShapes()
    drawStops()
  }
})

function initMap() {
  if (!mapContainer.value) return

  // Initialize Leaflet Map centered on Nantes
  map.value = L.map(mapContainer.value, { zoomControl: false }).setView([47.2184, -1.5536], 13)

  // Move zoom control to bottom right for better UX
  L.control.zoom({ position: 'bottomright' }).addTo(map.value)

  // Load the visual map tiles (CartoDB Light)
  L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
    attribution: 'CartoDB',
    subdomains: 'abcd',
    maxZoom: 20,
  }).addTo(map.value)

  // Initialize Layers
  shapesLayer.value = L.layerGroup().addTo(map.value) // Standard Layer for Lines

  // MarkerClusterGroup is special: it groups nearby pins into a single "Cluster" bubble
  // @ts-ignore
  markersLayer.value = L.markerClusterGroup({
    maxClusterRadius: 40,
    // Custom function to style the cluster bubble (Blue Circle)
    iconCreateFunction: function (cluster: any) {
      const count = cluster.getChildCount()
      return L.divIcon({
        html: `<div class="blue-cluster"><span>${count}</span></div>`,
        className: 'custom-cluster-icon',
        iconSize: [40, 40],
      })
    },
  })
  if (markersLayer.value && map.value) map.value.addLayer(markersLayer.value)
}

// --- 8. DRAWING FUNCTIONS --------------------------------
/**
 * Draws the colored poly-lines (routes) on the map.
 */
function drawShapes() {
  if (!shapesLayer.value) return
  shapesLayer.value.clearLayers() // Clear old lines before redrawing

  Object.values(shapesData).forEach((shape: Shape) => {
    // Skip drawing if the category (Bus/Tram) is hidden
    if (!showBus.value && !showTram.value) return
    // Skip if user has selected specific lines, but this shape isn't one of them
    if (selectedLines.value.size > 0 && !selectedLines.value.has(shape.route_name)) return

    L.polyline(shape.points as L.LatLngExpression[], {
      color: shape.color,
      weight: 3,
      opacity: 0.6,
    })
      .bindPopup(`Ligne ${shape.route_name}`)
      .addTo(shapesLayer.value!)
  })
}

/**
 * Draws the stop markers on the map.
 */
function drawStops() {
  if (!markersLayer.value) return
  markersLayer.value.clearLayers() // Clear old markers

  const colors = lineColors.value // Cache colors for performance

  visibleStops.value.forEach((stop) => {
    const meta = stopsMeta[stop.stop_id]
    const isTram = meta?.is_tram

    // A. Generate Line Badges (e.g. [C1] [23])
    let linesHtml = ''
    if (meta && meta.lines) {
      meta.lines.forEach((line) => {
        const lineColor = colors[line] || '#3b82f6'
        linesHtml += `<span class="popup-line-badge" style="background-color: ${lineColor};">${line}</span>`
      })
    }

    // B. Accessibility Status
    const isAccessible = true // Placeholder (Data source usually provides this)
    const accessText = isAccessible ? 'Oui' : 'Non'
    const handicapRow = `<div class="popup-row"><strong>Accès handicapé :</strong> ${accessText}</div>`

    // C. Bonus: Specific Images for Key Stations
    let imageHtml = ''
    // Image for Commerce
    if (stop.stop_name === 'Commerce') {
      imageHtml = `<div class="popup-image"><img src="https://i.imgur.com/4GEU71O.png" alt="Commerce Station" style="width:100%; height:auto; border-radius:4px; margin-top:10px;"></div>`
    }
    // Image for Vincent Gâche
    if (stop.stop_name === 'Vincent Gâche') {
      imageHtml = `<div class="popup-image"><img src="https://i.imgur.com/WbbaBkg.jpeg" alt="Vincent Gâche" style="width:100%; height:auto; border-radius:4px; margin-top:10px;"></div>`
    }

    // D. Construct Popup HTML
    const popupContent = `
      <div class="custom-popup">
        <div class="popup-header">Nom du station : <strong>${stop.stop_name}</strong></div>
        <div class="popup-body">
          <div class="popup-row">
            <strong>Lignes :</strong>
            <div class="badges-container">${linesHtml}</div>
          </div>
          ${handicapRow}
          ${imageHtml}
        </div>
      </div>
    `

    // E. Create Custom Icon (Bus or Tram)
    const iconSvg = isTram ? tramSVG : busSVG
    const colorClass = isTram ? 'icon-tram' : 'icon-bus'
    const customIcon = L.divIcon({
      html: `<div class="stop-icon-wrapper ${colorClass}">${iconSvg}</div>`,
      className: 'custom-fa-icon',
      iconSize: [28, 28],
      iconAnchor: [14, 14],
    })

    // F. Add Marker to Cluster Layer
    L.marker([stop.stop_lat, stop.stop_lon], { icon: customIcon })
      .bindPopup(popupContent)
      .addTo(markersLayer.value!)
  })
}

// --- 9. USER ACTIONS -------------------------------------
const toggleStopsMaster = () => {
  showStops.value = !showStops.value
  drawStops()
}

const toggleType = (type: 'bus' | 'tram') => {
  if (type === 'bus') showBus.value = !showBus.value
  if (type === 'tram') showTram.value = !showTram.value
  redraw()
}

const toggleLine = (lineName: string) => {
  if (selectedLines.value.has(lineName)) {
    selectedLines.value.delete(lineName)
  } else {
    selectedLines.value.add(lineName)
  }
  redraw()
}

const clearLineSelection = () => {
  selectedLines.value.clear()
  redraw()
}

const redraw = () => {
  drawStops()
  drawShapes()
}
</script>

<template>
  <div class="dashboard">
    <!-- SIDEBAR (Left Panel) -->
    <div class="sidebar">
      <!-- Header & Master Toggle -->
      <div class="header" @click="toggleStopsMaster">
        <div class="toggle-switch" :class="{ active: showStops }">
          <div class="toggle-circle"></div>
        </div>
        <h1>Afficher les stations</h1>
      </div>

      <!-- Section 1: Transport Type Filters -->
      <div class="section">
        <h3>Type de ligne</h3>
        <div class="stat-row" @click="toggleType('bus')">
          <div class="checkbox" :class="{ active: showBus }"></div>
          <span class="label-with-icon">
            <span class="sidebar-icon bus" v-html="busSVG"></span>
            Bus
          </span>
          <span class="count">{{ stats.busCount }}</span>
        </div>
        <div class="stat-row" @click="toggleType('tram')">
          <div class="checkbox" :class="{ active: showTram }"></div>
          <span class="label-with-icon">
            <span class="sidebar-icon tram" v-html="tramSVG"></span>
            Tram
          </span>
          <span class="count">{{ stats.tramCount }}</span>
        </div>
      </div>

      <!-- Section 2: Specific Line Selection -->
      <div class="section">
        <h3>Sélectionner Lignes</h3>
        <span v-if="selectedLines.size > 0" @click.stop="clearLineSelection" class="clear-btn">
          Effacer
        </span>

        <div class="search-box">
          <span class="search-icon">🔍</span>
          <input type="text" v-model="searchQuery" placeholder="Rechercher une ligne..." />
        </div>

        <div class="lines-list">
          <div
            class="stat-row line-row"
            v-for="line in stats.displayedLines"
            :key="line.name"
            @click="toggleLine(line.name)"
            :class="{ selected: selectedLines.has(line.name) }"
          >
            <div class="line-row-left">
              <div class="checkbox" :class="{ active: selectedLines.has(line.name) }"></div>
              <span class="line-badge">{{ line.name }}</span>
            </div>
            <span class="count">{{ line.count }}</span>
          </div>
        </div>

        <!-- "Show More" Link -->
        <div
          v-if="searchQuery === '' && stats.totalLines > 6"
          class="more-link"
          @click="showAllLines = !showAllLines"
        >
          {{ showAllLines ? '- Reduire' : `+ Afficher (${stats.totalLines - 6} autres)` }}
        </div>
      </div>
    </div>

    <!-- MAP CONTAINER (Right Panel) -->
    <div class="map-wrapper">
      <div ref="mapContainer" class="map-container"></div>
    </div>
  </div>
</template>

<style>
/* --- GLOBAL STYLES --- */
* {
  box-sizing: border-box;
}
body,
html {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
  overflow: hidden;
  font-family: 'Segoe UI', Roboto, Arial, sans-serif;
  background: #1a1a1a;
}

/* --- LAYOUT --- */
.dashboard {
  display: grid;
  grid-template-columns: 350px 1fr; /* Sidebar fixed width, Map takes rest */
  height: 100vh;
  width: 100vw;
}

/* --- MAP WRAPPER --- */
.map-wrapper {
  position: relative;
  height: 100%;
  width: 100%;
}
.map-container {
  height: 100%;
  width: 100%;
  background: #e5e5e5;
}

/* --- SIDEBAR STYLING --- */
.sidebar {
  background: #1e1e1e;
  color: #f0f0f0;
  padding: 25px;
  box-shadow: 5px 0 15px rgba(0, 0, 0, 0.3);
  overflow-y: auto;
  z-index: 10;
  border-right: 1px solid #333;
  display: flex;
  flex-direction: column;
}
.label-with-icon {
  display: flex;
  align-items: center;
}

/* Headers & Text */
h1 {
  font-size: 20px;
  margin: 0;
  color: #ffffff;
  font-weight: 700;
}
h3 {
  font-size: 14px;
  margin-top: 15px;
  margin-bottom: 12px;
  border-bottom: 1px solid #333;
  padding-bottom: 8px;
  text-transform: uppercase;
  color: #aaa;
  font-weight: 600;
  letter-spacing: 1px;
}

/* Search Input */
.search-box {
  position: relative;
  margin-bottom: 15px;
}
.search-box input {
  width: 100%;
  padding: 10px 10px 10px 35px;
  border: 1px solid #444;
  background-color: #2a2a2a;
  color: #fff;
  border-radius: 6px;
  font-size: 14px;
}
.search-box input:focus {
  outline: none;
  border-color: #3b82f6;
  background-color: #333;
}
.search-icon {
  position: absolute;
  left: 12px;
  top: 10px;
  color: #888;
  font-size: 14px;
}

/* List Rows */
.stat-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 8px;
  cursor: pointer;
  border-bottom: 1px solid #2a2a2a;
  transition: background 0.2s;
  border-radius: 6px;
}
.stat-row:hover {
  background: #2a2a2a;
}
.line-row {
  padding-left: 10px;
}
.line-row.selected {
  background-color: #1a365d;
  border: 1px solid #2c5282;
}
.line-row-left {
  display: flex;
  align-items: center;
}

.more-link {
  margin-top: 10px;
  color: #3b82f6;
  cursor: pointer;
  font-size: 13px;
  font-weight: 600;
  display: block;
  text-align: center;
  padding: 10px;
  background: #2a2a2a; /* Dark background */
  border: 1px solid #333; /* Subtle border */
  border-radius: 6px;
  transition: all 0.2s ease;
}

.more-link:hover {
  background: #333; /* Lighter on hover */
  border-color: #444;
  color: #60a5fa; /* Brighter blue on hover */
}

/* UI Components (Badges, Checkboxes) */
.line-badge {
  font-weight: bold;
  color: #e0e0e0;
  font-size: 15px;
}
.count {
  color: #666;
  font-size: 14px;
  font-weight: 500;
}
.checkbox {
  width: 18px;
  height: 18px;
  border: 2px solid #555;
  background: #2a2a2a;
  border-radius: 4px;
  margin-right: 12px;
}
.checkbox.active {
  background: #3b82f6;
  border-color: #3b82f6;
}
.clear-btn {
  font-size: 11px;
  color: #ef4444;
  cursor: pointer;
  font-weight: bold;
  text-transform: uppercase;
}
.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

/* Toggle Switch */
.header {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
  cursor: pointer;
}
.toggle-switch {
  width: 44px;
  height: 24px;
  background-color: #444;
  border-radius: 25px;
  position: relative;
  transition: background-color 0.3s;
  margin-right: 15px;
  flex-shrink: 0;
}
.toggle-switch.active {
  background-color: #3b82f6;
}
.toggle-circle {
  width: 20px;
  height: 20px;
  background-color: white;
  border-radius: 50%;
  position: absolute;
  top: 2px;
  left: 2px;
  transition: transform 0.3s;
}
.toggle-switch.active .toggle-circle {
  transform: translateX(20px);
}

/* --- MAP ICONS STYLING --- */
.stop-icon-wrapper {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: white;
  border: 2px solid white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}
.stop-icon-wrapper.icon-bus {
  color: #0735ed;
}
.stop-icon-wrapper.icon-tram {
  color: #009933;
}

.blue-cluster {
  background-color: rgba(59, 130, 246, 0.9);
  border: 3px solid white;
  color: white;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 14px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
}
.sidebar-icon {
  width: 20px;
  height: 20px;
  margin-right: 8px;
  display: flex;
  align-items: center;
}
.sidebar-icon.bus {
  color: #3b82f6;
}
.sidebar-icon.tram {
  color: #22c55e;
}

/* --- POPUP STYLING --- */
.leaflet-popup-content-wrapper {
  border-radius: 8px;
  padding: 0;
  overflow: hidden;
  box-shadow: 0 3px 14px rgba(0, 0, 0, 0.2);
}
.leaflet-popup-content {
  margin: 0 !important;
  width: 280px !important;
}

.custom-popup {
  font-family: 'Segoe UI', sans-serif;
  font-size: 13px;
  color: #333;
}
.popup-header {
  background: #f8f9fa;
  padding: 12px 15px;
  font-size: 14px;
  border-bottom: 1px solid #eee;
}
.popup-body {
  padding: 12px 15px;
}
.popup-row {
  margin-bottom: 10px;
}
.popup-line-badge {
  display: inline-block;
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: bold;
  font-size: 12px;
  color: white;
  min-width: 24px;
  text-align: center;
}
.badges-container {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
  margin-top: 6px;
}
</style>
