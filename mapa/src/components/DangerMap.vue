<template>
    <div class="map-container">
      <l-map
        ref="map"
        :zoom="zoom"
        :center="center"
        @ready="onMapReady"
      >
        <l-tile-layer
          url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
          layer-type="base"
          name="OpenStreetMap"
        />
  
        <l-geo-json
          v-if="boundaryData"
          :geojson="boundaryData"
          :options-style="boundaryStyle"
        />
  
        <!-- Tymczasowy marker podglądu -->
        <l-marker
          v-if="tempMarkerPosition"
          :lat-lng="tempMarkerPosition"
          :icon="previewIcon"
        />
  
        <l-marker
          v-for="(danger, index) in filteredDangers"
          :key="'danger-' + index"
          :lat-lng="[danger.lat, danger.lng]"
        >
          <l-icon :icon-url="getIcon(danger.category)" />
          <l-popup>{{ danger.description }}</l-popup>
        </l-marker>
  
        <l-control position="topright">
          <div class="map-alert" v-if="alertMessage">
            {{ alertMessage }}
          </div>
        </l-control>
      </l-map>
  
      <div class="controls">
        <select v-model="selectedCategory" class="category-select">
          <option value="all">Wszystkie kategorie</option>
          <option v-for="category in categories" :key="category" :value="category">
            {{ category }}
          </option>
        </select>
  
        <button @click="initReport" class="report-button">Zgłoś zagrożenie</button>
      </div>
  
      <div v-if="showReportForm" class="report-form">
        <h3>Nowe zgłoszenie</h3>
        <div class="form-group">
          <label>Opis:</label>
          <textarea v-model="newDanger.description" rows="3"></textarea>
        </div>
        
        <div class="form-group">
          <label>Kategoria:</label>
          <select v-model="newDanger.category">
            <option v-for="category in categories" :key="category" :value="category">
              {{ category }}
            </option>
          </select>
        </div>
  
        <div class="form-actions">
          <button @click="confirmDanger" class="confirm-button">Potwierdź</button>
          <button @click="cancelReport" class="cancel-button">Anuluj</button>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, computed,  } from 'vue'
  import { LMap, LTileLayer, LMarker, LPopup, LIcon, LGeoJson, LControl } from '@vue-leaflet/vue-leaflet'
  import 'leaflet/dist/leaflet.css'
  import axios from 'axios'
  import { booleanPointInPolygon } from '@turf/turf'
  import L from 'leaflet'
  
  // Konfiguracja mapy
  const center = ref([54.1557, 19.4048])
  const zoom = ref(13)
  const map = ref(null)
  const boundaryData = ref(null)
  const loading = ref(true)
  const alertMessage = ref('')
  
  // Dane
  const dangers = ref([])
  const categories = ["droga", "oświetlenie", "zieleń", "inne"]
  const selectedCategory = ref("all")
  
  // Formularz zgłoszeń
  const showReportForm = ref(false)
  const tempMarkerPosition = ref(null)
  const newDanger = ref({
    lat: 0,
    lng: 0,
    description: "",
    category: "droga"
  })
  
  // Style
  const boundaryStyle = {
    color: "#FF0000",
    weight: 3,
    opacity: 0.8,
    fillColor: "#FF0000",
    fillOpacity: 0.1
  }
  
  const previewIcon = L.icon({
    iconUrl: 'https://cdn-icons-png.flaticon.com/512/149/149059.png',
    iconSize: [32, 32],
    iconAnchor: [16, 32],
    className: 'preview-marker'
  })
  
  // Pobierz granice miasta
  const loadBoundaries = async () => {
    try {
      const response = await axios.get(
        'https://overpass-api.de/api/interpreter?data=[out:json][timeout:25];' +
        'rel(1623145);out geom;'
      )
      processBoundaryData(response.data)
    } catch (error) {
      showAlert('Błąd ładowania granic miasta', 'error')
    } finally {
      loading.value = false
    }
  }
  
  const processBoundaryData = (osmData) => {
  const geojson = {
    type: "FeatureCollection",
    features: []
  }

  osmData.elements.forEach(element => {
    if (element.type === "relation") {
      element.members.forEach(member => {
        if (member.type === "way" && member.role === "outer") {
          let coordinates = member.geometry.map(point => [point.lon, point.lat])
          
          // Napraw problem z niezamkniętym poligonem
          const firstCoord = coordinates[0]
          const lastCoord = coordinates[coordinates.length - 1]
          if (firstCoord[0] !== lastCoord[0] || firstCoord[1] !== lastCoord[1]) {
            coordinates.push([...firstCoord])
          }

          geojson.features.push({
            type: "Feature",
            properties: {},
            geometry: {
              type: "Polygon",
              coordinates: [coordinates]
            }
          })
        }
      })
    }
  })

  boundaryData.value = geojson
}
  
  // Obsługa mapy
  const onMapReady = async () => {
    await loadBoundaries()
    const mapInstance = map.value.leafletObject
  
    mapInstance.on('click', async (e) => {
      if (!showReportForm.value) return
  
      try {
        const isValid = await checkLocationValidity(e.latlng)
        
        if (isValid) {
          tempMarkerPosition.value = [e.latlng.lat, e.latlng.lng]
          newDanger.value.lat = e.latlng.lat
          newDanger.value.lng = e.latlng.lng
        } else {
          showAlert('Lokalizacja poza granicami Elbląga!', 'warning')
          tempMarkerPosition.value = null
        }
      } catch (error) {
        showAlert('Błąd weryfikacji lokalizacji', 'error')
      }
    })
  }
  
  const checkLocationValidity = async (latlng) => {
    if (!boundaryData.value) return false
  
    try {
      const point = {
        type: "Feature",
        geometry: {
          type: "Point",
          coordinates: [latlng.lng, latlng.lat]
        }
      }
  
      return boundaryData.value.features.some(feature => 
        booleanPointInPolygon(point, feature))
    } catch (error) {
      console.error('Błąd sprawdzania lokalizacji:', error)
      return false
    }
  }
  
  // Metody zgłoszeń
  const initReport = () => {
    showReportForm.value = true
    tempMarkerPosition.value = null
  }
  
  const confirmDanger = () => {
    if (!newDanger.value.description.trim()) {
      showAlert('Wprowadź opis zgłoszenia!', 'warning')
      return
    }
  
    dangers.value.push({ ...newDanger.value })
    cancelReport()
    showAlert('Zgłoszenie dodane pomyślnie!', 'success')
  }
  
  const cancelReport = () => {
    showReportForm.value = false
    tempMarkerPosition.value = null
    newDanger.value = {
      lat: 0,
      lng: 0,
      description: "",
      category: "droga"
    }
  }
  
  const showAlert = (message, type = 'info') => {
    alertMessage.value = message
    setTimeout(() => alertMessage.value = '', 3000)
  }
  
  const getIcon = (category) => {
    const icons = {
      droga: 'https://cdn-icons-png.flaticon.com/512/4476/4476951.png',
      oświetlenie: 'https://cdn-icons-png.flaticon.com/512/6976/6976906.png',
      zieleń: 'https://cdn-icons-png.flaticon.com/512/2052/2052839.png',
      inne: 'https://cdn-icons-png.flaticon.com/512/681/681138.png'
    }
    return icons[category] || icons.inne
  }
  
  const filteredDangers = computed(() => {
    return selectedCategory.value === 'all' 
      ? dangers.value 
      : dangers.value.filter(d => d.category === selectedCategory.value)
  })
  </script>
  
  <style>
  .map-container {
    height: 100vh;
    width: 100%;
    position: relative;
  }
  
  .controls {
    position: absolute;
    top: 20px;
    left: 20px;
    z-index: 1000;
    background: rgba(255, 255, 255, 0.95);
    padding: 15px;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    display: flex;
    gap: 10px;
  }
  
  .report-form {
    position: absolute;
    top: 80px;
    left: 20px;
    z-index: 1000;
    background: white;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 15px rgba(0,0,0,0.2);
    width: 300px;
  }
  
  .form-group {
    margin-bottom: 15px;
  }
  
  .form-group label {
    display: block;
    margin-bottom: 5px;
    font-weight: 500;
  }
  
  .form-group textarea {
    width: 100%;
    padding: 8px;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
  
  .form-actions {
    display: flex;
    gap: 10px;
    margin-top: 15px;
  }
  
  .preview-marker {
    filter: hue-rotate(90deg);
    opacity: 0.8 !important;
  }
  
  .map-alert {
    background: #fff;
    padding: 10px 15px;
    border-radius: 5px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.2);
    border-left: 4px solid;
  }
  
  .map-alert[data-type="error"] { border-color: #ff4444; }
  .map-alert[data-type="warning"] { border-color: #ffbb33; }
  .map-alert[data-type="success"] { border-color: #00C851; }
  
  .report-button {
    background: #2196F3;
    color: white;
    border: none;
    padding: 8px 15px;
    border-radius: 4px;
    cursor: pointer;
  }
  
  .category-select {
    padding: 6px 10px;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
  </style>