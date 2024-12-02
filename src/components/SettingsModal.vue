<template>
  <el-dialog
    :model-value="show"
    @update:model-value="handleClose"
    title="API 设置"
    width="360px"
    :modal="true"
    :append-to-body="true"
    :lock-scroll="false"
    class="settings-dialog"
  >
    <el-form 
      label-position="top" 
      :model="localConfig"
      :rules="rules"
      ref="formRef"
      class="settings-form"
    >
      <el-form-item 
        label="API 地址" 
        prop="baseUrl"
        required
      >
        <el-input 
          v-model="localConfig.baseUrl" 
          placeholder="默认: http://localhost"
          :prefix-icon="Link"
        />
      </el-form-item>
      
      <el-form-item 
        label="端口号" 
        prop="port"
        required
      >
        <div class="port-input-group">
          <el-input-number
            v-model="localConfig.port" 
            :min="1"
            :max="65535"
            placeholder="默认: 8000"
            controls-position="right"
            class="port-input"
          />
          <el-button 
            :icon="Connection" 
            @click="testConnection" 
            :loading="testing"
            class="test-btn"
            size="small"
            tabindex="-1"
          >
            测试连接
          </el-button>
        </div>
      </el-form-item>
    </el-form>
    
    <template #footer>
      <el-space>
        <el-button @click="handleCancel">取消</el-button>
        <el-button type="primary" @click="handleSave">保存</el-button>
      </el-space>
    </template>
  </el-dialog>
</template>

<script setup>
import { ref, watch } from 'vue'
import { Link, Connection } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'

const props = defineProps({
  show: Boolean,
  config: Object
})

const emit = defineEmits(['update:show', 'save'])
const formRef = ref(null)
const testing = ref(false)
const localConfig = ref({
  baseUrl: props.config?.baseUrl || 'http://localhost',
  port: Number(props.config?.port) || 8000
})

const rules = {
  baseUrl: [
    { required: true, message: '请输入 API 地址', trigger: 'blur' },
  ],
  port: [
    { required: true, message: '请输入端口号', trigger: 'blur' },
    { 
      validator: (rule, value, callback) => {
        const port = Number(value)
        if (isNaN(port) || port < 1 || port > 65535) {
          callback(new Error('端口号范围为 1-65535'))
        } else {
          callback()
        }
      },
      trigger: 'blur'
    }
  ]
}

watch(() => props.show, (newVal) => {
  if (newVal) {
    localConfig.value = {
      baseUrl: props.config?.baseUrl || 'http://localhost',
      port: Number(props.config?.port) || 8000
    }
  }
})

const handleCancel = () => {
  emit('update:show', false)
}

const handleClose = (val) => {
  if (!val) {
    handleCancel()
  }
}

const handleSave = async () => {
  if (!formRef.value) return
  
  try {
    await formRef.value.validate()
    const config = {
      baseUrl: localConfig.value.baseUrl,
      port: String(localConfig.value.port)
    }
    
    emit('save', config)
    emit('update:show', false)
    
    ElMessage({
      message: '配置已保存',
      type: 'success',
      offset: 60,
      position: 'bottom'
    })
  } catch (error) {
    ElMessage({
      message: '请检查配置信息',
      type: 'error',
      offset: 60,
      position: 'bottom'
    })
    return
  }
}

const testConnection = async () => {
  if (!localConfig.value.baseUrl || !localConfig.value.port) {
    ElMessage({
      message: '请先填写 API 地址和端口号',
      type: 'warning',
      offset: 60,
      position: 'bottom-left'
    })
    return
  }

  testing.value = true
  const baseUrl = `${localConfig.value.baseUrl}:${localConfig.value.port}`

  try {
    const response = await fetch(baseUrl, {
      method: 'HEAD',
      timeout: 3000
    })
    
    ElMessage({
      message: '连接成功',
      type: 'success',
      offset: 60,
      position: 'bottom-left'
    })
  } catch (error) {
    ElMessage({
      message: '连接失败，请检查 API 地址和端口号',
      type: 'error',
      offset: 60,
      position: 'bottom-left'
    })
  } finally {
    testing.value = false
  }
}
</script> 

<style scoped>
.settings-form {
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 16px;
  overflow: visible !important;
}

.input-with-button {
  display: flex;
  width: 100%;
}

:deep(.el-input-group__append) {
  padding: 0 !important;
  background: transparent !important;
  border-color: rgba(0, 198, 255, 0.2) !important;
  white-space: nowrap !important;
}

:deep(.test-connection-btn) {
  background: rgba(0, 198, 255, 0.15) !important;
  border-color: rgba(0, 198, 255, 0.3) !important;
  color: #00c6ff !important;
  padding: 0 16px !important;
  min-width: 100px !important;
  width: auto !important;
  display: flex !important;
  align-items: center !important;
  gap: 4px !important;
}

:deep(.test-connection-btn:hover) {
  background: rgba(0, 198, 255, 0.25) !important;
  border-color: rgba(0, 198, 255, 0.4) !important;
  color: #fff !important;
}

:deep(.test-connection-btn .el-icon) {
  font-size: 16px;
  margin-right: 4px;
}

:deep(.el-form-item) {
  margin-bottom: 0 !important;
  position: relative;
}

:deep(.el-form-item:last-child) {
  margin-bottom: 0;
}

:deep(.el-form-item__label) {
  padding-bottom: 4px !important;
  line-height: 1.2 !important;
  color: #e0e0e0 !important;
  font-size: 13px !important;
  position: relative;
  z-index: 1;
  margin-top: 4px !important;
  display: block !important;
}

:deep(.el-input__wrapper),
:deep(.el-input-number__wrapper) {
  height: 32px !important;
  position: relative;
  z-index: 0;
  margin-top: 4px !important;
}

:deep(.el-input__inner) {
  line-height: 32px !important;
}

:deep(.test-connection-btn) {
  height: 32px !important;
  padding: 0 8px !important;
}

:deep(.el-dialog) {
  margin-top: 15vh !important;
  min-height: auto !important;
}

:deep(.el-dialog__body) {
  padding: 20px 24px !important;
  min-height: 160px !important;
  height: auto !important;
  background: rgba(30, 35, 45, 0.95) !important;
  overflow: visible !important;
  max-height: none !important;
}

:deep(.el-dialog__header) {
  padding: 16px 24px !important;
  margin-right: 0 !important;
}

:deep(.el-dialog__footer) {
  padding: 16px 24px !important;
  border-top: 1px solid rgba(255, 255, 255, 0.05);
}

:deep(.el-input-number),
:deep(.el-input),
.input-with-button {
  width: 100% !important;
}

.settings-dialog {
  isolation: isolate;
  z-index: 2000;
}

:deep(.el-dialog__body) {
  isolation: isolate;
  position: relative;
  z-index: 1;
}

.port-input-group {
  display: flex;
  gap: 8px;
  align-items: center;
  margin-top: 2px;
  width: 100%;
}

.port-input {
  width: 140px !important;
}

.test-btn {
  flex-shrink: 0;
  height: 28px !important;
  padding: 0 10px !important;
  font-size: 12px !important;
  position: absolute;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
  user-select: none !important;
  pointer-events: auto !important;
  transition: background-color 0.2s, border-color 0.2s !important;
}

.test-btn:hover,
.test-btn:focus,
.test-btn:active {
  transform: translateY(-50%) !important;
}

:deep(.el-form-item__content) {
  margin-top: 2px !important;
  position: relative;
}
</style> 