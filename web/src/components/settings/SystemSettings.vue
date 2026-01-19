<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import {
  NCard,
  NSpace,
  NButton,
  NInputNumber,
  NFormItem,
  NInput,
  NTag,
  NAlert,
  NDivider,
  NIcon,
  NCollapse,
  NCollapseItem,
  useMessage,
} from 'naive-ui'
import {
  CheckmarkCircleOutline,
  CloseCircleOutline,
  AlertCircleOutline,
  KeyOutline,
  OpenOutline,
  RefreshOutline,
  TrashOutline,
  TimeOutline,
} from '@vicons/ionicons5'
import { configApi } from '@/api/config'
import type { ApiTokenStatus } from '@/api/types'
import TmdbSetupWizard from './TmdbSetupWizard.vue'

const message = useMessage()

// 系统配置
const loading = ref(false)
const saving = ref(false)
const scrapeThreads = ref(4)
const taskTimeout = ref(30)
const retryCount = ref(3)
const concurrentDownloads = ref(3)

// TMDB API Token
const tokenLoading = ref(false)
const tokenSaving = ref(false)
const tokenInput = ref('')
const tokenStatus = ref<ApiTokenStatus | null>(null)
const showWizard = ref(false)
const showAdvanced = ref(false)

// Token 状态配置
const statusConfig = computed(() => {
  if (!tokenStatus.value) {
    return { type: 'default' as const, icon: AlertCircleOutline, text: '加载中...', color: '#999' }
  }
  if (tokenStatus.value.is_configured && tokenStatus.value.is_valid) {
    return { type: 'success' as const, icon: CheckmarkCircleOutline, text: '已配置且有效', color: '#34C759' }
  }
  if (tokenStatus.value.is_configured && tokenStatus.value.is_valid === false) {
    return { type: 'error' as const, icon: CloseCircleOutline, text: '配置无效', color: '#FF3B30' }
  }
  if (tokenStatus.value.is_configured) {
    return { type: 'warning' as const, icon: AlertCircleOutline, text: '已配置但未验证', color: '#FF9500' }
  }
  return { type: 'default' as const, icon: KeyOutline, text: '未配置', color: '#8E8E93' }
})

// 格式化时间
const formatVerifyTime = (time: string | null) => {
  if (!time) return null
  const date = new Date(time)
  return date.toLocaleString('zh-CN', {
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  })
}

// 加载系统配置
const loadConfig = async () => {
  loading.value = true
  try {
    const config = await configApi.getSystemConfig()
    scrapeThreads.value = config.scrape_threads
    taskTimeout.value = config.task_timeout
    retryCount.value = config.retry_count
    concurrentDownloads.value = config.concurrent_downloads
  } catch (error) {
    console.error(error)
  } finally {
    loading.value = false
  }
}

// 保存系统配置
const saveConfig = async () => {
  saving.value = true
  try {
    await configApi.saveSystemConfig({
      scrape_threads: scrapeThreads.value,
      task_timeout: taskTimeout.value,
      retry_count: retryCount.value,
      concurrent_downloads: concurrentDownloads.value,
    })
    message.success('系统配置已保存')
  } catch (error) {
    message.error('保存失败')
    console.error(error)
  } finally {
    saving.value = false
  }
}

// 加载 Token 状态
const loadTokenStatus = async () => {
  tokenLoading.value = true
  try {
    tokenStatus.value = await configApi.getApiTokenStatus()
  } catch (error) {
    console.error(error)
  } finally {
    tokenLoading.value = false
  }
}

// 保存 Token
const saveToken = async () => {
  if (!tokenInput.value.trim()) {
    message.warning('请输入 API Token')
    return
  }
  tokenSaving.value = true
  try {
    const response = await configApi.saveApiToken(tokenInput.value.trim())
    if (response.success) {
      message.success(response.message)
      tokenStatus.value = response.status
      tokenInput.value = ''
      showAdvanced.value = false
    } else {
      message.error(response.message)
    }
  } catch (error) {
    message.error('保存失败')
    console.error(error)
  } finally {
    tokenSaving.value = false
  }
}

// 重新验证 Token
const revalidateToken = async () => {
  if (!tokenStatus.value?.is_configured) return

  tokenLoading.value = true
  try {
    // 重新获取状态会触发后端验证
    tokenStatus.value = await configApi.getApiTokenStatus()
    if (tokenStatus.value.is_valid) {
      message.success('Token 验证通过')
    } else {
      message.error(tokenStatus.value.error_message || 'Token 验证失败')
    }
  } catch (error) {
    message.error('验证失败')
    console.error(error)
  } finally {
    tokenLoading.value = false
  }
}

// 删除 Token
const deleteToken = async () => {
  try {
    const response = await configApi.deleteApiToken()
    if (response.success) {
      message.success(response.message)
      tokenStatus.value = { is_configured: false, is_valid: null, last_verified: null, error_message: null }
    } else {
      message.warning(response.message)
    }
  } catch (error) {
    message.error('删除失败')
    console.error(error)
  }
}

// 向导成功回调
const handleWizardSuccess = () => {
  loadTokenStatus()
}

// 打开外部链接
const openExternal = (url: string) => {
  window.open(url, '_blank', 'noopener,noreferrer')
}

onMounted(() => {
  loadConfig()
  loadTokenStatus()
})
</script>

<template>
  <NCard title="系统设置" size="small">
    <NSpace vertical size="large">
      <!-- TMDB 认证 -->
      <NDivider title-placement="left">TMDB 认证</NDivider>

      <!-- 状态卡片 -->
      <div class="tmdb-status-card" :class="statusConfig.type">
        <div class="status-header">
          <div class="status-icon">
            <NIcon :component="statusConfig.icon" :size="24" :color="statusConfig.color" />
          </div>
          <div class="status-info">
            <div class="status-title">
              <NTag :type="statusConfig.type" size="small" round>
                {{ statusConfig.text }}
              </NTag>
            </div>
            <div v-if="tokenStatus?.last_verified" class="status-meta">
              <NIcon :component="TimeOutline" :size="14" />
              <span>上次验证: {{ formatVerifyTime(tokenStatus.last_verified) }}</span>
            </div>
            <div v-if="tokenStatus?.error_message" class="status-error">
              {{ tokenStatus.error_message }}
            </div>
          </div>
        </div>

        <div class="status-actions">
          <template v-if="tokenStatus?.is_configured">
            <NButton
              size="small"
              quaternary
              :loading="tokenLoading"
              @click="revalidateToken"
            >
              <template #icon>
                <NIcon :component="RefreshOutline" />
              </template>
              重新验证
            </NButton>
            <NButton
              size="small"
              quaternary
              type="error"
              @click="deleteToken"
            >
              <template #icon>
                <NIcon :component="TrashOutline" />
              </template>
              删除
            </NButton>
          </template>
          <NButton
            type="primary"
            size="small"
            @click="showWizard = true"
          >
            <template #icon>
              <NIcon :component="KeyOutline" />
            </template>
            {{ tokenStatus?.is_configured ? '重新配置' : '配置向导' }}
          </NButton>
        </div>
      </div>

      <!-- 快速帮助 -->
      <NAlert type="info" class="help-alert">
        <template #icon>
          <NIcon :component="AlertCircleOutline" />
        </template>
        <div class="help-content">
          <span>TMDB API 用于获取剧集的元数据信息，您需要一个免费的 API Token。</span>
          <NButton
            text
            type="primary"
            size="small"
            @click="openExternal('https://www.themoviedb.org/settings/api')"
          >
            <template #icon>
              <NIcon :component="OpenOutline" />
            </template>
            前往 TMDB 获取
          </NButton>
        </div>
      </NAlert>

      <!-- 高级选项：手动输入 Token -->
      <NCollapse>
        <NCollapseItem title="手动输入 Token" name="manual">
          <template #header-extra>
            <span class="collapse-hint">如果向导无法使用，可以手动粘贴</span>
          </template>
          <NSpace vertical style="width: 100%">
            <NInput
              v-model:value="tokenInput"
              type="textarea"
              placeholder="粘贴 TMDB API Read Access Token (以 eyJ 开头)..."
              :rows="3"
              class="token-input"
            />
            <NSpace>
              <NButton
                type="primary"
                :loading="tokenSaving"
                :disabled="!tokenInput.trim()"
                @click="saveToken"
              >
                保存并验证
              </NButton>
            </NSpace>
          </NSpace>
        </NCollapseItem>
      </NCollapse>

      <!-- 性能设置 -->
      <NDivider title-placement="left">性能设置</NDivider>

      <div class="settings-grid">
        <NFormItem label="刮削任务线程数">
          <div class="setting-row">
            <NInputNumber v-model:value="scrapeThreads" :min="1" :max="16" style="width: 120px" />
            <span class="setting-hint">建议 2-8，过高可能导致请求被限制</span>
          </div>
        </NFormItem>

        <NFormItem label="任务超时设置 (秒)">
          <div class="setting-row">
            <NInputNumber v-model:value="taskTimeout" :min="10" :max="300" style="width: 120px" />
            <span class="setting-hint">适用于 TMDB 请求、图片下载等所有网络操作</span>
          </div>
        </NFormItem>

        <NFormItem label="失败重试次数">
          <div class="setting-row">
            <NInputNumber v-model:value="retryCount" :min="0" :max="10" style="width: 120px" />
            <span class="setting-hint">网络请求失败后的重试次数</span>
          </div>
        </NFormItem>

        <NFormItem label="并发下载数">
          <div class="setting-row">
            <NInputNumber v-model:value="concurrentDownloads" :min="1" :max="10" style="width: 120px" />
            <span class="setting-hint">同时下载图片的最大数量</span>
          </div>
        </NFormItem>
      </div>

      <NSpace>
        <NButton type="primary" :loading="saving" @click="saveConfig">
          保存配置
        </NButton>
      </NSpace>
    </NSpace>
  </NCard>

  <!-- TMDB 配置向导 -->
  <TmdbSetupWizard
    v-model:show="showWizard"
    :initial-status="tokenStatus"
    @success="handleWizardSuccess"
  />
</template>

<style scoped>
/* TMDB 状态卡片 */
.tmdb-status-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 16px 20px;
  border-radius: 14px;
  background: var(--ios-bg-secondary);
  border: 1px solid var(--ios-separator);
  transition: all 0.25s ease;
}

.tmdb-status-card.success {
  background: rgba(52, 199, 89, 0.08);
  border-color: rgba(52, 199, 89, 0.2);
}

.tmdb-status-card.error {
  background: rgba(255, 59, 48, 0.08);
  border-color: rgba(255, 59, 48, 0.2);
}

.tmdb-status-card.warning {
  background: rgba(255, 149, 0, 0.08);
  border-color: rgba(255, 149, 0, 0.2);
}

.status-header {
  display: flex;
  align-items: flex-start;
  gap: 14px;
}

.status-icon {
  width: 44px;
  height: 44px;
  border-radius: 12px;
  background: var(--ios-bg-tertiary);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.tmdb-status-card.success .status-icon {
  background: rgba(52, 199, 89, 0.15);
}

.tmdb-status-card.error .status-icon {
  background: rgba(255, 59, 48, 0.15);
}

.tmdb-status-card.warning .status-icon {
  background: rgba(255, 149, 0, 0.15);
}

.status-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.status-title {
  font-weight: 600;
}

.status-meta {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: var(--ios-text-tertiary);
}

.status-error {
  font-size: 12px;
  color: #FF3B30;
  margin-top: 2px;
}

.status-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

/* 帮助提示 */
.help-alert {
  border-radius: 12px;
}

.help-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  flex-wrap: wrap;
}

/* 折叠面板 */
.collapse-hint {
  font-size: 12px;
  color: var(--ios-text-tertiary);
}

.token-input :deep(.n-input__textarea-el) {
  font-family: 'SF Mono', Monaco, monospace;
  font-size: 12px;
}

/* 设置网格 */
.settings-grid {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.setting-row {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.setting-hint {
  font-size: 13px;
  color: var(--ios-text-tertiary);
}

/* 响应式 */
@media (max-width: 640px) {
  .tmdb-status-card {
    flex-direction: column;
    align-items: flex-start;
  }

  .status-actions {
    width: 100%;
    justify-content: flex-end;
    margin-top: 8px;
    padding-top: 12px;
    border-top: 1px solid var(--ios-separator);
  }

  .help-content {
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>
