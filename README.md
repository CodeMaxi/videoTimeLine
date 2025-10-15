# Vue3 Video Timeline Component

一个基于 Canvas 实现的 Vue3 视频录像时间轴组件，支持缩放、拖拽、点击跳转等交互功能。
![Uploading image.png…]()

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Vue](https://img.shields.io/badge/vue-3.x-brightgreen.svg)

## ✨ 特性

- 🎯 **24小时时间轴显示** - 完整展示一天的时间范围
- 📊 **录像片段可视化** - 金色块直观显示录像时间段
- 🔍 **鼠标滚轮缩放** - 支持从24小时到10分钟的缩放级别
- 👆 **拖拽平移** - 流畅的拖拽体验
- 🎯 **点击跳转** - 点击时间轴快速定位
- ⏱️ **实时播放指示器** - 红色指示线显示当前播放位置
- 💡 **悬停提示** - 鼠标悬停显示精确时间
- 📱 **响应式设计** - 自适应容器宽度
- 🎨 **高性能渲染** - Canvas 绘制，支持大量录像片段

## 📦 安装

```bash
# 复制组件文件到你的项目
cp VideoTimeline/index.vue src/components/VideoTimeline/index.vue
```

## 🚀 快速开始

### 基础用法

```vue
<template>
  <div class="app">
    <VideoTimeline
      :recordings="recordings"
      :current-time="currentTime"
      @seek="handleSeek"
    />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import VideoTimeline from '@/components/VideoTimeline/index.vue'

// 录像数据：包含开始时间戳和持续时长（毫秒）
const recordings = ref([
  { start: 1728806400000, duration: 60000 },   // 08:00:00, 1分钟
  { start: 1728806520000, duration: 120000 },  // 08:02:00, 2分钟
  { start: 1728806700000, duration: 180000 },  // 08:05:00, 3分钟
])

// 当前播放时间戳
const currentTime = ref(1728806400000)

// 处理时间轴点击跳转
const handleSeek = (timestamp) => {
  console.log('跳转到:', new Date(timestamp))
  currentTime.value = timestamp
  // 在这里通知你的视频播放器跳转到指定时间
}
</script>

<style scoped>
.app {
  width: 100%;
  padding: 20px;
}
</style>
```

## 📖 API 文档

### Props

| 属性 | 类型 | 默认值 | 必填 | 说明 |
|------|------|--------|------|------|
| recordings | Array | `[]` | 是 | 录像数据数组 |
| currentTime | Number | `0` | 否 | 当前播放时间戳（毫秒） |
| date | String | `''` | 否 | 日期字符串（保留字段，暂未使用） |

#### recordings 数据格式

```typescript
interface Recording {
  start: number      // 开始时间戳（毫秒）
  duration: number   // 持续时长（毫秒）
}

// 示例
const recordings = [
  {
    start: 1728806400000,  // 2024-10-13 08:00:00
    duration: 60000        // 1分钟
  }
]
```

### Events

| 事件名 | 参数 | 说明 |
|--------|------|------|
| seek | `timestamp: number` | 点击时间轴时触发，返回目标时间戳（毫秒） |

### 交互说明

| 操作 | 功能 |
|------|------|
| 鼠标滚轮 | 缩放时间轴（向上放大，向下缩小） |
| 鼠标拖拽 | 左右平移时间轴 |
| 鼠标点击 | 跳转到点击位置的时间（仅在录像区域有效） |
| 鼠标悬停 | 显示当前位置的时间提示 |

## 💡 使用示例

### 示例 1：基础时间轴

```vue
<template>
  <div class="timeline-demo">
    <h3>录像时间轴</h3>
    <VideoTimeline
      :recordings="recordings"
      :current-time="currentTime"
      @seek="onSeek"
    />
    <div class="info">
      <p>当前时间: {{ formatTime(currentTime) }}</p>
      <p>录像片段数: {{ recordings.length }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import VideoTimeline from '@/components/VideoTimeline/index.vue'

const recordings = ref([
  { start: 1728806400000, duration: 3600000 },  // 08:00-09:00
  { start: 1728810000000, duration: 7200000 },  // 09:00-11:00
  { start: 1728817200000, duration: 1800000 },  // 11:00-11:30
])

const currentTime = ref(1728806400000)

const onSeek = (timestamp) => {
  currentTime.value = timestamp
}

const formatTime = (timestamp) => {
  return new Date(timestamp).toLocaleTimeString('zh-CN')
}
</script>
```

### 示例 2：动态更新录像数据

```vue
<template>
  <div class="dynamic-demo">
    <VideoTimeline
      :recordings="recordings"
      :current-time="currentTime"
      @seek="onSeek"
    />
    <button @click="addRecording">添加录像片段</button>
    <button @click="clearRecordings">清空录像</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import VideoTimeline from '@/components/VideoTimeline/index.vue'

const recordings = ref([])
const currentTime = ref(Date.now())

const addRecording = () => {
  const now = new Date()
  now.setHours(Math.floor(Math.random() * 24), 0, 0, 0)
  
  recordings.value.push({
    start: now.getTime(),
    duration: Math.random() * 3600000 // 随机时长（0-1小时）
  })
}

const clearRecordings = () => {
  recordings.value = []
}

const onSeek = (timestamp) => {
  currentTime.value = timestamp
}
</script>
```

### 示例 3：模拟播放进度

```vue
<template>
  <div class="playback-demo">
    <VideoTimeline
      :recordings="recordings"
      :current-time="currentTime"
      @seek="onSeek"
    />
    <div class="controls">
      <button @click="togglePlay">{{ isPlaying ? '暂停' : '播放' }}</button>
      <button @click="reset">重置</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onUnmounted } from 'vue'
import VideoTimeline from '@/components/VideoTimeline/index.vue'

const recordings = ref([
  { start: 1728806400000, duration: 3600000 },
])

const currentTime = ref(1728806400000)
const isPlaying = ref(false)
let timer = null

const togglePlay = () => {
  isPlaying.value = !isPlaying.value
  
  if (isPlaying.value) {
    timer = setInterval(() => {
      currentTime.value += 1000 // 每秒前进1秒
      
      // 到达录像结束时停止
      const recording = recordings.value[0]
      if (currentTime.value >= recording.start + recording.duration) {
        isPlaying.value = false
        clearInterval(timer)
      }
    }, 100) // 10倍速播放
  } else {
    clearInterval(timer)
  }
}

const reset = () => {
  isPlaying.value = false
  clearInterval(timer)
  currentTime.value = recordings.value[0].start
}

const onSeek = (timestamp) => {
  currentTime.value = timestamp
  isPlaying.value = false
  clearInterval(timer)
}

onUnmounted(() => {
  clearInterval(timer)
})
</script>
```

## 🎨 样式定制

组件使用 scoped 样式，可以通过外层容器控制宽度和间距：

```vue
<template>
  <div class="custom-timeline">
    <VideoTimeline :recordings="recordings" />
  </div>
</template>

<style scoped>
.custom-timeline {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  background: #f5f5f5;
  border-radius: 8px;
}
</style>
```

### 修改录像块颜色

如需修改录像块颜色，可以编辑组件源码中的 `drawRecordings` 函数：

```javascript
// 在 drawRecordings 函数中
ctx.value.fillStyle = '#ffd700'  // 修改为你想要的颜色
```

## 🔧 工具函数

### 时间戳转换

```javascript
// 获取当天的毫秒数（从 00:00:00 开始）
const getTimeOfDay = (timestamp) => {
  const date = new Date(timestamp)
  return date.getHours() * 3600000 + 
         date.getMinutes() * 60000 + 
         date.getSeconds() * 1000 + 
         date.getMilliseconds()
}

// 示例
const timestamp = 1728806400000  // 2024-10-13 08:00:00
const timeOfDay = getTimeOfDay(timestamp)  // 28800000 (8小时 = 8 * 3600000)
```

### 生成测试数据

```javascript
// 生成一天的随机录像数据
const generateRecordings = (count = 10) => {
  const recordings = []
  const today = new Date()
  today.setHours(0, 0, 0, 0)
  const dayStart = today.getTime()
  
  for (let i = 0; i < count; i++) {
    const start = dayStart + Math.random() * 86400000 // 随机时间
    const duration = Math.random() * 3600000 // 0-1小时
    recordings.push({ start, duration })
  }
  
  // 按时间排序
  return recordings.sort((a, b) => a.start - b.start)
}
```

## 📐 技术细节

### 缩放级别

- **最小缩放**: `zoomLevel = 1` (显示完整24小时)
- **最大缩放**: `zoomLevel = 60` (显示约10分钟)
- **缩放步进**: 每次滚轮操作缩放 10% (0.9 或 1.1)

### 时间刻度间隔

根据缩放级别自动调整：

| 缩放级别 | 刻度间隔 |
|---------|---------|
| 1-12 | 1小时 |
| 13-24 | 30分钟 |
| 25-48 | 20分钟 |
| 49+ | 10分钟 |

### 性能优化

- 使用 Canvas 绘制，支持大量录像片段
- 只绘制可见区域的内容
- 使用 `devicePixelRatio` 适配高清屏幕
- 防抖处理窗口大小变化

## ⚠️ 注意事项

1. **时间戳格式**: 所有时间戳必须使用毫秒（JavaScript Date 标准格式）
2. **同一天录像**: 录像数据必须是同一天的，跨天录像需要分开处理
3. **时区问题**: 组件使用本地时区，确保时间戳与本地时区一致
4. **浏览器兼容**: 需要支持 Canvas API 的现代浏览器

## 🐛 常见问题

### Q: 时间轴不显示？

**A**: 检查以下几点：
- 确保 `recordings` 数组不为空
- 确保 `start` 和 `duration` 都是数字类型
- 检查时间戳是否在合理范围内

```javascript
// 正确的数据格式
const recordings = [
  { start: 1728806400000, duration: 60000 }
]

// 错误的数据格式
const recordings = [
  { start: "1728806400000", duration: "60000" }  // ❌ 字符串类型
]
```

### Q: 点击跳转不准确？

**A**: 确保：
- `currentTime` 和 `recordings` 的时间戳在同一天
- 点击位置在录像块上（金色区域）

### Q: 如何处理跨天录像？

**A**: 将录像按日期分组，每天使用独立的时间轴组件：

```vue
<template>
  <div v-for="date in dates" :key="date">
    <h3>{{ date }}</h3>
    <VideoTimeline
      :recordings="getRecordingsByDate(date)"
      :current-time="getCurrentTime(date)"
    />
  </div>
</template>
```

### Q: 如何自定义颜色？

**A**: 修改组件源码中的颜色值：

```javascript
// 背景色
ctx.value.fillStyle = '#1a1a1a'  // 深色背景
ctx.value.fillStyle = '#2a2a2a'  // 录像区域背景

// 录像块颜色
ctx.value.fillStyle = '#ffd700'  // 金色

// 刻度线颜色
ctx.value.strokeStyle = '#555'   // 刻度线
ctx.value.fillStyle = '#999'     // 时间文字
```

## 📄 License

MIT License

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📮 联系方式

如有问题或建议，请通过 GitHub Issues 联系。

---

**Enjoy coding! 🎉**
