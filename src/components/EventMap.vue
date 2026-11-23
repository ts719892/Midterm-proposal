<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue'
import L from 'leaflet'
const props = defineProps({
  events: { type:Array, required:true },
  selectedKey: { type:String, default:'' }
})
const emit = defineEmits(['select-point'])
const el = ref(null)
let map = null, layer=null
const bounds=L.latLngBounds([21.85,119.40],[25.4,122.20])
function keyOf(e) { return e.station_code ? `S:${e.station_code}` : e.section_from&&e.section_to?`R:${e.section_from}-${e.section_to}`:'' }
function safe(v) { return String(v??'').replace(/[&<>"']/g, c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#039;'}[c])) }
function draw(){
  if(!map)return
  if(layer) layer.remove()
  layer=L.layerGroup().addTo(map)
  const groups = new Map()
  for(const e of props.events){
    if(e.display_lat==null||e.display_lon==null) continue
    const key=keyOf(e)
    if(!key)continue
    if(!groups.has(key))groups.set(key,{key,lat:e.display_lat,lon:e.display_lon,label:e.location_raw,location_type:e.location_type,count:0})
    groups.get(key).count++
  }
  for(const pt of groups.values()){
    const selected=props.selectedKey===pt.key
    const section=pt.location_type==='section'
    const marker=L.circleMarker([pt.lat,pt.lon],{
      radius:Math.min(19,5+Math.sqrt(pt.count)*2.0)+(selected?3:0),
      color:selected?'#ffffff':section?'#ffcb75':'#54e0d2',
      fillColor:section?'#c97f30':'#1eb2aa',fillOpacity:selected?.94:.67,
      weight:selected?3:1.5, dashArray:section?'4 4':null
    })
    marker.bindTooltip(`<b>${safe(pt.label)}</b><br>${pt.count} 筆${section?'（區間中點示意）':'（車站點位）'}`)
    marker.on('click',()=>emit('select-point',props.selectedKey===pt.key?'':pt.key))
    marker.addTo(layer)
  }
}
onMounted(()=>{
  map=L.map(el.value,{zoomControl:true,scrollWheelZoom:false})
  map.fitBounds(bounds)
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png',{attribution:'&copy; OpenStreetMap contributors',maxZoom:18}).addTo(map)
  draw()
})
watch(()=>[props.events,props.selectedKey],()=>{draw();if(props.selectedKey){const e=props.events.find(x=>keyOf(x)===props.selectedKey);if(e?.display_lat!=null)map?.flyTo([e.display_lat,e.display_lon],Math.max(map.getZoom(),8),{duration:.4})}})
onBeforeUnmount(()=>{if(map){map.remove();map=null}})
</script>
<template><div ref="el" class="event-map" aria-label="逐筆事故地理分布地圖，橘色虛線圓圈為站間中點示意"></div></template>
