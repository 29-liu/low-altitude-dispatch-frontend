<template>
  <main class="dashboard-shell">
    <header class="topbar glass-card">
      <div class="brand-block">
        <div class="brand-kicker">LOW-ALTITUDE INTELLIGENT CHAIN</div>
        <h1>低空智链 · 城市无人机智能协同调度平台</h1>
        <p>Cesium 三维态势 · Java V3.4 Multi-Constraint Space-Time A* · 当前调度快照</p>
      </div>

      <div class="topbar-actions">
        <span class="system-badge" :class="{ offline: dataError }"><i></i>{{ dataError ? '数据接口异常' : 'Java V3.4 已连接' }}</span>
        <span class="mode-badge">{{ scenarioMode === 'live' ? '当前数据' : '验证演示' }}</span>

        <div class="scenario-switch" title="当前数据来自 Java V3.4；冲突演示保留已验证的双机案例">
          <button :class="{ active: scenarioMode === 'live' }" @click="switchScenario('live')">当前数据</button>
          <button :class="{ active: scenarioMode === 'demo' }" @click="switchScenario('demo')">冲突演示</button>
        </div>

        <button v-if="scenarioMode === 'live'" class="secondary-btn" @click="refreshDashboard(true)">{{ dataLoading ? '同步中…' : '刷新数据' }}</button>
        <button class="secondary-btn" @click="resetCamera">重置视角</button>
        <button class="primary-btn" @click="togglePlay">{{ playing ? '暂停仿真' : '播放仿真' }}</button>
      </div>
    </header>

    <aside class="left-panel glass-card">
      <div class="section-heading">
        <div>
          <span class="eyebrow">FLEET STATUS</span>
          <h2>无人机运行状态</h2>
        </div>
        <span class="fleet-count">{{ fleet.length }} 架</span>
      </div>

      <div class="fleet-list">
        <article
          v-for="drone in fleet"
          :key="drone.id"
          class="drone-card"
          :class="{ active: selectedDroneId === drone.id }"
          @click="focusDrone(drone.id)"
        >
          <div class="drone-card-head">
            <div class="drone-name-row">
              <span class="drone-dot" :class="statusClass(drone.status)"></span>
              <strong>{{ drone.id }}</strong>
            </div>
            <span class="status-pill" :class="statusClass(drone.status)">{{ drone.status }}</span>
          </div>
          <div class="battery-row">
            <span>电量</span>
            <b>{{ drone.battery }}%</b>
          </div>
          <div class="battery-track"><i :style="{ width: drone.battery + '%' }"></i></div>
          <div class="drone-meta">
            <span>载荷 {{ drone.payload }}/{{ drone.capacity }} kg</span>
            <span>{{ drone.currentTask || '无任务' }}</span>
          </div>
        </article>
      </div>

      <div class="mini-summary">
        <div><span>可用</span><b>{{ fleet.filter(d => d.status === '可用').length }}</b></div>
        <div><span>执行中</span><b>{{ fleet.filter(d => d.status === '执行中').length }}</b></div>
        <div><span>已分配</span><b>{{ fleet.filter(d => d.status === '已分配').length }}</b></div>
      </div>
    </aside>

    <section class="map-stage">
      <div id="cesiumContainer"></div>

      <div class="map-toolbar glass-card">
        <button :class="{ active: layerState.routes }" @click="layerState.routes = !layerState.routes; syncLayerVisibility()">航迹</button>
        <button :class="{ active: layerState.zones }" @click="layerState.zones = !layerState.zones; syncLayerVisibility()">约束区</button>
        <button :class="{ active: layerState.labels }" @click="layerState.labels = !layerState.labels; syncLayerVisibility()">标注</button>
        <span class="toolbar-sep"></span>
        <button @click="setTopCamera">俯视</button>
        <button class="camera-accent" @click="setLowCamera">低空视角</button>
      </div>

      <div class="control-hint glass-card">左键拖动平移 · 滚轮缩放 · 中键倾斜</div>

      <div class="altitude-badge glass-card">
        <span>LOW-ALTITUDE LAYER</span>
        <b>120 m</b>
        <i>仿真飞行高度</i>
      </div>

      <div class="legend glass-card" v-if="scenarioMode === 'live'">
        <span><i class="legend-line reposition"></i>前往取货点</span>
        <span><i class="legend-line baseline"></i>未约束基准航迹</span>
        <span><i class="legend-line planned"></i>实际规划航迹</span>
        <span><i class="legend-zone nofly"></i>禁飞区</span>
        <span><i class="legend-zone obstacle"></i>障碍区</span>
        <span><i class="legend-zone crowd"></i>人群风险区</span>
      </div>

      <div class="legend glass-card" v-else>
        <span><i class="legend-line task-a"></i>UAV-05 已有计划</span>
        <span><i class="legend-line task-b"></i>UAV-04 避让后航迹</span>
        <span><i class="legend-line reposition"></i>UAV-04 前往取货点</span>
        <span><i class="legend-line ghost"></i>未避让预测位置</span>
        <span><i class="legend-point conflict"></i>动态冲突位置</span>
      </div>

      <div class="timeline-panel glass-card">
        <div class="timeline-top">
          <div>
            <span>仿真时间</span>
            <strong>{{ formatSeconds(simTimeMs) }}</strong>
          </div>
          <div class="timeline-status">
            <span>{{ playing ? 'RUNNING' : 'PAUSED' }}</span>
            <b>{{ maxEndMs > 0 ? Math.round((simTimeMs / maxEndMs) * 100) : 0 }}%</b>
          </div>
        </div>
        <input
          v-model.number="simTimeMs"
          type="range"
          min="0"
          :max="Math.max(1000, maxEndMs)"
          step="1000"
          @input="renderByTime"
        />
        <div class="timeline-scale"><span>0s</span><span>{{ formatSeconds(maxEndMs) }}</span></div>
      </div>
    </section>

    <aside class="right-panel">
      <section class="glass-card task-card">
        <div class="section-heading compact">
          <div>
            <span class="eyebrow">SELECTED DRONE / MISSION</span>
            <h2>无人机与当前任务</h2>
          </div>
          <span class="mission-tag">{{ currentMission.tag }}</span>
        </div>

        <div class="selected-drone-strip">
          <span class="selected-drone-dot" :style="{ background: droneHex(selectedDroneId) }"></span>
          <div><small>当前查看</small><b>{{ selectedDroneId }}</b></div>
          <em>{{ currentMission.status }}</em>
        </div>

        <div class="route-title" :class="{ muted: !currentMission.hasRoute }">
          <div><span>取货点</span><b>{{ currentMission.pickup }}</b></div>
          <div class="route-arrow">→</div>
          <div><span>配送点</span><b>{{ currentMission.delivery }}</b></div>
        </div>

        <div class="task-grid">
          <div><span>任务编号</span><b>{{ currentMission.taskId }}</b></div>
          <div><span>任务时限</span><b>{{ currentMission.deadline }}</b></div>
          <div><span>预计时间</span><b>{{ currentMission.eta }}</b></div>
          <div><span>路径点</span><b>{{ currentMission.points }}</b></div>
        </div>

        <div v-if="currentMission.hasRoute" class="algorithm-row">
          <span>规划算法</span>
          <b>{{ currentMission.algorithm }}</b>
          <em>{{ scenarioMode === 'live' ? 'V3.4' : '验证案例' }}</em>
        </div>
        <div v-else class="mission-empty-note">{{ currentMission.note }}</div>

        <div class="mission-switch-hint">点击左侧无人机，可切换对应状态/任务。当前数据模式来自 Java V3.4 最近一次调度快照，不代表真实飞控遥测。</div>
      </section>

      <section class="glass-card constraint-card">
        <div class="section-heading compact">
          <div><span class="eyebrow warning">STATIC AIRSPACE CONSTRAINTS</span><h2>三层空域约束</h2></div>
          <span class="resolved-badge">{{ constraintSatisfiedText }}</span>
        </div>

        <div class="constraint-zone-grid">
          <div class="zone-stat nofly"><span>禁飞区</span><b>{{ constraintStats.noFly }}</b><small>硬约束</small></div>
          <div class="zone-stat obstacle"><span>障碍区</span><b>{{ constraintStats.obstacle }}</b><small>硬约束</small></div>
          <div class="zone-stat crowd"><span>人群风险区</span><b>{{ constraintStats.crowd }}</b><small>软约束</small></div>
        </div>

        <div class="constraint-compare">
          <div><span>硬约束违规</span><b class="danger-text">{{ constraintStats.hardBefore }}</b><i>→</i><b class="success-text">{{ constraintStats.hardAfter }}</b></div>
          <div><span>人群风险点</span><b class="danger-text">{{ constraintStats.riskPointBefore }}</b><i>→</i><b class="success-text">{{ constraintStats.riskPointAfter }}</b></div>
          <div><span>人群风险代价</span><b class="danger-text">{{ formatNumber(constraintStats.riskCostBefore) }}</b><i>→</i><b class="success-text">{{ formatNumber(constraintStats.riskCostAfter) }}</b></div>
        </div>

        <div class="resolution-note static-note">
          <span class="pulse-dot"></span>
          <p v-if="scenarioMode === 'live' && hasLiveData">本次加载 <strong>{{ constraintStats.total }} 个</strong>静态约束区域；基准硬约束违规 <strong>{{ constraintStats.hardBefore }}→{{ constraintStats.hardAfter }}</strong>，人群风险代价 <strong>{{ formatNumber(constraintStats.riskCostBefore) }}→{{ formatNumber(constraintStats.riskCostAfter) }}</strong>。以上均来自当前 Java 调度结果。</p>
          <p v-else>禁飞区、障碍区和人群风险区持续作为全局环境约束；冲突演示在此基础上额外展示多机动态时空占用与时间避让。</p>
        </div>
      </section>

      <section v-if="activePlanDisplayCount > 0" class="glass-card active-plan-card">
        <div class="section-heading compact">
          <div><span class="eyebrow">ACTIVE FLIGHT PLAN</span><h2>已有活动飞行计划</h2></div>
          <span class="active-badge">{{ activePlanDisplayCount }} 条占用</span>
        </div>
        <div class="plan-stack">
          <div v-for="plan in activePlanSummaries" :key="plan.id" class="plan-row plan-a"><span>{{ plan.status }}</span><b>{{ plan.droneId }}</b><em>{{ plan.taskId }}</em></div>
          <div v-if="scenarioMode === 'demo'" class="plan-row plan-b"><span>新任务</span><b>UAV-04</b><em>急救站H → 医院G</em></div>
        </div>
        <div class="neutral-note">静态空域约束持续生效；已有飞行计划进一步形成“栅格位置 + 时间窗”的动态占用。</div>
      </section>

      <section class="glass-card conflict-card">
        <div class="section-heading compact">
          <div><span class="eyebrow">DYNAMIC CONFLICT</span><h2>动态时空冲突</h2></div>
          <span v-if="conflictStats.detected && conflictStats.remaining === 0" class="resolved-badge">冲突已消解</span>
          <span v-else-if="conflictStats.detected" class="status-pill running">检测到冲突</span>
          <span v-else class="idle-badge">本轮未触发</span>
        </div>

        <div class="conflict-flow">
          <div class="flow-node"><span>活动计划</span><b>{{ conflictStats.activePlans }}</b><small>条</small></div>
          <i>→</i>
          <div class="flow-node" :class="{ danger: conflictStats.baseline > 0 }"><span>基准冲突</span><b>{{ conflictStats.baseline }}</b><small>处</small></div>
          <i>→</i>
          <div class="flow-node success"><span>剩余冲突</span><b>{{ conflictStats.remaining }}</b><small>处</small></div>
        </div>

        <div class="metric-list">
          <div><span>避让调整</span><b>{{ conflictStats.adjustments }} 次</b></div>
          <div><span>时间避让延迟</span><b :class="{ accent: conflictStats.delayMs > 0 }">{{ conflictStats.delayMs > 0 ? '+' : '' }}{{ (conflictStats.delayMs / 1000).toFixed(0) }} s</b></div>
          <div><span>时限满足</span><b class="success-text">{{ conflictStats.deadlineSatisfied ? '是' : '否' }}</b></div>
        </div>

        <div v-if="!conflictStats.detected" class="neutral-note">当前结果未检测到与已有活动飞行计划之间的动态时空冲突，因此没有触发动态冲突避让。</div>
        <div v-else class="resolution-note conflict-note">
          <span class="pulse-dot"></span>
          <p>基准航迹检测到 <strong>{{ conflictStats.baseline }} 处</strong>动态时空冲突，算法实施 <strong>{{ conflictStats.adjustments }} 次</strong>调整，时间避让延迟 <strong>{{ (conflictStats.delayMs / 1000).toFixed(0) }} s</strong>，最终剩余冲突为 <strong>{{ conflictStats.remaining }}</strong>。</p>
        </div>
      </section>

      <section class="glass-card evidence-card">
        <div class="section-heading compact"><div><span class="eyebrow">ALGORITHM EVIDENCE</span><h2>算法证据链</h2></div></div>
        <div class="evidence-chain">
          <div><b>01</b><span>加载静态约束</span><em>{{ constraintStats.total }} 个区域</em></div>
          <div><b>02</b><span>基准硬约束违规</span><em>{{ constraintStats.hardBefore }} 处</em></div>
          <div><b>03</b><span>规划后硬约束违规</span><em>{{ constraintStats.hardAfter }} 处</em></div>
          <div><b>04</b><span>人群风险代价</span><em>{{ formatNumber(constraintStats.riskCostBefore) }} → {{ formatNumber(constraintStats.riskCostAfter) }}</em></div>
          <div><b>05</b><span>动态时空冲突</span><em>{{ conflictStats.baseline }} → {{ conflictStats.remaining }}</em></div>
          <div><b>06</b><span>预计任务时间</span><em>{{ currentMission.eta }}</em></div>
          <div v-if="scenarioMode === 'live'"><b>07</b><span>数据快照更新</span><em>{{ lastUpdatedText }}</em></div>
        </div>
      </section>
    </aside>

    <footer class="bottom-log glass-card">
      <div class="log-title">
        <span class="live-dot"></span>
        <b>系统运行日志</b>
        <em>{{ scenarioMode === 'live' ? 'Java V3.4 / 当前调度快照' : '已验证双机冲突案例' }}</em>
      </div>
      <div class="log-stream">
        <span v-for="(log, index) in visibleLogs" :key="index"><i>{{ log.time }}</i>{{ log.text }}</span>
      </div>
    </footer>
  </main>
</template>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import * as Cesium from 'cesium'
import 'cesium/Build/Cesium/Widgets/widgets.css'

const GRID_ORIGIN = { lon: 121.2200, lat: 31.1200 }
const GRID_STEP_DEG = 0.0001
const FLIGHT_HEIGHT = 120
const DRONE_HEIGHT = 132
const CONFLICT_HEIGHT = 112
const DEFAULT_API_BASE = 'https://low-altitude-dispatch-api.onrender.com'
const API_BASE = String(import.meta.env.VITE_DISPATCH_API || DEFAULT_API_BASE).replace(/\/$/, '')

let viewer = null
let timer = null
let syncTimer = null
let panHandler = null
let panActive = false
let conflictGhostEntity = null
let waitMarkerEntity = null
let liveConflictPoint = null

const droneEntities = new Map()
const routeEntities = []
const routeVolumeEntities = []
const zoneEntities = []
const labelEntities = []
const projectionEntities = []
const conflictTimedEntities = []

const playing = ref(false)
const simTimeMs = ref(0)
const maxEndMs = ref(99000)
const selectedDroneId = ref('UAV-05')
const scenarioMode = ref('live')
const layerState = reactive({ routes: true, zones: true, labels: true })

const liveSnapshot = ref(null)
const dataLoading = ref(false)
const dataError = ref('')
const lastSnapshotUpdatedAt = ref(0)
const lastSnapshotSignature = ref('')

const fallbackFleet = [
  { id: 'UAV-01', status: '换电', battery: 18, payload: 0, capacity: 5, currentTask: '', x: 15, y: 15 },
  { id: 'UAV-02', status: '执行中', battery: 63, payload: 2, capacity: 5, currentTask: 'TASK-001', x: 30, y: 45 },
  { id: 'UAV-03', status: '已分配', battery: 72, payload: 0, capacity: 5, currentTask: 'TASK-002', x: 50, y: 60 },
  { id: 'UAV-04', status: '可用', battery: 85, payload: 0, capacity: 5, currentTask: '', x: 18, y: 22 },
  { id: 'UAV-05', status: '可用', battery: 88, payload: 0, capacity: 5, currentTask: '', x: 22, y: 18 }
]
const fleet = ref(fallbackFleet.map(d => ({ ...d })))
const plans = ref([])

const demoZones = [
  { id: 'NFZ-001', name: '仿真禁飞区A', type: 'NO_FLY', enabled: true, xMin: 35, xMax: 42, yMin: 18, yMax: 24, riskWeight: 0 },
  { id: 'OBS-001', name: '固定障碍区B', type: 'OBSTACLE', enabled: true, xMin: 50, xMax: 55, yMin: 18, yMax: 26, riskWeight: 0 },
  { id: 'CROWD-001', name: '人群密集风险区C', type: 'CROWD_RISK', enabled: true, xMin: 68, xMax: 72, yMin: 40, yMax: 50, riskWeight: 4 }
]

const demoTaskAPath = buildPath([22, 18], [[20, 20], [20, 34], [60, 34], [60, 55], [70, 55], [70, 65]], null)
const demoTaskBBaselinePath = buildPath([18, 22], [[18, 65], [70, 65], [20, 65], [20, 20]], null)
const demoTaskBResolvedPath = buildPath([18, 22], [[18, 65], [70, 65], [20, 65], [20, 20]], { x: 70, y: 65, extraDelayMs: 11000 })
const demoPickupIndex = demoTaskBResolvedPath.findIndex(p => p.x === 70 && p.y === 65)

const liveDispatch = computed(() => liveSnapshot.value?.dispatchResult || {})
const hasLiveData = computed(() => Boolean(liveSnapshot.value && liveDispatch.value?.success))

const constraintStats = computed(() => {
  if (scenarioMode.value === 'demo') {
    return { total: 3, noFly: 1, obstacle: 1, crowd: 1, hardBefore: 14, hardAfter: 0, riskPointBefore: 11, riskPointAfter: 0, riskCostBefore: 44, riskCostAfter: 0, satisfied: true }
  }
  const r = liveDispatch.value || {}
  return {
    total: Number(r.constraintZoneCount ?? 0),
    noFly: Number(r.noFlyZoneCount ?? 0),
    obstacle: Number(r.obstacleZoneCount ?? 0),
    crowd: Number(r.crowdRiskZoneCount ?? 0),
    hardBefore: Number(r.baselineHardConstraintViolationCount ?? 0),
    hardAfter: Number(r.plannedHardConstraintViolationCount ?? 0),
    riskPointBefore: Number(r.baselineCrowdRiskPointCount ?? 0),
    riskPointAfter: Number(r.plannedCrowdRiskPointCount ?? 0),
    riskCostBefore: Number(r.baselineCrowdRiskCost ?? 0),
    riskCostAfter: Number(r.plannedCrowdRiskCost ?? 0),
    satisfied: Boolean(r.hardConstraintSatisfied ?? true)
  }
})

const conflictStats = computed(() => {
  if (scenarioMode.value === 'demo') {
    return { detected: true, activePlans: 1, baseline: 2, adjustments: 1, delayMs: 11000, remaining: 0, deadlineSatisfied: true }
  }
  const r = liveDispatch.value || {}
  return {
    detected: Boolean(r.conflictDetected),
    activePlans: Number(r.activeFlightPlanCount ?? 0),
    baseline: Number(r.baselineConflictCount ?? 0),
    adjustments: Number(r.avoidanceAdjustmentCount ?? 0),
    delayMs: Number(r.avoidanceDelayMs ?? 0),
    remaining: Number(r.remainingConflictCount ?? 0),
    deadlineSatisfied: Boolean(r.deadlineSatisfied)
  }
})

const constraintSatisfiedText = computed(() => constraintStats.value.satisfied ? '约束满足' : '存在违规')

const currentMission = computed(() => {
  const droneId = selectedDroneId.value
  const drone = fleet.value.find(d => d.id === droneId)
  const fallbackStatus = drone?.status || '未知'
  const fallbackTask = drone?.currentTask || '无'

  if (scenarioMode.value === 'demo') {
    if (droneId === 'UAV-04') return { tag: '双机冲突避让', status: '执行中', taskId: 'TASK-B', pickup: '急救站H', delivery: '医院G', deadline: '20 min', eta: '3.35 min', points: 191, algorithm: 'Multi-Constraint Space-Time A*', hasRoute: true, note: '' }
    if (droneId === 'UAV-05') return { tag: '已有活动计划', status: '已分配', taskId: 'TASK-A', pickup: '医院G', delivery: '急救站H', deadline: '20 min', eta: '1.65 min', points: 100, algorithm: 'Multi-Constraint Space-Time A*', hasRoute: true, note: '' }
  }

  if (scenarioMode.value === 'live' && liveSnapshot.value) {
    const d = liveSnapshot.value
    const activePlans = Array.isArray(d.flightPlans) ? d.flightPlans : []
    const activePlan = activePlans.find(p => String(p.droneId || '') === droneId)

    // 第一优先级：正式“已分配/执行中”飞行计划。
    // 它代表当前生命周期状态，不能被后续的最新 dispatchResult 覆盖。
    if (activePlan) {
      const path = Array.isArray(activePlan.path) ? activePlan.path : []
      const routeMatched = activePlanMatchesSnapshotRoute(activePlan, d)
      const etaMin = getFlightPlanDurationMin(activePlan)
      const status = String(activePlan.status || fallbackStatus || '活动')
      return {
        tag: status === '执行中' ? '当前执行计划' : '当前活动计划',
        status,
        taskId: String(activePlan.taskId || fallbackTask || activePlan.planId || '活动任务'),
        pickup: routeMatched ? String(d.pickup?.name || '—') : '—',
        delivery: routeMatched ? String(d.delivery?.name || '—') : '—',
        deadline: routeMatched && d.deadlineMin !== undefined ? `${formatNumber(d.deadlineMin)} min` : '—',
        eta: etaMin !== null ? `${formatNumber(etaMin)} min` : '—',
        points: path.length || '—',
        algorithm: routeMatched ? String(liveDispatch.value.algorithm || 'Multi-Constraint Space-Time A*') : '—',
        hasRoute: path.length > 0,
        note: routeMatched ? '' : '该任务的正式飞行计划已同步；当前快照未携带与该 task_id 绑定的取配送点名称，因此不使用最近一次调度结果替代。'
      }
    }

    // 第二优先级：最近一次 Java 调度结果。仅作为尚未正式入库的规划预览。
    if (hasLiveData.value && droneId === String(liveDispatch.value.assignedDrone || '')) {
      const taskLabel = fallbackTask && fallbackTask !== '无' ? fallbackTask : '规划预览（未入库）'
      return {
        tag: '当前算法结果',
        status: fallbackStatus === '可用' ? '规划完成' : fallbackStatus,
        taskId: taskLabel,
        pickup: String(d.pickup?.name || '—'),
        delivery: String(d.delivery?.name || '—'),
        deadline: `${formatNumber(d.deadlineMin)} min`,
        eta: `${formatNumber(liveDispatch.value.estimatedTimeMin)} min`,
        points: Number(liveDispatch.value.pathPointCount ?? 0),
        algorithm: String(liveDispatch.value.algorithm || 'Multi-Constraint Space-Time A*'),
        hasRoute: true,
        note: ''
      }
    }
  }

  return {
    tag: fallbackStatus,
    status: fallbackStatus,
    taskId: fallbackTask === '无' ? '无当前任务' : fallbackTask,
    pickup: '—', delivery: '—', deadline: '—', eta: '—', points: '—', algorithm: '—', hasRoute: false,
    note: fallbackTask === '无' ? '该无人机当前没有绑定本次调度航迹；此处展示其当前数据表状态。' : '该无人机存在任务状态，但当前 dashboard 快照中没有对应的正式飞行计划明细。'
  }
})

function getFlightPlanDurationMin(plan) {
  const path = Array.isArray(plan?.path) ? plan.path : []
  if (path.length < 2) return path.length === 1 ? 0 : null
  const first = Number(path[0]?.tMs)
  const last = Number(path[path.length - 1]?.tMs)
  if (Number.isFinite(first) && Number.isFinite(last) && last >= first) return (last - first) / 60000
  const stepMs = Number(plan?.stepMs || 1000)
  return ((path.length - 1) * stepMs) / 60000
}

function activePlanMatchesSnapshotRoute(plan, snapshot) {
  const path = Array.isArray(plan?.path) ? plan.path : []
  if (!path.length || !snapshot?.pickup || !snapshot?.delivery) return false
  const pickupX = Number(snapshot.pickup.x), pickupY = Number(snapshot.pickup.y)
  const deliveryX = Number(snapshot.delivery.x), deliveryY = Number(snapshot.delivery.y)
  const hasPickup = path.some(p => Number(p.x) === pickupX && Number(p.y) === pickupY)
  const last = path[path.length - 1]
  const endsAtDelivery = Number(last?.x) === deliveryX && Number(last?.y) === deliveryY
  return hasPickup && endsAtDelivery
}

const activePlanSummaries = computed(() => {
  if (scenarioMode.value === 'demo') return [{ id: 'TASK-A', droneId: 'UAV-05', taskId: 'TASK-A · 医院G→急救站H', status: '已有计划' }]
  const rows = Array.isArray(liveSnapshot.value?.flightPlans) ? liveSnapshot.value.flightPlans : []
  return rows.map((p, i) => ({ id: p.planId || `plan-${i}`, droneId: String(p.droneId || ''), taskId: String(p.taskId || p.planId || '活动计划'), status: String(p.status || '活动') }))
})
const activePlanDisplayCount = computed(() => {
  if (scenarioMode.value === 'demo') return 1
  return Array.isArray(liveSnapshot.value?.flightPlans) ? liveSnapshot.value.flightPlans.length : 0
})

const visibleLogs = computed(() => {
  if (scenarioMode.value === 'demo') {
    return [
      { time: 'DEMO', text: '全局静态约束持续生效：禁飞区1、障碍区1、人群风险区1。' },
      { time: 'DEMO', text: '读取已有活动飞行计划：1条，UAV-05 医院G→急救站H。' },
      { time: 'DEMO', text: 'UAV-04 未避让基准航迹检测到2处动态时空冲突。' },
      { time: 'DEMO', text: 'Space-Time A*执行1次时序避让调整，增加11 s等待。' },
      { time: 'DEMO', text: '避让后剩余冲突降至0，预计3.35 min。' }
    ]
  }
  if (!hasLiveData.value) return [{ time: 'SYNC', text: dataError.value || '等待 Java V3.4 当前调度快照…' }]
  const s = constraintStats.value
  const c = conflictStats.value
  return [
    { time: 'SYNC', text: `读取当前调度快照：${s.total}个静态约束区域，${fleet.value.length}架无人机。` },
    { time: 'PLAN', text: `基准硬约束违规 ${s.hardBefore}→${s.hardAfter}，人群风险代价 ${formatNumber(s.riskCostBefore)}→${formatNumber(s.riskCostAfter)}。` },
    { time: 'DYN', text: c.detected ? `动态冲突 ${c.baseline}→${c.remaining}，调整${c.adjustments}次，延迟${Math.round(c.delayMs / 1000)}s。` : '本轮未检测到动态时空冲突。' },
    { time: 'PATH', text: `生成${Number(liveDispatch.value.pathPointCount ?? 0)}个路径点，预计${formatNumber(liveDispatch.value.estimatedTimeMin)} min。` },
    { time: 'DATA', text: `快照更新时间：${lastUpdatedText.value}。` }
  ]
})

const lastUpdatedText = computed(() => {
  const ms = Number(liveSnapshot.value?.updatedAtMs || 0)
  if (!ms) return '—'
  try { return new Date(ms).toLocaleTimeString('zh-CN', { hour12: false }) } catch { return '—' }
})

onMounted(async () => {
  await nextTick()
  initViewer()
  await refreshDashboard(false)
  applyScenarioState()
  drawScenario()
  renderByTime()
  syncTimer = window.setInterval(() => {
    if (scenarioMode.value === 'live' && !playing.value) refreshDashboard(false)
  }, 8000)
})

onBeforeUnmount(() => {
  stopPlayback()
  if (syncTimer) window.clearInterval(syncTimer)
  if (panHandler && !panHandler.isDestroyed()) panHandler.destroy()
  window.removeEventListener('mouseup', stopPan)
  if (viewer && !viewer.isDestroyed()) viewer.destroy()
})

async function refreshDashboard(forceRedraw = false) {
  if (dataLoading.value) return
  dataLoading.value = true
  try {
    const response = await fetch(`${API_BASE}/api/dashboard/state`, { cache: 'no-store' })
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    const payload = await response.json()
    if (!payload?.success) throw new Error('接口返回 success=false')
    dataError.value = ''
    if (!payload.hasData || !payload.data) {
      liveSnapshot.value = null
      if (forceRedraw && scenarioMode.value === 'live') redrawFromState()
      return
    }
    const incomingUpdated = Number(payload.data.updatedAtMs || 0)
    const incomingSignature = buildSnapshotSignature(payload.data)
    const changed = incomingSignature !== lastSnapshotSignature.value
    liveSnapshot.value = payload.data
    lastSnapshotUpdatedAt.value = incomingUpdated
    lastSnapshotSignature.value = incomingSignature
    if (scenarioMode.value === 'live' && (changed || forceRedraw)) redrawFromState()
  } catch (error) {
    dataError.value = `无法读取 Java V3.4：${error?.message || error}`
  } finally {
    dataLoading.value = false
  }
}

function buildSnapshotSignature(data) {
  const drones = Array.isArray(data?.drones) ? data.drones : []
  const flightPlans = Array.isArray(data?.flightPlans) ? data.flightPlans : []
  const droneState = drones.map(d => [d.droneId, d.status, d.currentTask, d.currentPayload, d.battery].join(':')).join('|')
  const planState = flightPlans.map(p => [p.planId, p.droneId, p.taskId, p.status, Array.isArray(p.path) ? p.path.length : 0].join(':')).join('|')
  const dispatch = data?.dispatchResult || {}
  return [
    Number(data?.updatedAtMs || 0),
    droneState,
    planState,
    String(dispatch.assignedDrone || ''),
    Number(dispatch.pathPointCount || 0),
    Number(dispatch.planningStartTimeMs || 0)
  ].join('||')
}

function redrawFromState() {
  if (!viewer) return
  stopPlayback()
  simTimeMs.value = 0
  applyScenarioState()
  clearScenarioEntities()
  drawScenario()
  renderByTime()
}

function initViewer() {
  const token = import.meta.env.VITE_CESIUM_TOKEN
  if (token && token !== 'replace_with_your_cesium_ion_token') Cesium.Ion.defaultAccessToken = token

  viewer = new Cesium.Viewer('cesiumContainer', {
    animation: false, timeline: false, baseLayerPicker: false, geocoder: false, homeButton: false,
    sceneModePicker: false, navigationHelpButton: false, infoBox: false, selectionIndicator: false,
    fullscreenButton: false, baseLayer: false
  })

  viewer.imageryLayers.addImageryProvider(new Cesium.OpenStreetMapImageryProvider({ url: 'https://tile.openstreetmap.org/' }))
  viewer.scene.globe.depthTestAgainstTerrain = false
  viewer.scene.globe.enableLighting = false
  viewer.scene.backgroundColor = Cesium.Color.fromCssColorString('#050c18')

  if (token && token !== 'replace_with_your_cesium_ion_token') {
    Cesium.createOsmBuildingsAsync().then((tileset) => viewer.scene.primitives.add(tileset)).catch(() => {})
  }

  const controller = viewer.scene.screenSpaceCameraController
  controller.enableRotate = false
  controller.enableZoom = true
  controller.enableTilt = true
  controller.enableLook = true
  controller.minimumZoomDistance = 80
  controller.maximumZoomDistance = 20000000
  enableLocalPan()
  resetCamera()
}

function enableLocalPan() {
  if (!viewer) return
  const canvas = viewer.scene.canvas
  canvas.style.cursor = 'grab'
  panHandler = new Cesium.ScreenSpaceEventHandler(canvas)
  panHandler.setInputAction(() => { panActive = true; canvas.style.cursor = 'grabbing' }, Cesium.ScreenSpaceEventType.LEFT_DOWN)
  panHandler.setInputAction(stopPan, Cesium.ScreenSpaceEventType.LEFT_UP)
  panHandler.setInputAction((movement) => {
    if (!panActive || !viewer) return
    const dx = movement.endPosition.x - movement.startPosition.x
    const dy = movement.endPosition.y - movement.startPosition.y
    if (dx === 0 && dy === 0) return
    const camera = viewer.camera
    const ellipsoid = viewer.scene.globe.ellipsoid
    const up = ellipsoid.geodeticSurfaceNormal(camera.positionWC, new Cesium.Cartesian3())
    let east = Cesium.Cartesian3.cross(Cesium.Cartesian3.UNIT_Z, up, new Cesium.Cartesian3())
    if (Cesium.Cartesian3.magnitudeSquared(east) < 1e-8) east = Cesium.Cartesian3.cross(Cesium.Cartesian3.UNIT_Y, up, east)
    Cesium.Cartesian3.normalize(east, east)
    const north = Cesium.Cartesian3.cross(up, east, new Cesium.Cartesian3())
    Cesium.Cartesian3.normalize(north, north)
    const height = Math.max(120, camera.positionCartographic.height)
    const metersPerPixel = Math.max(0.25, (height * 1.10) / Math.max(400, canvas.clientHeight))
    camera.move(east, -dx * metersPerPixel)
    camera.move(north, dy * metersPerPixel)
  }, Cesium.ScreenSpaceEventType.MOUSE_MOVE)
  window.addEventListener('mouseup', stopPan)
}

function stopPan() {
  panActive = false
  if (viewer?.scene?.canvas) viewer.scene.canvas.style.cursor = 'grab'
}

async function switchScenario(mode) {
  if (scenarioMode.value === mode) return
  stopPlayback()
  scenarioMode.value = mode
  simTimeMs.value = 0
  if (mode === 'live') await refreshDashboard(false)
  applyScenarioState()
  clearScenarioEntities()
  drawScenario()
  renderByTime()
  resetCamera()
}

function applyScenarioState() {
  liveConflictPoint = null
  if (scenarioMode.value === 'demo') {
    fleet.value = fallbackFleet.map(d => {
      if (d.id === 'UAV-05') return { ...d, status: '已分配', currentTask: 'TASK-A' }
      if (d.id === 'UAV-04') return { ...d, status: '执行中', currentTask: 'TASK-B' }
      return { ...d }
    })
    selectedDroneId.value = 'UAV-04'
    plans.value = [
      { id: 'TASK-A', droneId: 'UAV-05', path: demoTaskAPath, color: '#25d0ff' },
      { id: 'TASK-B', droneId: 'UAV-04', path: demoTaskBResolvedPath, color: '#ffb020' }
    ]
    maxEndMs.value = 201000
    return
  }

  const d = liveSnapshot.value
  if (!d) {
    fleet.value = fallbackFleet.map(x => ({ ...x }))
    plans.value = []
    selectedDroneId.value = 'UAV-05'
    maxEndMs.value = 99000
    return
  }

  fleet.value = (Array.isArray(d.drones) ? d.drones : []).map(row => ({
    id: String(row.droneId || ''), status: String(row.status || ''), battery: Number(row.battery ?? 0),
    payload: Number(row.currentPayload ?? 0), capacity: Number(row.payloadCapacity ?? 0), currentTask: String(row.currentTask || ''),
    x: Number(row.x ?? 0), y: Number(row.y ?? 0)
  }))

  const result = d.dispatchResult || {}
  const currentPath = Array.isArray(result.path) ? result.path.map(p => ({ x: Number(p.x), y: Number(p.y), tMs: Number(p.tMs ?? 0) })) : []
  const assigned = String(result.assignedDrone || '')
  const rawActivePlans = Array.isArray(d.flightPlans) ? d.flightPlans : []
  const currentPlans = []

  for (const fp of rawActivePlans) {
    const normalized = normalizeActivePlan(fp)
    if (normalized.length) currentPlans.push({ id: String(fp.planId || fp.taskId || 'active'), droneId: String(fp.droneId || ''), path: normalized, color: droneColorCss(String(fp.droneId || '')), source: 'active', status: String(fp.status || '') })
  }

  // 动画只绑定正式活动飞行计划。
  // 最近一次 dispatchResult 仍由 drawLiveRoutes() 作为“规划预览航迹”绘制，
  // 但绝不能再驱动真实无人机图标，否则会出现“UAV-04 标签跟着 UAV-05 航迹移动”的错位。
  plans.value = currentPlans

  const maxPathEnd = currentPlans.reduce((m, p) => Math.max(m, Number(p.path[p.path.length - 1]?.tMs || 0)), 0)
  maxEndMs.value = Math.max(1000, maxPathEnd || Number(currentPath[currentPath.length - 1]?.tMs || 99000))

  // 优先查看正在执行的正式计划；没有活动计划时再查看最近一次算法分配对象。
  const runningPlan = rawActivePlans.find(fp => String(fp.status || '') === '执行中') || rawActivePlans[0]
  if (runningPlan?.droneId) selectedDroneId.value = String(runningPlan.droneId)
  else if (assigned) selectedDroneId.value = assigned

  const wait = findWaitEvent(currentPath, Number(result.stepMs || 1000))
  if (wait) liveConflictPoint = wait
}

function normalizeActivePlan(fp) {
  const path = Array.isArray(fp.path) ? fp.path : []
  if (!path.length) return []
  const firstT = Number(path[0]?.tMs ?? 0)
  const absThreshold = 1_000_000_000_000
  const stepMs = Number(fp.stepMs || 1000)

  return path.map((p, index) => {
    const raw = p.tMs !== undefined ? Number(p.tMs) : index * stepMs
    let relative
    if (Number.isFinite(raw) && raw >= absThreshold && Number.isFinite(firstT) && firstT >= absThreshold) relative = raw - firstT
    else if (Number.isFinite(raw) && Number.isFinite(firstT)) relative = raw - firstT
    else relative = index * stepMs
    return { x: Number(p.x ?? 0), y: Number(p.y ?? 0), tMs: Math.max(0, Math.round(relative)) }
  })
}

function sameGridPath(a, b) {
  const p1 = Array.isArray(a) ? a : []
  const p2 = Array.isArray(b) ? b : []
  if (!p1.length || p1.length !== p2.length) return false
  for (let i = 0; i < p1.length; i++) {
    if (Number(p1[i]?.x) !== Number(p2[i]?.x) || Number(p1[i]?.y) !== Number(p2[i]?.y)) return false
  }
  return true
}

function clearScenarioEntities() {
  if (!viewer) return
  viewer.entities.removeAll()
  droneEntities.clear(); routeEntities.length = 0; routeVolumeEntities.length = 0; zoneEntities.length = 0
  labelEntities.length = 0; projectionEntities.length = 0; conflictTimedEntities.length = 0
  conflictGhostEntity = null; waitMarkerEntity = null
}

function drawScenario() {
  drawConstraintZones()
  if (scenarioMode.value === 'demo') {
    drawLocation('医院G', 20, 20, '#4ade80')
    drawLocation('急救站H', 70, 65, '#fb923c')
    drawDemoRoutes()
  } else if (hasLiveData.value) {
    const d = liveSnapshot.value
    drawLocation(String(d.pickup?.name || '取货点'), Number(d.pickup?.x || 0), Number(d.pickup?.y || 0), '#4ade80')
    drawLocation(String(d.delivery?.name || '配送点'), Number(d.delivery?.x || 0), Number(d.delivery?.y || 0), '#fb923c')
    drawLiveRoutes()
  }
  initDroneEntities()
  drawAltitudeReference()
  if (scenarioMode.value === 'demo') drawDemoConflictVisuals()
  else if (conflictStats.value.detected && liveConflictPoint) drawLiveConflictVisuals()
  syncLayerVisibility()
}

function drawConstraintZones() {
  const zones = scenarioMode.value === 'live' && Array.isArray(liveSnapshot.value?.constraintZones) && liveSnapshot.value.constraintZones.length
    ? liveSnapshot.value.constraintZones
    : demoZones

  for (const z of zones) {
    if (z.enabled === false) continue
    const type = String(z.type || '')
    const minX = Number(z.xMin ?? 0), maxX = Number(z.xMax ?? 0), minY = Number(z.yMin ?? 0), maxY = Number(z.yMax ?? 0)
    const sw = gridToLonLat(minX, minY), ne = gridToLonLat(maxX, maxY)
    const center = gridToLonLat((minX + maxX) / 2, (minY + maxY) / 2)
    const css = type === 'NO_FLY' ? '#ff405c' : type === 'OBSTACLE' ? '#8b5cf6' : '#f59e0b'
    const height = type === 'NO_FLY' ? 155 : type === 'OBSTACLE' ? 105 : 72
    const color = Cesium.Color.fromCssColorString(css)
    const entity = viewer.entities.add({ name: String(z.name || type), rectangle: { coordinates: Cesium.Rectangle.fromDegrees(sw.lon, sw.lat, ne.lon, ne.lat), height: 0, extrudedHeight: height, material: color.withAlpha(type === 'CROWD_RISK' ? 0.15 : 0.19), outline: true, outlineColor: color.withAlpha(0.92) } })
    entity.__layerType = 'zone'; zoneEntities.push(entity)
    const label = viewer.entities.add({ position: Cesium.Cartesian3.fromDegrees(center.lon, center.lat, height + 12), label: { ...makeLabel(`${String(z.name || type)} · ${type}`, color, -10, 12), showBackground: true, backgroundColor: Cesium.Color.fromCssColorString('#04101d').withAlpha(0.78), backgroundPadding: new Cesium.Cartesian2(7, 4) } })
    label.__layerType = 'label'; labelEntities.push(label)
  }
}

function drawLocation(name, x, y, color) {
  const p = gridToLonLat(x, y)
  const entity = viewer.entities.add({ name, position: Cesium.Cartesian3.fromDegrees(p.lon, p.lat, 10), point: { pixelSize: 12, color: Cesium.Color.fromCssColorString(color), outlineColor: Cesium.Color.WHITE, outlineWidth: 2, disableDepthTestDistance: Number.POSITIVE_INFINITY }, label: makeLabel(name, Cesium.Color.WHITE, -22) })
  entity.__layerType = 'label'; labelEntities.push(entity)
}

function drawLiveRoutes() {
  const d = liveSnapshot.value
  const result = d?.dispatchResult || {}
  const latestPath = Array.isArray(result.path) ? result.path.map(p => ({ x: Number(p.x), y: Number(p.y), tMs: Number(p.tMs ?? 0) })) : []
  const assigned = String(result.assignedDrone || '')
  const rawActivePlans = Array.isArray(d?.flightPlans) ? d.flightPlans : []

  // 1) 正式活动飞行计划优先绘制。选中的/执行中的计划使用完整高亮航迹。
  for (const fp of rawActivePlans) {
    const normalized = normalizeActivePlan(fp)
    if (!normalized.length) continue
    const droneId = String(fp.droneId || '')
    const status = String(fp.status || '活动')
    const color = droneColorCss(droneId)
    const isPrimary = droneId === selectedDroneId.value || status === '执行中'
    if (isPrimary) drawPlannedRoute(normalized, color, `${droneId} ${status === '执行中' ? '执行航迹' : '活动航迹'}`, false)
    else drawPolylineRoute(normalized, color, 3.2, 'solid', `${droneId} 已有活动计划`, FLIGHT_HEIGHT + 3)
  }

  // 2) 最近一次 Java 调度结果作为“规划预览”。如果已经与正式计划完全相同则不重复绘制。
  if (latestPath.length && assigned) {
    const duplicatedByActivePlan = rawActivePlans.some(fp => String(fp.droneId || '') === assigned && sameGridPath(fp.path, latestPath))
    if (!duplicatedByActivePlan) {
      const pickupX = Number(d.pickup?.x ?? 0), pickupY = Number(d.pickup?.y ?? 0)
      let pickupIndex = latestPath.findIndex(p => p.x === pickupX && p.y === pickupY)
      if (pickupIndex < 0) pickupIndex = 0
      const color = droneColorCss(assigned)
      drawPolylineRoute(latestPath.slice(0, pickupIndex + 1), '#d6edf7', 3, 'dash', `${assigned} 前往取货点`)
      if (assigned === selectedDroneId.value && !rawActivePlans.some(fp => String(fp.droneId || '') === assigned)) {
        drawPlannedRoute(latestPath.slice(pickupIndex), color, `${assigned} 实际规划航迹`, false)
      } else {
        drawPolylineRoute(latestPath.slice(pickupIndex), color, 3.4, 'solid', `${assigned} 最新调度预览`, FLIGHT_HEIGHT + 8)
      }

      const start = latestPath[0]
      const baseline = buildPath([start.x, start.y], [[pickupX, pickupY], [Number(d.delivery?.x ?? 0), Number(d.delivery?.y ?? 0)]], null)
      drawPolylineRoute(baseline, '#ff6577', 3.0, 'dash', '未约束基准航迹', FLIGHT_HEIGHT - 4)
    }
  }
}

function drawDemoRoutes() {
  drawPolylineRoute(demoTaskAPath.slice(0, 5), '#d6edf7', 3, 'dash', 'UAV-05 调机段')
  drawPlannedRoute(demoTaskAPath.slice(4), '#25d0ff', 'UAV-05 已有活动计划', false)
  drawPolylineRoute(demoTaskBResolvedPath.slice(0, demoPickupIndex + 1), '#ffcf66', 3.2, 'dash', 'UAV-04 前往取货点')
  drawPlannedRoute(demoTaskBResolvedPath.slice(demoPickupIndex), '#ffb020', 'UAV-04 配送航段', false)
  const near = demoTaskBBaselinePath.filter(p => p.tMs >= 85000 && p.tMs <= 103000)
  drawPolylineRoute(near, '#ff5d73', 2.5, 'dash', 'UAV-04 未避让冲突预测段', FLIGHT_HEIGHT + 3)
}

function drawPolylineRoute(path, colorCss, width, mode, name, height = FLIGHT_HEIGHT) {
  if (!path.length) return
  const coords = []
  for (const point of path) { const geo = gridToLonLat(point.x, point.y); coords.push(geo.lon, geo.lat, height) }
  const material = mode === 'dash' ? new Cesium.PolylineDashMaterialProperty({ color: Cesium.Color.fromCssColorString(colorCss).withAlpha(0.9), dashLength: 14 }) : Cesium.Color.fromCssColorString(colorCss)
  const entity = viewer.entities.add({ name, polyline: { positions: Cesium.Cartesian3.fromDegreesArrayHeights(coords), width, material } })
  entity.__layerType = 'route'; routeEntities.push(entity)
}

function simplifyPathForCorridor(path) {
  if (!Array.isArray(path) || path.length <= 2) return Array.isArray(path) ? path : []

  // 航路体只保留真正的转折点，避免把 Java A* 返回的逐栅格密集点
  // 全部交给 Cesium Corridor 后出现自交/几何生成不稳定。
  const simplified = [path[0]]
  let prevDx = null
  let prevDy = null

  for (let i = 1; i < path.length; i++) {
    const a = path[i - 1]
    const b = path[i]
    const dx = Math.sign(Number(b.x) - Number(a.x))
    const dy = Math.sign(Number(b.y) - Number(a.y))

    if (prevDx === null) {
      prevDx = dx
      prevDy = dy
      continue
    }

    if (dx !== prevDx || dy !== prevDy) {
      simplified.push(path[i - 1])
      prevDx = dx
      prevDy = dy
    }
  }

  simplified.push(path[path.length - 1])
  return simplified
}

function drawPlannedRoute(path, colorCss, name, withCorridor = true) {
  if (!Array.isArray(path) || path.length < 2) return

  // 真实算法路径始终使用全部路径点绘制，不对算法结果做删减。
  // 提高几米并使用“底层高亮 + 发光 + 核心线”三层绘制，确保在地形/区域体上方清晰可见。
  const routeHeight = FLIGHT_HEIGHT + 12
  const coords = []
  for (const point of path) {
    const geo = gridToLonLat(Number(point.x), Number(point.y))
    coords.push(geo.lon, geo.lat, routeHeight)
  }
  const positions = Cesium.Cartesian3.fromDegreesArrayHeights(coords)
  const color = Cesium.Color.fromCssColorString(colorCss)

  // 航路体只使用关键转折点。它是可视化辅助，不改变真实100点航迹与动画。
  if (withCorridor) {
    const corridorPath = simplifyPathForCorridor(path)
    if (corridorPath.length >= 2) {
      const corridorPositions = corridorPath.map(point => {
        const geo = gridToLonLat(Number(point.x), Number(point.y))
        return Cesium.Cartesian3.fromDegrees(geo.lon, geo.lat)
      })

      const corridor = viewer.entities.add({
        name: `${name} 航路体`,
        corridor: {
          positions: corridorPositions,
          width: 14,
          height: FLIGHT_HEIGHT - 5,
          extrudedHeight: FLIGHT_HEIGHT + 6,
          material: color.withAlpha(0.10),
          outline: true,
          outlineColor: color.withAlpha(0.30)
        }
      })
      corridor.__layerType = 'route'
      routeVolumeEntities.push(corridor)
    }
  }

  // 深色底线提供轮廓，即使地图底图很亮也能看清。
  const halo = viewer.entities.add({
    name: `${name} 高对比底线`,
    polyline: {
      positions,
      width: 12,
      arcType: Cesium.ArcType.NONE,
      material: Cesium.Color.fromCssColorString('#03111d').withAlpha(0.72),
      depthFailMaterial: Cesium.Color.fromCssColorString('#03111d').withAlpha(0.72)
    }
  })
  halo.__layerType = 'route'
  routeEntities.push(halo)

  const glow = viewer.entities.add({
    name: `${name} glow`,
    polyline: {
      positions,
      width: 9,
      arcType: Cesium.ArcType.NONE,
      material: new Cesium.PolylineGlowMaterialProperty({
        glowPower: 0.32,
        taperPower: 0.65,
        color: color.withAlpha(0.88)
      }),
      depthFailMaterial: color.withAlpha(0.72)
    }
  })
  glow.__layerType = 'route'
  routeEntities.push(glow)

  const core = viewer.entities.add({
    name,
    polyline: {
      positions,
      width: 5.6,
      arcType: Cesium.ArcType.NONE,
      material: color,
      depthFailMaterial: color
    }
  })
  core.__layerType = 'route'
  routeEntities.push(core)

  // V3.11：真实路径额外绘制“面包屑”节点。Cesium Point 使用
  // disableDepthTestDistance=Infinity，即使复杂折线或几何体异常，
  // 真实路径仍会以连续高亮节点清楚显示。
  const breadcrumbStep = path.length > 80 ? 2 : 1
  for (let i = 0; i < path.length; i += breadcrumbStep) {
    const point = path[i]
    const geo = gridToLonLat(Number(point.x), Number(point.y))
    const crumb = viewer.entities.add({
      name: `${name} 路径点 ${i + 1}`,
      position: Cesium.Cartesian3.fromDegrees(geo.lon, geo.lat, routeHeight + 2),
      point: {
        pixelSize: 5.5,
        color: color.withAlpha(0.98),
        outlineColor: Cesium.Color.WHITE.withAlpha(0.72),
        outlineWidth: 1,
        disableDepthTestDistance: Number.POSITIVE_INFINITY
      }
    })
    crumb.__layerType = 'route'
    routeEntities.push(crumb)
  }

  // 在航迹中部直接加一个标签，避免“路径已载入但视觉上无法确认”。
  const midPoint = path[Math.floor((path.length - 1) / 2)]
  if (midPoint) {
    const midGeo = gridToLonLat(Number(midPoint.x), Number(midPoint.y))
    const routeLabel = viewer.entities.add({
      name: `${name} 标签`,
      position: Cesium.Cartesian3.fromDegrees(midGeo.lon, midGeo.lat, routeHeight + 8),
      label: {
        ...makeLabel(`${name} · ${path.length}点`, color, -18, 14),
        showBackground: true,
        backgroundColor: Cesium.Color.fromCssColorString('#04101d').withAlpha(0.82),
        backgroundPadding: new Cesium.Cartesian2(8, 5),
        disableDepthTestDistance: Number.POSITIVE_INFINITY
      }
    })
    routeLabel.__layerType = 'route'
    routeEntities.push(routeLabel)
  }

  // 在起点、取货后航段中点和终点加少量方向锚点，让实际规划航迹更容易辨认。
  const markerIndices = [...new Set([0, Math.floor((path.length - 1) / 2), path.length - 1])]
  for (const index of markerIndices) {
    const point = path[index]
    const geo = gridToLonLat(Number(point.x), Number(point.y))
    const marker = viewer.entities.add({
      name: `${name} 路径锚点`,
      position: Cesium.Cartesian3.fromDegrees(geo.lon, geo.lat, routeHeight + 1),
      point: {
        pixelSize: index === path.length - 1 ? 8 : 6,
        color,
        outlineColor: Cesium.Color.WHITE.withAlpha(0.92),
        outlineWidth: 1.5,
        disableDepthTestDistance: Number.POSITIVE_INFINITY
      }
    })
    marker.__layerType = 'route'
    routeEntities.push(marker)
  }
}

function drawDemoConflictVisuals() {
  drawConflictMarkerAt({ x: 70, y: 65 }, '动态冲突位置 · 基准2处', 86000, 108000)
  drawWaitMarkerAt({ x: 69, y: 65 }, 94000, 106000, '等待避让 +11 s')
  drawConflictGhostEntity(demoTaskBBaselinePath, 86000, 102000)
}

function drawLiveConflictVisuals() {
  const wait = liveConflictPoint
  const point = wait.next || wait.current
  drawConflictMarkerAt(point, `动态冲突避让 · 基准${conflictStats.value.baseline}处`, Math.max(0, wait.beforeMs - 8000), wait.afterMs + 4000)
  drawWaitMarkerAt(wait.current, wait.beforeMs, wait.afterMs, `等待避让 +${Math.round(wait.extraDelayMs / 1000)} s`)
  const path = Array.isArray(liveDispatch.value.path) ? liveDispatch.value.path.map(p => ({ x: Number(p.x), y: Number(p.y), tMs: Number(p.tMs ?? 0) })) : []
  const noWait = removePathDelays(path, Number(liveDispatch.value.stepMs || 1000))
  drawConflictGhostEntity(noWait, Math.max(0, wait.beforeMs - 8000), wait.afterMs)
}

function drawConflictMarkerAt(point, labelText, startMs, endMs) {
  const p = gridToLonLat(point.x, point.y), color = Cesium.Color.fromCssColorString('#ff445f')
  const marker = viewer.entities.add({ position: Cesium.Cartesian3.fromDegrees(p.lon, p.lat, CONFLICT_HEIGHT), point: { pixelSize: 8, color: color.withAlpha(0.18), outlineColor: Cesium.Color.fromCssColorString('#ff6b80'), outlineWidth: 3, disableDepthTestDistance: Number.POSITIVE_INFINITY }, label: { ...makeLabel(labelText, Cesium.Color.fromCssColorString('#ffd166'), -36, 14), showBackground: true, backgroundColor: Cesium.Color.fromCssColorString('#42101a').withAlpha(0.75), backgroundPadding: new Cesium.Cartesian2(8, 5) }, show: false })
  marker.__layerType = 'zone'; marker.__startMs = startMs; marker.__endMs = endMs; zoneEntities.push(marker); conflictTimedEntities.push(marker)
  const pulseEpoch = Cesium.JulianDate.now()
  ;[0, 0.33, 0.66].forEach((phase) => {
    const radius = new Cesium.CallbackProperty((time) => { const e = Math.max(0, Cesium.JulianDate.secondsDifference(time, pulseEpoch)); return 10 + (((e / 1.8) + phase) % 1) * 55 }, false)
    const ring = viewer.entities.add({ position: Cesium.Cartesian3.fromDegrees(p.lon, p.lat, CONFLICT_HEIGHT + 1), ellipse: { semiMajorAxis: radius, semiMinorAxis: radius, height: CONFLICT_HEIGHT, material: new Cesium.ColorMaterialProperty(new Cesium.CallbackProperty((time) => { const e = Math.max(0, Cesium.JulianDate.secondsDifference(time, pulseEpoch)); const t = ((e / 1.8) + phase) % 1; return color.withAlpha(Math.max(0.02, 0.34 * (1 - t))) }, false)), outline: true, outlineColor: color.withAlpha(0.55) }, show: false })
    ring.__layerType = 'zone'; ring.__startMs = startMs; ring.__endMs = endMs; zoneEntities.push(ring); conflictTimedEntities.push(ring)
  })
}

function drawWaitMarkerAt(point, startMs, endMs, text) {
  const p = gridToLonLat(point.x, point.y)
  waitMarkerEntity = viewer.entities.add({ position: Cesium.Cartesian3.fromDegrees(p.lon, p.lat, DRONE_HEIGHT + 4), point: { pixelSize: 12, color: Cesium.Color.fromCssColorString('#ffb020').withAlpha(0.18), outlineColor: Cesium.Color.fromCssColorString('#ffd166'), outlineWidth: 3, disableDepthTestDistance: Number.POSITIVE_INFINITY }, label: { ...makeLabel(text, Cesium.Color.fromCssColorString('#ffd166'), -35, 14), showBackground: true, backgroundColor: Cesium.Color.fromCssColorString('#3a2a08').withAlpha(0.76), backgroundPadding: new Cesium.Cartesian2(8, 5) }, show: false })
  waitMarkerEntity.__layerType = 'zone'; waitMarkerEntity.__startMs = startMs; waitMarkerEntity.__endMs = endMs; zoneEntities.push(waitMarkerEntity)
}

function drawConflictGhostEntity(path, startMs, endMs) {
  if (!path.length) return
  const start = resolvePointAtTime(path, 0), geo = gridToLonLat(start.x, start.y)
  conflictGhostEntity = viewer.entities.add({ position: Cesium.Cartesian3.fromDegrees(geo.lon, geo.lat, DRONE_HEIGHT + 5), billboard: { image: droneIconDataUri('#ff7b8d'), width: 39, height: 39, color: Cesium.Color.WHITE.withAlpha(0.38), disableDepthTestDistance: Number.POSITIVE_INFINITY }, label: { ...makeLabel('未避让预测', Cesium.Color.fromCssColorString('#ff8797'), -44, 12), showBackground: true, backgroundColor: Cesium.Color.fromCssColorString('#48151d').withAlpha(0.64), backgroundPadding: new Cesium.Cartesian2(6, 4) }, show: false })
  conflictGhostEntity.__layerType = 'route'; conflictGhostEntity.__path = path; conflictGhostEntity.__startMs = startMs; conflictGhostEntity.__endMs = endMs; routeEntities.push(conflictGhostEntity)
}

const DRONE_COLORS = { 'UAV-01': '#ff667a', 'UAV-02': '#3dd6c6', 'UAV-03': '#b18cff', 'UAV-04': '#ffb020', 'UAV-05': '#25d0ff' }
function droneColorCss(id) { return DRONE_COLORS[id] || '#8da2bd' }
function droneHex(id) { return droneColorCss(id) }

function droneIconDataUri(color) {
  const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="96" height="96" viewBox="0 0 96 96"><defs><filter id="g" x="-60%" y="-60%" width="220%" height="220%"><feGaussianBlur stdDeviation="3.2" result="b"/><feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge></filter></defs><g filter="url(#g)" stroke="${color}" stroke-width="5" stroke-linecap="round" stroke-linejoin="round" fill="none"><circle cx="23" cy="23" r="12" fill="rgba(4,16,29,.88)"/><circle cx="73" cy="23" r="12" fill="rgba(4,16,29,.88)"/><circle cx="23" cy="73" r="12" fill="rgba(4,16,29,.88)"/><circle cx="73" cy="73" r="12" fill="rgba(4,16,29,.88)"/><path d="M32 32 L41 41 M64 32 L55 41 M32 64 L41 55 M64 64 L55 55"/><rect x="37" y="37" width="22" height="22" rx="7" fill="${color}" fill-opacity=".95" stroke="#eaf8ff" stroke-width="2.5"/><path d="M43 59 L39 68 M53 59 L57 68" stroke="#eaf8ff" stroke-width="3"/></g></svg>`
  return `data:image/svg+xml;charset=UTF-8,${encodeURIComponent(svg)}`
}

function initDroneEntities() {
  for (const drone of fleet.value) {
    const x = Number(drone.x ?? 0), y = Number(drone.y ?? 0)
    const geo = gridToLonLat(x, y), css = droneColorCss(drone.id), color = Cesium.Color.fromCssColorString(css)
    const offsetX = drone.id === 'UAV-04' ? -56 : drone.id === 'UAV-05' ? 56 : 0
    const offsetY = drone.id === 'UAV-04' ? -40 : drone.id === 'UAV-05' ? -52 : -34
    const entity = viewer.entities.add({ name: drone.id, position: Cesium.Cartesian3.fromDegrees(geo.lon, geo.lat, DRONE_HEIGHT), billboard: { image: droneIconDataUri(css), width: drone.id === 'UAV-04' || drone.id === 'UAV-05' ? 46 : 38, height: drone.id === 'UAV-04' || drone.id === 'UAV-05' ? 46 : 38, verticalOrigin: Cesium.VerticalOrigin.CENTER, horizontalOrigin: Cesium.HorizontalOrigin.CENTER, disableDepthTestDistance: Number.POSITIVE_INFINITY, scaleByDistance: new Cesium.NearFarScalar(100, 1.18, 8000, 0.72) }, label: { ...makeLabel(`${drone.id}  ·  ${DRONE_HEIGHT}m`, color, offsetY, drone.id === 'UAV-04' || drone.id === 'UAV-05' ? 15 : 13), pixelOffset: new Cesium.Cartesian2(offsetX, offsetY), showBackground: true, backgroundColor: color.withAlpha(0.18), backgroundPadding: new Cesium.Cartesian2(8, 5), outlineColor: Cesium.Color.fromCssColorString('#04101d'), outlineWidth: 4, disableDepthTestDistance: Number.POSITIVE_INFINITY } })
    entity.__layerType = 'label'; droneEntities.set(drone.id, entity); labelEntities.push(entity)
    const projection = viewer.entities.add({ name: `${drone.id} 高度投影`, polyline: { positions: Cesium.Cartesian3.fromDegreesArrayHeights([geo.lon, geo.lat, 0, geo.lon, geo.lat, DRONE_HEIGHT]), width: drone.id === 'UAV-04' || drone.id === 'UAV-05' ? 2 : 1, material: new Cesium.PolylineDashMaterialProperty({ color: color.withAlpha(drone.id === 'UAV-04' || drone.id === 'UAV-05' ? 0.58 : 0.22), dashLength: 10 }) } })
    projection.__layerType = 'label'; projectionEntities.push(projection); labelEntities.push(projection)
  }
}

function drawAltitudeReference() {
  const p = gridToLonLat(47, 18), base = Cesium.Cartesian3.fromDegrees(p.lon, p.lat, 0), top = Cesium.Cartesian3.fromDegrees(p.lon, p.lat, FLIGHT_HEIGHT)
  const ruler = viewer.entities.add({ polyline: { positions: [base, top], width: 2, material: new Cesium.PolylineDashMaterialProperty({ color: Cesium.Color.fromCssColorString('#6fe7ff').withAlpha(0.58), dashLength: 12 }) }, position: top, label: { ...makeLabel('120 m 低空层', Cesium.Color.fromCssColorString('#9eeeff'), -16, 12), showBackground: true, backgroundColor: Cesium.Color.fromCssColorString('#04101d').withAlpha(0.72), backgroundPadding: new Cesium.Cartesian2(6, 4) } })
  ruler.__layerType = 'label'; labelEntities.push(ruler)
}

function renderByTime() {
  // 当前数据模式下，只允许“正式活动飞行计划”驱动无人机实体。
  // dispatchResult 只是最近一次规划快照，只画航迹，不移动无人机。
  const animatablePlans = scenarioMode.value === 'live'
    ? plans.value.filter(plan => plan.source === 'active')
    : plans.value

  // 同一架无人机只允许一个动画源，防止重复计划互相覆盖位置。
  const animatedDroneIds = new Set()
  for (const plan of animatablePlans) {
    const droneId = String(plan.droneId || '')
    if (!droneId || animatedDroneIds.has(droneId)) continue
    const entity = droneEntities.get(droneId)
    if (!entity || !plan.path?.length) continue

    const point = resolvePointAtTime(plan.path, simTimeMs.value)
    const geo = gridToLonLat(point.x, point.y)
    const nextPosition = Cesium.Cartesian3.fromDegrees(geo.lon, geo.lat, DRONE_HEIGHT)

    // 显式更新原来的唯一无人机实体，不创建第二个“半透明动画机”。
    if (entity.position && typeof entity.position.setValue === 'function') entity.position.setValue(nextPosition)
    else entity.position = nextPosition

    if (entity.billboard) entity.billboard.color = Cesium.Color.WHITE.withAlpha(1.0)

    const projection = projectionEntities.find(p => p.name === `${droneId} 高度投影`)
    if (projection) {
      const nextProjection = Cesium.Cartesian3.fromDegreesArrayHeights([geo.lon, geo.lat, 0, geo.lon, geo.lat, DRONE_HEIGHT])
      if (projection.polyline?.positions && typeof projection.polyline.positions.setValue === 'function') projection.polyline.positions.setValue(nextProjection)
      else projection.polyline.positions = nextProjection
    }

    animatedDroneIds.add(droneId)
  }

  for (const entity of conflictTimedEntities) {
    const start = Number(entity.__startMs ?? 0), end = Number(entity.__endMs ?? 0)
    entity.show = layerState.zones && simTimeMs.value >= start && simTimeMs.value <= end
  }
  if (waitMarkerEntity) waitMarkerEntity.show = layerState.zones && simTimeMs.value >= Number(waitMarkerEntity.__startMs || 0) && simTimeMs.value < Number(waitMarkerEntity.__endMs || 0)
  if (conflictGhostEntity) {
    const start = Number(conflictGhostEntity.__startMs || 0), end = Number(conflictGhostEntity.__endMs || 0)
    const show = layerState.routes && simTimeMs.value >= start && simTimeMs.value <= end
    conflictGhostEntity.show = show
    if (show) {
      const point = resolvePointAtTime(conflictGhostEntity.__path || [], simTimeMs.value), geo = gridToLonLat(point.x, point.y)
      conflictGhostEntity.position = Cesium.Cartesian3.fromDegrees(geo.lon, geo.lat, DRONE_HEIGHT + 5)
    }
  }
}

function resolvePointAtTime(path, timeMs) {
  if (!path.length) return { x: 0, y: 0 }
  let current = path[0]
  for (const p of path) { if (p.tMs <= timeMs) current = p; else break }
  return current
}

function findWaitEvent(path, stepMs) {
  if (!Array.isArray(path) || path.length < 2) return null
  for (let i = 1; i < path.length; i++) {
    const diff = Number(path[i].tMs || 0) - Number(path[i - 1].tMs || 0)
    if (diff > stepMs) return { current: path[i - 1], next: path[i], beforeMs: Number(path[i - 1].tMs || 0), afterMs: Number(path[i].tMs || 0), extraDelayMs: diff - stepMs }
  }
  return null
}

function removePathDelays(path, stepMs) {
  if (!path.length) return []
  return path.map((p, i) => ({ x: p.x, y: p.y, tMs: i * stepMs }))
}

function buildPath(start, waypoints, hold) {
  const result = [{ x: start[0], y: start[1], tMs: 0 }]
  let x = start[0], y = start[1], t = 0
  for (const [tx, ty] of waypoints) {
    while (x !== tx) { x += Math.sign(tx - x); t += 1000; if (hold && x === hold.x && y === hold.y) t += hold.extraDelayMs; result.push({ x, y, tMs: t }) }
    while (y !== ty) { y += Math.sign(ty - y); t += 1000; if (hold && x === hold.x && y === hold.y) t += hold.extraDelayMs; result.push({ x, y, tMs: t }) }
  }
  return result
}

function gridToLonLat(x, y) { return { lon: GRID_ORIGIN.lon + x * GRID_STEP_DEG, lat: GRID_ORIGIN.lat + y * GRID_STEP_DEG } }

function togglePlay() {
  if (playing.value) { stopPlayback(); return }
  if (simTimeMs.value >= maxEndMs.value) simTimeMs.value = 0
  playing.value = true
  timer = window.setInterval(() => {
    simTimeMs.value += 1000
    if (simTimeMs.value >= maxEndMs.value) { simTimeMs.value = maxEndMs.value; stopPlayback() }
    renderByTime()
  }, 220)
}

function stopPlayback() { playing.value = false; if (timer) window.clearInterval(timer); timer = null }

function focusDrone(id) {
  selectedDroneId.value = id
  const entity = droneEntities.get(id)
  if (!entity || !viewer) return
  viewer.flyTo(entity, { duration: 0.8, offset: new Cesium.HeadingPitchRange(0, Cesium.Math.toRadians(-35), 900) })
}

function resetCamera() { setLowCamera() }
function setLowCamera() {
  if (!viewer) return
  const center = gridToLonLat(47, 42)
  viewer.camera.flyTo({ destination: Cesium.Cartesian3.fromDegrees(center.lon - 0.0060, center.lat - 0.0045, 820), orientation: { heading: Cesium.Math.toRadians(28), pitch: Cesium.Math.toRadians(-34), roll: 0 }, duration: 0.9 })
}
function setTopCamera() {
  if (!viewer) return
  const center = gridToLonLat(45, 42)
  viewer.camera.flyTo({ destination: Cesium.Cartesian3.fromDegrees(center.lon, center.lat, 1900), orientation: { heading: 0, pitch: Cesium.Math.toRadians(-89), roll: 0 }, duration: 0.9 })
}

function syncLayerVisibility() {
  for (const entity of routeEntities) { if (entity === conflictGhostEntity) continue; entity.show = layerState.routes }
  for (const entity of routeVolumeEntities) entity.show = layerState.routes
  for (const entity of zoneEntities) { if (conflictTimedEntities.includes(entity) || entity === waitMarkerEntity) continue; entity.show = layerState.zones }
  for (const entity of labelEntities) entity.show = layerState.labels
  renderByTime()
}

function makeLabel(text, color, offsetY = -20, fontSize = 13) {
  return { text, font: `${fontSize}px Microsoft YaHei`, fillColor: color, outlineColor: Cesium.Color.BLACK, outlineWidth: 4, style: Cesium.LabelStyle.FILL_AND_OUTLINE, verticalOrigin: Cesium.VerticalOrigin.BOTTOM, pixelOffset: new Cesium.Cartesian2(0, offsetY), disableDepthTestDistance: Number.POSITIVE_INFINITY }
}

function statusClass(status) { if (status === '可用') return 'available'; if (status === '执行中') return 'running'; if (status === '已分配') return 'assigned'; return 'charging' }
function formatSeconds(ms) { return `${Math.round(Number(ms || 0) / 1000)} s` }
function formatNumber(value) { const n = Number(value ?? 0); return Number.isInteger(n) ? String(n) : n.toFixed(2).replace(/0+$/, '').replace(/\.$/, '') }
</script>
