<template>
  <div class="video-timeline-container">
    <div
      ref="timelineRef"
      class="timeline-wrapper"
      @wheel.prevent="handleWheel"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @mouseleave="handleMouseLeave"
    >
      <canvas ref="canvasRef" class="timeline-canvas"></canvas>

      <!-- 当前播放时间指示器 -->
      <div v-if="currentTime" class="current-time-indicator" :style="{ left: currentTimePosition + 'px' }">
        <div class="time-label">{{ currentTimeLabel }}</div>
      </div>

      <!-- 悬停提示 -->
      <div v-if="hoverInfo.show" class="hover-tooltip" :style="{ left: hoverInfo.x + 'px', top: hoverInfo.y + 'px' }">
        {{ hoverInfo.time }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch, computed } from 'vue'

const props = defineProps({
  // 录像数据 [{start: timestamp, duration: ms}, ...]
  recordings: {
    type: Array,
    default: () => [],
  },
  // 当前播放时间戳
  currentTime: {
    type: Number,
    default: 0,
  },
  // 日期（用于显示）
  date: {
    type: String,
    default: '',
  },
})


const emit = defineEmits(['seek'])


const timelineRef = ref(null)
const canvasRef = ref(null)
const ctx = ref(null)

// 缩放级别：1小时到10分钟
const zoomLevel = ref(1) // 1 = 24小时, 最大 144 = 10分钟视图
const MAX_ZOOM_LEVEL = 60 
const scrollOffset = ref(0) // 水平滚动偏移
const isDragging = ref(false)
const dragStartX = ref(0)
const dragStartOffset = ref(0)

// 悬停信息
const hoverInfo = ref({
  show: false,
  x: 0,
  y: 0,
  time: '',
})

// 时间轴配置
const TIMELINE_HEIGHT = 60
const RECORDING_HEIGHT = 35
const RECORDING_TOP = 25

// 计算当前播放时间在画布上的位置
const currentTimePosition = computed(() => {
  if (!props.currentTime || !canvasRef.value) return 0

  const date = new Date(props.currentTime)
  const msFromDayStart = date.getHours() * 3600000 + date.getMinutes() * 60000 + date.getSeconds() * 1000 + date.getMilliseconds()

  const width = canvasRef.value.width / (window.devicePixelRatio || 1)
  const msPerDay = 24 * 60 * 60 * 1000
  const viewWidth = width * zoomLevel.value
  const msPerPixel = msPerDay / viewWidth

  return msFromDayStart / msPerPixel - scrollOffset.value
})

// 当前播放时间标签
const currentTimeLabel = computed(() => {
  if (!props.currentTime) return ''
  const date = new Date(props.currentTime)
  const hours = String(date.getHours()).padStart(2, '0')
  const minutes = String(date.getMinutes()).padStart(2, '0')
  const seconds = String(date.getSeconds()).padStart(2, '0')
  return `${hours}:${minutes}:${seconds}`
})

// 初始化画布
const initCanvas = () => {
  if (!canvasRef.value || !timelineRef.value) return

  const dpr = window.devicePixelRatio || 1
  const rect = timelineRef.value.getBoundingClientRect()

  canvasRef.value.width = rect.width * dpr
  canvasRef.value.height = TIMELINE_HEIGHT * dpr
  canvasRef.value.style.width = rect.width + 'px'
  canvasRef.value.style.height = TIMELINE_HEIGHT + 'px'

  ctx.value = canvasRef.value.getContext('2d')
  ctx.value.scale(dpr, dpr)

  drawTimeline()
}

// 绘制时间轴
const drawTimeline = () => {
  if (!ctx.value || !canvasRef.value) return

  const width = canvasRef.value.width / (window.devicePixelRatio || 1)
  const height = TIMELINE_HEIGHT

  // 清空画布
  ctx.value.clearRect(0, 0, width, height)

  // 背景
  ctx.value.fillStyle = '#1a1a1a'
  ctx.value.fillRect(0, 0, width, height)

  // 绘制录像区域背景（浅灰色）
  ctx.value.fillStyle = '#2a2a2a'
  ctx.value.fillRect(0, RECORDING_TOP, width, RECORDING_HEIGHT)

  // 计算可见时间范围
  const msPerDay = 24 * 60 * 60 * 1000
  const viewWidth = width * zoomLevel.value
  const msPerPixel = msPerDay / viewWidth
  const startMs = scrollOffset.value * msPerPixel
  const endMs = (scrollOffset.value + width) * msPerPixel

  // 绘制时间刻度
  drawTimeRuler(width, startMs, endMs, msPerPixel)

  // 绘制录像块
  drawRecordings(width, startMs, endMs, msPerPixel)
}

// 绘制时间刻度尺
const drawTimeRuler = (width, startMs, endMs, msPerPixel) => {
  const msPerHour = 60 * 60 * 1000
  const msPerMinute = 60 * 1000
  const msPerDay = 24 * 60 * 60 * 1000

  // 根据缩放级别决定刻度间隔，最小间隔为10分钟
  let interval = msPerHour
  if (zoomLevel.value > 12) interval = msPerMinute * 30
  if (zoomLevel.value > 24) interval = msPerMinute * 20
  if (zoomLevel.value > 48) interval = msPerMinute * 10
  // 达到最大缩放级别时，保持10分钟间隔

  const startInterval = Math.floor(startMs / interval) * interval

  ctx.value.strokeStyle = '#555'
  ctx.value.fillStyle = '#999'
  ctx.value.font = '11px Arial'
  ctx.value.textAlign = 'left'

  for (let ms = startInterval; ms <= Math.min(endMs, msPerDay); ms += interval) {
    const x = ms / msPerPixel - scrollOffset.value
    if (x < -50 || x > width) continue

    // 刻度线
    ctx.value.beginPath()
    ctx.value.moveTo(x, 0)
    ctx.value.lineTo(x, 8)
    ctx.value.stroke()

    // 时间文字
    const hours = Math.floor(ms / msPerHour) % 24
    const minutes = Math.floor((ms % msPerHour) / msPerMinute)
    const timeStr = `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}`

    // 确保文字不超出边界
    if (x >= 0 && x + 40 <= width) {
      ctx.value.fillText(timeStr, x + 2, 18)
    }
  }
}

// 绘制录像块
const drawRecordings = (width, startMs, endMs, msPerPixel) => {
  if (!props.recordings || props.recordings.length === 0) return

  ctx.value.fillStyle = '#ffd700'

  props.recordings.forEach(record => {
    // 直接使用时间戳计算当天的毫秒数
    const date = new Date(record.start)
    const recordStart = date.getHours() * 3600000 + date.getMinutes() * 60000 + date.getSeconds() * 1000 + date.getMilliseconds()
    const recordEnd = recordStart + record.duration

    // 检查是否在可见范围内
    if (recordEnd < startMs || recordStart > endMs) return

    const x = recordStart / msPerPixel - scrollOffset.value
    const w = record.duration / msPerPixel

    if (x + w > 0 && x < width) {
      const drawX = Math.max(0, x)
      const drawW = Math.min(w, width - x)
      if (drawW > 0) {
        ctx.value.fillRect(drawX, RECORDING_TOP, drawW, RECORDING_HEIGHT)
      }
    }
  })
}

// 获取时间戳在当天的毫秒数
const getTimeOfDay = timestamp => {
  const date = new Date(timestamp)
  return date.getHours() * 3600000 + date.getMinutes() * 60000 + date.getSeconds() * 1000 + date.getMilliseconds()
}

// 鼠标滚轮缩放
const handleWheel = e => {
  const delta = e.deltaY > 0 ? 0.9 : 1.1
  const newZoom = Math.max(1, Math.min(MAX_ZOOM_LEVEL, zoomLevel.value * delta))

  // 如果已经达到最大缩放级别，不再继续缩放
  if (zoomLevel.value >= MAX_ZOOM_LEVEL && delta > 1) {
    return
  }

  // 以鼠标位置为中心缩放
  const rect = timelineRef.value.getBoundingClientRect()
  const mouseX = e.clientX - rect.left
  const mouseRatio = (scrollOffset.value + mouseX) / (rect.width * zoomLevel.value)

  zoomLevel.value = newZoom
  scrollOffset.value = mouseRatio * rect.width * newZoom - mouseX
  scrollOffset.value = Math.max(0, Math.min(scrollOffset.value, rect.width * (newZoom - 1)))

  drawTimeline()
}

// 鼠标拖拽
const handleMouseDown = e => {
  isDragging.value = false
  dragStartX.value = e.clientX
  dragStartOffset.value = scrollOffset.value
}

const handleMouseMove = e => {
  if (!timelineRef.value) return
  
  const rect = timelineRef.value.getBoundingClientRect()
  const mouseX = e.clientX - rect.left

  // 检查是否开始拖拽（移动超过5像素才算拖拽）
  if (dragStartX.value && Math.abs(e.clientX - dragStartX.value) > 5) {
    isDragging.value = true
  }

  if (isDragging.value) {
    const deltaX = dragStartX.value - e.clientX
    const maxOffset = rect.width * (zoomLevel.value - 1)
    scrollOffset.value = Math.max(0, Math.min(dragStartOffset.value + deltaX, maxOffset))
    drawTimeline()
    hoverInfo.value.show = false
  } else if (!dragStartX.value) {
    // 只有在没有按下鼠标时才显示悬停提示
    const msPerDay = 24 * 60 * 60 * 1000
    const viewWidth = rect.width * zoomLevel.value
    const msPerPixel = msPerDay / viewWidth
    const hoverMs = (scrollOffset.value + mouseX) * msPerPixel

    const hours = Math.floor(hoverMs / 3600000)
    const minutes = Math.floor((hoverMs % 3600000) / 60000)
    const seconds = Math.floor((hoverMs % 60000) / 1000)

    hoverInfo.value = {
      show: true,
      x: mouseX,
      y: -25,
      time: `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`,
    }
  }
}

const handleMouseUp = e => {
  const wasDragging = isDragging.value
  
  // 立即重置拖拽状态
  isDragging.value = false
  const tempDragStartX = dragStartX.value
  dragStartX.value = 0
  dragStartOffset.value = 0

  // 只有在没有拖拽的情况下才触发点击
  if (!wasDragging && tempDragStartX) {
    const rect = timelineRef.value.getBoundingClientRect()
    const mouseX = e.clientX - rect.left
    const msPerDay = 24 * 60 * 60 * 1000
    const viewWidth = rect.width * zoomLevel.value
    const msPerPixel = msPerDay / viewWidth
    const clickMs = (scrollOffset.value + mouseX) * msPerPixel

    // 查找对应的录像
    const recording = props.recordings.find(r => {
      const start = getTimeOfDay(r.start)
      const end = start + r.duration
      return clickMs >= start && clickMs <= end
    })

    if (recording) {
      const offset = clickMs - getTimeOfDay(recording.start)
      emit('seek', recording.start + offset)
    }
  }
}

const handleMouseLeave = () => {
  // 鼠标离开时重置所有状态
  isDragging.value = false
  dragStartX.value = 0
  dragStartOffset.value = 0
  hoverInfo.value.show = false
}

// 监听数据变化
watch(
  () => props.recordings,
  () => {
    drawTimeline()
  },
  { deep: true }
)

watch(
  () => props.currentTime,
  () => {
    drawTimeline()
  }
)

// 窗口大小变化
const handleResize = () => {
  initCanvas()
}

onMounted(() => {
  initCanvas()
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped lang="scss">
.video-timeline-container {
  width: 100%;
  padding-bottom: 16px;
  padding-top: 8px;
}

.timeline-wrapper {
  position: relative;
  width: 100%;
  height: 60px;
  cursor: grab;
  user-select: none;
  border-radius: 4px;
  &:active {
    cursor: grabbing;
  }
}

.timeline-canvas {
  display: block;
  width: 100%;
  height: 100%;
  border-radius: 4px;
}

.current-time-indicator {
  position: absolute;
  top: 0;
  width: 2px;
  height: 100%;
  background: #ff4444;
  pointer-events: none;
  z-index: 10;

  &::before {
    content: '';
    position: absolute;
    top: -4px;
    left: -4px;
    width: 0;
    height: 0;
    border-left: 5px solid transparent;
    border-right: 5px solid transparent;
    border-top: 6px solid #ff4444;
  }

  .time-label {
    position: absolute;
    bottom: -24px;
    left: 50%;
    transform: translateX(-50%);
    padding: 2px 6px;
    border: 1px solid #ff4444;
    background: #ffffff;
    font-size: 11px;
    border-radius: 3px;
    white-space: nowrap;
    font-weight: 500;
  }
}

.hover-tooltip {
  position: absolute;
  padding: 4px 8px;
  background: rgba(0, 0, 0, 0.85);
  color: #fff;
  font-size: 12px;
  border-radius: 3px;
  pointer-events: none;
  white-space: nowrap;
  transform: translateX(-50%);
  z-index: 20;

  &::after {
    content: '';
    position: absolute;
    bottom: -4px;
    left: 50%;
    transform: translateX(-50%);
    width: 0;
    height: 0;
    border-left: 4px solid transparent;
    border-right: 4px solid transparent;
    border-top: 4px solid rgba(0, 0, 0, 0.85);
  }
}
</style>
