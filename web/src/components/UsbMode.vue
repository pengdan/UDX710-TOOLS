<script setup>
import { ref, onMounted } from 'vue'
import { getUsbMode, setUsbMode, usbAdvanceSwitch, deviceControl } from '../composables/useApi'
import { useToast } from '../composables/useToast'
import { useConfirm } from '../composables/useConfirm'

const { success, error } = useToast()
const { confirm } = useConfirm()

const loading = ref(null) // 当前加载的按钮标识
const currentMode = ref(null) // 当前硬件模式
const currentModeName = ref('') // 当前模式名称
const isTemporary = ref(false) // 是否临时模式
const hotSwitching = ref(false) // 热切换中

// 模式值到名称的映射
const modeValueToName = {
  1: 'CDC-NCM',
  2: 'CDC-ECM', 
  3: 'RNDIS'
}

const modeIdToValue = {
  'cdc_ncm': 1,
  'cdc_ecm': 2,
  'rndis': 3
}

// 获取当前USB模式
async function fetchCurrentMode() {
  try {
    const res = await getUsbMode()
    if (res.Code === 0 && res.Data) {
      const modeVal = res.Data.mode_value
      currentMode.value = modeVal
      // 处理无效模式值的情况
      if (modeVal > 0 && modeValueToName[modeVal]) {
        currentModeName.value = modeValueToName[modeVal]
      } else {
        currentModeName.value = '未知'
      }
      isTemporary.value = res.Data.is_temporary || false
    } else {
      currentModeName.value = '未知'
    }
  } catch (err) {
    console.error('获取USB模式失败:', err)
    currentModeName.value = '获取失败'
  }
}

// 热切换USB模式
async function handleHotSwitch(mode) {
  const modeValue = modeIdToValue[mode.id]
  if (!modeValue || hotSwitching.value) return
  
  const confirmed = await confirm({
    title: '热切换确认',
    message: `即将热切换到 ${mode.name} 模式，USB连接会短暂断开（约2秒），确定继续吗？`
  })
  if (!confirmed) return
  
  hotSwitching.value = true
  loading.value = `${mode.id}_hot`
  
  try {
    const res = await usbAdvanceSwitch(modeValue)
    if (res.Code === 0) {
      success(`已热切换到 ${mode.name} 模式`)
      currentMode.value = modeValue
      currentModeName.value = mode.name
    } else {
      throw new Error(res.Error || '热切换失败')
    }
  } catch (err) {
    error('热切换失败: ' + err.message)
  } finally {
    hotSwitching.value = false
    loading.value = null
  }
}

onMounted(() => {
  fetchCurrentMode()
})

const modes = [
  {
    id: 'cdc_ecm',
    name: 'CDC-ECM',
    icon: 'network-wired',
    desc: '兼容性差，不支持绝大部分Windows设备，仅建议Android、MacOS和老旧Openwrt、Padavan路由器使用',
    gradient: 'from-blue-500 to-cyan-500',
    gradientLight: 'from-blue-50 to-cyan-50',
    gradientDark: 'from-blue-500/20 to-cyan-500/20',
    btnTemp: 'from-blue-500 to-blue-600',
    btnPerm: 'from-blue-700 to-blue-800',
    shadow: 'shadow-blue-500/30',
    text: 'text-blue-500'
  },
  {
    id: 'cdc_ncm',
    name: 'CDC-NCM',
    icon: 'broadcast-tower',
    desc: '仅测试，勿使用，兼容性差，不支持绝大部分Windows设备（DHCP可能异常）',
    gradient: 'from-red-500 to-orange-500',
    gradientLight: 'from-red-50 to-orange-50',
    gradientDark: 'from-red-500/20 to-orange-500/20',
    btnTemp: 'from-red-500 to-red-600',
    btnPerm: 'from-red-700 to-red-800',
    shadow: 'shadow-red-500/30',
    text: 'text-red-500'
  },
  {
    id: 'rndis',
    name: 'RNDIS',
    icon: ['fab', 'usb'],
    desc: '兼容性好，支持Windows、Openwrt、Padavan路由器，不支持绝大部分Android和MacOS',
    gradient: 'from-green-500 to-emerald-500',
    gradientLight: 'from-green-50 to-emerald-50',
    gradientDark: 'from-green-500/20 to-emerald-500/20',
    btnTemp: 'from-green-500 to-green-600',
    btnPerm: 'from-green-700 to-green-800',
    shadow: 'shadow-green-500/30',
    text: 'text-green-500'
  }
]

async function handleSwitch(mode, permanent) {
  const btnKey = `${mode.id}_${permanent ? 'perm' : 'temp'}`
  if (loading.value) return

  // 切换确认
  const confirmTitle = permanent ? '永久模式变更' : '临时模式变更'
  const confirmMsg = permanent 
    ? `警告：这将永久更改为${mode.name}模式，重启后永久生效，确定要继续吗？`
    : `确定要临时切换到${mode.name}模式吗？重启后生效，再次重启将恢复默认。`
  
  const confirmed = await confirm({ title: confirmTitle, message: confirmMsg })
  if (!confirmed) return

  loading.value = btnKey
  try {
    const res = await setUsbMode(mode.id, permanent)
    if (res.Code === 0) {
      success(`${permanent ? '永久' : '临时'}切换到${mode.name}模式成功`)
      
      // 重新获取当前模式状态
      await fetchCurrentMode()
      
      // 询问是否重启
      const reboot = await confirm({
        title: '重启设备',
        message: permanent ? '必须重启才能生效，是否立即重启？' : '是否立即重启以应用变更？'
      })
      if (reboot) {
        await deviceControl('reboot')
        success('系统正在重启...')
      }
    } else {
      throw new Error(res.Error || '切换失败')
    }
  } catch (err) {
    error('切换失败: ' + err.message)
  } finally {
    loading.value = null
  }
}
</script>

<template>
  <div class="space-y-6">
    <!-- 标题卡片 + 当前模式显示 -->
    <div class="relative overflow-hidden rounded-3xl bg-gradient-to-br from-indigo-50/80 via-purple-50/60 to-pink-50/40 dark:from-indigo-600/20 dark:via-purple-600/20 dark:to-pink-600/20 border border-slate-200/60 dark:border-white/10 p-6 shadow-xl">
      <div class="absolute top-0 right-0 w-64 h-64 bg-purple-500/10 rounded-full blur-3xl -translate-y-1/2 translate-x-1/2"></div>
      <div class="relative flex flex-col sm:flex-row sm:items-center sm:justify-between gap-4">
        <div class="flex items-center space-x-4">
          <div class="w-14 h-14 rounded-2xl bg-gradient-to-br from-purple-500 to-pink-500 flex items-center justify-center shadow-lg shadow-purple-500/30">
            <font-awesome-icon :icon="['fab', 'usb']" class="text-white text-2xl" />
          </div>
          <div>
            <h2 class="text-slate-800 dark:text-white font-bold text-xl">USB模式切换</h2>
            <p class="text-slate-600 dark:text-white/50 text-sm mt-1">选择适合您设备的USB网络模式</p>
          </div>
        </div>
        <!-- 当前模式状态 -->
        <div v-if="currentModeName" class="flex items-center space-x-3 px-4 py-2 rounded-xl bg-white/60 dark:bg-white/10 border border-slate-200/60 dark:border-white/10">
          <div class="w-3 h-3 rounded-full bg-green-500 animate-pulse"></div>
          <div class="text-sm">
            <span class="text-slate-500 dark:text-white/50">当前模式：</span>
            <span class="font-semibold text-slate-800 dark:text-white">{{ currentModeName }}</span>
            <span v-if="isTemporary" class="ml-2 px-2 py-0.5 text-xs bg-amber-100 dark:bg-amber-500/20 text-amber-600 dark:text-amber-400 rounded-full">临时</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 模式卡片网格 -->
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-6">
      <div 
        v-for="mode in modes" 
        :key="mode.id"
        class="group relative overflow-hidden rounded-3xl bg-white/95 dark:bg-white/5 backdrop-blur-xl border border-slate-200/60 dark:border-white/10 shadow-xl hover:shadow-2xl transition-all duration-500 hover:-translate-y-1"
      >
        <!-- 背景装饰 -->
        <div :class="`absolute top-0 right-0 w-32 h-32 bg-gradient-to-br ${mode.gradient} opacity-10 rounded-full blur-2xl -translate-y-1/2 translate-x-1/2 group-hover:scale-150 transition-transform duration-500`"></div>
        
        <div class="relative p-5 md:p-6">
          <!-- 头部：图标和标题 -->
          <div class="flex items-center space-x-4 mb-4">
            <div :class="`w-12 h-12 md:w-14 md:h-14 rounded-2xl bg-gradient-to-br ${mode.gradient} flex items-center justify-center shadow-lg ${mode.shadow}`">
              <font-awesome-icon :icon="mode.icon" class="text-white text-lg md:text-xl" />
            </div>
            <div>
              <h3 class="text-slate-800 dark:text-white font-bold text-lg">{{ mode.name }}</h3>
              <span :class="`text-xs font-medium ${mode.text}`">USB网络模式</span>
            </div>
          </div>
          
          <!-- 描述 -->
          <p class="text-slate-600 dark:text-white/60 text-sm leading-relaxed mb-5 min-h-[60px]">
            {{ mode.desc }}
          </p>
          
          <!-- 按钮组 -->
          <div class="space-y-3">
            <!-- 热切换按钮（立即生效） -->
            <button
              @click="handleHotSwitch(mode)"
              :disabled="loading !== null || currentMode === modeIdToValue[mode.id]"
              :class="`w-full py-3 px-4 rounded-xl bg-gradient-to-r from-violet-500 to-purple-600 text-white font-medium text-sm shadow-lg shadow-violet-500/30 hover:shadow-xl hover:scale-[1.02] active:scale-[0.98] transition-all duration-300 disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center space-x-2`"
            >
              <font-awesome-icon v-if="loading === `${mode.id}_hot`" icon="spinner" spin />
              <font-awesome-icon v-else icon="bolt" />
              <span>热切换（立即生效）</span>
            </button>
            
            <!-- 临时切换按钮 -->
            <button
              @click="handleSwitch(mode, false)"
              :disabled="loading !== null"
              :class="`w-full py-3 px-4 rounded-xl bg-gradient-to-r ${mode.btnTemp} text-white font-medium text-sm shadow-lg ${mode.shadow} hover:shadow-xl hover:scale-[1.02] active:scale-[0.98] transition-all duration-300 disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center space-x-2`"
            >
              <font-awesome-icon v-if="loading === `${mode.id}_temp`" icon="spinner" spin />
              <span>临时模式（重启生效）</span>
            </button>
            
            <!-- 永久切换按钮 -->
            <button
              @click="handleSwitch(mode, true)"
              :disabled="loading !== null"
              :class="`w-full py-3 px-4 rounded-xl bg-gradient-to-r ${mode.btnPerm} text-white font-medium text-sm shadow-lg hover:shadow-xl hover:scale-[1.02] active:scale-[0.98] transition-all duration-300 disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center space-x-2`"
            >
              <font-awesome-icon v-if="loading === `${mode.id}_perm`" icon="spinner" spin />
              <span>永久模式（重启生效）</span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- 注意事项 -->
    <div class="rounded-2xl bg-amber-50 dark:bg-amber-500/10 border border-amber-200 dark:border-amber-500/20 p-4">
      <div class="flex items-start space-x-3">
        <font-awesome-icon icon="exclamation-triangle" class="text-amber-500 mt-0.5" />
        <div class="text-sm text-amber-800 dark:text-amber-200">
          <p class="font-medium mb-1">注意事项</p>
          <ul class="list-disc list-inside space-y-1 text-amber-700 dark:text-amber-300/80">
            <li><span class="font-medium text-violet-600 dark:text-violet-400">热切换</span>：立即生效，USB连接会短暂断开约2秒</li>
            <li><span class="font-medium">临时模式</span>：重启后生效，再次重启恢复默认</li>
            <li><span class="font-medium">永久模式</span>：重启后永久生效</li>
          </ul>
        </div>
      </div>
    </div>

    <!-- 免责声明 -->
    <div class="rounded-2xl bg-red-50 dark:bg-red-500/10 border border-red-200 dark:border-red-500/20 p-4">
      <div class="flex items-start space-x-3">
        <font-awesome-icon icon="shield-halved" class="text-red-500 mt-0.5" />
        <div class="text-sm text-red-800 dark:text-red-200">
          <p class="font-bold mb-2">⚠️ 免责声明</p>
          <ul class="list-disc list-inside space-y-1 text-red-700 dark:text-red-300/80">
            <li>部分设备可能不兼容某些USB模式，请根据自身设备情况谨慎选择</li>
            <li>切换模式可能导致设备无法正常联网，请确保了解恢复方法</li>
            <li>使用本功能即表示您已了解风险并自行承担后果</li>
            <li class="font-medium">本功能不提供任何售后支持，请自行解决使用中遇到的问题</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>
