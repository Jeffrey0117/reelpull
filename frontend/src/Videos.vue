<script setup>
import { ref, onMounted } from 'vue'

const API_BASE = '/api'
const videos = ref([])
const loading = ref(true)
const saving = ref({})

async function loadVideos() {
  loading.value = true
  try {
    const res = await fetch(`${API_BASE}/videos`)
    videos.value = await res.json()
    // 為每個影片初始化編輯用的名稱（不含副檔名）
    videos.value.forEach(v => {
      const lastDot = v.filename.lastIndexOf('.')
      v.editName = lastDot > 0 ? v.filename.substring(0, lastDot) : v.filename
      v.ext = lastDot > 0 ? v.filename.substring(lastDot) : ''
      v.originalName = v.editName
    })
  } catch (e) {
    console.error('載入失敗', e)
  } finally {
    loading.value = false
  }
}

async function saveRename(video) {
  const newName = video.editName.trim()
  if (!newName || newName === video.originalName) return

  saving.value[video.filename] = true
  try {
    const res = await fetch(
      `${API_BASE}/videos/${encodeURIComponent(video.filename)}/rename?new_name=${encodeURIComponent(newName)}`,
      { method: 'PUT' }
    )
    if (res.ok) {
      const result = await res.json()
      video.filename = result.new_filename
      const lastDot = result.new_filename.lastIndexOf('.')
      video.editName = lastDot > 0 ? result.new_filename.substring(0, lastDot) : result.new_filename
      video.originalName = video.editName
    } else {
      const error = await res.json()
      alert('重命名失敗：' + (error.detail || '未知錯誤'))
      video.editName = video.originalName
    }
  } catch (e) {
    alert('重命名失敗：' + e.message)
    video.editName = video.originalName
  } finally {
    saving.value[video.filename] = false
  }
}

function formatFileSize(bytes) {
  if (!bytes || isNaN(bytes)) return ''
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB'
}

onMounted(() => {
  loadVideos()
})
</script>

<template>
  <div class="container">
    <header class="header">
      <h1>影片管理</h1>
    </header>

    <div v-if="loading" class="loading">
      <div class="spinner"></div>
    </div>

    <div v-else-if="videos.length === 0" class="empty">
      <div class="empty-icon">🎬</div>
      <div>還沒有影片</div>
    </div>

    <div v-else class="video-list">
      <div v-for="video in videos" :key="video.filename" class="video-item">
        <video
          :src="'/api/videos/' + encodeURIComponent(video.filename)"
          controls
          preload="metadata"
        ></video>
        <div class="video-bottom">
          <input
            v-model="video.editName"
            type="text"
            @blur="saveRename(video)"
            @keyup.enter="$event.target.blur()"
            :disabled="saving[video.filename]"
          /><span class="ext">{{ video.ext }}</span>
          <span class="size">{{ formatFileSize(video.size) }}</span>
        </div>
      </div>
    </div>
  </div>
</template>
