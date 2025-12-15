<script setup>
import { ref, onMounted, computed } from 'vue'
import { getApnList, setApnConfig } from '../composables/useApi'
import { useToast } from '../composables/useToast'
import { useConfirm } from '../composables/useConfirm'

const { success, error } = useToast()
const { confirm } = useConfirm()

const loading = ref(false)
const saving = ref(false)
const contexts = ref([])
const selectedContext = ref(null)

// 表单数据
const form = ref({
  apn: '',
  protocol: 'dual',
  username: '',
  password: '',
  auth_method: 'chap'
})

// 协议选项
const protocolOptions = [
  { value: 'ip', label: 'IPv4' },
  { value: 'ipv6', label: 'IPv6' },
  { value: 'dual', label: 'IPv4/IPv6 双栈' }
]

// 认证方式选项
const authOptions = [
  { value: 'none', label: '无认证' },
  { value: 'pap', label: 'PAP' },
  { value: 'chap', label: 'CHAP' }
]

// 常用运营商配置
const presets = [
  { name: '中国移动', apn: 'cmnet', icon: 'mobile-alt', color: 'from-blue-500 to-cyan-500' },
  { name: '中国联通', apn: '3gnet', icon: 'signal', color: 'from-red-500 to-orange-500' },
  { name: '中国电信', apn: 'ctnet', icon: 'broadcast-tower', color: 'from-green-500 to-emerald-500' }
]

// 当前激活的 context
const activeContext = computed(() => contexts.value.find(c => c.active))

// 获取 APN 列表
async function fetchApnList() {
  loading.value = true
  try {
    const res = await getApnList()
    if (res.status === 'ok' && res.data?.contexts) {
      contexts.value = res.data.contexts
      // 默认选择第一个或激活的
      if (contexts.value.length > 0) {
        const active = contexts.value.find(c => c.active) || contexts.value[0]
        selectContext(active)
      }
    }
  } catch (err) {
    error('获取APN配置失败: ' + err.message)
  } finally {
    loading.value = false
  }
}

// 选择 context
function selectContext(ctx) {
  selectedContext.value = ctx
  form.value = {
    apn: ctx.apn || '',
    protocol: ctx.protocol || 'dual',
    username: ctx.username || '',
    password: ctx.password || '',
    auth_method: ctx.auth_method || 'chap'
  }
}

// 应用预设配置
function applyPreset(preset) {
  form.value.apn = preset.apn
  form.value.protocol = 'dual'
  form.value.auth_method = 'chap'
  form.value.username = ''
  form.value.password = ''
}

// 保存配置
async function saveConfig() {
  if (!selectedContext.value) {
    error('请先选择一个APN配置')
    return
  }

  const confirmed = await confirm({
    title: '保存APN配置',
    message: `确定要将APN修改为 "${form.value.apn}" 吗？修改后可能需要重新连接网络。`
  })
  if (!confirmed) return

  saving.value = true
  try {
    const res = await setApnConfig({
      context_path: selectedContext.value.path,
      apn: form.value.apn,
      protocol: form.value.protocol,
      username: form.value.username,
      password: form.value.password,
      auth_method: form.value.auth_method
    })
    if (res.status === 'ok') {
      success('APN配置已保存')
      await fetchApnList()
    } else {
      throw new Error(res.message || '保存失败')
    }
  } catch (err) {
    error('保存失败: ' + err.message)
  } finally {
    saving.value = false
  }
}

onMounted(() => {
  fetchApnList()
})
</script>

<template>
  <div class="space-y-6">
    <!-- 标题卡片 -->
    <div class="relative overflow-hidden rounded-3xl bg-gradient-to-br from-teal-50/80 via-cyan-50/60 to-blue-50/40 dark:from-teal-600/20 dark:via-cyan-600/20 dark:to-blue-600/20 border border-slate-200/60 dark:border-white/10 p-6 shadow-xl">
      <div class="absolute top-0 right-0 w-64 h-64 bg-teal-500/10 rounded-full blur-3xl -translate-y-1/2 translate-x-1/2"></div>
      <div class="relative flex flex-col sm:flex-row sm:items-center sm:justify-between gap-4">
        <div class="flex items-center space-x-4">
          <div class="w-14 h-14 rounded-2xl bg-gradient-to-br from-teal-500 to-cyan-500 flex items-center justify-center shadow-lg shadow-teal-500/30">
            <font-awesome-icon icon="globe" class="text-white text-2xl" />
          </div>
          <div>
            <h2 class="text-slate-800 dark:text-white font-bold text-xl">APN配置</h2>
            <p class="text-slate-600 dark:text-white/50 text-sm mt-1">配置移动网络接入点</p>
          </div>
        </div>
        <!-- 当前状态 -->
        <div v-if="activeContext" class="flex items-center space-x-3 px-4 py-2 rounded-xl bg-white/60 dark:bg-white/10 border border-slate-200/60 dark:border-white/10">
          <div class="w-3 h-3 rounded-full bg-green-500 animate-pulse"></div>
          <div class="text-sm">
            <span class="text-slate-500 dark:text-white/50">当前APN：</span>
            <span class="font-semibold text-slate-800 dark:text-white">{{ activeContext.apn || '未配置' }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 快捷配置 -->
    <div class="grid grid-cols-1 sm:grid-cols-3 gap-4">
      <button
        v-for="preset in presets"
        :key="preset.apn"
        @click="applyPreset(preset)"
        class="group relative overflow-hidden rounded-2xl bg-white/95 dark:bg-white/5 backdrop-blur-xl border border-slate-200/60 dark:border-white/10 p-4 shadow-lg hover:shadow-xl transition-all duration-300 hover:-translate-y-0.5"
      >
        <div :class="`absolute top-0 right-0 w-20 h-20 bg-gradient-to-br ${preset.color} opacity-10 rounded-full blur-xl -translate-y-1/2 translate-x-1/2 group-hover:scale-150 transition-transform duration-500`"></div>
        <div class="relative flex items-center space-x-3">
          <div :class="`w-10 h-10 rounded-xl bg-gradient-to-br ${preset.color} flex items-center justify-center shadow-lg`">
            <font-awesome-icon :icon="preset.icon" class="text-white" />
          </div>
          <div class="text-left">
            <div class="font-medium text-slate-800 dark:text-white text-sm">{{ preset.name }}</div>
            <div class="text-xs text-slate-500 dark:text-white/50">{{ preset.apn }}</div>
          </div>
        </div>
      </button>
    </div>

    <!-- 配置表单 -->
    <div class="rounded-3xl bg-white/95 dark:bg-white/5 backdrop-blur-xl border border-slate-200/60 dark:border-white/10 shadow-xl overflow-hidden">
      <div class="p-6 space-y-5">
        <!-- APN 名称 -->
        <div>
          <label class="block text-sm font-medium text-slate-700 dark:text-white/80 mb-2">
            <font-awesome-icon icon="tag" class="mr-2 text-teal-500" />APN 名称
          </label>
          <input
            v-model="form.apn"
            type="text"
            placeholder="例如: cmnet, 3gnet, ctnet"
            class="w-full px-4 py-3 rounded-xl bg-slate-100 dark:bg-white/10 border border-slate-200 dark:border-white/10 text-slate-800 dark:text-white placeholder-slate-400 dark:placeholder-white/30 focus:outline-none focus:ring-2 focus:ring-teal-500/50 transition-all"
          />
        </div>

        <!-- 协议 -->
        <div>
          <label class="block text-sm font-medium text-slate-700 dark:text-white/80 mb-2">
            <font-awesome-icon icon="network-wired" class="mr-2 text-teal-500" />协议类型
          </label>
          <div class="grid grid-cols-3 gap-3">
            <button
              v-for="opt in protocolOptions"
              :key="opt.value"
              @click="form.protocol = opt.value"
              :class="[
                'py-3 px-4 rounded-xl text-sm font-medium transition-all duration-300',
                form.protocol === opt.value
                  ? 'bg-gradient-to-r from-teal-500 to-cyan-500 text-white shadow-lg shadow-teal-500/30'
                  : 'bg-slate-100 dark:bg-white/10 text-slate-600 dark:text-white/60 hover:bg-slate-200 dark:hover:bg-white/20'
              ]"
            >
              {{ opt.label }}
            </button>
          </div>
        </div>

        <!-- 用户名和密码 -->
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
          <div>
            <label class="block text-sm font-medium text-slate-700 dark:text-white/80 mb-2">
              <font-awesome-icon icon="user" class="mr-2 text-teal-500" />用户名
            </label>
            <input
              v-model="form.username"
              type="text"
              placeholder="可选"
              class="w-full px-4 py-3 rounded-xl bg-slate-100 dark:bg-white/10 border border-slate-200 dark:border-white/10 text-slate-800 dark:text-white placeholder-slate-400 dark:placeholder-white/30 focus:outline-none focus:ring-2 focus:ring-teal-500/50 transition-all"
            />
          </div>
          <div>
            <label class="block text-sm font-medium text-slate-700 dark:text-white/80 mb-2">
              <font-awesome-icon icon="lock" class="mr-2 text-teal-500" />密码
            </label>
            <input
              v-model="form.password"
              type="password"
              placeholder="可选"
              class="w-full px-4 py-3 rounded-xl bg-slate-100 dark:bg-white/10 border border-slate-200 dark:border-white/10 text-slate-800 dark:text-white placeholder-slate-400 dark:placeholder-white/30 focus:outline-none focus:ring-2 focus:ring-teal-500/50 transition-all"
            />
          </div>
        </div>

        <!-- 认证方式 -->
        <div>
          <label class="block text-sm font-medium text-slate-700 dark:text-white/80 mb-2">
            <font-awesome-icon icon="shield-alt" class="mr-2 text-teal-500" />认证方式
          </label>
          <div class="grid grid-cols-3 gap-3">
            <button
              v-for="opt in authOptions"
              :key="opt.value"
              @click="form.auth_method = opt.value"
              :class="[
                'py-3 px-4 rounded-xl text-sm font-medium transition-all duration-300',
                form.auth_method === opt.value
                  ? 'bg-gradient-to-r from-teal-500 to-cyan-500 text-white shadow-lg shadow-teal-500/30'
                  : 'bg-slate-100 dark:bg-white/10 text-slate-600 dark:text-white/60 hover:bg-slate-200 dark:hover:bg-white/20'
              ]"
            >
              {{ opt.label }}
            </button>
          </div>
        </div>

        <!-- 保存按钮 -->
        <button
          @click="saveConfig"
          :disabled="saving || !form.apn"
          class="w-full py-4 px-6 rounded-xl bg-gradient-to-r from-teal-500 to-cyan-500 text-white font-medium shadow-lg shadow-teal-500/30 hover:shadow-xl hover:scale-[1.02] active:scale-[0.98] transition-all duration-300 disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center space-x-2"
        >
          <font-awesome-icon v-if="saving" icon="spinner" spin />
          <font-awesome-icon v-else icon="save" />
          <span>{{ saving ? '保存中...' : '保存配置' }}</span>
        </button>
      </div>
    </div>

    <!-- 提示信息 -->
    <div class="rounded-2xl bg-blue-50 dark:bg-blue-500/10 border border-blue-200 dark:border-blue-500/20 p-4">
      <div class="flex items-start space-x-3">
        <font-awesome-icon icon="info-circle" class="text-blue-500 mt-0.5" />
        <div class="text-sm text-blue-800 dark:text-blue-200">
          <p class="font-medium mb-1">使用说明</p>
          <ul class="list-disc list-inside space-y-1 text-blue-700 dark:text-blue-300/80">
            <li>点击运营商快捷按钮可快速填充常用APN配置</li>
            <li>大多数情况下只需填写APN名称，用户名和密码可留空</li>
            <li>修改配置后可能需要重新连接网络才能生效</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>
