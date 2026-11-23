<script setup>
import { computed, ref, watch } from 'vue'
import monthly from './data/monthly.json'
import stationData from './data/stations.json'
import events from './data/events.json'
import quality from './data/quality.json'
import CalendarHeatmap from './components/CalendarHeatmap.vue'
import EventMap from './components/EventMap.vue'
import MonthlyChart from './components/MonthlyChart.vue'
import StationMap from './components/StationMap.vue'

const activeView = ref('overview')
const selectedMonth = ref(null)
const selectedLevel = ref('all')
const selectedDate = ref('')
const incidentQuery = ref('')
const selectedPoint = ref('')
const openedEvent = ref(null)
const page = ref(1)
const pageSize = 16
const levelLabels = { major:'重大行車事故', general:'一般行車事故', abnormal:'行車異常事件' }
const locationKey = (e) => e.station_code ? `S:${e.station_code}` : e.section_from && e.section_to ? `R:${e.section_from}-${e.section_to}` : ''
const aggregateRows = computed(() => events.filter(e =>
  (selectedLevel.value === 'all' || e.event_level === selectedLevel.value) &&
  (!incidentQuery.value.trim() || [e.location_raw,e.description,e.cause_raw,e.event_id].join(' ').toLowerCase().includes(incidentQuery.value.trim().toLowerCase()))))
const incidentRows = computed(() => aggregateRows.value.filter(e =>
  (selectedMonth.value === null || e.month === selectedMonth.value) &&
  (!selectedDate.value || e.date === selectedDate.value) &&
  (!selectedPoint.value || locationKey(e)===selectedPoint.value)))
const pagedIncidents = computed(() => incidentRows.value.slice((page.value-1)*pageSize,page.value*pageSize))
const pageCount = computed(() => Math.max(1,Math.ceil(incidentRows.value.length/pageSize)))
const geoRows = computed(() => aggregateRows.value.filter(e =>
  (selectedMonth.value === null || e.month === selectedMonth.value) &&
  (!selectedDate.value || e.date === selectedDate.value)))
const mapMatched = computed(() => incidentRows.value.filter(e=>e.display_lat!==null).length)
const datedRows = computed(() => incidentRows.value.filter(e=>e.date).length)
const officialTotal = Object.values(quality).reduce((total,row)=>total+Object.values(row.official).reduce((s,n)=>s+n,0),0)
const mismatch = officialTotal - events.length
function chooseDate(v) { selectedDate.value = v; if(v) selectedMonth.value = Number(v.slice(5,7)); page.value = 1 }
function choosePoint(v) { selectedPoint.value = v; page.value = 1 }
function clearEventFilters(){ selectedMonth.value=null;selectedLevel.value='all';selectedDate.value='';incidentQuery.value='';selectedPoint.value='';page.value=1 }

const stationQuery = ref('')
const activeCode = ref('')
const rows = computed(() => selectedMonth.value === null
  ? monthly
  : monthly.filter(d => d.month === selectedMonth.value))
const sum = key => rows.value.reduce((total, row) => total + row[key], 0)
const numbers = computed(() => ({
  total: sum('total'), abnormal: sum('abnormal'), general: sum('general'), major: sum('major'),
}))
const titleMonth = computed(() => selectedMonth.value ? `2025 年 ${selectedMonth.value} 月` : '2025 年全年')
const visibleStations = computed(() => {
  const q = stationQuery.value.trim().toLowerCase()
  return !q ? stationData : stationData.filter(s =>
    s.name.toLowerCase().includes(q) ||
    s.ename.toLowerCase().includes(q) ||
    s.code.includes(q) || s.address.toLowerCase().includes(q))
})
const activeStation = computed(() => stationData.find(s => s.code === activeCode.value))
const highest = computed(() => [...monthly].sort((a,b) => b.total - a.total)[0])
const fmt = n => new Intl.NumberFormat('zh-TW').format(n)
watch([selectedMonth,selectedLevel,incidentQuery,selectedDate,selectedPoint],()=>{page.value=1})

function toggleMonth(month) {
  selectedMonth.value = selectedMonth.value === month ? null : month
}
function reset() { selectedMonth.value = null; stationQuery.value = ''; activeCode.value = '' }
function chooseStation(code) { activeCode.value = code }
</script>

<template>
  <div class="shell">
    <aside class="sidebar">
      <div class="brand"><div class="brand-icon">T</div><div><strong>TRA INSIGHT</strong><span>Railway Visual Analytics</span></div></div>
      <div class="sidebar-group">
        <div class="side-label">研究儀表板</div>
        <button :class="['nav-link',activeView==='overview'?'selected':'muted']" @click="activeView='overview'">▣ <span>事故概覽</span></button>
        <button :class="['nav-link',activeView==='explore'?'selected':'muted']" @click="activeView='explore'">◉ <span>事件探索 V2</span></button>
      </div>
      <div class="sidebar-spacer"></div>
      <div class="sidebar-note">
        <span class="dot"></span> Data quality note
        <p>官方月報標題統計 861 件，逐筆表格辨識 860 件；地圖上的站間位置是兩站中點示意，不是事故精確座標。</p>
      </div>
      <div class="sidebar-bottom">臺鐵事故與異常事件時空視覺化<br/>Vue 3 · D3 · Leaflet</div>
    </aside>

    <main class="main">
      <template v-if="activeView==='overview'">
      <header class="topbar">
        <div><span class="eyebrow">DATA EXPLORATION / 2025</span><h1>臺鐵事故與異常事件 <em>分析總覽</em></h1>
        <p>以官方月報統計與車站地理資料，探索事件類型與時間分布。</p></div>
        <button class="reset" @click="reset">↻ 重設所有篩選</button>
      </header>

      <div class="filterbar"><div class="filter-label">資料範圍</div>
        <select v-model="selectedMonth" aria-label="選擇月份">
          <option :value="null">2025 年全年</option>
          <option v-for="n in 12" :key="n" :value="n">2025 年 {{ n }} 月</option>
        </select>
        <span class="filter-hint">可點擊下方月份長條圖進行篩選</span>
        <div class="source-pill">● 官方月報（114 年）</div>
      </div>

      <section class="stats-grid">
        <div class="stat stat-primary"><span class="stat-label">{{ titleMonth }} / 事件總數</span><div class="stat-val">{{ fmt(numbers.total) }} <small>件</small></div><div class="stat-sub">事故與行車異常事件合計</div></div>
        <div class="stat"><span class="stat-label"><i class="color-dot teal"></i> 行車異常事件</span><div class="stat-val">{{ fmt(numbers.abnormal) }} <small>件</small></div><div class="stat-sub">類別在官方月報中標示</div></div>
        <div class="stat"><span class="stat-label"><i class="color-dot gold"></i> 一般行車事故</span><div class="stat-val">{{ fmt(numbers.general) }} <small>件</small></div><div class="stat-sub">按官方事故等級彙整</div></div>
        <div class="stat"><span class="stat-label"><i class="color-dot red"></i> 重大行車事故</span><div class="stat-val">{{ fmt(numbers.major) }} <small>件</small></div><div class="stat-sub">按官方事故等級彙整</div></div>
      </section>

      <section class="content-grid">
        <div class="panel chart-panel">
          <div class="panel-head"><div><div class="section-eyebrow">TEMPORAL ANALYSIS</div><h2>每月事故與異常事件趨勢</h2><p>堆疊長條圖 · 2025 年 1–12 月</p></div><span class="panel-tag">D3.js</span></div>
          <div class="legend"><span><i class="color-dot teal"></i>行車異常事件</span><span><i class="color-dot gold"></i>一般行車事故</span><span><i class="color-dot red"></i>重大行車事故</span></div>
          <MonthlyChart :rows="monthly" :selected-month="selectedMonth" @change-month="toggleMonth" />
          <div class="chart-footer"><span>全年最高：<b>{{ highest.month }} 月 / {{ highest.total }} 件</b></span><span>點擊長條可篩選；再點一次取消</span></div>
        </div>

        <div class="panel insight-panel">
          <div class="panel-head"><div><div class="section-eyebrow">DISTRIBUTION</div><h2>事件類型占比</h2><p>{{ titleMonth }}</p></div></div>
          <div class="donut-wrap">
            <div class="css-donut" :style="{'--a': numbers.total ? numbers.abnormal / numbers.total * 100 + '%' : '0%', '--b': numbers.total ? (numbers.abnormal+numbers.general) / numbers.total * 100 + '%' : '0%'}"><div class="donut-center"><b>{{fmt(numbers.total)}}</b><small>總事件數</small></div></div>
          </div>
          <div class="dist-list">
            <div v-for="item in [{ key:'abnormal',name:'行車異常',color:'teal'},{key:'general',name:'一般事故',color:'gold'},{key:'major',name:'重大事故',color:'red'}]" :key="item.key">
              <span><i :class="['color-dot',item.color]"></i>{{item.name}}</span><strong>{{ numbers.total ? (numbers[item.key] / numbers.total * 100).toFixed(1) : 0 }}%</strong>
            </div>
          </div>
        </div>

        <div class="panel map-panel">
          <div class="panel-head"><div><div class="section-eyebrow">GEOSPATIAL DATA</div><h2>臺灣鐵路車站分布圖</h2><p>車站 GPS 點位資料 · 共 {{ stationData.length }} 筆</p></div><span class="panel-tag">Leaflet</span></div>
          <div class="map-content">
            <StationMap :stations="visibleStations" :active-code="activeCode" @choose-station="chooseStation" />
            <div class="station-side">
              <label for="station-search">搜尋車站</label>
              <input id="station-search" v-model="stationQuery" placeholder="輸入站名、車站代碼或縣市" />
              <div class="station-count">符合條件：{{visibleStations.length}} 筆車站</div>
              <div v-if="activeStation" class="selected-station"><span class="side-label">目前選取</span><h3>{{activeStation.name}}站</h3><p>{{activeStation.address}}</p><p>車站代碼：{{activeStation.code}}</p><p>GPS：{{activeStation.lat}}, {{activeStation.lon}}</p><button @click="activeCode = ''">取消選取</button></div>
              <div v-else class="station-placeholder">在地圖上點擊車站，或使用搜尋框尋找車站，就能查看名稱、代碼與 GPS 座標。</div>
              <div class="station-list">
                <button v-for="s in visibleStations.slice(0,8)" :key="s.code" :class="{chosen:s.code===activeCode}" @click="chooseStation(s.code)"><span>{{s.name}}</span><small>{{s.code}}</small></button>
              </div>
            </div>
          </div>
          <div class="map-warning"><strong>重要：</strong>目前地圖只展示車站位置。因尚未將 861 件事件逐筆匹配至車站／區間，不能把車站密集程度解讀為事故熱點；月份篩選也不會改變車站位置。</div>
        </div>
      </section>

      <section class="next-step"><div class="step-icon">↗</div><div><b>下一步：真正的事故熱點分析</b><p>先把 12 份官方 PDF 的逐筆事件整理成 <code>events.json</code>，包含日期、事件等級、車站／站間區間、原因與來源頁碼。完成地理匹配後，才加入 Calendar Heatmap、事故熱點與跨圖表連動。</p></div></section>
      <footer>資料來源：臺鐵 114 年 1–12 月行車事故（件）月報表；使用者提供的車站基本資料集 JSON。地圖圖磚 © OpenStreetMap contributors。</footer>

      </template>
      <template v-else>
        <header class="topbar">
          <div><span class="eyebrow">INCIDENT EXPLORER / 2025</span><h1>逐筆事件 <em>時空探索</em></h1>
            <p>官方事故月報逐筆表格擷取 · 日期熱圖 / 地理分布 / 原因與改善措施</p></div>
          <button class="reset" @click="clearEventFilters">↻ 清除事件篩選</button>
        </header>
        <div class="quality-banner"><b>資料來源透明</b>：官方月報月份標題合計 <strong>{{ fmt(officialTotal) }}</strong> 件，表格辨識 <strong>{{ fmt(events.length) }}</strong> 筆；7 月標題與可辨識逐筆編號相差 <strong>{{ mismatch }}</strong> 件。未用假資料補齊。</div>
        <section class="event-stats">
          <div><span>目前篩選紀錄</span><strong>{{ fmt(incidentRows.length) }}</strong><small>筆逐筆事件</small></div>
          <div><span>有效日期</span><strong>{{ fmt(datedRows) }}</strong><small>筆可繪製時間熱圖</small></div>
          <div><span>有示意座標</span><strong>{{ fmt(mapMatched) }}</strong><small>站點 / 站間中點</small></div>
          <div><span>官方月報合計</span><strong>{{fmt(officialTotal)}}</strong><small>標題統計值</small></div>
        </section>
        <section class="explore-filters">
          <label>月份<select v-model="selectedMonth"><option :value="null">全年</option><option v-for="n in 12" :key="n" :value="n">{{n}} 月</option></select></label>
          <label>事故等級<select v-model="selectedLevel"><option value="all">全部等級</option><option value="major">重大行車事故</option><option value="general">一般行車事故</option><option value="abnormal">行車異常事件</option></select></label>
          <label class="wide-input">地點 / 事件內容搜尋<input v-model="incidentQuery" placeholder="例如：彰化、號誌、轉轍器、豪雨" /></label>
          <button v-if="selectedDate" class="small-filter" @click="selectedDate=''">日期：{{selectedDate}} ×</button>
          <button v-if="selectedPoint" class="small-filter" @click="selectedPoint=''">地圖位置篩選 ×</button>
        </section>
        <section class="explore-stack">
          <div class="panel"><div class="panel-head"><div><div class="section-eyebrow">DAILY PATTERN</div><h2>2025 年每日事件熱圖</h2><p>點擊日期 → 下方地圖與事件列表同步篩選 · 色階越亮代表逐筆事件數越多</p></div><span class="panel-tag">D3.js</span></div>
            <CalendarHeatmap :events="aggregateRows" :selected-month="selectedMonth" :selected-date="selectedDate" @select-date="chooseDate" />
          </div>
          <div class="panel"><div class="panel-head"><div><div class="section-eyebrow">EVENT LOCATIONS</div><h2>事件地理分布（非風險率）</h2><p>青綠實心：車站事件；橘色虛線：站間兩端點的直線中點，僅表示區間</p></div><span class="panel-tag">Leaflet</span></div>
            <div class="map-meta"><span>可定位 {{ geoRows.filter(e=>e.display_lat!==null).length }} 筆 / {{geoRows.length}} 筆</span><span>點選圓圈 → 篩選事件列表</span></div>
            <EventMap :events="geoRows" :selected-key="selectedPoint" @select-point="choosePoint" />
            <p class="map-warning"><strong>解讀限制：</strong>站間中點不是實際事故位置；統計件數未校正班次、里程或運量，不代表事故風險。無法對應座標的事件仍保留在列表。</p>
          </div>
          <div class="panel"><div class="panel-head"><div><div class="section-eyebrow">SOURCE-TRACEABLE RECORDS</div><h2>逐筆事故事件清單</h2><p>依月報原始表格逐筆顯示；可核對月份與來源頁碼</p></div><span class="panel-tag">{{fmt(incidentRows.length)}} 筆</span></div>
            <div v-if="!incidentRows.length" class="empty-list">目前沒有符合條件的紀錄。可以清除日期或地圖位置篩選。</div>
            <div v-else class="table-scroller"><table class="event-table"><thead><tr><th>日期 / 時間</th><th>事故等級</th><th>發生位置</th><th>事件摘錄</th><th>原始來源</th><th></th></tr></thead>
              <tbody><tr v-for="e in pagedIncidents" :key="e.event_id"><td>{{e.date || '未辨識'}}<br/><small>{{e.time||'時間待核'}}</small></td><td><span :class="['lev',e.event_level]">{{levelLabels[e.event_level]}}</span></td><td>{{e.location_raw}}</td><td class="desc-cell">{{e.description||'原文待核'}}</td><td>{{e.source_month}} 月 · 第 {{e.source_page}} 頁</td><td><button class="details-btn" @click="openedEvent=e">詳情</button></td></tr></tbody>
            </table></div>
            <div class="pagination"><span>第 {{page}} / {{pageCount}} 頁 · 每頁 {{pageSize}} 筆</span><div><button :disabled="page<=1" @click="page--">上一頁</button><button :disabled="page>=pageCount" @click="page++">下一頁</button></div></div>
          </div>
          <div class="quality-info"><strong>資料品質檢查</strong><p>2025 年 12 份官方月報逐筆擷取 {{events.length}} 筆；{{events.filter(e=>!e.date).length}} 筆日期待人工確認、{{events.filter(e=>!e.time).length}} 筆時間待人工確認。地點採嚴格匹配，不確定位置不強行定位。7 月事件編號 1–94，與標題 95 件不一致，需回頭核對官方紀錄。</p></div>
        </section>
        <div v-if="openedEvent" class="event-modal" @click.self="openedEvent=null">
          <div class="event-modal-panel" role="dialog" aria-modal="true" aria-label="事故事件詳情"><div class="event-modal-head"><div><span class="eyebrow">{{openedEvent.event_id}}</span><h2>{{openedEvent.location_raw}} · {{openedEvent.date||'日期待核'}}</h2></div><button class="reset" @click="openedEvent=null">關閉 ×</button></div>
            <p><strong>時間：</strong>{{openedEvent.time||'待核對'}}　<strong>事件層級：</strong>{{levelLabels[openedEvent.event_level]}}</p>
            <h3>事故（件）概況</h3><p>{{openedEvent.description||'該欄位尚無可辨識文字'}}</p>
            <h3>原因</h3><p>{{openedEvent.cause_raw||'待核對'}}</p>
            <h3>改善對策</h3><p>{{openedEvent.countermeasure||'待核對'}}</p>
            <h3>資料來源及空間定位</h3><p>{{openedEvent.source_file}}，第 {{openedEvent.source_page}} 頁</p>
            <p>定位狀態：{{openedEvent.location_confidence==='station-coordinate'?'對應車站 GPS':openedEvent.location_confidence==='section-midpoint-approx'?'站間兩端車站直線中點（示意）':'尚無可靠座標'}}</p>
          </div>
        </div>
        <footer>資料來源：臺鐵 114 年行車事故事件月報（1–12 月）與車站基本資料集。原始月報表格 OCR-free 文字擷取，尚須逐筆人工稽核；本站不提供安全風險排名。</footer>
      </template>
    </main>
  </div>
</template>
