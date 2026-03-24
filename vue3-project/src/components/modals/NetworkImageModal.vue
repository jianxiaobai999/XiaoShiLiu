<template>
  <div v-if="visible" class="network-image-modal-overlay" v-click-outside.mousedown="closeModal"
    v-escape-key="closeModal">
    <div class="network-image-modal" @mousedown.stop @wheel.stop>
      <div class="modal-header">
        <h4>添加网络图片</h4>
        <button @click="closeModal" class="close-btn">
          <SvgIcon name="close" width="16" height="16" />
        </button>
      </div>

      <div class="modal-body">
        <div class="input-section">
          <label class="input-label">请输入图片链接：</label>
          <textarea
            ref="urlInput"
            v-model="inputText"
            class="url-textarea"
            placeholder="请输入图片链接，每行一个，支持批量粘贴&#10;&#10;示例：&#10;https://example.com/image1.jpg&#10;https://example.com/image2.png"
            @dragover.prevent
            @drop.prevent="handleTextDrop"
            @input="handleInput"
            :disabled="isProcessing"
          ></textarea>
          <div class="input-hints">
            <p>• 每行一个图片链接</p>
            <p>• 支持 JPG、PNG、GIF、WebP 格式</p>
            <p>• 最多添加 {{ remainingCount }} 张图片</p>
            <p>• 可直接粘贴或拖拽包含链接的文本</p>
          </div>
        </div>

        <div v-if="invalidUrls.length > 0" class="invalid-section">
          <div class="invalid-header">
            <SvgIcon name="warning" class="warning-icon" />
            <span>以下链接格式无效（{{ invalidUrls.length }}个）：</span>
          </div>
          <div class="invalid-list">
            <div v-for="(url, index) in invalidUrls" :key="index" class="invalid-item">
              {{ url }}
            </div>
          </div>
        </div>

        <div v-if="error" class="error-message">
          {{ error }}
        </div>

        <div v-if="isProcessing" class="processing-state">
          <SvgIcon name="loading" class="loading-icon" />
          <p>正在验证链接...</p>
        </div>
      </div>

      <div class="modal-footer">
        <button @click="closeModal" class="cancel-btn" :disabled="isProcessing">
          取消
        </button>
        <button @click="handleConfirm" class="confirm-btn" :disabled="!canConfirm || isProcessing">
          确认添加{{ validUrls.length > 0 ? `(${validUrls.length}张)` : '' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, nextTick } from 'vue'
import SvgIcon from '@/components/SvgIcon.vue'
import { useScrollLock } from '@/composables/useScrollLock'

const props = defineProps({
  visible: {
    type: Boolean,
    default: false
  },
  existingUrls: {
    type: Array,
    default: () => []
  },
  maxCount: {
    type: Number,
    default: 9
  }
})

const emit = defineEmits(['close', 'confirm'])

const { lock, unlock } = useScrollLock()

// 状态管理
const inputText = ref('')
const error = ref('')
const isProcessing = ref(false)
const invalidUrls = ref([])
const validUrls = ref([])
const urlInput = ref(null)

// 防滚动穿透
watch(() => props.visible, (newValue) => {
  if (newValue) {
    lock()
    // 清空之前的状态
    inputText.value = ''
    error.value = ''
    invalidUrls.value = []
    validUrls.value = []
    nextTick(() => {
      urlInput.value?.focus()
    })
  } else {
    unlock()
  }
})

// 计算剩余可添加数量
const remainingCount = computed(() => {
  return props.maxCount
})

// 判断是否可以确认
const canConfirm = computed(() => {
  return validUrls.value.length > 0 && validUrls.value.length <= remainingCount.value
})

// URL 验证正则
const isValidImageUrl = (url) => {
  try {
    // 验证是否为有效的 URL
    new URL(url.trim())
    
    // 验证是否为图片链接（检查扩展名或查询参数）
    const imagePattern = /\.(jpeg|jpg|png|gif|webp|bmp|svg)(\?.*)?$/i
    return imagePattern.test(url.trim())
  } catch (_) {
    return false
  }
}

// 防抖定时器
let debounceTimer = null

// 处理输入变化（带防抖的自动验证）
const handleInput = () => {
  // 清除之前的定时器
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  
  // 设置新的定时器，延迟 300ms 验证
  debounceTimer = setTimeout(() => {
    processUrls()
  }, 300)
}

// 处理文本拖拽
const handleTextDrop = (event) => {
  const text = event.dataTransfer.getData('text/plain')
  if (text) {
    // 提取所有 URL（使用正则匹配）
    const urlRegex = /(https?:\/\/[^\s]+)/g
    const matches = text.match(urlRegex) || []
    
    if (matches.length > 0) {
      // 将提取到的 URL 填充到输入框
      inputText.value = matches.join('\n')
      // 立即触发验证
      handleInput()
    } else {
      // 如果没有提取到 URL，直接使用原始文本
      inputText.value = text
      // 立即触发验证
      handleInput()
    }
  }
}

// 验证并处理 URLs
const processUrls = async () => {
  isProcessing.value = true
  error.value = ''
  invalidUrls.value = []
  validUrls.value = []

  try {
    // 按行分割并过滤空行
    const lines = inputText.value.split('\n')
      .map(line => line.trim())
      .filter(line => line.length > 0)

    if (lines.length === 0) {
      error.value = '请输入至少一个图片链接'
      isProcessing.value = false
      return
    }

    // 去重
    const uniqueUrls = [...new Set(lines)]

    // 验证每个 URL
    const valid = []
    const invalid = []

    for (const url of uniqueUrls) {
      if (isValidImageUrl(url)) {
        // 检查是否已存在
        if (!props.existingUrls.includes(url)) {
          valid.push(url)
        }
        // 已存在的 URL 不加入任何列表（静默跳过）
      } else {
        invalid.push(url)
      }
    }

    // 检查数量限制
    if (valid.length > remainingCount.value) {
      error.value = `最多只能再添加${remainingCount.value}张图片，当前输入了${valid.length}张有效链接`
      validUrls.value = valid.slice(0, remainingCount.value)
      invalidUrls.value = invalid
    } else {
      validUrls.value = valid
      invalidUrls.value = invalid
    }

    // 如果有无效链接，显示警告
    if (invalid.length > 0) {
      error.value = `发现${invalid.length}个无效链接，已自动跳过`
    }

  } catch (err) {
    console.error('处理链接失败:', err)
    error.value = '处理链接失败：' + err.message
  } finally {
    isProcessing.value = false
  }
}

// 确认添加
const handleConfirm = async () => {
  if (!canConfirm.value || isProcessing.value) {
    return
  }

  // 先验证一次确保数据最新
  await processUrls()

  if (validUrls.value.length === 0) {
    error.value = '没有有效的图片链接可添加'
    return
  }

  // 发送确认事件
  emit('confirm', validUrls.value)
  
  // 清空输入并关闭
  inputText.value = ''
  closeModal()
}

// 关闭模态框
const closeModal = () => {
  // 清除防抖定时器
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  emit('close')
}
</script>

<style scoped>
.network-image-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: var(--overlay-bg);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
}

.network-image-modal {
  background: var(--bg-color-primary);
  border-radius: 12px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
  width: 100%;
  max-width: 600px;
  max-height: 80vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px;
  border-bottom: 1px solid var(--border-color-primary);
  background: var(--bg-color-primary);
}

.modal-header h4 {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
  color: var(--text-color-primary);
}

.close-btn {
  width: 30px;
  height: 30px;
  background: var(--bg-color-secondary);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  border: none;
  cursor: pointer;
  padding: 5px;
  color: var(--text-color-primary);
  transition: all 0.2s ease;
}

.close-btn:hover {
  color: var(--text-color-secondary);
  transform: scale(1.1);
  transition: all 0.2s ease;
}

.modal-body {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
  min-height: 0;
}

.input-section {
  margin-bottom: 20px;
}

.input-label {
  display: block;
  margin-bottom: 8px;
  font-size: 14px;
  font-weight: 500;
  color: var(--text-color-primary);
}

.url-textarea {
  width: 100%;
  min-height: 200px;
  padding: 12px;
  border: 1px solid var(--border-color-primary);
  border-radius: 8px;
  background: var(--bg-color-primary);
  color: var(--text-color-primary);
  font-size: 14px;
  font-family: 'Consolas', 'Monaco', monospace;
  line-height: 1.6;
  resize: vertical;
  transition: all 0.2s ease;
  box-sizing: border-box;
}

.url-textarea:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 2px rgba(255, 36, 66, 0.1);
}

.url-textarea:disabled {
  background: var(--bg-color-secondary);
  cursor: not-allowed;
  opacity: 0.6;
}

.url-textarea::placeholder {
  color: var(--text-color-secondary);
  font-family: var(--font-family);
}

.input-hints {
  margin-top: 8px;
  font-size: 12px;
  color: var(--text-color-secondary);
  line-height: 1.6;
}

.input-hints p {
  margin: 2px 0;
}

.invalid-section {
  margin-top: 16px;
  padding: 16px;
  background: var(--bg-color-secondary);
  border-radius: 8px;
  border-left: 3px solid var(--danger-color);
}

.invalid-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 12px;
  font-size: 14px;
  color: var(--text-color-primary);
  font-weight: 500;
}

.warning-icon {
  width: 18px;
  height: 18px;
  color: var(--danger-color);
}

.invalid-list {
  max-height: 150px;
  overflow-y: auto;
}

.invalid-item {
  padding: 6px 10px;
  margin: 4px 0;
  background: var(--bg-color-primary);
  border-radius: 4px;
  font-size: 12px;
  color: var(--danger-color);
  font-family: 'Consolas', 'Monaco', monospace;
  word-break: break-all;
  animation: fadeIn 0.2s ease;
}

.invalid-item:not(:last-child) {
  margin-bottom: 6px;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-5px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.error-message {
  margin-top: 16px;
  padding: 12px 16px;
  background: rgba(255, 12, 48, 0.1);
  border-radius: 8px;
  border-left: 3px solid var(--danger-color);
  font-size: 14px;
  color: var(--danger-color);
  animation: shake 0.3s ease;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}

.processing-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 20px;
  color: var(--text-color-secondary);
}

.loading-icon {
  width: 32px;
  height: 32px;
  margin-bottom: 12px;
  animation: spin 1s linear infinite;
  color: var(--primary-color);
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.processing-state p {
  margin: 0;
  font-size: 14px;
}

.modal-footer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 10px;
  padding: 20px 24px;
  border-top: 1px solid var(--border-color-primary);
  background: var(--bg-color-primary);
}

.cancel-btn,
.confirm-btn {
  padding: 8px 16px;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  text-decoration: none;
}

.cancel-btn {
  background-color: transparent;
  color: var(--text-color-secondary);
  border: 1px solid var(--border-color-primary);
}

.cancel-btn:hover:not(:disabled) {
  background-color: var(--bg-color-secondary);
}

.cancel-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.confirm-btn {
  background-color: var(--primary-color);
  color: white;
  border: none;
}

.confirm-btn:hover:not(:disabled) {
  background-color: var(--primary-color-dark);
}

.confirm-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .network-image-modal-overlay {
    padding: 10px;
  }

  .network-image-modal {
    max-height: 90vh;
  }

  .modal-header,
  .modal-footer {
    padding: 16px 20px;
  }

  .modal-body {
    padding: 20px;
  }

  .url-textarea {
    min-height: 150px;
  }

  .invalid-section {
    padding: 12px;
  }

  .invalid-list {
    max-height: 120px;
  }
}
</style>
