<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue'
import L from 'leaflet'

const props = defineProps({
  stations: { type: Array, required: true },
  activeCode: { type: String, default: '' }
})
const emit = defineEmits(['choose-station'])
const mapEl = ref(null)
let map = null
let markers = null
const allBounds = L.latLngBounds([21.85,119.40],[25.40,122.20])

function paintMarkers() {
  if (!map) return
  if (markers) markers.remove()
  markers = L.layerGroup().addTo(map)
  for (const station of props.stations) {
    const active = station.code === props.activeCode
    const marker = L.circleMarker([station.lat, station.lon], {
      radius: active ? 8 : 4,
      color: active ? '#ffce71' : '#7fe2dd',
      weight: active ? 2.8 : 1.2,
      fillColor: active ? '#ffc24e' : '#48d0c5',
      fillOpacity: active ? 1 : 0.7,
    })
    marker.bindTooltip(`${station.name}站`, {direction: 'top', offset: [0,-8]})
    marker.on('click', () => emit('choose-station', station.code))
    marker.addTo(markers)
  }
}

onMounted(() => {
  map = L.map(mapEl.value, { zoomControl: true, scrollWheelZoom: false })
  map.fitBounds(allBounds)
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors',
    maxZoom: 18,
  }).addTo(map)
  paintMarkers()
})

watch(() => [props.stations, props.activeCode], () => {
  paintMarkers()
  if (map && props.activeCode) {
    const selected = props.stations.find(s => s.code === props.activeCode)
    if (selected) map.flyTo([selected.lat, selected.lon], Math.max(map.getZoom(), 10), {duration:0.5})
  }
})

onBeforeUnmount(() => { if (map) { map.remove(); map = null } })
</script>

<template>
  <div ref="mapEl" class="station-map" aria-label="臺灣鐵路車站點位互動地圖"></div>
</template>
