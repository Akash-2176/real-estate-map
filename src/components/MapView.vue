<!-- src/components/MapView.vue -->
<template>
  <div id="map" style="height: 100vh; width: 100vw;"></div>
</template>

<script setup>
import { onMounted } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

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
  const map = L.map('map').setView([12.9616, 77.5946], 14)

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
    }
  })
})
</script>

<style scoped>
/* Fixes Leaflet icons not showing properly */
@import "leaflet/dist/leaflet.css";
</style>
