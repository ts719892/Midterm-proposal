<script setup>
import { ref, onMounted, watch, nextTick } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  events: { type: Array, required: true },
  selectedMonth: { type: Number, default: null },
  selectedDate: { type: String, default: '' },
})
const emit = defineEmits(['select-date'])
const canvas = ref(null)
const dims = { left: 50, top: 36, cellW: 27.5, cellH: 24, rowGap: 7 }
const W = 965, H = 425
function draw() {
  if (!canvas.value) return
  const count = d3.rollup(props.events.filter(e=>e.date), v=>v.length, d=>d.date)
  const data = []
  for (let month=1; month<=12; month++) {
    const days = new Date(2025,month,0).getDate()
    for (let day=1;day<=31;day++) {
      const date=`2025-${String(month).padStart(2,'0')}-${String(day).padStart(2,'0')}`
      data.push({month,day,date,valid:day<=days,total:count.get(date)||0})
    }
  }
  const max = Math.max(1,d3.max(data,d=>d.total)||1)
  const color = d3.scaleSequential([0,max],d3.interpolateRgbBasis(['#1a3444','#207f8a','#34d3c6']))
  const svg = d3.select(canvas.value).attr('viewBox',`0 0 ${W} ${H}`)
  svg.selectAll('*').remove()
  svg.append('g').selectAll('text.month').data(d3.range(1,13)).join('text')
    .attr('x',8).attr('y',m=>dims.top+(m-1)*(dims.cellH+dims.rowGap)+16)
    .attr('fill','#c0d4df').attr('font-size',12).text(m=>`${m}月`)
  svg.append('g').selectAll('text.tick').data([1,5,10,15,20,25,31]).join('text')
    .attr('x',d=>dims.left+(d-1)*dims.cellW+9).attr('y',20).attr('fill','#859db0')
    .attr('text-anchor','middle').attr('font-size',11).text(d=>d)
  svg.append('g').selectAll('rect.day').data(data).join('rect')
    .attr('class','day')
    .attr('x',d=>dims.left+(d.day-1)*dims.cellW)
    .attr('y',d=>dims.top+(d.month-1)*(dims.cellH+dims.rowGap))
    .attr('width',dims.cellW-3).attr('height',dims.cellH)
    .attr('rx',4)
    .attr('fill',d=>!d.valid?'transparent':d.total===0?'#1d3545':color(d.total))
    .attr('opacity',d=>(props.selectedMonth && props.selectedMonth!==d.month) ? 0.26 : 1)
    .attr('stroke',d=>props.selectedDate===d.date?'#fff':'transparent')
    .attr('stroke-width',d=>props.selectedDate===d.date?2.5:0)
    .style('cursor',d=>d.valid?'pointer':'default')
    .attr('role',d=>d.valid?'button':null)
    .attr('tabindex',d=>d.valid?'0':null)
    .attr('aria-label',d=>d.valid?`${d.date} ${d.total} 筆事件`:null)
    .on('click',(_,d)=>{if(d.valid)emit('select-date',props.selectedDate===d.date?'':d.date)})
    .on('keydown',(event,d)=>{if(d.valid&&(event.key==='Enter'||event.key===' ')){event.preventDefault();emit('select-date',props.selectedDate===d.date?'':d.date)}})
    .append('title').text(d=>d.valid?`${d.date}：${d.total} 筆逐筆事件`:'')
  svg.append('text').attr('x',dims.left).attr('y',H-1)
    .attr('fill','#7fa0ad').attr('font-size',10)
    .text('每日件數來自可辨識日期的逐筆事件；與官方月份標題統計可能有差異')
}
onMounted(draw)
watch(()=>[props.events,props.selectedMonth,props.selectedDate],async()=>{await nextTick();draw()})
</script>
<template><svg ref="canvas" class="calendar-svg" aria-label="2025 年每日臺鐵事故事件熱圖，可點擊日期篩選"></svg></template>
