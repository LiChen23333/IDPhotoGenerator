<template>
  <div id="app">
    <div class="mobile-container">
      <!-- 顶部标题 -->
      <div class="header">
        <h1>
          <el-icon><Camera /></el-icon>
          证件照生成
        </h1>
      </div>
      
      <!-- 主要内容区域 -->
      <div class="main-content">
        <!-- 图片上传区域 -->
        <div class="upload-section">
          <div v-if="modelLoading" class="loading-tip">
            <el-icon class="loading-icon"><Loading /></el-icon>
            <span>模型加载中，请稍候...</span>
          </div>
          <el-upload
            v-else
            class="upload-demo"
            drag
            :auto-upload="false"
            :on-change="handleImageUpload"
            :show-file-list="false"
            accept="image/*"
          >
            <el-icon class="el-icon--upload" size="32"><upload-filled /></el-icon>
            <div class="el-upload__text">
              点击或拖拽上传图片
            </div>
            <template #tip>
              <div class="el-upload__tip">
                支持 jpg/png 格式
              </div>
            </template>
          </el-upload>
        </div>

        <!-- 预览区域 -->
        <div class="preview-section" v-show="originalImage || processedImage">
          <!-- 原图和结果图并排显示 -->
          <div class="image-row" v-show="originalImage">
            <div class="image-item">
              <div class="image-title">原图</div>
              <div class="image-wrapper">
                <img :src="originalImage" alt="原图" />
              </div>
            </div>
            <div class="image-item" v-show="processedImage">
              <div class="image-title">证件照</div>
              <div class="image-wrapper">
                <canvas ref="resultCanvas"></canvas>
              </div>
            </div>
          </div>
          
          <!-- 打印排版预览 -->
          <div class="print-preview" v-show="printLayoutImage">
            <div class="image-title">打印排版</div>
            <div class="image-wrapper">
              <canvas ref="printCanvas"></canvas>
            </div>
          </div>
        </div>

        <!-- 控制面板 -->
        <div class="control-panel">
          <!-- 背景颜色选择 -->
          <div class="control-group">
            <div class="group-title">背景颜色</div>
            <div class="color-grid">
              <div 
                v-for="color in backgroundColors" 
                :key="color.name"
                class="color-item"
                :class="{ active: selectedBgColor === color.value }"
                :style="{ backgroundColor: color.value }"
                @click="selectedBgColor = color.value"
              >
                <span class="color-name">{{ color.name }}</span>
              </div>
              <div class="custom-color">
              <el-color-picker v-model="selectedBgColor" size="small" />
              <span>自定义</span>
            </div>
            </div>
            
          </div>

          <!-- 证件照尺寸选择 -->
          <div class="control-group">
            <div class="group-title">照片尺寸</div>
            <el-select v-model="selectedSize" placeholder="选择尺寸" size="large" value-key="name">
              <el-option
                v-for="size in photoSizes"
                :key="size.name"
                :label="`${size.name} (${size.width}×${size.height}mm)`"
                :value="size"
              />
            </el-select>
          </div>

          <!-- 操作按钮 -->
          <div class="action-buttons">
            <el-button 
              type="primary" 
              size="large" 
              :loading="processing"
              :disabled="!originalImage"
              @click="processImage"
              class="action-btn"
            >
              <el-icon><Magic /></el-icon>
              {{ processing ? '处理中...' : '生成证件照' }}
            </el-button>
            
            <div class="download-buttons" v-show="processedImage">
              <el-button 
                type="success" 
                size="large" 
                @click="downloadImage"
                class="action-btn"
              >
                <el-icon><Download /></el-icon>
                下载证件照
              </el-button>
              <el-button 
                type="warning" 
                size="large" 
                @click="downloadPrintLayout"
                class="action-btn"
              >
                <el-icon><Printer /></el-icon>
                下载排版
              </el-button>
            </div>
          </div>
        </div>

        <!-- 空状态 -->
        <div v-if="!originalImage && !modelLoading" class="empty-state">
          <el-icon size="60" color="#C0C4CC"><Picture /></el-icon>
          <p>上传图片开始制作证件照</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import * as ort from '/node_modules/onnxruntime-web/dist/ort.mjs'
window.ort = ort

// 响应式数据
const originalImage = ref(null)
const processedImage = ref(null)
const printLayoutImage = ref(null)
const processing = ref(false)
const modelLoading = ref(true)
const selectedBgColor = ref('#FFFFFF')
const selectedSize = ref(null)
const resultCanvas = ref(null)
const printCanvas = ref(null)
const session = ref(null)

// 背景颜色选项
const backgroundColors = [
  { name: '白色', value: '#FFFFFF' },
  { name: '蓝色', value: '#438EDB' },
  { name: '红色', value: '#FF0000' },
  { name: '灰色', value: '#C0C0C0' }
]

// 证件照尺寸选项
const photoSizes = [
  { name: '一寸', width: 25, height: 35, pixelWidth: 295, pixelHeight: 413 },
  { name: '二寸', width: 35, height: 49, pixelWidth: 413, pixelHeight: 579 },
  { name: '小二寸', width: 33, height: 48, pixelWidth: 390, pixelHeight: 567 },
  { name: '大二寸', width: 35, height: 53, pixelWidth: 413, pixelHeight: 626 }
]

// 初始化
onMounted(async () => {
  selectedSize.value = photoSizes[0]
  await initONNX()
})

// 初始化ONNX
async function initONNX() {
  try {
    modelLoading.value = true
    // // 设置ONNX运行时
    // if (ort.env && ort.env.wasm) {
    //   ort.env.wasm.wasmPaths = 'https://cdn.jsdelivr.net/npm/onnxruntime-web@1.15.1/dist/'
    // }
    
    // 加载模型
    let modelPath = '/IDPhotoGenerator/modnet/model.onnx'
    if (import.meta.env.DEV) {
      modelPath = '/modnet/model.onnx'
    }

    if (import.meta.env.PROD) {
      // modelPath = '/IDPhotoGenerator/modnet/model.onnx'
      modelPath = '/modnet/model.onnx'
    }
    session.value = await ort.InferenceSession.create(modelPath)
    console.log('ModNet模型加载成功')
    ElMessage.success('模型加载完成，可以开始使用！')
  } catch (error) {
    console.error('模型加载失败:', error)
    ElMessage.error('模型加载失败，请检查网络连接')
  } finally {
    modelLoading.value = false
  }
}

// 处理图片上传
function handleImageUpload(file) {
  const reader = new FileReader()
  reader.onload = (e) => {
    originalImage.value = e.target.result
    processedImage.value = null
    printLayoutImage.value = null
  }
  reader.readAsDataURL(file.raw)
}

// 图片预处理
function preprocessImage(imageElement) {
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')
  
  // 设置输入尺寸 (ModNet通常使用512x512)
  canvas.width = 512
  canvas.height = 512
  
  // 绘制并缩放图片
  ctx.drawImage(imageElement, 0, 0, 512, 512)
  
  // 获取图片数据
  const imageData = ctx.getImageData(0, 0, 512, 512)
  const data = imageData.data
  
  // 转换为模型输入格式 [1, 3, 512, 512]
  const input = new Float32Array(1 * 3 * 512 * 512)
  
  for (let i = 0; i < 512 * 512; i++) {
    // 归一化到 [0, 1] 并转换为 RGB
    input[i] = data[i * 4] / 255.0 // R
    input[i + 512 * 512] = data[i * 4 + 1] / 255.0 // G
    input[i + 512 * 512 * 2] = data[i * 4 + 2] / 255.0 // B
  }
  
  return { input, originalWidth: imageElement.naturalWidth, originalHeight: imageElement.naturalHeight }
}

// 处理图片
async function processImage() {
  if (!originalImage.value || !session.value) {
    ElMessage.warning('请先上传图片')
    return
  }
  
  processing.value = true
  
  try {
    // 创建图片元素
    const img = new Image()
    img.crossOrigin = 'anonymous'
    
    await new Promise((resolve, reject) => {
      img.onload = resolve
      img.onerror = reject
      img.src = originalImage.value
    })
    
    // 预处理图片
    const preprocessResult = preprocessImage(img)
    
    // 创建输入张量
    const inputTensor = new ort.Tensor('float32', preprocessResult.input, [1, 3, 512, 512])
    
    // 运行推理
    const results = await session.value.run({ input: inputTensor })
    
    // 获取mask结果
    const mask = results.output.data
    
    // 生成最终图片
    generateFinalImage(img, mask, preprocessResult.originalWidth, preprocessResult.originalHeight)
    
    ElMessage.success('证件照生成成功！')
  } catch (error) {
    console.error('处理失败:', error)
    ElMessage.error('图片处理失败，请重试')
  } finally {
    processing.value = false
  }
}

// 生成最终图片
function generateFinalImage(originalImg, mask, originalWidth, originalHeight) {
  const canvas = resultCanvas.value
  const ctx = canvas.getContext('2d')
  
  // 设置画布尺寸为选定的证件照尺寸
  canvas.width = selectedSize.value.pixelWidth
  canvas.height = selectedSize.value.pixelHeight
  
  // 填充背景色
  ctx.fillStyle = selectedBgColor.value
  ctx.fillRect(0, 0, canvas.width, canvas.height)
  
  // 创建临时画布处理mask，使用原图分辨率
  const tempCanvas = document.createElement('canvas')
  const tempCtx = tempCanvas.getContext('2d')
  tempCanvas.width = originalWidth
  tempCanvas.height = originalHeight
  
  // 绘制原图到临时画布（保持原分辨率）
  tempCtx.drawImage(originalImg, 0, 0, originalWidth, originalHeight)
  const imageData = tempCtx.getImageData(0, 0, originalWidth, originalHeight)
  
  // 将512x512的mask缩放到原图尺寸
  const maskCanvas = document.createElement('canvas')
  const maskCtx = maskCanvas.getContext('2d')
  maskCanvas.width = 512
  maskCanvas.height = 512
  
  // 创建mask图像数据
  const maskImageData = maskCtx.createImageData(512, 512)
  for (let i = 0; i < mask.length; i++) {
    const alpha = Math.round(mask[i] * 255)
    maskImageData.data[i * 4] = 255     // R
    maskImageData.data[i * 4 + 1] = 255 // G
    maskImageData.data[i * 4 + 2] = 255 // B
    maskImageData.data[i * 4 + 3] = alpha // A
  }
  maskCtx.putImageData(maskImageData, 0, 0)
  
  // 创建缩放后的mask画布
  const scaledMaskCanvas = document.createElement('canvas')
  const scaledMaskCtx = scaledMaskCanvas.getContext('2d')
  scaledMaskCanvas.width = originalWidth
  scaledMaskCanvas.height = originalHeight
  
  // 将mask缩放到原图尺寸
  scaledMaskCtx.drawImage(maskCanvas, 0, 0, originalWidth, originalHeight)
  const scaledMaskData = scaledMaskCtx.getImageData(0, 0, originalWidth, originalHeight)
  
  // 应用缩放后的mask到原图
  for (let i = 0; i < originalWidth * originalHeight; i++) {
    imageData.data[i * 4 + 3] = scaledMaskData.data[i * 4 + 3] // 使用mask的alpha通道
  }
  
  tempCtx.putImageData(imageData, 0, 0)
  
  // 计算等比缩放，保持原图宽高比
  const scaleX = canvas.width / originalWidth
  const scaleY = canvas.height / originalHeight
  const scale = Math.min(scaleX, scaleY)
  
  const scaledWidth = originalWidth * scale
  const scaledHeight = originalHeight * scale
  
  // 水平居中，垂直底部对齐（人像下方与照片下方对齐）
  const x = (canvas.width - scaledWidth) / 2
  const y = canvas.height - scaledHeight
  
  // 绘制到结果画布
  ctx.drawImage(tempCanvas, x, y, scaledWidth, scaledHeight)
  
  processedImage.value = canvas.toDataURL('image/png')
  
  // 生成打印排版
  generatePrintLayout()
}

// 生成打印排版
function generatePrintLayout() {
  if (!processedImage.value) return
  
  const printCanvasEl = printCanvas.value
  const ctx = printCanvasEl.getContext('2d')
  
  // 6寸相纸尺寸 (300 DPI): 1795 x 1205 像素 (152mm × 102mm)
  const paperWidth = 1795
  const paperHeight = 1205
  printCanvasEl.width = paperWidth
  printCanvasEl.height = paperHeight
  
  // 填充白色背景
  ctx.fillStyle = '#FFFFFF'
  ctx.fillRect(0, 0, paperWidth, paperHeight)
  
  // 获取单张证件照尺寸
  const photoWidth = selectedSize.value.pixelWidth
  const photoHeight = selectedSize.value.pixelHeight
  
  // 计算排版参数
  const margin = 50 // 边距
  const spacing = 20 // 照片间距
  
  // 计算可排列的行列数
  const availableWidth = paperWidth - 2 * margin
  const availableHeight = paperHeight - 2 * margin
  
  const cols = Math.floor((availableWidth + spacing) / (photoWidth + spacing))
  const rows = Math.floor((availableHeight + spacing) / (photoHeight + spacing))
  
  // 计算实际起始位置（居中排列）
  const totalWidth = cols * photoWidth + (cols - 1) * spacing
  const totalHeight = rows * photoHeight + (rows - 1) * spacing
  const startX = (paperWidth - totalWidth) / 2
  const startY = (paperHeight - totalHeight) / 2
  
  // 创建图片对象
  const img = new Image()
  img.onload = () => {
    // 排列证件照
    for (let row = 0; row < rows; row++) {
      for (let col = 0; col < cols; col++) {
        const x = startX + col * (photoWidth + spacing)
        const y = startY + row * (photoHeight + spacing)
        ctx.drawImage(img, x, y, photoWidth, photoHeight)
      }
    }
    
    // 添加裁切线
    ctx.strokeStyle = '#CCCCCC'
    ctx.lineWidth = 1
    ctx.setLineDash([5, 5])
    
    // 绘制垂直裁切线
    for (let col = 1; col < cols; col++) {
      const x = startX + col * (photoWidth + spacing) - spacing / 2
      ctx.beginPath()
      ctx.moveTo(x, startY - 10)
      ctx.lineTo(x, startY + totalHeight + 10)
      ctx.stroke()
    }
    
    // 绘制水平裁切线
    for (let row = 1; row < rows; row++) {
      const y = startY + row * (photoHeight + spacing) - spacing / 2
      ctx.beginPath()
      ctx.moveTo(startX - 10, y)
      ctx.lineTo(startX + totalWidth + 10, y)
      ctx.stroke()
    }
    
    // 添加尺寸标注
    ctx.setLineDash([])
    ctx.fillStyle = '#666666'
    ctx.font = '12px Arial'
    ctx.textAlign = 'center'
    
    const infoText = `${selectedSize.value.name} (${selectedSize.value.width}×${selectedSize.value.height}mm) - ${rows}×${cols}=${rows*cols}张`
    ctx.fillText(infoText, paperWidth / 2, paperHeight - 20)
    
    printLayoutImage.value = printCanvasEl.toDataURL('image/png')
  }
  
  img.src = processedImage.value
}

// 检测是否为移动设备
function isMobileDevice() {
  return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)
}

// 下载图片
function downloadImage() {
  if (!processedImage.value) return
  
  const filename = `证件照_${selectedSize.value.name}_${Date.now()}.png`
  
  if (isMobileDevice()) {
    // 移动端：在新窗口打开图片，用户可以长按保存
    const newWindow = window.open()
    if (newWindow) {
      newWindow.document.write(`
        <html>
          <head>
            <title>${filename}</title>
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <style>
              body { margin: 0; padding: 20px; text-align: center; background: #f5f5f5; }
              img { max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.15); }
              .tip { margin-top: 20px; color: #666; font-size: 14px; }
            </style>
          </head>
          <body>
            <img src="${processedImage.value}" alt="${filename}" />
            <div class="tip">长按图片保存到相册</div>
          </body>
        </html>
      `)
      newWindow.document.close()
    } else {
      // 如果无法打开新窗口，尝试传统下载方式
      fallbackDownload(processedImage.value, filename)
    }
  } else {
    // 桌面端：使用传统下载方式
    fallbackDownload(processedImage.value, filename)
  }
}

// 下载打印排版
function downloadPrintLayout() {
  if (!printLayoutImage.value) return
  
  const filename = `证件照排版_${selectedSize.value.name}_${Date.now()}.png`
  
  if (isMobileDevice()) {
    // 移动端：在新窗口打开图片，用户可以长按保存
    const newWindow = window.open()
    if (newWindow) {
      newWindow.document.write(`
        <html>
          <head>
            <title>${filename}</title>
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <style>
              body { margin: 0; padding: 20px; text-align: center; background: #f5f5f5; }
              img { max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.15); }
              .tip { margin-top: 20px; color: #666; font-size: 14px; }
            </style>
          </head>
          <body>
            <img src="${printLayoutImage.value}" alt="${filename}" />
            <div class="tip">长按图片保存到相册</div>
          </body>
        </html>
      `)
      newWindow.document.close()
    } else {
      // 如果无法打开新窗口，尝试传统下载方式
      fallbackDownload(printLayoutImage.value, filename)
    }
  } else {
    // 桌面端：使用传统下载方式
    fallbackDownload(printLayoutImage.value, filename)
  }
}

// 传统下载方式（兜底方案）
function fallbackDownload(dataUrl, filename) {
  try {
    const link = document.createElement('a')
    link.download = filename
    link.href = dataUrl
    link.style.display = 'none'
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
  } catch (error) {
    console.error('下载失败:', error)
    ElMessage.warning('下载失败，请尝试长按图片保存')
  }
}
</script>

<style scoped>
#app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.mobile-container {
  max-width: 100vw;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* 顶部标题 */
.header {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  padding: 10px 15px;
  text-align: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  position: sticky;
  top: 0;
  z-index: 100;
}

.header h1 {
  margin: 0;
  color: #409EFF;
  font-size: 18px;
  font-weight: 600;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

/* 主要内容区域 */
.main-content {
  flex: 1;
  padding: 10px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

/* 图片上传区域 */
.upload-section {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 10px;
  padding: 10px 20px 0 20px;
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.1);
}

.loading-tip {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 20px;
  color: #409EFF;
  font-size: 14px;
}

.loading-icon {
  animation: rotate 2s linear infinite;
}

@keyframes rotate {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.upload-demo {
  width: 100%;
}

.upload-demo .el-upload-dragger {
  width: 100%;
  height: 90px;
  border-radius: 8px;
  border: 2px dashed #d9d9d9;
  background: #fafafa;
}

.el-upload__text {
  font-size: 13px;
  margin-top: 6px;
}

.el-upload__tip {
  font-size: 11px;
  color: #999;
  margin-top: 3px;
}

/* 预览区域 */
.preview-section {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 10px;
  padding: 12px;
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.1);
}

.image-row {
  display: flex;
  flex-direction: row;
  gap: 8px;
  margin-bottom: 10px;
}

.image-item {
  flex: 1;
  text-align: center;
}

.image-title {
  font-size: 13px;
  font-weight: 600;
  color: #409EFF;
  margin-bottom: 6px;
}

.image-wrapper {
  border: 2px dashed #E4E7ED;
  border-radius: 6px;
  padding: 8px;
  background: #fafafa;
  min-height: 120px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.image-wrapper img,
.image-wrapper canvas {
  max-width: 100%;
  max-height: 150px;
  border-radius: 4px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

.print-preview {
  margin-top: 10px;
  text-align: center;
}

.print-preview .image-wrapper {
  min-height: 80px;
}

.print-preview .image-wrapper canvas {
  max-height: 100px;
}

/* 控制面板 */
.control-panel {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 10px;
  padding: 15px;
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.control-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.group-title {
  font-size: 14px;
  font-weight: 600;
  color: #409EFF;
}

/* 颜色选择 */
.color-grid {
  display: flex;
  flex-direction: row;
}

.color-item {
  height: 32px;
  width: auto;
  flex:1;
  border-radius: 6px;
  margin-right: 6px;
  cursor: pointer;
  border: 2px solid transparent;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  position: relative;
}

.color-item:active {
  transform: scale(0.95);
}

.color-item.active {
  border-color: #409EFF;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2);
}

.color-name {
  font-size: 12px;
  font-weight: 600;
  color: #333;
  text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8);
}

.custom-color {
  display: flex;
  align-items: center;
  flex:2;
}

.custom-color span {
  font-size: 14px;
  color: #666;
}

/* 尺寸选择 */
.el-select {
  width: 100%;
}

/* 操作按钮 */
.action-buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.action-btn {
  width: 100%;
  height: 42px;
  border-radius: 8px;
  font-weight: 600;
  font-size: 14px;
}

.download-buttons {
  display: flex;
  gap: 8px;
}

.download-buttons .action-btn {
  flex: 1;
  height: 38px;
  font-size: 13px;
}

/* 空状态 */
.empty-state {
  text-align: center;
  color: #909399;
  padding: 30px 15px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 10px;
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.1);
}

.empty-state p {
  margin-top: 12px;
  font-size: 14px;
}

/* 响应式设计 */
@media (max-width: 480px) {
  .main-content {
    padding: 8px;
    gap: 8px;
  }
  
  .header {
    padding: 8px 12px;
  }
  
  .header h1 {
    font-size: 16px;
  }
  
  .control-panel,
  .upload-section,
  .preview-section {
    padding: 12px;
  }
  
  .image-row {
    flex-direction: row;
    gap: 6px;
  }
  
  .download-buttons {
    flex-direction: column;
    gap: 6px;
  }
  
  .action-btn {
    font-size: 13px;
    height: 38px;
  }
  
  .upload-demo .el-upload-dragger {
    height: 80px;
  }
  
  .image-wrapper {
    min-height: 100px;
  }
  
  .image-wrapper img,
  .image-wrapper canvas {
    max-height: 120px;
  }
}

@media (min-width: 768px) {
  .mobile-container {
    max-width: 600px;
    margin: 0 auto;
  }
  
  .main-content {
    padding: 10px;
  }
}

:deep(.el-upload-dragger)
{
  padding: 0;
}
</style>