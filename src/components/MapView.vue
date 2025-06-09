<template>
  <div>
    <div class="toolbar">
      <button @click="setDrawMode('none')" :disabled="isDrawing" :class="{active: drawMode === 'none'}">None</button>
      <button @click="startPolygonMode" :disabled="isDrawing" :class="{active: drawMode === 'polygon'}">Polygon</button>
      <button @click="setDrawMode('circle')" :disabled="isDrawing" :class="{active: drawMode === 'circle'}">Circle</button>
      <button @click="setDrawMode('rectangle')" :disabled="isDrawing" :class="{active: drawMode === 'rectangle'}">Rectangle</button>
      <button @click="resetMap" :disabled="isDrawing">Reset</button>
    </div>
    <div id="map" style="height: 100vh; width: 100vw;"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import 'leaflet/dist/leaflet.css'
import L from 'leaflet'
import markerIcon2x from '@/assets/leaflet/marker-icon-2x.png'
import markerIcon from '@/assets/leaflet/marker-icon.png'
import markerShadow from '@/assets/leaflet/marker-shadow.png'


delete L.Icon.Default.prototype._getIconUrl
L.Icon.Default.mergeOptions({
  iconRetinaUrl: markerIcon2x,
  iconUrl: markerIcon,
  shadowUrl: markerShadow,
})

let map = null
const hospitalMarkers = ref([])

const isDrawing = ref(false)
const drawMode = ref('none')

const polygonPoints = ref([])
const tempMarkers = ref([])
let polygonLayer = null
let followLine = null

let currentLayer = null
let startLatLng = null
const activeLayers = ref([])

function isPointInsidePolygon(point, polygon) {
  const x = point.lng, y = point.lat
  let inside = false
  for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
    const xi = polygon[i].lng, yi = polygon[i].lat
    const xj = polygon[j].lng, yj = polygon[j].lat
    const intersect = ((yi > y) !== (yj > y)) &&
                      (x < ((xj - xi) * (y - yi)) / (yj - yi) + xi)
    if (intersect) inside = !inside
  }
  return inside
}

function isPointInsideShape(marker, layer) {
  const latlng = marker.getLatLng()
  if (layer instanceof L.Circle) {
    return layer.getLatLng().distanceTo(latlng) <= layer.getRadius()
  }
  if (layer instanceof L.Polygon || layer instanceof L.Rectangle) {
    const latlngs = layer.getLatLngs()[0]
    return isPointInsidePolygon(latlng, latlngs)
  }
  return false
}

function filterMarkers() {
  if (activeLayers.value.length === 0) {
    hospitalMarkers.value.forEach(m => {
      m.setOpacity(1)
      m.options.interactive = true
    })
    return
  }

  hospitalMarkers.value.forEach(marker => {
    let inside = false
    for (const shape of activeLayers.value) {
      if (isPointInsideShape(marker, shape)) {
        inside = true
        break
      }
    }

    if (inside) {
      marker.setOpacity(1)
      marker.options.interactive = true
    } else {
      marker.setOpacity(0)
      marker.options.interactive = false
    }
  })
}

function setDrawMode(mode) {
  if (isDrawing.value) return
  drawMode.value = mode
  stopDrawing()
  if (mode !== 'none') startDrawing()
}

function startPolygonMode() {
  if (isDrawing.value) return
  drawMode.value = 'polygon'
  startDrawing()
}

function resetMap() {
  if (isDrawing.value) return

  if (polygonLayer) {
    map.removeLayer(polygonLayer)
    polygonLayer = null
  }

  if (currentLayer) {
    map.removeLayer(currentLayer)
    currentLayer = null
  }

  tempMarkers.value.forEach(m => {
    if (m.rect) map.removeLayer(m.rect)
    if (m.line) map.removeLayer(m.line)
  })
  tempMarkers.value = []

  if (followLine) {
    map.removeLayer(followLine)
    followLine = null
  }

  polygonPoints.value = []
  activeLayers.value = []

  hospitalMarkers.value.forEach(m => {
    m.setOpacity(1)
    m.options.interactive = true
    m.addTo(map)
  })

  drawMode.value = 'none'
}

function stopDrawing() {
  isDrawing.value = false
  map.getContainer().style.cursor = ''
  map.off('click', onMapClick)
  map.off('dblclick', onDoubleClick)
  map.off('mousedown', onMouseDown)
  map.off('mousemove', onMouseMove)
  map.off('mouseup', onMouseUp)
  map.off('mousemove', onMouseFollow)
  map.dragging.enable()

  if (followLine) {
    map.removeLayer(followLine)
    followLine = null
  }
}

function startDrawing() {
  isDrawing.value = true

  if (drawMode.value === 'polygon') {
    map.getContainer().style.cursor = 'crosshair'
    polygonPoints.value = []
    tempMarkers.value.forEach(m => {
      if (m.rect) map.removeLayer(m.rect)
      if (m.line) map.removeLayer(m.line)
    })
    tempMarkers.value = []
    followLine = null

    if (polygonLayer) {
      map.removeLayer(polygonLayer)
      polygonLayer = null
    }

    map.on('click', onMapClick)
    map.on('dblclick', onDoubleClick)
    map.on('mousemove', onMouseFollow)
  }

  else if (drawMode.value === 'circle' || drawMode.value === 'rectangle') {
    map.dragging.disable()
    map.on('mousedown', onMouseDown)
    map.on('mousemove', onMouseMove)
    map.on('mouseup', onMouseUp)
    map.getContainer().style.cursor = 'crosshair'
  }
}

function addPoint(latlng) {
  const size = getRectSize()
  const bounds = [
    [latlng.lat - size, latlng.lng - size],
    [latlng.lat + size, latlng.lng + size]
  ]

  const rect = L.rectangle(bounds, {
    color: 'blue',
    weight: 1,
    fillColor: 'white',
    fillOpacity: 1,
    interactive: true
  }).addTo(map)

  tempMarkers.value.push({ rect, latlng })
  polygonPoints.value.push(latlng)

  const pointCount = polygonPoints.value.length

  if (pointCount > 1) {
    const line = L.polyline(
      [polygonPoints.value[pointCount - 2], latlng],
      { color: 'blue', dashArray: '5,5' }
    ).addTo(map)
    tempMarkers.value.push({ line })
  }

  rect.on('click', () => {
    if (!isDrawing.value) return
    const firstLatLng = polygonPoints.value[0]
    const distToFirst = latlng.distanceTo(firstLatLng)

    if (distToFirst < 15 && polygonPoints.value.length >= 3) {
      closePolygon()
    }
  })
}

function getRectSize() {
  const zoom = map.getZoom()
  return 0.0007 / Math.pow(2, zoom - 14)
}

function onMouseFollow(e) {
  if (!isDrawing.value || drawMode.value !== 'polygon') return
  if (polygonPoints.value.length === 0) return

  const lastPoint = polygonPoints.value[polygonPoints.value.length - 1]
  const mousePoint = e.latlng

  if (followLine) {
    followLine.setLatLngs([lastPoint, mousePoint])
  } else {
    followLine = L.polyline([lastPoint, mousePoint], {
      color: 'black',
      dashArray: '4, 6',
      interactive: false
    }).addTo(map)
  }
}

function closePolygon() {
  isDrawing.value = false

  polygonLayer = L.polygon(polygonPoints.value, {
    color: 'red',
    fillOpacity: 0.3
  }).addTo(map)

  tempMarkers.value.forEach(m => {
    if (m.rect) map.removeLayer(m.rect)
    if (m.line) map.removeLayer(m.line)
  })
  tempMarkers.value = []

  activeLayers.value = [polygonLayer]

  const polygonLatLngs = polygonLayer.getLatLngs()[0]

  hospitalMarkers.value.forEach(marker => {
    const latlng = marker.getLatLng()
    const inside = isPointInsidePolygon(latlng, polygonLatLngs)

    if (!inside) {
      marker.setOpacity(0)
      marker.closePopup()
      marker.options.interactive = false
    } else {
      marker.setOpacity(1)
      marker.options.interactive = true
    }
  })

  stopDrawing()
}

function onMouseDown(e) {
  if (!isDrawing.value) return
  startLatLng = e.latlng

  if (drawMode.value === 'circle') {
    currentLayer = L.circle(startLatLng, {
      radius: 0,
      color: 'red',
      fillOpacity: 0.3
    }).addTo(map)
  } else if (drawMode.value === 'rectangle') {
    currentLayer = L.rectangle([startLatLng, startLatLng], {
      color: 'green',
      fillOpacity: 0.3
    }).addTo(map)
  }
}

function onMouseMove(e) {
  if (!isDrawing.value || !currentLayer) return

  if (drawMode.value === 'circle') {
    const radius = startLatLng.distanceTo(e.latlng)
    currentLayer.setRadius(radius)
  } else if (drawMode.value === 'rectangle') {
    const bounds = L.latLngBounds(startLatLng, e.latlng)
    currentLayer.setBounds(bounds)
  }
}

function onMouseUp() {
  if (!isDrawing.value) return

  isDrawing.value = false
  activeLayers.value = [currentLayer]
  filterMarkers()
  stopDrawing()
}

function onMapClick(e) {
  if (drawMode.value !== 'polygon' || !isDrawing.value) return
  if (polygonPoints.value.length < 10) {
    addPoint(e.latlng)
  }
}

function onDoubleClick() {
  if (drawMode.value !== 'polygon' || !isDrawing.value) return
  if (polygonPoints.value.length < 3) {
    alert('Polygon needs at least 3 points')
    return
  }
  closePolygon()
}

async function fetchHospitals() {
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

  map.on('zoomend', () => {
    tempMarkers.value.forEach(entry => {
      if (entry.latlng && entry.rect) {
        const size = getRectSize()
        const bounds = [
          [entry.latlng.lat - size, entry.latlng.lng - size],
          [entry.latlng.lat + size, entry.latlng.lng + size]
        ]
        entry.rect.setBounds(bounds)
      }
    })
  })
})
</script>

<style scoped>
.toolbar {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1000;
  background: white;
  padding: 8px;
  border-radius: 4px;
  display: flex;
  gap: 8px;
}

.toolbar button {
  padding: 6px 12px;
  background: #2c3e50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.toolbar button:disabled {
  background: #95a5a6;
  cursor: not-allowed;
}

.toolbar button.active {
  background: #3498db;
}
</style>
