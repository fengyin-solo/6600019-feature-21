<template>
  <div class="flex h-screen">
    <!-- Sidebar -->
    <div class="w-72 bg-gray-900 p-4 flex flex-col gap-3 border-r border-gray-800 overflow-y-auto">
      <h1 class="text-lg font-bold text-cyan-400">地震波形 P/S 波分析</h1>

      <div>
        <label class="block bg-cyan-500 text-black text-center py-2 rounded cursor-pointer hover:bg-cyan-400 text-sm font-medium">
          上传 SAC/miniSEED
          <input type="file" @change="onUpload" class="hidden" />
        </label>
      </div>
      <button @click="store.loadMockData()" class="bg-gray-800 py-2 rounded text-sm hover:bg-gray-700">
        加载模拟数据
      </button>

      <!-- STA/LTA Parameters -->
      <div class="bg-gray-800 rounded-xl p-3 space-y-3">
        <h3 class="text-cyan-300 font-bold text-sm">STA/LTA 参数</h3>

        <div>
          <div class="flex justify-between items-baseline">
            <label class="text-gray-300 text-xs font-medium">STA 窗口: {{ store.staWindow.toFixed(1) }}s</label>
            <span class="text-cyan-400 text-[10px]">推荐 0.5–2.0s</span>
          </div>
          <div class="relative mt-1">
            <input type="range" v-model.number="store.staWindow" min="0.5" max="5" step="0.1" class="w-full slider" />
            <div class="range-highlight" style="left: 0%; width: 33.3%"></div>
          </div>
          <p class="text-gray-500 text-[10px] mt-1 leading-tight">短时平均窗口 · 值越小对信号突变越敏感，易检测微弱初至；值越大更平滑，减少误触发</p>
        </div>

        <div>
          <div class="flex justify-between items-baseline">
            <label class="text-gray-300 text-xs font-medium">LTA 窗口: {{ store.ltaWindow.toFixed(1) }}s</label>
            <span class="text-cyan-400 text-[10px]">推荐 8–15s</span>
          </div>
          <div class="relative mt-1">
            <input type="range" v-model.number="store.ltaWindow" min="5" max="30" step="0.5" class="w-full slider" />
            <div class="range-highlight" style="left: 12%; width: 28%"></div>
          </div>
          <p class="text-gray-500 text-[10px] mt-1 leading-tight">长时平均窗口 · 值越大背景基线越稳定，适合低信噪比检测；值过小基线波动增大</p>
        </div>

        <div>
          <div class="flex justify-between items-baseline">
            <label class="text-gray-300 text-xs font-medium">触发阈值: {{ store.threshold.toFixed(1) }}</label>
            <span class="text-cyan-400 text-[10px]">推荐 3.0–5.0</span>
          </div>
          <div class="relative mt-1">
            <input type="range" v-model.number="store.threshold" min="1" max="10" step="0.5" class="w-full slider" />
            <div class="range-highlight" style="left: 22.2%; width: 22.2%"></div>
          </div>
          <p class="text-gray-500 text-[10px] mt-1 leading-tight">STA/LTA 比值门限 · 值越高仅拾取高信噪比事件；值越低拾取更多事件但可能增加误触发</p>
        </div>

        <button @click="runPick" class="w-full bg-cyan-600 py-2 rounded text-sm hover:bg-cyan-500">
          运行自动拾取
        </button>
      </div>

      <!-- Picks -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">震相拾取结果</h3>
        <div v-for="p in store.picks" :key="p.id" class="flex justify-between bg-gray-700 rounded p-2 mb-1 text-sm">
          <span :class="p.type === 'P' ? 'text-red-400' : 'text-blue-400'">{{ p.type }} 波</span>
          <span>{{ p.time.toFixed(2) }}s</span>
          <span class="text-gray-400">{{ (p.confidence * 100).toFixed(0) }}%</span>
        </div>
        <div v-if="!store.picks.length" class="text-gray-600 text-xs">加载数据后运行拾取</div>
      </div>

      <!-- Stations -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">台站分布</h3>
        <div v-for="s in store.stations" :key="s.id"
          @click="store.selectedStation = s"
          class="bg-gray-700 rounded p-2 mb-1 text-sm cursor-pointer hover:bg-gray-600"
          :class="store.selectedStation?.id === s.id ? 'ring-1 ring-cyan-500' : ''">
          {{ s.name }} <span class="text-gray-400 text-xs">({{ s.latitude.toFixed(1) }}, {{ s.longitude.toFixed(1) }})</span>
        </div>
      </div>

      <!-- Events -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">地震事件目录</h3>
        <div v-for="e in store.events" :key="e.id" class="bg-gray-700 rounded p-2 mb-1 text-xs">
          M{{ e.magnitude }} {{ e.location }}
          <div class="text-gray-500">深度 {{ e.depth }}km | {{ e.originTime.slice(0, 16) }}</div>
        </div>
      </div>
    </div>

    <!-- Main: Waveform Charts -->
    <div class="flex-1 flex flex-col gap-2 p-4 overflow-y-auto">
      <WaveformChart v-if="store.waveform" />
      <div v-else class="flex-1 flex items-center justify-center text-gray-600">
        请上传数据或加载模拟波形
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useSeismicStore } from './store/seismic'
import WaveformChart from './components/WaveformChart.vue'

const store = useSeismicStore()

function onUpload(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) store.uploadAndAnalyze(file)
}

function runPick() {
  store.picks = store.staLtaPicking()
}
</script>

<style scoped>
.slider {
  -webkit-appearance: none;
  appearance: none;
  height: 4px;
  border-radius: 2px;
  background: #374151;
  outline: none;
  position: relative;
  z-index: 1;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #22d3ee;
  cursor: pointer;
  border: 2px solid #0e7490;
  box-shadow: 0 0 4px rgba(34, 211, 238, 0.4);
}

.slider::-moz-range-thumb {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #22d3ee;
  cursor: pointer;
  border: 2px solid #0e7490;
  box-shadow: 0 0 4px rgba(34, 211, 238, 0.4);
}

.range-highlight {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  height: 4px;
  border-radius: 2px;
  background: rgba(34, 211, 238, 0.25);
  pointer-events: none;
  z-index: 0;
}
</style>
