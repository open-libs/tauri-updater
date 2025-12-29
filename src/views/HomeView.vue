<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { check } from '@tauri-apps/plugin-updater'
import { relaunch } from '@tauri-apps/plugin-process'
import { getVersion } from '@tauri-apps/api/app'

const currentVersion = ref('')
const updateStatus = ref('')
const updateProgress = ref(0)
const isChecking = ref(false)
const isDownloading = ref(false)
const hasUpdate = ref(false)
const newVersion = ref('')
const downloadedBytes = ref(0)
const totalBytes = ref(0)

onMounted(async () => {
  try {
    currentVersion.value = await getVersion()
  } catch (e) {
    currentVersion.value = '开发模式'
  }
})

async function checkForUpdate() {
  isChecking.value = true
  updateStatus.value = '正在检查更新...'
  
  try {
    const update = await check()
    
    if (update) {
      hasUpdate.value = true
      newVersion.value = update.version
      updateStatus.value = `发现新版本: ${update.version}`
      
      // 开始下载和安装
      isDownloading.value = true
      updateStatus.value = '正在下载更新...'
      
      await update.downloadAndInstall((event) => {
        switch (event.event) {
          case 'Started':
            totalBytes.value = event.data.contentLength || 0
            updateStatus.value = `开始下载 (${formatBytes(totalBytes.value)})`
            break
          case 'Progress':
            downloadedBytes.value += event.data.chunkLength
            if (totalBytes.value > 0) {
              updateProgress.value = Math.round((downloadedBytes.value / totalBytes.value) * 100)
            }
            updateStatus.value = `下载中: ${updateProgress.value}% (${formatBytes(downloadedBytes.value)}/${formatBytes(totalBytes.value)})`
            break
          case 'Finished':
            updateStatus.value = '下载完成，准备重启...'
            break
        }
      })
      
      updateStatus.value = '更新已安装，即将重启应用...'
      await new Promise(resolve => setTimeout(resolve, 2000))
      await relaunch()
    } else {
      updateStatus.value = '当前已是最新版本'
      hasUpdate.value = false
    }
  } catch (e) {
    updateStatus.value = `检查更新失败: ${e}`
    console.error('Update check failed:', e)
  } finally {
    isChecking.value = false
    isDownloading.value = false
  }
}

function formatBytes(bytes: number): string {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}
</script>

<template>
  <div class="container">
    <header>
      <h1>🚀 Tauri Updater Demo</h1>
      <p class="version">当前版本: <strong>{{ currentVersion }}</strong></p>
    </header>

    <main>
      <div class="card">
        <h2>自动更新功能演示</h2>
        <p>此应用演示了 Tauri 2.0 的自动更新功能，基于 GitHub Releases 进行版本分发。</p>
        
        <div class="update-section">
          <button 
            @click="checkForUpdate" 
            :disabled="isChecking || isDownloading"
            class="update-btn"
          >
            {{ isChecking ? '检查中...' : isDownloading ? '下载中...' : '检查更新' }}
          </button>
          
          <div v-if="updateStatus" class="status-box" :class="{ 'has-update': hasUpdate }">
            <p>{{ updateStatus }}</p>
            <div v-if="isDownloading && totalBytes > 0" class="progress-bar">
              <div class="progress" :style="{ width: updateProgress + '%' }"></div>
            </div>
          </div>
        </div>
      </div>

      <div class="card info-card">
        <h3>功能说明</h3>
        <ul>
          <li>点击"检查更新"按钮检测 GitHub 上的新版本</li>
          <li>如果有新版本，会自动下载并安装</li>
          <li>安装完成后应用会自动重启</li>
        </ul>
      </div>
    </main>

    <footer>
      <p>Powered by Tauri 2.0 + Vue 3</p>
    </footer>
  </div>
</template>

<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 2rem;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

header h1 {
  font-size: 2rem;
  margin-bottom: 0.5rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.version {
  color: #666;
  font-size: 0.9rem;
}

.version strong {
  color: #333;
  font-family: monospace;
  background: #f0f0f0;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
}

main {
  flex: 1;
}

.card {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  margin-bottom: 1rem;
}

.card h2 {
  margin-top: 0;
  color: #333;
}

.card p {
  color: #666;
  line-height: 1.6;
}

.update-section {
  margin-top: 1.5rem;
}

.update-btn {
  width: 100%;
  padding: 1rem 2rem;
  font-size: 1.1rem;
  font-weight: 600;
  color: white;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}

.update-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.update-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.status-box {
  margin-top: 1rem;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #667eea;
}

.status-box.has-update {
  border-left-color: #28a745;
  background: #f0fff4;
}

.status-box p {
  margin: 0;
  font-size: 0.95rem;
}

.progress-bar {
  margin-top: 0.75rem;
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.progress {
  height: 100%;
  background: linear-gradient(90deg, #667eea, #764ba2);
  transition: width 0.3s ease;
}

.info-card {
  background: #f8f9fa;
}

.info-card h3 {
  margin-top: 0;
  color: #333;
  font-size: 1rem;
}

.info-card ul {
  margin: 0;
  padding-left: 1.5rem;
  color: #666;
}

.info-card li {
  margin-bottom: 0.5rem;
  line-height: 1.5;
}

footer {
  text-align: center;
  padding-top: 2rem;
  color: #999;
  font-size: 0.85rem;
}
</style>
