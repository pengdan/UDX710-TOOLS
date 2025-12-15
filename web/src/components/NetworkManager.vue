<script setup>
import { inject, ref, watch, onMounted } from 'vue'
import { toggleAirplaneMode, setNetworkMode, switchSlot, getDataStatus, setDataStatus, getRoamingStatus, setRoamingAllowed } from '../composables/useApi'
import { useToast } from '../composables/useToast'

const { success, error } = useToast()

const systemInfo = inject('systemInfo')

const airplaneMode = ref(false)
const selectedNetworkMode = ref('')
const currentSlot = ref('slot1')
const saving = ref(false)

// 数据连接和漫游状态
const dataActive = ref(false)
const dataLoading = ref(false)
const roamingAllowed = ref(false)
const isRoaming = ref(false)
const roamingLoading = ref(false)

const networkModes = [
  { value: 'nr_5g_lte_auto', label: '5G/LTE Auto', icon: 'fa-a', desc: '自动切换' },
  { value: 'nr_5g_only', label: 'NR 5G Only', icon: 'fa-5', desc: '仅5G网络' },
  { value: 'lte_only', label: 'LTE Only', icon: 'fa-4', desc: '仅4G网络' },
  { value: 'nsa_only', label: 'NSA Only', icon: 'fa-n', desc: '非独立组网' }
]

watch(() => systemInfo.value, (info) => {
  if (info) {
    airplaneMode.value = info.airplane_mode === true || info.airplane_mode === 1
    currentSlot.value = info.sim_slot || 'slot1'
    if (info.select_network_mode) {
      if (info.select_network_mode.includes('LTE only')) selectedNetworkMode.value = 'lte_only'
      else if (info.select_network_mode.includes('NR 5G only')) selectedNetworkMode.value = 'nr_5g_only'
      else if (info.select_network_mode.includes('NR 5G/LTE auto')) selectedNetworkMode.value = 'nr_5g_lte_auto'
      else if (info.select_network_mode.includes('NSA only')) selectedNetworkMode.value = 'nsa_only'
    }
  }
}, { immediate: true })

async function handleAirplaneToggle() {
  try {
    await toggleAirplaneMode(airplaneMode.value)
    success(airplaneMode.value ? '飞行模式已开启' : '飞行模式已关闭')
  } catch (err) {
    airplaneMode.value = !airplaneMode.value
    error('切换失败: ' + err.message)
  }
}

async function handleNetworkModeChange(mode) {
  try {
    await setNetworkMode(mode)
    success('网络模式设置已提交')
  } catch (err) {
    error('切换失败: ' + err.message)
  }
}

async function handleSlotSwitch(slot) {
  if (currentSlot.value === slot || saving.value) return
  saving.value = true
  try {
    await switchSlot(slot)
    currentSlot.value = slot
    success('卡槽切换成功')
  } catch (err) {
    error('切换失败: ' + err.message)
  } finally {
    saving.value = false
  }
}

// 获取数据连接状态
async function fetchDataStatus() {
  try {
    const res = await getDataStatus()
    if (res.status === 'ok' && res.data) {
      dataActive.value = res.data.active
    }
  } catch (err) {
    console.error('获取数据连接状态失败:', err)
  }
}

// 切换数据连接
async function handleDataToggle() {
  dataLoading.value = true
  try {
    const res = await setDataStatus(dataActive.value)
    if (res.status === 'ok') {
      success(dataActive.value ? '数据连接已开启' : '数据连接已关闭')
    } else {
      dataActive.value = !dataActive.value
      error(res.message || '操作失败')
    }
  } catch (err) {
    dataActive.value = !dataActive.value
    error('切换失败: ' + err.message)
  } finally {
    dataLoading.value = false
  }
}

// 获取漫游状态
async function fetchRoamingStatus() {
  try {
    const res = await getRoamingStatus()
    if (res.status === 'ok' && res.data) {
      roamingAllowed.value = res.data.roaming_allowed
      isRoaming.value = res.data.is_roaming
    }
  } catch (err) {
    console.error('获取漫游状态失败:', err)
  }
}

// 切换漫游开关
async function handleRoamingToggle() {
  roamingLoading.value = true
  try {
    const res = await setRoamingAllowed(roamingAllowed.value)
    if (res.status === 'ok') {
      if (res.data) {
        isRoaming.value = res.data.is_roaming
      }
      success(roamingAllowed.value ? '漫游已开启' : '漫游已关闭')
    } else {
      roamingAllowed.value = !roamingAllowed.value
      error(res.message || '操作失败')
    }
  } catch (err) {
    roamingAllowed.value = !roamingAllowed.value
    error('切换失败: ' + err.message)
  } finally {
    roamingLoading.value = false
  }
}

// 初始化获取状态
onMounted(() => {
  fetchDataStatus()
  fetchRoamingStatus()
})

</script>

<template>
  <div class="space-y-6">
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
      <!-- 飞行模式 & 网络模式 -->
      <div class="rounded-2xl bg-white/95 dark:bg-white/5 backdrop-blur border border-slate-200/60 dark:border-white/10 p-6 shadow-lg shadow-slate-200/40 dark:shadow-black/20 hover:shadow-xl hover:shadow-slate-300/50 dark:hover:shadow-black/30 transition-all duration-300">
        <h3 class="text-slate-900 dark:text-white font-semibold mb-6 flex items-center">
          <i class="fas fa-sliders-h text-cyan-400 mr-2"></i>
          网络设置
        </h3>
        
        <!-- 飞行模式 -->
        <div class="flex items-center justify-between p-4 bg-slate-100 dark:bg-white/5 rounded-xl mb-3">
          <div class="flex items-center space-x-3">
            <div class="w-10 h-10 rounded-lg bg-blue-500/20 flex items-center justify-center">
              <i class="fas fa-plane text-blue-400"></i>
            </div>
            <div>
              <p class="text-slate-900 dark:text-white font-medium">飞行模式</p>
              <p class="text-slate-500 dark:text-white/50 text-sm">关闭所有无线连接</p>
            </div>
          </div>
          <label class="relative cursor-pointer">
            <input type="checkbox" v-model="airplaneMode" @change="handleAirplaneToggle" class="sr-only peer">
            <div class="w-14 h-7 bg-slate-300 dark:bg-white/20 rounded-full peer peer-checked:bg-blue-500 transition-colors"></div>
            <div class="absolute top-0.5 left-0.5 w-6 h-6 bg-white rounded-full shadow transition-transform peer-checked:translate-x-7"></div>
          </label>
        </div>

        <!-- 数据连接 -->
        <div class="flex items-center justify-between p-4 bg-slate-100 dark:bg-white/5 rounded-xl mb-3">
          <div class="flex items-center space-x-3">
            <div class="w-10 h-10 rounded-lg bg-green-500/20 flex items-center justify-center">
              <i class="fas fa-signal text-green-400"></i>
            </div>
            <div>
              <p class="text-slate-900 dark:text-white font-medium">数据连接</p>
              <p class="text-slate-500 dark:text-white/50 text-sm">开启/关闭移动数据</p>
            </div>
          </div>
          <label class="relative cursor-pointer" :class="{ 'opacity-50 pointer-events-none': dataLoading }">
            <input type="checkbox" v-model="dataActive" @change="handleDataToggle" class="sr-only peer" :disabled="dataLoading">
            <div class="w-14 h-7 bg-slate-300 dark:bg-white/20 rounded-full peer peer-checked:bg-green-500 transition-colors"></div>
            <div class="absolute top-0.5 left-0.5 w-6 h-6 bg-white rounded-full shadow transition-transform peer-checked:translate-x-7"></div>
          </label>
        </div>

        <!-- 数据漫游 -->
        <div class="flex items-center justify-between p-4 bg-slate-100 dark:bg-white/5 rounded-xl mb-4">
          <div class="flex items-center space-x-3">
            <div class="w-10 h-10 rounded-lg bg-orange-500/20 flex items-center justify-center">
              <i class="fas fa-globe text-orange-400"></i>
            </div>
            <div>
              <p class="text-slate-900 dark:text-white font-medium">数据漫游</p>
              <p class="text-slate-500 dark:text-white/50 text-sm">
                {{ roamingAllowed ? '漫游开关: 已开启' : '漫游开关: 已关闭' }}
              </p>
            </div>
          </div>
          <div class="flex items-center space-x-2">
            <span v-if="isRoaming" class="px-2 py-0.5 bg-orange-500/20 text-orange-400 text-xs rounded-full">漫游中</span>
            <label class="relative cursor-pointer" :class="{ 'opacity-50 pointer-events-none': roamingLoading }">
              <input type="checkbox" v-model="roamingAllowed" @change="handleRoamingToggle" class="sr-only peer" :disabled="roamingLoading">
              <div class="w-14 h-7 bg-slate-300 dark:bg-white/20 rounded-full peer peer-checked:bg-orange-500 transition-colors"></div>
              <div class="absolute top-0.5 left-0.5 w-6 h-6 bg-white rounded-full shadow transition-transform peer-checked:translate-x-7"></div>
            </label>
          </div>
        </div>

        <!-- 网络模式选择 -->
        <p class="text-slate-600 dark:text-white/60 text-sm mb-3">网络模式</p>
        <div class="grid grid-cols-2 gap-3">
          <button
            v-for="mode in networkModes"
            :key="mode.value"
            @click="handleNetworkModeChange(mode.value); selectedNetworkMode = mode.value"
            class="p-4 rounded-xl border transition-all duration-300"
            :class="selectedNetworkMode === mode.value 
              ? 'bg-gradient-to-br from-cyan-500/20 to-blue-500/20 border-cyan-500/50' 
              : 'bg-slate-100 dark:bg-white/5 border-slate-200 dark:border-white/10 hover:border-slate-400 dark:hover:border-white/30'"
          >
            <div class="text-left">
              <p class="text-slate-900 dark:text-white font-medium">{{ mode.label }}</p>
              <p class="text-slate-500 dark:text-white/50 text-xs mt-1">{{ mode.desc }}</p>
            </div>
          </button>
        </div>
      </div>

      <!-- SIM卡管理 -->
      <div class="rounded-2xl bg-white/95 dark:bg-white/5 backdrop-blur border border-slate-200/60 dark:border-white/10 p-6 shadow-lg shadow-slate-200/40 dark:shadow-black/20 hover:shadow-xl hover:shadow-slate-300/50 dark:hover:shadow-black/30 transition-all duration-300">
        <h3 class="text-slate-900 dark:text-white font-semibold mb-6 flex items-center">
          <i class="fas fa-sim-card text-purple-400 mr-2"></i>
          SIM卡管理
        </h3>
        
        <div class="space-y-4">
          <!-- SIM卡槽1 -->
          <div 
            class="p-4 rounded-xl border transition-all cursor-pointer"
            :class="currentSlot === 'slot1' 
              ? 'bg-gradient-to-r from-green-500/20 to-emerald-500/20 border-green-500/50' 
              : 'bg-slate-100 dark:bg-white/5 border-slate-200 dark:border-white/10 hover:border-slate-400 dark:hover:border-white/30'"
            @click="handleSlotSwitch('slot1')"
          >
            <div class="flex items-center justify-between">
              <div class="flex items-center space-x-3">
                <div class="w-12 h-12 rounded-xl flex items-center justify-center"
                  :class="currentSlot === 'slot1' ? 'bg-green-500/30' : 'bg-slate-200 dark:bg-white/10'">
                  <i class="fas fa-sim-card text-xl" :class="currentSlot === 'slot1' ? 'text-green-400' : 'text-slate-400 dark:text-white/40'"></i>
                </div>
                <div>
                  <p class="text-slate-900 dark:text-white font-medium">SIM卡槽 1</p>
                  <p class="text-slate-500 dark:text-white/50 text-sm">{{ currentSlot === 'slot1' ? '当前使用中' : '点击切换' }}</p>
                </div>
              </div>
              <div v-if="currentSlot === 'slot1'" class="flex items-center space-x-2">
                <span class="px-2 py-1 bg-green-500/20 text-green-400 text-xs rounded-full">已激活</span>
                <i class="fas fa-check-circle text-green-400"></i>
              </div>
            </div>
            <div v-if="currentSlot === 'slot1'" class="mt-3 pt-3 border-t border-slate-200 dark:border-white/10">
              <p class="text-slate-500 dark:text-white/50 text-sm">信号强度: <span class="text-green-600 dark:text-green-400 font-medium">{{ systemInfo?.signal_strength || 'N/A' }}</span></p>
            </div>
          </div>

          <!-- SIM卡槽2 -->
          <div 
            class="p-4 rounded-xl border transition-all cursor-pointer"
            :class="currentSlot === 'slot2' 
              ? 'bg-gradient-to-r from-green-500/20 to-emerald-500/20 border-green-500/50' 
              : 'bg-slate-100 dark:bg-white/5 border-slate-200 dark:border-white/10 hover:border-slate-400 dark:hover:border-white/30'"
            @click="handleSlotSwitch('slot2')"
          >
            <div class="flex items-center justify-between">
              <div class="flex items-center space-x-3">
                <div class="w-12 h-12 rounded-xl flex items-center justify-center"
                  :class="currentSlot === 'slot2' ? 'bg-green-500/30' : 'bg-slate-200 dark:bg-white/10'">
                  <i class="fas fa-sim-card text-xl" :class="currentSlot === 'slot2' ? 'text-green-400' : 'text-slate-400 dark:text-white/40'"></i>
                </div>
                <div>
                  <p class="text-slate-900 dark:text-white font-medium">SIM卡槽 2</p>
                  <p class="text-slate-500 dark:text-white/50 text-sm">{{ currentSlot === 'slot2' ? '当前使用中' : '点击切换' }}</p>
                </div>
              </div>
              <div v-if="currentSlot === 'slot2'" class="flex items-center space-x-2">
                <span class="px-2 py-1 bg-green-500/20 text-green-400 text-xs rounded-full">已激活</span>
                <i class="fas fa-check-circle text-green-400"></i>
              </div>
            </div>
            <div v-if="currentSlot === 'slot2'" class="mt-3 pt-3 border-t border-slate-200 dark:border-white/10">
              <p class="text-slate-500 dark:text-white/50 text-sm">信号强度: <span class="text-green-600 dark:text-green-400 font-medium">{{ systemInfo?.signal_strength || 'N/A' }}</span></p>
            </div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>
