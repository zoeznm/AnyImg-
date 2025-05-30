<template>
  <div id="app">
    <h1>이미지 편집 및 포맷 변환기</h1>

    <!-- 업로드 섹션 -->
    <div v-if="!imageUrl" class="upload-section">
      <ImageUploader @file-selected="onFileSelected" />
    </div>

    <!-- 편집 섹션 -->
    <div v-else class="edit-section">
      <h2>원본 이미지</h2>
      <img :src="imageUrl" alt="Original" class="original-preview" />

      <!-- 모드 선택 탭 -->
      <div class="tabs">
        <button
          :class="{ active: mode === 'crop' }"
          @click="mode = 'crop'"
        >
          자르기
        </button>
        <button
          :class="{ active: mode === 'convert' }"
          @click="mode = 'convert'"
        >
          포맷 변환
        </button>
      </div>

      <!-- 자르기 모드 -->
      <div v-if="mode === 'crop'" class="crop-mode">
        <h3>이미지 자르기</h3>
        <div class="cropper-wrapper">
          <ImageCropper
            :src="imageUrl"
            :aspectRatio="null"
            @ready="onCropReady"
            @cropped="onCropped"
          />
        </div>
        <!-- 크롭 컨트롤: 오직 자르기 모드에서만 노출 -->
        <div v-if="croppedBlob" class="controls">
          <img :src="croppedUrl" alt="Cropped Preview" class="preview" />
          <p>크롭 크기: {{ cropData.width }} x {{ cropData.height }}</p>
          <button @click="downloadCropped">크롭 다운로드</button>
        </div>
      </div>

      <!-- 변환 모드 -->
      <div v-if="mode === 'convert'" class="convert-mode">
        <h3>포맷 변환</h3>
        <p>원본 포맷: {{ originalFormatLabel }}</p>
        <div class="controls">
          <label>
            변환 포맷:
            <select v-model="convertedFormat">
              <option
                v-for="fmt in availableFormats"
                :key="fmt"
                :value="fmt"
              >
                {{ formatLabel(fmt) }}
              </option>
            </select>
          </label>
          <button @click="convertOriginal">변환하기</button>
        </div>
        <div v-if="convertedUrl" class="controls">
          <img :src="convertedUrl" alt="Converted Preview" class="preview" />
          <a :href="convertedUrl" :download="downloadOriginalName">
            <button>다운로드</button>
          </a>
        </div>
      </div>

      <!-- 다시 업로드 버튼 -->
      <button class="reset-all" @click="reset">다시 업로드</button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, reactive, computed } from 'vue'
import ImageUploader from './components/ImageUploader.vue'
import ImageCropper from './components/ImageCropper.vue'

// 상태 정의
const imageUrl = ref<string>('')
const originalFormat = ref<string>('')
const mode = ref<'crop' | 'convert'>('crop')

// Crop 상태
const croppedBlob = ref<Blob | null>(null)
const croppedUrl = ref<string>('')
const cropData = reactive({ width: 0, height: 0 })

// Convert 상태
const convertedFormat = ref<string>('')
const convertedUrl = ref<string>('')

/** 파일 업로드 핸들러 */
function onFileSelected(file: File) {
  cleanupAll()
  imageUrl.value = URL.createObjectURL(file)
  originalFormat.value = file.type
  // 변환 포맷은 원본과 다른 것 중 첫 번째
  const formats = ['image/png', 'image/jpeg', 'image/webp', 'image/jpg']
  convertedFormat.value = formats.find(f => f !== file.type) || formats[0]
}

/** Cropper 준비 완료 */
function onCropReady({ detail }: CustomEvent) {
  cropData.width = detail.width
  cropData.height = detail.height
}

/** Blob 형태 크롭 결과 받기 */
function onCropped(blob: Blob) {
  if (croppedUrl.value) URL.revokeObjectURL(croppedUrl.value)
  croppedBlob.value = blob
  croppedUrl.value = URL.createObjectURL(blob)
}

/** 크롭 이미지 다운로드 */
function downloadCropped() {
  if (!croppedUrl.value) return
  const a = document.createElement('a')
  a.href = croppedUrl.value
  const ext = originalFormat.value.split('/')[1] || 'png'
  a.download = `cropped.${ext}`
  a.click()
}

/** 변환 가능한 포맷 목록(원본 제외) */
const availableFormats = computed(() =>
  ['image/png', 'image/jpeg', 'image/webp', 'image/jpg'].filter(f => f !== originalFormat.value)
)

/** 포맷 라벨 */
function formatLabel(fmt: string) {
  return fmt.split('/')[1].toUpperCase()
}

/** 원본 이미지 포맷 변환 */
async function convertOriginal() {
  if (!imageUrl.value) return
  const img = new Image()
  img.src = imageUrl.value
  await img.decode()
  const canvas = document.createElement('canvas')
  canvas.width = img.naturalWidth
  canvas.height = img.naturalHeight
  const ctx = canvas.getContext('2d')!
  ctx.drawImage(img, 0, 0)
  canvas.toBlob(blob => {
    if (blob) {
      if (convertedUrl.value) URL.revokeObjectURL(convertedUrl.value)
      convertedUrl.value = URL.createObjectURL(blob)
    }
  }, convertedFormat.value)
}

/** 변환된 파일 다운로드 이름 */
const downloadOriginalName = computed(() => {
  const ext = convertedFormat.value.split('/')[1] || 'img'
  return `converted.${ext}`
})

/** 초기화 */
function reset() {
  cleanupAll()
}

/** URL 해제 및 상태 reset */
function cleanupAll() {
  if (imageUrl.value) URL.revokeObjectURL(imageUrl.value)
  if (croppedUrl.value) URL.revokeObjectURL(croppedUrl.value)
  if (convertedUrl.value) URL.revokeObjectURL(convertedUrl.value)
  imageUrl.value = ''
  originalFormat.value = ''
  croppedBlob.value = null
  croppedUrl.value = ''
  convertedUrl.value = ''
  mode.value = 'crop'
  convertedFormat.value = ''
}
</script>

<style lang="scss">
#app {
  max-width: 700px;
  margin: 2rem auto;
  text-align: center;

  .upload-section {
    margin: 2rem 0;
  }

  .edit-section {
    margin: 2rem 0;
    position: relative;

    .original-preview {
      max-width: 200px;
      border: 1px solid #ddd;
      margin-bottom: 1rem;
    }

    .tabs {
      display: flex;
      justify-content: center;
      gap: 1rem;
      margin-bottom: 1rem;

      button {
        padding: 0.5rem 1rem;
        border: none;
        cursor: pointer;

        &.active {
          background: #409eff;
          color: #fff;
        }
      }
    }

    .crop-mode,
    .convert-mode {
      margin-bottom: 1.5rem;
    }

    .cropper-wrapper {
      max-width: 600px;
      margin: 0 auto;
    }

    .controls {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 0.5rem;
      margin-top: 1rem;

      .preview {
        max-width: 300px;
        border: 1px solid #ddd;
      }

      button {
        padding: 0.5rem 1rem;
        background: #409eff;
        color: #fff;
        border: none;
        border-radius: 4px;
        cursor: pointer;

        &:hover {
          background: #66b1ff;
        }
      }
    }

    .reset-all {
      position: absolute;
      top: 0.5rem;
      right: 0.5rem;
      padding: 0.3rem 0.6rem;
      background: #ddd;
      border: none;
      cursor: pointer;
    }
  }
}
</style>
