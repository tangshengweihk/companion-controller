<template>
  <div class="container">
    <div class="author-wrapper">
      <span class="author-text">Created by</span>
      <span class="author-name">Kid</span>
    </div>
    
    <div class="top-buttons">
      <el-button
        @click="showSettings = true"
        :icon="Setting"
        circle
        type="primary"
      />
      <el-button
        @click="exportConfig"
        :icon="Download"
        circle
        type="success"
      />
      <el-button
        @click="importConfig"
        :icon="Upload"
        circle
        type="warning"
      />
      <el-button
        @click="resetAll"
        :icon="Delete"
        circle
        type="danger"
      />
    </div>
    
    <div class="header">
      <div class="title-group">
        <div class="title-line">
          <span class="dash">—</span>
          <h1 class="page-title">API控制面板</h1>
          <span class="dash">—</span>
        </div>
        <div class="subtitle">COMPANION CONTROLLER</div>
      </div>
    </div>
    
    <div class="buttons-grid">
      <div v-for="button in buttons" :key="button.id" class="button-wrapper">
        <el-button
          class="action-button"
          :class="{ 'last-executed': button.id === lastExecutedButtonId }"
          :data-button-id="button.id"
          :style="{
            '--el-button-text-color': button?.textColor || '#ffffff',
            color: button?.textColor || '#ffffff'
          }"
          @click="executeButtonApis(button)"
          size="large"
        >
          {{ button.name }}
        </el-button>
        <el-button
          @click="showEditor(button)"
          :icon="Edit"
          circle
        />
      </div>
    </div>

    <ButtonEditor
      v-model="showEditorDialog"
      :button-data="currentButton"
      @save="handleSave"
    />

    <SettingsModal
      v-if="showSettings"
      v-model:show="showSettings"
      :config="apiConfig"
      @save="saveApiConfig"
    />
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { Setting, Edit, Download, Upload, Delete } from '@element-plus/icons-vue'
import ButtonEditor from './components/ButtonEditor.vue'
import SettingsModal from './components/SettingsModal.vue'
import { ElMessage, ElMessageBox } from 'element-plus'

// 从 localStorage 加载按钮配置，修改为20个按钮
const buttons = ref(JSON.parse(localStorage.getItem('buttons')) || Array(20).fill().map((_, i) => ({
  id: i + 1,
  name: `按钮 ${i + 1}`,
  textColor: '#ffffff',
  apis: [],
  hotkey: ''
})))

// 从 localStorage 加载 API 配置，设置默认值
const apiConfig = ref(JSON.parse(localStorage.getItem('apiConfig')) || {
  baseUrl: 'http://localhost',  // 默认使用本机地址
  port: '8000'                  // 默认端口 8000
})

const showSettings = ref(false)
const showEditorDialog = ref(false)
const currentButton = ref(null)

// 修改 ElMessage 配置
const messageConfig = {
  offset: 20,
  position: 'bottom',
  grouping: true,
  duration: 2000,
  showClose: true,
  customClass: 'message-custom'
}

const loading = ref(false)

// 保存按钮配置
const saveButtons = () => {
  localStorage.setItem('buttons', JSON.stringify(buttons.value))
}

// 保存 API 配置
const saveApiConfig = (config) => {
  // 确保配置对象的完整性
  if (!config.baseUrl || !config.port) {
    ElMessage({
      message: '配置信息不完整',
      type: 'error',
      offset: 60,
      position: 'bottom'
    })
    return
  }
  
  apiConfig.value = config
  localStorage.setItem('apiConfig', JSON.stringify(config))
  
  ElMessage({
    message: '配置已保存',
    type: 'success',
    offset: 60,
    position: 'bottom'
  })
}

// 添加最后执行的按钮 ID 的 ref
const lastExecutedButtonId = ref(null)

// 修改后的 executeButtonApis 函数
const executeButtonApis = async (button) => {
  if (!apiConfig.value.baseUrl || !apiConfig.value.port) {
    ElMessage({
      ...messageConfig,
      message: '请先配置 API 接口',
      type: 'warning'
    })
    return
  }

  // 设置最后执行的按钮
  lastExecutedButtonId.value = button.id
  
  // 添加按钮点击效果
  const buttonEl = document.querySelector(`[data-button-id="${button.id}"]`)
  if (buttonEl) {
    buttonEl.classList.add('button-click-effect')
    setTimeout(() => {
      buttonEl.classList.remove('button-click-effect')
    }, 200)
  }

  loading.value = true
  try {
    const baseUrl = `${apiConfig.value.baseUrl}:${apiConfig.value.port}`
    let successCount = 0
    let failCount = 0

    for (const api of button.apis) {
      try {
        await fetch(`${baseUrl}/api/location/${api.page}/${api.row}/${api.column}/press`, {
          method: 'POST'
        })
        successCount++
      } catch (error) {
        console.error('API 调用失败:', error)
        failCount++
      }
    }

    // 合并显示消息
    if (successCount > 0) {
      ElMessage({
        ...messageConfig,
        message: `${successCount} 个 API 调用成功`,
        type: 'success'
      })
    }
    
    if (failCount > 0) {
      ElMessage({
        ...messageConfig,
        message: `${failCount} 个 API 调用失败`,
        type: 'error'
      })
    }
  } finally {
    loading.value = false
  }
}

// 导出配置
const exportConfig = () => {
  const config = {
    buttons: buttons.value,
    apiConfig: apiConfig.value,
    exportTime: new Date().toLocaleString()
  }
  
  const blob = new Blob([JSON.stringify(config, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  
  const a = document.createElement('a')
  a.href = url
  a.download = `companion-config-${new Date().toISOString().split('T')[0]}.json`
  a.click()
  
  URL.revokeObjectURL(url)
}

// 导入配置
const importConfig = () => {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = '.json'
  
  input.onchange = async (e) => {
    try {
      const file = e.target.files[0]
      const text = await file.text()
      const config = JSON.parse(text)
      
      if (config.buttons && config.apiConfig) {
        buttons.value = config.buttons
        apiConfig.value = config.apiConfig
        localStorage.setItem('buttons', JSON.stringify(config.buttons))
        localStorage.setItem('apiConfig', JSON.stringify(config.apiConfig))
        
        ElMessage({
          ...messageConfig,
          message: '配置导入成功',
          type: 'success'
        })
      } else {
        throw new Error('配置文件格式错误')
      }
    } catch (error) {
      ElMessage({
        ...messageConfig,
        message: '配置导入失败：' + error.message,
        type: 'error'
      })
    }
  }
  
  input.click()
}

// 添加重置功能
const resetAll = () => {
  ElMessageBox.confirm(
    '确定要恢初始配置吗？这将清除所有配置并恢复默认状态。',
    '警告',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning',
    }
  )
    .then(() => {
      // 重置按钮到初始状态，修改为20个按钮
      buttons.value = Array(20).fill().map((_, i) => ({
        id: i + 1,
        name: `按钮 ${i + 1}`,
        textColor: '#ffffff',
        apis: [],
        hotkey: ''
      }))
      
      // 重置 API 配置
      apiConfig.value = {
        baseUrl: 'http://localhost',
        port: '8000'
      }
      
      // 保存到 localStorage
      localStorage.setItem('buttons', JSON.stringify(buttons.value))
      localStorage.setItem('apiConfig', JSON.stringify(apiConfig.value))
      
      ElMessage({
        ...messageConfig,
        type: 'success',
        message: '已恢复初始配置'
      })
    })
    .catch(() => {
      // 用户取消操作
    })
}

// 显示编辑器
const showEditor = (button) => {
  currentButton.value = button
  showEditorDialog.value = true
}

// 处理保存
const handleSave = (updatedButton) => {
  const index = buttons.value.findIndex(b => b.id === updatedButton.id)
  if (index !== -1) {
    buttons.value[index] = {
      ...updatedButton,
      textColor: updatedButton.textColor || '#ffffff'
    }
    saveButtons()
  }
}

// 添加快捷键监听
onMounted(() => {
  window.addEventListener('keydown', handleGlobalKeydown)
})

// 清理事件监听
onUnmounted(() => {
  window.removeEventListener('keydown', handleGlobalKeydown)
})

// 处理全局快捷键
const handleGlobalKeydown = (e) => {
  // 如果是在输入框中，不触发快捷键
  if (e.target.tagName === 'INPUT' || e.target.tagName === 'TEXTAREA') {
    return
  }
  
  // 构建快捷键字符串
  const keys = []
  if (e.ctrlKey) keys.push('Ctrl')
  if (e.altKey) keys.push('Alt')
  if (e.shiftKey) keys.push('Shift')
  if (e.key !== 'Control' && e.key !== 'Alt' && e.key !== 'Shift') {
    keys.push(e.key.toUpperCase())
  }
  
  const hotkey = keys.join('+')
  console.log('当前按下的快捷键:', hotkey) // 添加调试日志
  
  // 查找匹配的按钮
  const matchedButton = buttons.value.find(btn => btn.hotkey === hotkey)
  if (matchedButton) {
    console.log('找到匹配的按钮:', matchedButton.name) // 添加调试日志
    e.preventDefault() // 默认行为
    executeButtonApis(matchedButton)
  }
}
</script>

<style scoped>
.container {
  max-width: 1200px;
  width: 100%;
  padding: 40px 20px;
  height: auto;
  min-height: 500px;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 0 auto;
}

.header {
  width: 100%;
  display: flex;
  justify-content: center;
  margin-bottom: 40px;
  padding: 20px;
  position: relative;
  height: 120px;
}

.title-group {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.title-line {
  display: flex;
  align-items: center;
  gap: 16px;
}

.dash {
  color: #00c6ff;
  opacity: 0.6;
  font-weight: 300;
  font-size: 24px;
}

.page-title {
  font-size: 32px;
  font-weight: 600;
  background: linear-gradient(135deg, #00c6ff, #0072ff);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  margin: 0;
  letter-spacing: 2px;
  text-shadow: 0 2px 10px rgba(0, 198, 255, 0.2);
}

.subtitle {
  font-size: 14px;
  color: rgba(255, 255, 255, 0.4);
  letter-spacing: 4px;
  text-transform: uppercase;
  margin-top: 4px;
}

.author-wrapper {
  position: absolute;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  right: 280px;
  top: 120px;
}

.author-text {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.4);
  letter-spacing: 1px;
}

.author-name {
  font-family: 'Brush Script MT', 'Comic Sans MS', cursive;
  font-size: 28px;
  color: rgba(255, 255, 255, 0.4);
  font-weight: 500;
  position: relative;
  margin-top: -4px;
}

.author-name::after {
  content: '';
  position: absolute;
  bottom: -2px;
  left: 0;
  width: 100%;
  height: 1px;
  background: rgba(255, 255, 255, 0.1);
  opacity: 0.3;
}

.author-name:hover {
  color: rgba(255, 255, 255, 0.6);
  transition: all 0.3s ease;
}

.author-name:hover::after {
  opacity: 0.5;
  transition: all 0.3s ease;
}

.top-buttons {
  position: absolute;
  display: flex;
  gap: 8px;
  right: 20px;
  top: 20px;
  align-items: center;
  z-index: 1;
}

.buttons-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 20px;
  width: 100%;
  max-width: 1200px;
  margin-top: 20px;
  padding: 20px;
  background: rgba(255, 255, 255, 0.02);
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
}

.button-wrapper {
  display: flex;
  gap: 8px;
  align-items: center;
}

.action-button {
  flex: 1;
  height: 50px;
  min-width: 120px;
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
  user-select: none;
  -webkit-user-select: none;
}

.action-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  opacity: 0.9;
}

.action-button:active {
  transform: translateY(1px);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  filter: brightness(85%);
  transition: all 0.1s ease;
}

.button-wrapper .el-button.is-circle {
  transition: all 0.3s ease;
}

.button-wrapper .el-button.is-circle:hover {
  transform: rotate(180deg);
  background-color: #f0f0f0;
}

.button-wrapper .el-button.is-circle:active {
  background-color: #e0e0e0;
  transform: rotate(180deg) scale(0.95);
}

@media (max-width: 1200px) {
  .buttons-grid {
    grid-template-columns: repeat(4, 1fr);
  }
}

@media (max-width: 1024px) {
  .buttons-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}

@media (max-width: 768px) {
  .buttons-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 480px) {
  .buttons-grid {
    grid-template-columns: repeat(1, 1fr);
  }
}

/* 添加全局样式来调整消息位置 */
:global(.el-message) {
  justify-content: center;  /* 水平居中 */
  margin-bottom: 20px;     /* 距离底部距离 */
}

/* 顶部工具栏样式 */
.top-buttons {
  position: absolute;
  display: flex;
  gap: 8px;
  right: 20px;
  top: 20px;
  align-items: center;
  z-index: 1;
}

/* 作者署名容器 */
.author-wrapper {
  position: absolute;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  right: 280px;
  top: 120px;
}

.author-text {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.4);
  letter-spacing: 1px;
}

.author-name {
  font-family: 'Brush Script MT', 'Comic Sans MS', cursive;
  font-size: 28px;
  color: rgba(255, 255, 255, 0.4);
  font-weight: 500;
  position: relative;
  margin-top: -4px;
}

.author-name::after {
  content: '';
  position: absolute;
  bottom: -2px;
  left: 0;
  width: 100%;
  height: 1px;
  background: rgba(255, 255, 255, 0.1);
  opacity: 0.3;
}

.author-name:hover {
  color: rgba(255, 255, 255, 0.6);
  transition: all 0.3s ease;
}

.author-name:hover::after {
  opacity: 0.5;
  transition: all 0.3s ease;
}

.action-button.last-executed {
  box-shadow: 0 0 0 2px rgba(0, 198, 255, 0.5);
  animation: pulse 2s infinite;
}

.button-click-effect {
  transform: scale(0.95) !important;
  opacity: 0.8;
  transition: all 0.2s ease !important;
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 2px rgba(0, 198, 255, 0.5);
  }
  50% {
    box-shadow: 0 0 0 4px rgba(0, 198, 255, 0.3);
  }
  100% {
    box-shadow: 0 0 0 2px rgba(0, 198, 255, 0.5);
  }
}

/* 确保动画效果平滑 */
.action-button {
  transition: all 0.2s ease;
}

/* 添加这个新的样式规则，使用更高的优先级 */
.buttons-grid .button-wrapper .action-button {
  color: var(--el-button-text-color) !important;
}

/* 添加一个全局样式来覆盖 Element Plus 的默认样式 */
:deep(.el-button) {
  --el-button-text-color: v-bind('button?.textColor || "#ffffff"');
}
</style>
