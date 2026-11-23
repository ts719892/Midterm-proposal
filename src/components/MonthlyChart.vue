<script setup>
import { ref, onMounted, watch, nextTick } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  rows: { type: Array, required: true },
  selectedMonth: { type: Number, default: null },
})
const emit = defineEmits(['change-month'])
const svgEl = ref(null)

function draw() {
  if (!svgEl.value || !props.rows.length) return
  const W = 850, H = 304
  const m = { top: 25, right: 16, bottom: 39, left: 42 }
  const svg = d3.select(svgEl.value)
  svg.selectAll('*').remove()
  svg.attr('viewBox', `0 0 ${W} ${H}`).attr('role', 'img')
    .attr('aria-label', '2025 年臺鐵每月行車事故與異常事件堆疊長條圖。點選月份進行篩選')

  const x = d3.scaleBand().domain(props.rows.map(d => d.month))
    .range([m.left, W - m.right]).padding(0.28)
  const y = d3.scaleLinear().domain([0, 110]).range([H - m.bottom, m.top])
  const layers = d3.stack().keys(['abnormal', 'general', 'major'])(props.rows)
  const colors = { abnormal: '#3dc8c0', general: '#f4bc5e', major: '#f07879' }

  svg.append('g').attr('class', 'grid')
    .attr('transform', `translate(${m.left},0)`)
    .call(d3.axisLeft(y).ticks(5).tickSize(-(W-m.left-m.right)).tickFormat(d => d))
    .call(g => g.select('.domain').remove())
    .call(g => g.selectAll('text').attr('fill', '#90a4b8').attr('font-size', 11))
    .call(g => g.selectAll('line').attr('stroke', '#22364b').attr('stroke-dasharray', '3 5'))

  const label = svg.append('g')
  label.selectAll('text.month-label').data(props.rows).join('text')
    .attr('x', d => x(d.month) + x.bandwidth() / 2)
    .attr('y', H - 14).attr('text-anchor', 'middle')
    .attr('font-size', 12).attr('fill', '#aabccd')
    .text(d => `${d.month}月`)

  svg.append('g').selectAll('g').data(layers).join('g')
    .attr('fill', layer => colors[layer.key])
    .selectAll('rect').data(layer => layer).join('rect')
    .attr('x', d => x(d.data.month)).attr('width', x.bandwidth())
    .attr('y', d => y(d[1])).attr('height', d => Math.max(0, y(d[0])-y(d[1])))
    .attr('opacity', d => (props.selectedMonth !== null && props.selectedMonth !== d.data.month) ? 0.28 : 1)
    .attr('rx', 2).style('cursor', 'pointer')
    .on('click', (_, d) => emit('change-month', d.data.month))
    .append('title')
      .text(d => `${d.data.month}月：總計 ${d.data.total} 件
重大 ${d.data.major}／一般 ${d.data.general}／異常 ${d.data.abnormal}`)

  svg.append('g').selectAll('text.total-label').data(props.rows).join('text')
    .attr('x', d => x(d.month) + x.bandwidth() / 2)
    .attr('y', d => y(d.total) - 8)
    .attr('text-anchor','middle').attr('font-size', 12)
    .attr('font-weight', d => d.month === props.selectedMonth ? 800 : 600)
    .attr('fill', d => d.month === props.selectedMonth ? '#ffffff' : '#a8b9c9')
    .text(d => d.total)
}

onMounted(draw)
watch(() => [props.selectedMonth, props.rows], async () => { await nextTick(); draw() })
</script>

<template>
  <svg ref="svgEl" class="monthly-chart"></svg>
</template>
