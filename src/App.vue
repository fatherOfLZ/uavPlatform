<template>
  <div class="dashboard-container">
    <header class="dashboard-header">
      <div class="title-logo-row">
        <div class="logo-placeholder"></div>

        <div class="title-area">
          <h1>双无人机协同导航可视化仿真平台</h1>
        </div>

        <div class="logo-container">
          <img src="./assets/images/sdu.png" class="logo-image" />
        </div>
      </div>

      <div class="controls-producer-row">
        <div class="simulation-controls">
          <button @click="togglePlay" :class="{ active: isPlaying }">
            {{ isPlaying ? '⏸ 暂停仿真' : '▶ 开始仿真' }}
          </button>
          <button @click="stopSimulation">⏹ 终止任务</button>

          <div class="progress-container">
            <span class="time-label">{{ formattedProgress }}%</span>
            <input
                type="range"
                min="0"
                max="100"
                v-model="simProgress"
                class="progress-bar"
                @input="pauseOnDrag"
            />
          </div>
        </div>

        <div class="producer-info">
          <span>制作人：朱源 刘宸宇 李冠赫</span>
        </div>
      </div>
    </header>

    <div class="main-content">
      <div class="left-column">
        <div class="card terminal-card">
          <h3>[SYS] 自然语言指令 (Prompt)</h3>
          <div class="terminal-text">
            <p class="sys-log">> object_description_with_help:</p>
            <p class="highlight-text">"Compass north corresponds to the top of the bird's-eye-view image. The target location is 50.39 degrees north by east from the starting point. The description of the target and its surrounding is shown below."</p>
            <p class="sys-log">> object_description:</p>
            <p class="highlight-text">"The red car is positioned on a street next to a house with a garage, surrounded by tall green hedges on one side and power lines overhead. The street is open with clear skies above, and there is a shadow of the vehicle visible on the pavement."</p>
          </div>
        </div>
      </div>

      <div class="panel panel-center">
        <div class="video-wrapper card">
          <video
              ref="mainVideo"
              src="./assets/video/video1.mp4"
              loop
              muted
              class="main-video"
          ></video>
        </div>

        <div class="card data-stream-card">
          <h3>实时飞行轨迹与动作流 (Live Telemetry)</h3>
          <div class="terminal-text scroll-auto">
            <p class="sys-log">--- 低空 UAV Trajectory ---</p>
            <p v-for="(pos, index) in highUavTraj" :key="'h'+index">[{{ pos.join(', ') }}]</p>
            <p class="sys-log mt-2">--- 低空 UAV Pose & Actions ---</p>
            <p v-for="(pos, index) in lowUavTraj" :key="'l'+index">
              POS: [{{ pos[0].toFixed(2) }}, {{ pos[1].toFixed(2) }}, {{ pos[2].toFixed(2) }}] | ACT: {{ actions[index] || 8 }}
            </p>
          </div>
        </div>
      </div>

      <div class="right-column">
        <div class="card image-card">
          <h3>高空端推理：Ortho & Heatmap</h3>
          <div class="img-stack">
            <img src="./assets/images/1/step_1778005768049_1_ortho.png" class="base-img" />
            <img src="./assets/images/1/step_1778005768049_3_heatmap.png" class="overlay-img" />
          </div>
        </div>

        <div class="card image-card">
          <h3>高空端感知：Depth Map</h3>
          <div class="img-stack">
            <img src="./assets/images/1/step_1778005768049_2_depth.png" class="base-img" />
          </div>
        </div>

        <div class="card image-card">
          <h3>低空端感知：FPV Camera</h3>
          <div class="img-stack">
            <img :src="currentFpvImage" class="base-img" />
            <div class="camera-rec">● REC</div>
          </div>
        </div>
      </div>
    </div>

    <div class="bottom-evaluation">
      <div class="eval-left">
        <div class="card">
          <h3>当前任务评估 (Current Task)</h3>
          <div class="metrics-grid">
            <div class="metric-item"><span>Success:</span> <span :class="currentEval.success ? 'text-green' : 'text-red'">{{ currentEval.success }}</span></div>
            <div class="metric-item"><span>Oracle Success:</span> <span :class="currentEval.oracle_success ? 'text-green' : 'text-red'">{{ currentEval.oracle_success }}</span></div>
            <div class="metric-item"><span>Collision:</span> <span :class="currentEval.collision ? 'text-red' : 'text-green'">{{ currentEval.collision }}</span></div>
            <div class="metric-item"><span>AirSim Collision:</span> <span :class="currentEval.airsim_collision ? 'text-red' : 'text-green'">{{ currentEval.airsim_collision }}</span></div>
            <div class="metric-item"><span>SPL:</span> <span class="text-blue">{{ currentEval.spl.toFixed(4) }}</span></div>
            <div class="metric-item"><span>SST:</span> <span class="text-blue">{{ currentEval.sst.toFixed(4) }}</span></div>
            <div class="metric-item"><span>Steps:</span> <span class="text-yellow">{{ currentEval.steps }}</span></div>
            <div class="metric-item"><span>Path Len:</span> <span class="text-yellow">{{ currentEval.path_length.toFixed(1) }}m</span></div>
            <div class="metric-item"><span>Time:</span> <span class="text-yellow">{{ currentEval.time.toFixed(1) }}s</span></div>
            <div class="metric-item"><span>Dist2End:</span> <span class="text-yellow">{{ currentEval.distance_to_end.toFixed(1) }}m</span></div>
          </div>
        </div>
      </div>

      <div class="eval-right">
        <div class="card">
          <h3>累计任务评估 (Cumulative)</h3>
          <div class="metrics-grid">
            <div class="metric-item"><span>Total:</span> <span class="text-white">{{ totalEval.total_cnt }}</span></div>
            <div class="metric-item"><span>Success:</span> <span class="text-green">{{ totalEval.success_cnt }}</span></div>
            <div class="metric-item"><span>Oracle Success:</span> <span class="text-green">{{ totalEval.oracle_success_cnt }}</span></div>
            <div class="metric-item"><span>Collision:</span> <span class="text-red">{{ totalEval.collision_cnt }}</span></div>
            <div class="metric-item"><span>AirSim Collision:</span> <span class="text-red">{{ totalEval.airsim_collision_cnt }}</span></div>
            <div class="metric-item"><span>SPL (avg):</span> <span class="text-blue">{{ totalEval.spl_cnt.toFixed(2) }}</span></div>
            <div class="metric-item"><span>SST (avg):</span> <span class="text-blue">{{ totalEval.sst_cnt.toFixed(2) }}</span></div>
            <div class="metric-item"><span>Total Length:</span> <span class="text-yellow">{{ totalEval.total_length.toFixed(1) }}m</span></div>
            <div class="metric-item"><span>Total Error:</span> <span class="text-yellow">{{ totalEval.total_error.toFixed(1) }}m</span></div>
            <div class="metric-item"><span>Nav Time:</span> <span class="text-yellow">{{ totalEval.nav_time_stat.toFixed(0) }}s</span></div>
            <div class="metric-item"><span>Nav Velocity:</span> <span class="text-yellow">{{ totalEval.nav_velocity_stat.toFixed(1) }}m/s</span></div>
            <div class="metric-item"><span>Nav Count:</span> <span class="text-white">{{ totalEval.nav_stat_cnt }}</span></div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

// ==========================================
// 导入第一人称视角 (FPV) 图片资源
// ==========================================
import fpv1 from './assets/images/front/000005.png'
import fpv2 from './assets/images/front/000045.png'
import fpv3 from './assets/images/front/000105.png'

// ==========================================
// 状态与仿真控制逻辑
// ==========================================
const isPlaying = ref(false)
const simProgress = ref(0)
let timer = null
const mainVideo = ref(null)

const formattedProgress = computed(() => Number(simProgress.value).toFixed(1))

// 动态计算当前显示的 FPV 图片 (根据进度条 0-33%, 33-66%, 66-100%)
const currentFpvImage = computed(() => {
  if (simProgress.value < 33) return fpv1
  if (simProgress.value < 66) return fpv2
  return fpv3
})

const togglePlay = () => {
  isPlaying.value = !isPlaying.value
  if (isPlaying.value) {
    if (mainVideo.value) mainVideo.value.play()
    timer = setInterval(() => {
      simProgress.value = Math.min(Number(simProgress.value) + 0.5, 100)
      if (simProgress.value >= 100) stopSimulation()
    }, 100) // 进度条更新速度
  } else {
    pauseSimulation()
  }
}

const pauseSimulation = () => {
  isPlaying.value = false
  clearInterval(timer)
  if (mainVideo.value) mainVideo.value.pause()
}

const stopSimulation = () => {
  pauseSimulation()
  simProgress.value = 0
  if (mainVideo.value) {
    mainVideo.value.currentTime = 0
  }
}

const pauseOnDrag = () => {
  if (isPlaying.value) pauseSimulation()
  // 拖动进度条时，可以按比例快进视频
  if (mainVideo.value && mainVideo.value.duration) {
    mainVideo.value.currentTime = (simProgress.value / 100) * mainVideo.value.duration
  }
}

// ==========================================
// 评估数据注入
// ==========================================
const currentEval = ref({"success": true, "oracle_success": true, "collision": false, "airsim_collision": false, "spl": 0.8913942705045271, "sst": 0.9558702055720822, "steps": 54, "path_length": 279.7278724406281, "time": 28.32840433716774, "distance_to_end": 32.17969798422464})
const totalEval = ref({"total_cnt": 325, "success_cnt": 51, "oracle_success_cnt": 89, "collision_cnt": 252, "airsim_collision_cnt": 198, "spl_cnt": 48.02867998470429, "sst_cnt": 49.7319668654627, "total_length": 36691.85793748148, "total_error": 33700.99697302942, "nav_time_stat": 31207.039610862732, "nav_velocity_stat": 5477.3533074790475, "nav_stat_cnt": 682})

// ==========================================
// 轨迹数据注入
// ==========================================
const highUavTraj = ref([
  [-124.54, -17.80, -40.0], [-124.32, -17.55, -39.96], [-123.80, -16.91, -39.93],
  [-123.19, -16.18, -39.94], [-122.56, -15.42, -39.94], [-121.93, -14.66, -39.94]
])
const lowUavTraj = ref([
  [-124.54, -17.80, -2.59, -2.90], [-120.73, -17.91, -2.54, -0.02], [-115.91, -18.04, -2.52, -0.02],
  [-111.02, -18.18, -2.51, -0.02], [-106.07, -18.31, -2.49, -0.02], [-101.11, -18.45, -2.48, -0.02]
])
const actions = ref([2, 8, 8, 8, 8, 8, 8])

</script>

<style>
/* --- 全局基础设定 --- */
* { box-sizing: border-box; margin: 0; padding: 0; }
body { background-color: #0b1120; color: #e2e8f0; font-family: 'Helvetica Neue', sans-serif; overflow-y: auto; }
.dashboard-container { display: flex; flex-direction: column; padding: 10px 15px; min-height: 100vh; }

/* --- 头部控制台 --- */
.dashboard-header { display: flex; flex-direction: column; padding-bottom: 10px; border-bottom: 1px solid rgba(0, 240, 255, 0.2); margin-bottom: 15px;}

.title-logo-row { display: flex; justify-content: space-between; align-items: center; width: 100%; margin-bottom: 10px; }
.logo-placeholder { flex: 1; }
.title-area { flex: 1; text-align: center; }
.dashboard-header h1 { font-size: 23px; color: #00f0ff; text-shadow: 0 0 10px rgba(0, 240, 255, 0.4); }

.logo-container { flex: 1; display: flex; justify-content: flex-end; }
.logo-image { height: 50px; width: auto; object-fit: contain; }

.controls-producer-row { display: flex; justify-content: center; align-items: center; width: 100%; position: relative; }

.simulation-controls { display: flex; align-items: center; gap: 15px; background: rgba(15, 23, 42, 0.6); padding: 8px 15px; border-radius: 8px; border: 1px solid rgba(0, 240, 255, 0.3);}
.simulation-controls button { background: transparent; border: 1px solid #00f0ff; color: #00f0ff; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold; transition: 0.3s; }
.simulation-controls button:hover, .simulation-controls button.active { background: rgba(0, 240, 255, 0.2); box-shadow: 0 0 10px rgba(0, 240, 255, 0.5); }

.producer-info { position: absolute; right: 0; }
.producer-info span { font-size: 15px; color: #00f0ff; text-shadow: 0 0 10px rgba(0, 240, 255, 0.4); font-family: 'Helvetica Neue', sans-serif; }

/* --- 主内容区布局 --- */
.main-content { display: flex; gap: 15px; align-items: stretch; margin-bottom: 15px; }
.left-column { flex: 0 0 25%; display: flex; flex-direction: column; gap: 10px; }
.panel-center { flex: 1; display: flex; flex-direction: column; gap: 10px; }
.right-column { flex: 0 0 25%; display: flex; flex-direction: column; gap: 10px; }

/* --- 卡片通用样式 --- */
.card { background-color: rgba(30, 41, 59, 0.7); border: 1px solid rgba(0, 240, 255, 0.2); border-radius: 6px; padding: 12px; box-shadow: inset 0 0 20px rgba(0, 240, 255, 0.02); display: flex; flex-direction: column;}
.card h3 { color: #38bdf8; font-size: 14px; margin-bottom: 10px; border-bottom: 1px solid rgba(56, 189, 248, 0.3); padding-bottom: 4px; display: flex; justify-content: space-between;}

/* --- 终端风格框 (Prompt & 数据流) --- */
.terminal-text { font-family: 'Consolas', monospace; font-size: 12px; line-height: 1.5; background: #000; padding: 10px; border-radius: 4px; border: 1px solid #333; }
.scroll-auto { max-height: 150px; overflow-y: auto; }
.scroll-auto::-webkit-scrollbar { width: 4px; }
.scroll-auto::-webkit-scrollbar-thumb { background: #333; }
.sys-log { color: #22c55e; } /* 绿色系统日志 */
.highlight-text { color: #cbd5e1; margin-bottom: 8px; margin-left: 10px; border-left: 2px solid #00f0ff; padding-left: 8px;}
.mt-2 { margin-top: 10px; }

/* --- 评估指标 Grid 布局 --- */
.metrics-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 8px; font-family: monospace; font-size: 13px; }
.metric-item { background: rgba(0,0,0,0.3); padding: 4px 8px; border-radius: 4px; display: flex; justify-content: space-between; border-left: 2px solid rgba(0, 240, 255, 0.5);}
.text-green { color: #4ade80; font-weight: bold;}
.text-red { color: #f87171; font-weight: bold;}
.text-blue { color: #60a5fa; font-weight: bold;}
.text-yellow { color: #facc15; font-weight: bold;}
.text-white { color: #ffffff; font-weight: bold;}


/* =========================================================
   核心修改区：视频与图像填充修复
   ========================================================= */

.video-wrapper {
  height: 400px;
  padding: 4px;
  background: #000;
  border-radius: 6px;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}
.main-video {
  width: 100%;
  height: 100%;
  object-fit: contain; /* 关键修改：从 cover 改为 contain，画面100%保留不裁剪 */
}

.data-stream-card { height: 200px; flex-shrink: 0; }

.image-card { min-height: 180px; display: flex; flex-direction: column; flex-shrink: 0; }

.img-stack {
  position: relative;
  flex: 1;
  background: #000;
  border-radius: 4px;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}

.base-img, .overlay-img {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: contain; /* 关键修改：所有图片由 cover 改为 contain，确保边缘信息不丢失 */
}

.overlay-img { mix-blend-mode: screen; }

/* FPV 录制红点特效 */
.camera-rec { position: absolute; top: 10px; right: 10px; color: red; font-weight: bold; font-family: monospace; font-size: 12px; text-shadow: 0 0 5px red; animation: blink 1s infinite; z-index: 10;}
@keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0; } }

/* --- 底部评估区域布局 --- */
.bottom-evaluation { display: flex; gap: 15px; }
.eval-left { flex: 1; }
.eval-right { flex: 1; }
</style>
