<template>
  <div class="app">
    <h1>边框大师</h1>
    <div class="container">
      <div class="upload-section">
        <div class="upload-box" @click="triggerFileInput">
          <input
            ref="fileInput"
            type="file"
            multiple
            accept="image/*"
            @change="handleFileSelect"
            style="display: none"
          />
          <div class="upload-icon">+</div>
          <p>点击上传图片</p>
          <p class="upload-hint">支持JPG、PNG等格式</p>
        </div>
        
        <div v-if="uploadedFiles.length > 0" class="file-list">
          <h3>已上传的图片 ({{ uploadedFiles.length }})</h3>
          <div class="file-items">
            <div v-for="(file, index) in uploadedFiles" :key="index" class="file-item">
              <div class="file-thumbnail">
                <img :src="file.previewUrl" alt="预览" />
                <button @click="removeFile(index)" class="remove-btn">×</button>
              </div>
              <span class="file-name">{{ file.name }}</span>
            </div>
          </div>
        </div>
      </div>

      <div class="settings-section">
        <h2>边框设置</h2>
        <div class="setting-item">
          <label for="borderWidth">边框宽度 (px):</label>
          <input
            id="borderWidth"
            type="range"
            v-model.number="borderWidth"
            min="1"
            max="50"
            step="1"
          />
          <span class="value-display">{{ borderWidth }}</span>
        </div>

        <div class="setting-item">
          <label for="borderColor">边框颜色:</label>
          <input
            id="borderColor"
            type="color"
            v-model="borderColor"
          />
          <span class="color-value">{{ borderColor }}</span>
        </div>

        <div class="setting-item">
          <label for="borderRadius">圆角半径 (px):</label>
          <input
            id="borderRadius"
            type="range"
            v-model.number="borderRadius"
            min="0"
            max="50"
            step="1"
          />
          <span class="value-display">{{ borderRadius }}</span>
        </div>

        <div class="setting-item">
          <label for="borderStyle">边框样式:</label>
          <select id="borderStyle" v-model="borderStyle">
            <option value="solid">实线 (solid)</option>
            <option value="dashed">虚线 (dashed)</option>
            <option value="dotted">点线 (dotted)</option>
            <option value="double">双线 (double)</option>
          </select>
        </div>

        <button class="process-btn" @click="processAllImages" :disabled="uploadedFiles.length === 0">
          应用边框
        </button>

        <button class="download-btn" @click="downloadAllImages" :disabled="processedImages.length === 0">
          下载全部图片
        </button>
      </div>

      <div v-if="uploadedFiles.length > 0" class="preview-section">
        <h2>预览</h2>
        <div class="preview-container">
          <canvas ref="previewCanvas" class="preview-canvas"></canvas>
        </div>
      </div>

      <div v-else class="canvas-container">
        <canvas ref="previewCanvas" class="preview-canvas"></canvas>
      </div>
    </div>
  </div>
</template>

<script>
import JSZip from 'jszip';
import FileSaver from 'file-saver';

export default {
  name: 'App',
  data() {
    return {
      uploadedFiles: [],
      processedImages: [],
      borderWidth: 10,
      borderColor: '#000000',
      borderRadius: 0,
      borderStyle: 'solid'
    };
  },
  methods: {
    // 触发文件选择对话框
    triggerFileInput() {
      this.$refs.fileInput.click();
    },

    // 处理文件选择
    handleFileSelect(event) {
      const files = Array.from(event.target.files);
      files.forEach(file => {
        if (file.type.startsWith('image/')) {
          const reader = new FileReader();
          reader.onload = (e) => {
            this.uploadedFiles.push({
              name: file.name,
              previewUrl: e.target.result,
              file: file
            });
            // 如果是第一张图片，更新预览
            if (this.uploadedFiles.length === 1) {
              this.updatePreview();
            }
          };
          reader.readAsDataURL(file);
        }
      });
      // 清空input，允许重复选择同一文件
      event.target.value = '';
    },

    // 移除文件
    removeFile(index) {
      this.uploadedFiles.splice(index, 1);
      if (this.uploadedFiles.length > 0) {
        this.updatePreview();
      } else {
        this.clearPreview();
      }
    },

    // 更新预览
    updatePreview() {
      const canvas = this.$refs.previewCanvas;
      if (!canvas) return;
      const ctx = canvas.getContext('2d');
      
      if (this.uploadedFiles.length === 0) {
        this.clearPreview();
        return;
      }

      const img = new Image();
      img.onload = () => {
        // 设置canvas尺寸，考虑边框宽度
        const totalWidth = img.width + (this.borderWidth * 2);
        const totalHeight = img.height + (this.borderWidth * 2);
        
        canvas.width = Math.min(totalWidth, 400); // 限制最大宽度
        canvas.height = Math.min(totalHeight, 300); // 限制最大高度
        
        // 清空canvas
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        // 绘制边框背景（用于圆角效果）
        if (this.borderRadius > 0) {
          ctx.fillStyle = this.borderColor;
          ctx.beginPath();
          ctx.roundRect(0, 0, canvas.width, canvas.height, this.borderRadius);
          ctx.fill();
        }
        
        // 绘制边框
        ctx.strokeStyle = this.borderColor;
        ctx.lineWidth = this.borderWidth * 2;
        ctx.setLineDash(this.borderStyle === 'dashed' ? [10, 5] : 
                        this.borderStyle === 'dotted' ? [5, 5] : []);
        ctx.beginPath();
        const radius = this.borderRadius > 0 ? this.borderRadius - this.borderWidth : 0;
        ctx.roundRect(this.borderWidth, this.borderWidth, 
                     canvas.width - (this.borderWidth * 2), 
                     canvas.height - (this.borderWidth * 2), 
                     radius > 0 ? radius : 0);
        ctx.stroke();
        ctx.setLineDash([]); // 重置为实线
        
        // 计算图片缩放比例，确保图片加上边框能完整显示在canvas中
        const scale = Math.min(
          (canvas.width - (this.borderWidth * 2)) / img.width,
          (canvas.height - (this.borderWidth * 2)) / img.height,
          1 // 不放大图片
        );
        
        // 计算图片居中位置
        const imgX = this.borderWidth + (canvas.width - (this.borderWidth * 2) - img.width * scale) / 2;
        const imgY = this.borderWidth + (canvas.height - (this.borderWidth * 2) - img.height * scale) / 2;
        
        // 绘制图片
        ctx.drawImage(img, imgX, imgY, img.width * scale, img.height * scale);
      };
      img.src = this.uploadedFiles[0].previewUrl;
    },

    // 清除预览
    clearPreview() {
      const canvas = this.$refs.previewCanvas;
      if (!canvas) return;
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    },

    // 处理所有图片
    async processAllImages() {
      this.processedImages = [];
      
      for (let i = 0; i < this.uploadedFiles.length; i++) {
        const processedImg = await this.processSingleImage(this.uploadedFiles[i]);
        this.processedImages.push({
          name: this.uploadedFiles[i].name,
          url: processedImg
        });
      }
      
      // 显示处理完成提示
      alert(`已成功为 ${this.processedImages.length} 张图片添加边框！`);
    },

    // 处理单张图片
    processSingleImage(fileObj) {
      return new Promise((resolve) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        const img = new Image();
        
        img.onload = () => {
          // 设置canvas尺寸，考虑边框宽度
          const totalWidth = img.width + (this.borderWidth * 2);
          const totalHeight = img.height + (this.borderWidth * 2);
          
          canvas.width = totalWidth;
          canvas.height = totalHeight;
          
          // 清空canvas
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          
          // 绘制边框背景（用于圆角效果）
          if (this.borderRadius > 0) {
            ctx.fillStyle = this.borderColor;
            ctx.beginPath();
            ctx.roundRect(0, 0, canvas.width, canvas.height, this.borderRadius);
            ctx.fill();
          }
          
          // 绘制边框
          ctx.strokeStyle = this.borderColor;
          ctx.lineWidth = this.borderWidth * 2;
          ctx.setLineDash(this.borderStyle === 'dashed' ? [10, 5] : 
                          this.borderStyle === 'dotted' ? [5, 5] : []);
          ctx.beginPath();
          const radius = this.borderRadius > 0 ? this.borderRadius - this.borderWidth : 0;
          ctx.roundRect(this.borderWidth, this.borderWidth, 
                       canvas.width - (this.borderWidth * 2), 
                       canvas.height - (this.borderWidth * 2), 
                       radius > 0 ? radius : 0);
          ctx.stroke();
          ctx.setLineDash([]); // 重置为实线
          
          // 绘制图片（原图大小）
          ctx.drawImage(img, this.borderWidth, this.borderWidth, img.width, img.height);
          
          // 转换为数据URL
          resolve(canvas.toDataURL('image/png'));
        };
        img.src = fileObj.previewUrl;
      });
    },

    // 下载所有图片
    downloadAllImages() {
      if (this.processedImages.length === 0) {
        alert('没有可下载的图片，请先处理图片！');
        return;
      }
      
      // 如果只有一张图片，直接下载
      if (this.processedImages.length === 1) {
        const link = document.createElement('a');
        link.href = this.processedImages[0].url;
        link.download = this.processedImages[0].name;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        return;
      }
      
      // 多张图片打包下载
      const zip = new JSZip();
      
      this.processedImages.forEach((img, index) => {
        // 从数据URL中提取二进制数据
        const base64Data = img.url.split(',')[1];
        zip.file(img.name, base64Data, { base64: true });
      });
      
      zip.generateAsync({ type: 'blob' }).then((content) => {
        FileSaver.saveAs(content, 'border-images.zip');
      });
    }
  },
  watch: {
    // 监听边框设置变化，实时更新预览
    borderWidth() {
      this.updatePreview();
    },
    borderColor() {
      this.updatePreview();
    },
    borderRadius() {
      this.updatePreview();
    },
    borderStyle() {
      this.updatePreview();
    }
  },
  mounted() {
    // 组件挂载后初始化预览
    this.updatePreview();
  }
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background-color: #f5f5f5;
  color: #333;
  line-height: 1.6;
}

.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  text-align: center;
  margin-bottom: 30px;
  color: #2c3e50;
  font-size: 2.5rem;
}

.container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 30px;
  max-width: 900px;
  margin: 0 auto;
}

@media (max-width: 768px) {
  .container {
    grid-template-columns: 1fr;
  }
}

.upload-section {
  background-color: #fff;
  padding: 25px;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.upload-box {
  border: 2px dashed #ddd;
  border-radius: 8px;
  padding: 40px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background-color: #fafafa;
}

.upload-box:hover {
  border-color: #3498db;
  background-color: #f0f8ff;
}

.upload-icon {
  font-size: 48px;
  color: #999;
  margin-bottom: 10px;
}

.upload-box p {
  color: #666;
}

.upload-hint {
  font-size: 12px;
  color: #999;
  margin-top: 5px;
}

.file-list {
  margin-top: 20px;
}

.file-list h3 {
  margin-bottom: 15px;
  color: #2c3e50;
  font-size: 1.2rem;
}

.file-items {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 10px;
}

.file-item {
  text-align: center;
}

.file-thumbnail {
  position: relative;
  width: 100%;
  height: 80px;
  margin-bottom: 5px;
  border-radius: 4px;
  overflow: hidden;
  border: 1px solid #ddd;
}

.file-thumbnail img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.remove-btn {
  position: absolute;
  top: 5px;
  right: 5px;
  background-color: rgba(255, 0, 0, 0.7);
  color: white;
  border: none;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  cursor: pointer;
  font-size: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.remove-btn:hover {
  background-color: rgba(255, 0, 0, 1);
}

.file-name {
  font-size: 12px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  display: block;
}

.settings-section {
  background-color: #fff;
  padding: 25px;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.settings-section h2 {
  margin-bottom: 20px;
  color: #2c3e50;
  font-size: 1.5rem;
}

.setting-item {
  margin-bottom: 20px;
}

.setting-item label {
  display: block;
  margin-bottom: 8px;
  color: #555;
  font-weight: 500;
}

.setting-item input[type="range"] {
  width: calc(100% - 60px);
  margin-right: 10px;
}

.setting-item input[type="color"] {
  width: 50px;
  height: 30px;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
  vertical-align: middle;
}

.setting-item select {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #fff;
  cursor: pointer;
}

.value-display {
  display: inline-block;
  width: 40px;
  text-align: right;
  font-weight: bold;
  color: #3498db;
}

.color-value {
  margin-left: 10px;
  font-family: monospace;
  color: #3498db;
}

.process-btn,
.download-btn {
  width: 100%;
  padding: 12px;
  margin-top: 10px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
}

.process-btn {
  background-color: #3498db;
  color: white;
}

.process-btn:hover:not(:disabled) {
  background-color: #2980b9;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(52, 152, 219, 0.3);
}

.download-btn {
  background-color: #2ecc71;
  color: white;
}

.download-btn:hover:not(:disabled) {
  background-color: #27ae60;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(46, 204, 113, 0.3);
}

.process-btn:disabled,
.download-btn:disabled {
  background-color: #ccc;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.preview-section {
  grid-column: 1 / -1;
  background-color: #fff;
  padding: 25px;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.preview-section h2 {
  margin-bottom: 20px;
  color: #2c3e50;
  font-size: 1.5rem;
}

.preview-container,
.canvas-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 200px;
  background-color: #fafafa;
  border-radius: 8px;
  padding: 20px;
  border: 1px solid #e0e0e0;
}

.preview-canvas {
  max-width: 100%;
  max-height: 100%;
  border-radius: 4px;
}
</style>