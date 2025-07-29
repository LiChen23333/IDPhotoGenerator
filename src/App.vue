<template>
  <div id="app">
    <el-container>
      <el-header>
        <h1 style="text-align: center; margin: 0; color: #409EFF;">
          <el-icon><Camera /></el-icon>
          一键证件照生成工具
        </h1>
      </el-header>
      
      <el-main>   
        <el-row :gutter="20">
          <!-- 左侧控制面板 -->
          <el-col :span="8">
            <el-card shadow="hover">
              <template #header>
                <span>控制面板</span>
              </template>
              
              <!-- 图片上传 -->
              <div class="control-section">
                <h3>1. 上传图片</h3>
                <el-upload
                  class="upload-demo"
                  drag
                  :auto-upload="false"
                  :on-change="handleImageUpload"
                  :show-file-list="false"
                  accept="image/*"
                >
                  <el-icon class="el-icon--upload"><upload-filled /></el-icon>
                  <div class="el-upload__text">
                    拖拽图片到此处或 <em>点击上传</em>
                  </div>
                  <template #tip>
                    <div class="el-upload__tip">
                      支持 jpg/png 格式图片
                    </div>
                  </template>
                </el-upload>
              </div>

              <!-- 背景颜色选择 -->
              <div class="control-section">
                <h3>2. 选择背景颜色</h3>
                <el-row :gutter="10">
                  <el-col :span="6" v-for="color in backgroundColors" :key="color.name">
                    <div 
                      class="color-option"
                      :class="{ active: selectedBgColor === color.value }"
                      :style="{ backgroundColor: color.value }"
                      @click="selectedBgColor = color.value"
                    >
                      <span class="color-name">{{ color.name }}</span>
                    </div>
                  </el-col>
                </el-row>
                <div style="margin-top: 10px;">
                  <el-color-picker v-model="selectedBgColor" show-alpha />
                  <span style="margin-left: 10px;">自定义颜色</span>
                </div>
              </div>

              <!-- 证件照尺寸选择 -->
              <div class="control-section">
                <h3>3. 选择证件照尺寸</h3>
                <el-select v-model="selectedSize" placeholder="请选择尺寸" style="width: 100%;" value-key="name">
                  <el-option
                    v-for="size in photoSizes"
                    :key="size.name"
                    :label="`${size.name} (${size.width}×${size.height}mm)`"
                    :value="size"
                  />
                </el-select>
              </div>

              <!-- 处理按钮 -->
              <div class="control-section">
                <el-button 
                  type="primary" 
                  size="large" 
                  style="width: 100%;"
                  :loading="processing"
                  :disabled="!originalImage"
                  @click="processImage"
                >
                  <el-icon><Magic /></el-icon>
                  {{ processing ? '处理中...' : '生成证件照' }}
                </el-button>
              </div>

              <!-- 下载按钮 -->
              <div class="control-section" v-if="processedImage">
                <el-button 
                  type="success" 
                  size="large" 
                  style="width: 100%; margin-bottom: 10px;"
                  @click="downloadImage"
                >
                  <el-icon><Download /></el-icon>
                  下载证件照
                </el-button>
                <el-button 
                  type="warning" 
                  size="large" 
                  style="width: 100%;"
                  @click="downloadPrintLayout"
                >
                  <el-icon><Printer /></el-icon>
                  下载打印排版
                </el-button>
              </div>
            </el-card>
          </el-col>

          <!-- 右侧预览区域 -->
          <el-col :span="16">
            <el-card shadow="hover">
              <template #header>
                <span>预览效果</span>
              </template>
              
              <div class="preview-container">
                <!-- 原图预览 -->
                <div class="preview-item" v-show="originalImage">
                  <h4>原图</h4>
                  <div class="image-container">
                    <img :src="originalImage" alt="原图" />
                  </div>
                </div>

                <!-- 处理后预览 -->
                <div class="preview-item" v-show="processedImage">
                  <h4>证件照效果</h4>
                  <div class="image-container">
                    <canvas ref="resultCanvas" style="max-width: 100%; height: auto;"></canvas>
                  </div>
                </div>

                <!-- 打印排版预览 -->
                <div class="preview-item" v-show="printLayoutImage">
                  <h4>打印排版效果</h4>
                  <div class="image-container">
                    <canvas ref="printCanvas" style="max-width: 100%; height: auto;"></canvas>
                  </div>
                </div>

                <!-- 空状态 -->
                <div v-show="!originalImage" class="empty-state">
                  <el-icon size="64" color="#C0C4CC"><Picture /></el-icon>
                  <p>请上传图片开始制作证件照</p>
                </div>
              </div>
            </el-card>
          </el-col>
        </el-row>
      </el-main>
    </el-container>
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
      modelPath = '/IDPhotoGenerator/modnet/model.onnx'
    }
    session.value = await ort.InferenceSession.create(modelPath)
    console.log('ModNet模型加载成功')
  } catch (error) {
    console.error('模型加载失败:', error)
    ElMessage.error('模型加载失败，请检查网络连接')
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

// 下载图片
function downloadImage() {
  if (!processedImage.value) return
  
  const link = document.createElement('a')
  link.download = `证件照_${selectedSize.value.name}_${Date.now()}.png`
  link.href = processedImage.value
  link.click()
}

// 下载打印排版
function downloadPrintLayout() {
  if (!printLayoutImage.value) return
  
  const link = document.createElement('a')
  link.download = `证件照排版_${selectedSize.value.name}_${Date.now()}.png`
  link.href = printLayoutImage.value
  link.click()
}
</script>

<style scoped>
#app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.el-header {
  background-color: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-radius: 0 0 20px 20px;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.control-section {
  margin-bottom: 25px;
}

.control-section h3 {
  color: #409EFF;
  margin-bottom: 15px;
  font-size: 16px;
}

.color-option {
  width: 100%;
  height: 60px;
  border-radius: 8px;
  cursor: pointer;
  border: 2px solid transparent;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  margin-bottom: 10px;
}

.color-option:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.color-option.active {
  border-color: #409EFF;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2);
}

.color-name {
  color: #333;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8);
}

.preview-container {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  justify-content: center;
}

.preview-item {
  flex: 1;
  min-width: 300px;
  text-align: center;
}

.preview-item h4 {
  color: #409EFF;
  margin-bottom: 15px;
}

.image-container {
  border: 2px dashed #E4E7ED;
  border-radius: 8px;
  padding: 20px;
  background-color: #FAFAFA;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.image-container img {
  max-width: 100%;
  max-height: 400px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.empty-state {
  text-align: center;
  color: #909399;
  padding: 60px 20px;
}

.empty-state p {
  margin-top: 16px;
  font-size: 16px;
}

.upload-demo {
  width: 100%;
}

.el-card {
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.el-button {
  border-radius: 8px;
  font-weight: bold;
}

.el-select {
  border-radius: 8px;
}
</style>