<template>
  <el-dialog
    v-model="dialogVisible"
    title="编辑按钮"
    width="500px"
    :modal="true"
    :append-to-body="true"
    :lock-scroll="false"
    modal-class="no-scroll"
    @open="handleDialogOpen"
    @close="handleDialogClose"
  >
    <el-form label-position="top">
      <el-form-item label="按钮名称">
        <el-input v-model="localButton.name" />
      </el-form-item>
      
      <el-form-item label="文字颜色">
        <el-color-picker v-model="localButton.textColor" />
      </el-form-item>
      
      <el-form-item label="API 配置">
        <div v-for="(api, index) in localButton.apis" :key="index" class="api-item">
          <el-row :gutter="12">
            <el-col :span="5">
              <el-input 
                v-model="api.label" 
                placeholder="标签"
                size="default"
              />
            </el-col>
            <el-col :span="5">
              <el-input v-model="api.page" placeholder="页" />
            </el-col>
            <el-col :span="5">
              <el-input v-model="api.row" placeholder="行" />
            </el-col>
            <el-col :span="5">
              <el-input v-model="api.column" placeholder="列" />
            </el-col>
            <el-col :span="4">
              <el-button type="danger" :icon="Delete" circle @click="localButton.apis.splice(index, 1)" />
            </el-col>
          </el-row>
        </div>
        <el-button 
          type="primary" 
          :icon="Plus"
          @click="addApi"
          style="width: 100%; margin-top: 12px"
        >
          添加 API
        </el-button>
      </el-form-item>
      
      <el-form-item label="快捷键" prop="hotkey">
        <div class="hotkey-input-group">
          <el-input
            v-model="localButton.hotkey"
            placeholder="点击输入快捷键"
            readonly
            @keydown.stop.prevent="handleHotkeyInput"
            @focus="isRecordingHotkey = true"
            @blur="isRecordingHotkey = false"
          />
          <el-button
            v-if="localButton.hotkey"
            @click="clearHotkey"
            class="clear-hotkey-btn"
            :icon="Delete"
            size="small"
          />
        </div>
      </el-form-item>
    </el-form>
    
    <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleSave">保存</el-button>
      </span>
    </template>
  </el-dialog>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { Delete, Plus } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'

const props = defineProps({
  modelValue: Boolean,
  buttonData: Object
})

const emit = defineEmits(['update:modelValue', 'save'])

// 创建本数据副本
const localButton = ref({})

// 监听对话框显示状态
const dialogVisible = computed({
  get: () => props.modelValue,
  set: (value) => emit('update:modelValue', value)
})

// 初始化本地数据
const handleDialogOpen = () => {
  localButton.value = JSON.parse(JSON.stringify(props.buttonData || {}))
}

// 处理对话框关闭
const handleDialogClose = () => {
  localButton.value = {}
}

const validateApis = () => {
  if (!localButton.value.apis || localButton.value.apis.length === 0) {
    return true
  }
  
  for (const api of localButton.value.apis) {
    if (!api.page || !api.row || !api.column) {
      return false
    }
    
    if (api.page === '' || api.row === '' || api.column === '') {
      return false
    }
  }
  return true
}

const handleSave = () => {
  if (!localButton.value.name?.trim()) {
    ElMessage({
      message: '请输入按钮名称',
      type: 'warning',
      offset: 60,
      position: 'bottom-left'
    })
    return
  }

  if (!validateApis()) {
    ElMessage({
      message: '请完整填写所有 API 参数（页/行/列）',
      type: 'warning',
      offset: 60,
      position: 'bottom-left'
    })
    return
  }

  // 确保保存完整的按钮数据
  const buttonData = {
    ...localButton.value,
    id: localButton.value.id,
    name: localButton.value.name.trim(),
    textColor: localButton.value.textColor || '#ffffff',
    apis: localButton.value.apis || [],
    hotkey: localButton.value.hotkey || ''
  }
  
  console.log('保存按钮数据:', buttonData) // 调试日志
  
  emit('save', buttonData)
  dialogVisible.value = false
  ElMessage({
    message: '保存成功',
    type: 'success',
    offset: 60,
    position: 'bottom-left'
  })
}

// 添加 API
const addApi = () => {
  if (!localButton.value.apis) {
    localButton.value.apis = []
  }
  localButton.value.apis.push({
    label: '',
    page: '',
    row: '',
    column: ''
  })
}

const isRecordingHotkey = ref(false)

// 处理快捷键输入
const handleHotkeyInput = (e) => {
  if (!isRecordingHotkey.value) return
  
  const keys = []
  if (e.ctrlKey) keys.push('Ctrl')
  if (e.altKey) keys.push('Alt')
  if (e.shiftKey) keys.push('Shift')
  
  // 只记录常规按键，不记录修饰键
  if (e.key !== 'Control' && e.key !== 'Alt' && e.key !== 'Shift') {
    keys.push(e.key.toUpperCase())
  }
  
  if (keys.length > 0) {
    localButton.value.hotkey = keys.join('+')
  }
}

// 清除快捷键
const clearHotkey = () => {
  localButton.value.hotkey = ''
}
</script>

<style scoped>
.api-item {
  margin-bottom: 12px;
  background: rgba(255, 255, 255, 0.03);
  padding: 12px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

:deep(.el-form-item__label) {
  color: #e0e0e0 !important;
}

:deep(.el-input__inner) {
  transition: all 0.3s ease;
}

:deep(.el-input__inner:hover) {
  background: rgba(255, 255, 255, 0.08) !important;
}

.hotkey-input-group {
  display: flex;
  gap: 8px;
  align-items: center;
  position: relative;
}

/* 修改快捷键输入框样式 */
:deep(.el-input__wrapper) {
  background: rgba(0, 198, 255, 0.05) !important;
  border: 1px solid rgba(0, 198, 255, 0.2) !important;
  transition: all 0.3s ease !important;
}

:deep(.el-input__wrapper:hover),
:deep(.el-input__wrapper.is-focus) {
  background: rgba(0, 198, 255, 0.1) !important;
  border-color: rgba(0, 198, 255, 0.4) !important;
  box-shadow: 0 0 10px rgba(0, 198, 255, 0.1) !important;
}

/* 清除按钮样式 */
.clear-hotkey-btn {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  padding: 4px !important;
  height: 24px;
  width: 24px;
  background: transparent !important;
  border: none !important;
  color: rgba(255, 255, 255, 0.6) !important;
  transition: all 0.3s ease !important;
}

.clear-hotkey-btn:hover {
  color: #fff !important;
  background: rgba(255, 255, 255, 0.1) !important;
  transform: translateY(-50%) !important;
}

.clear-hotkey-btn:active {
  transform: translateY(-50%) scale(0.95) !important;
}
</style> 