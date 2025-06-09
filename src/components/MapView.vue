<!-- src/components/MapView.vue -->
<template>
    <div>
        <button @click="startPolygonMode" :disabled="isDrawing" class="polygon-btn">Create polygon</button>
        <div id="map" style="height: 100vh; width: 100vw;"></div>
    </div>
</template>

<script>
    function isPointInsidePolygon(point, polygon) {
    const x = point.lng, y = point.lat;
    let inside = false;

    for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
        const xi = polygon[i].lng, yi = polygon[i].lat;
        const xj = polygon[j].lng, yj = polygon[j].lat;

        const intersect = ((yi > y) !== (yj > y)) &&
                        (x < ((xj - xi) * (y - yi)) / (yj - yi) + xi);
        if (intersect) inside = !inside;
    }

    return inside;
    }
</script>

<script setup>
import { ref, onMounted } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

let map = null
const isDrawing = ref(false)
const polygonPoints = ref([])
const tempMarkers = ref([])
let polygonLayer = null

const hospitalMarkers = ref([])


const startPolygonMode = () => {
  isDrawing.value = true
  polygonPoints.value = []
  tempMarkers.value.forEach(m => map.removeLayer(m))
  tempMarkers.value = []

  if(polygonLayer) {
    map.removeLayer(polygonLayer)
    polygonLayer = null
  }
}

const addPoint = (latlng) => {
  const marker = L.circleMarker(latlng, { radius: 5, color: 'blue' }).addTo(map)
  tempMarkers.value.push(marker)
  polygonPoints.value.push(latlng)

  const pointCount = polygonPoints.value.length

  // Handle polygon closure
  if (pointCount === 1) {
    marker.on('click', () => {
      if (!isDrawing.value) return
      const distance = marker.getLatLng().distanceTo(polygonPoints.value[0])
      if (polygonPoints.value.length >= 3 && distance <= 150) {
        closePolygon()
      }
    })
  }

  // Max point limit
  if (polygonPoints.value.length >= 10) {
    closePolygon()
  }
}

const closePolygon = () => {
  isDrawing.value = false

  polygonLayer = L.polygon(polygonPoints.value, {
    color: 'red',
    fillOpacity: 0.3
  }).addTo(map)

  tempMarkers.value.forEach(m => map.removeLayer(m))
  tempMarkers.value = []

  const polygonLatLngs = polygonLayer.getLatLngs()[0]

  hospitalMarkers.value.forEach(marker => {
    const latlng = marker.getLatLng()
    const inside = isPointInsidePolygon(latlng, polygonLatLngs)

    if (!inside) {
      marker.setOpacity(0)
      marker.closePopup() // just in case popup is open
      marker.options.interactive = false
    } else {
      marker.setOpacity(1)
      marker.options.interactive = true
    }
  })
}




const fetchHospitals = async () => {
  const query = `
    [out:json];
    area[name="Bengaluru"]->.searchArea;
    (
      node["amenity"="hospital"](area.searchArea);
      way["amenity"="hospital"](area.searchArea);
      relation["amenity"="hospital"](area.searchArea);
    );
    out center;
  `
  const response = await fetch('https://overpass-api.de/api/interpreter', {
    method: 'POST',
    body: query,
  })

  const data = await response.json()
  return data.elements
}

onMounted(async () => {
  map = L.map('map').setView([12.9616, 77.5946], 14)

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors'
  }).addTo(map)

  const hospitals = await fetchHospitals()

  hospitals.forEach(place => {
    const lat = place.lat || place.center?.lat
    const lon = place.lon || place.center?.lon
    const name = place.tags?.name || "Unnamed Hospital"

    if (lat && lon) {
      const marker = L.marker([lat, lon]).addTo(map)
      marker.bindPopup(`<strong>${name}</strong>`)

    hospitalMarkers.value.push(marker)
    }
  })

    map.on('click', (e) => {
        if (isDrawing.value && polygonPoints.value.length < 10) {
            addPoint(e.latlng)
        }
    })

})
</script>

<style scoped>
.polygon-btn {
  position: absolute;
  z-index: 1000;
  top: 10px;
  left: 10px;
  padding: 8px 16px;
  background: #2c3e50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
.polygon-btn:disabled {
  background: #95a5a6;
  cursor: not-allowed;
}
</style>