<template>
  <div id="app">
    <!-- 헤더 -->
    <header class="header">
      <div class="header-content">
        <h1 class="logo">{{ $t('appTitle') }}</h1>
        <!-- 언어 선택 드롭다운 -->
        <select v-model="currentLocale" @change="changeLocale" class="locale-switcher">
          <option value="ko">한국어</option>
          <option value="en">English</option>
        </select>
      </div>
    </header>

    <!-- 메인 컨테이너 -->
    <main class="container">
      <!-- 1. 업로드 섹션 -->
      <section v-if="images.length === 0" class="card upload-card">
        <h2 class="section-title">{{ $t('uploadPrompt') }}</h2>
        <ImageUploader @files-selected="onFilesSelected" />
      </section>

      <!-- 2. 썸네일 목록 + 배치 처리 -->
      <section v-else class="card thumbnails-card">
        <h2 class="section-title">
          {{ $t('totalImages', { count: images.length }) }}
        </h2>

        <!-- 배치 선택 컨트롤 -->
        <div class="batch-controls">
          <button class="btn-secondary" @click="selectAll">
            {{ allSelected ? $t('deselectAll') : $t('selectAll') }}
          </button>
          <button
            class="btn-primary"
            :disabled="selectedBatch.length === 0"
            @click="applyBatchCrop"
          >
            {{ $t('applyBatchCrop') }}
          </button>
          <button
            class="btn-primary"
            :disabled="selectedBatch.length === 0"
            @click="applyBatchConvert"
          >
            {{ $t('applyBatchConvert') }}
          </button>
          <!-- 일괄 Undo/Redo 버튼 -->
          <button
            class="btn-secondary"
            :disabled="batchUndoDisabled"
            @click="applyBatchUndo"
          >
            {{ $t('undo') }}
          </button>
          <button
            class="btn-secondary"
            :disabled="batchRedoDisabled"
            @click="applyBatchRedo"
          >
            {{ $t('redo') }}
          </button>
        </div>

        <!-- 썸네일 목록 (각 썸네일에 체크박스 추가) -->
        <div class="thumb-list">
          <div
            v-for="(img, idx) in images"
            :key="img.id"
            class="thumb-wrapper"
            :class="{ selected: idx === selectedIndex }"
            @click="selectImage(idx)"
          >
            <input
              type="checkbox"
              class="batch-checkbox"
              :checked="selectedBatch.includes(idx)"
              @click.stop="toggleBatchSelection(idx)"
            />
            <img :src="img.url" alt="Thumbnail" class="thumb-img" />
            <p class="thumb-name">{{ img.name }}</p>
          </div>
        </div>

        <!-- 전체 초기화 + 추가 업로드 -->
        <div class="footer-controls">
          <button class="btn-secondary reset-all" @click="resetAll">
            {{ $t('resetAll') }}
          </button>
          <div class="add-upload">
            <ImageUploader @files-selected="onFilesSelected" />
          </div>
        </div>
      </section>

      <!-- 3. 편집 섹션 -->
      <section v-if="selectedImage" class="card edit-card">
        <h2 class="section-title">
          {{ $t('edit', { name: selectedImage.name }) }}
        </h2>
        <div class="edit-content">
          <!-- 왼쪽: 원본 프리뷰 & 제거 버튼 -->
          <div class="original-preview-wrapper">
            <img
              :src="selectedImage.url"
              alt="Original Preview"
              class="original-preview"
            />
            <p class="format-label">
              {{ $t('originalFormat', { format: originalFormatLabel }) }}
            </p>
            <button
              class="btn-secondary remove-selected"
              @click="removeImage(selectedIndex)"
            >
              {{ $t('removeImage') }}
            </button>
          </div>

          <!-- 오른쪽: 탭 및 에디터 영역 -->
          <div class="editor-wrapper">
            <!-- 모드 탭 -->
            <div class="tabs">
              <button
                :class="['tab-button', { active: mode === 'crop' }]"
                @click="mode = 'crop'"
              >
              {{ $t('cropImageLabel') }}
              </button>
              <button
                :class="['tab-button', { active: mode === 'convert' }]"
                @click="mode = 'convert'"
              >
                {{ $t('tabConvert') }}
              </button>
            </div>

            <!-- 자르기 모드 -->
            <div v-if="mode === 'crop'" class="crop-mode">
              <h3 class="sub-title">{{ $t('tabCrop') }}</h3>

              <!-- Undo / Redo -->
              <div class="history-controls">
                <button
                  class="btn-secondary"
                  :disabled="undoDisabled"
                  @click="undo"
                >
                  {{ $t('undo') }}
                </button>
                <button
                  class="btn-secondary"
                  :disabled="redoDisabled"
                  @click="redo"
                >
                  {{ $t('redo') }}
                </button>
              </div>

              <!-- 필터 컨트롤 -->
              <div class="filter-controls">
                <div class="filter-group">
                  <label>{{ $t('brightness', { percent: (brightness * 100).toFixed(0) }) }}</label>
                  <input
                    type="range"
                    min="0.5"
                    max="1.5"
                    step="0.01"
                    v-model.number="brightness"
                  />
                </div>
                <div class="filter-group">
                  <label>{{ $t('contrast', { percent: (contrast * 100).toFixed(0) }) }}</label>
                  <input
                    type="range"
                    min="0.5"
                    max="1.5"
                    step="0.01"
                    v-model.number="contrast"
                  />
                </div>
              </div>

              <!-- 픽셀 입력 컨트롤 -->
              <div class="dimension-controls">
                <div class="dim-group">
                  <label>{{ $t('widthPixels') }}</label>
                  <input
                    type="number"
                    min="1"
                    v-model.number="widthInput"
                    placeholder="ex) 200"
                  />
                </div>
                <div class="dim-group">
                  <label>{{ $t('heightPixels') }}</label>
                  <input
                    type="number"
                    min="1"
                    v-model.number="heightInput"
                    placeholder="ex) 100"
                  />
                </div>
                <button class="btn-primary apply-btn" @click="applyDimensions">
                  {{ $t('apply') }}
                </button>
              </div>

              <!-- Cropper 영역 -->
              <div class="cropper-wrapper">
                <ImageCropper
                  :key="selectedImage.url + (historyMap[selectedIndex]?.[historyIndexMap[selectedIndex]!] || '')"
                  :src="historyMap[selectedIndex]?.[historyIndexMap[selectedIndex]!] || selectedImage.url"
                  :cropperStyle="cropperStyle"
                  :aspectRatio="null"
                  :desiredWidth="desiredWidth"
                  :desiredHeight="desiredHeight"
                  :zoomable="false"
                  :zoomOnWheel="false"
                  :movable="false"
                  @ready="onCropReady"
                  @cropped="onCropped"
                />
              </div>

              <!-- 크롭 결과 -->
              <div v-if="croppedBlob" class="crop-result">
                <h4 class="sub-title-sm">{{ $t('cropResult') }}</h4>
                <img
                  :src="croppedUrl"
                  alt="Cropped Preview"
                  class="preview-img"
                />
                <button
                  class="btn-primary download-btn"
                  @click="downloadCropped"
                >
                  {{ $t('downloadCrop') }}
                </button>
              </div>
            </div>

            <!-- 포맷 변환 모드 -->
            <div v-if="mode === 'convert'" class="convert-mode">
              <h3 class="sub-title">{{ $t('tabConvert') }}</h3>
              <div class="convert-controls">
                <label>
                  {{ $t('tabConvert') }}:
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
                <button class="btn-primary convert-btn" @click="convertOriginal">
                  {{ $t('apply') }}
                </button>
              </div>
              <div v-if="convertedUrl" class="convert-result">
                <h4 class="sub-title-sm">{{ $t('convertResult') }}</h4>
                <img
                  :src="convertedUrl"
                  alt="Converted Preview"
                  class="preview-img"
                />
                <a :href="convertedUrl" :download="downloadOriginalName">
                  <button class="btn-primary download-btn">
                    {{ $t('downloadConverted') }}
                  </button>
                </a>
              </div>
            </div>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<script lang="ts" setup>
import { ref, reactive, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import ImageUploader from './components/ImageUploader.vue'
import ImageCropper from './components/ImageCropper.vue'

const { locale, t } = useI18n()
const currentLocale = ref(locale.value)
function changeLocale() {
  locale.value = currentLocale.value
}

// --- 상태 정의 ---
interface ImgItem {
  id: number
  url: string
  name: string
  type: string
}
const images = ref<ImgItem[]>([])
const selectedIndex = ref<number | null>(null)
const selectedBatch = ref<number[]>([]) // 배치 처리 대상 인덱스 배열

const originalFormat = ref<string>('')

const mode = ref<'crop' | 'convert'>('crop')
const croppedBlob = ref<Blob | null>(null)
const croppedUrl = ref<string>('')
const cropData = reactive({ width: 0, height: 0 })

const brightness = ref<number>(1)
const contrast = ref<number>(1)

const widthInput = ref<number | null>(null)
const heightInput = ref<number | null>(null)
const desiredWidth = ref<number | null>(null)
const desiredHeight = ref<number | null>(null)

const convertedFormat = ref<string>('')
const convertedUrl = ref<string>('')

// --- 히스토리(Undo/Redo) 관련 상태 ---
// 이미지별 히스토리 맵: idx → URL 배열
const historyMap = reactive<Record<number, string[]>>({})
// 이미지별 히스토리 인덱스 맵: idx → 현재 인덱스
const historyIndexMap = reactive<Record<number, number>>({})

// --- 변환 가능한 포맷 목록 (원본 제외) ---
const availableFormats = computed(() =>
  ['image/png', 'image/jpeg', 'image/webp', 'image/jpg'].filter(
    (f) => f !== originalFormat.value
  )
)
function formatLabel(fmt: string) {
  return fmt.split('/')[1].toUpperCase()
}

// 1) 다중 파일 업로드 핸들러
function onFilesSelected(files: File[]) {
  files.forEach((file, idx) => {
    const id = Date.now() + idx
    const url = URL.createObjectURL(file)
    images.value.push({
      id,
      url,
      name: file.name,
      type: file.type,
    })
    // 히스토리 맵 초기화: 새로 추가된 이미지의 히스토리와 인덱스 세팅
    const newIdx = images.value.length - 1
    historyMap[newIdx] = [url]
    historyIndexMap[newIdx] = 0
  })
  if (selectedIndex.value === null && images.value.length > 0) {
    selectImage(0)
  }
}

// 2) 썸네일 클릭 시 선택 (편집 상태 초기화)
function selectImage(idx: number) {
  selectedIndex.value = idx
  originalFormat.value = images.value[idx].type

  mode.value = 'crop'
  croppedBlob.value = null
  croppedUrl.value = ''
  convertedUrl.value = ''
  brightness.value = 1
  contrast.value = 1
  widthInput.value = null
  heightInput.value = null
  desiredWidth.value = null
  desiredHeight.value = null

  // 히스토리가 historyMap[idx]에 이미 저장되어 있으므로 별도 초기화는 필요 없음
  setConvertedFormat()
}

// 3) 배치 선택 토글
function toggleBatchSelection(idx: number) {
  const i = selectedBatch.value.indexOf(idx)
  if (i === -1) {
    selectedBatch.value.push(idx)
  } else {
    selectedBatch.value.splice(i, 1)
  }
}
const allSelected = computed(() => {
  return (
    images.value.length > 0 &&
    selectedBatch.value.length === images.value.length
  )
})
function selectAll() {
  if (allSelected.value) {
    selectedBatch.value = []
  } else {
    selectedBatch.value = images.value.map((_, i) => i)
  }
}

// 해당 idx에 대해 히스토리 추가
function pushToHistory(idx: number, newUrl: string) {
  const arr = historyMap[idx]
  const currentIdx = historyIndexMap[idx]
  if (currentIdx < arr.length - 1) {
    historyMap[idx] = arr.slice(0, currentIdx + 1)
  }
  historyMap[idx].push(newUrl)
  historyIndexMap[idx] = historyMap[idx].length - 1
}

// 4) 일괄 크롭: CENTER-CROP 로직 + 히스토리 기록
async function applyBatchCrop() {
  if (!desiredWidth.value || !desiredHeight.value) {
    alert('먼저 크롭할 픽셀 값을 입력하고 “적용” 버튼을 눌러주세요.')
    return
  }
  if (selectedBatch.value.length === 0) {
    alert('일괄 크롭할 이미지를 하나 이상 선택해주세요.')
    return
  }

  const w = desiredWidth.value
  const h = desiredHeight.value

  for (const idx of selectedBatch.value) {
    const imgItem = images.value[idx]
    const originalUrl = imgItem.url
    const originalType = imgItem.type

    // 이미지 로드
    const img = new Image()
    img.src = originalUrl
    await img.decode()

    // 캔버스 설정 (center-crop)
    const canvas = document.createElement('canvas')
    canvas.width = w
    canvas.height = h
    const ctx = canvas.getContext('2d')!

    const sx = Math.max((img.naturalWidth - w) / 2, 0)
    const sy = Math.max((img.naturalHeight - h) / 2, 0)
    ctx.drawImage(img, sx, sy, w, h, 0, 0, w, h)

    // 해당 idx 히스토리에 원본 URL 저장
    pushToHistory(idx, originalUrl)

    // 캔버스에서 Blob 생성 → 새 URL 얻기
    const blob = await new Promise<Blob | null>((resolve) => {
      canvas.toBlob((b) => resolve(b), originalType)
    })
    if (blob) {
      // newUrl을 생성하기 전에 기존 URL을 revoke하지 않습니다.
      const newUrl = URL.createObjectURL(blob)
      // 배열에 덮어쓰기
      images.value[idx].url = newUrl

      // 해당 idx 히스토리에 크롭 후 URL 저장
      pushToHistory(idx, newUrl)
    }
  }

  // 첫 번째 선택 이미지를 편집 모드로 전환
  if (selectedBatch.value.length > 0) {
    selectImage(selectedBatch.value[0])
  }
}

// 5) 일괄 변환: 동일 포맷 변환 로직 + 히스토리 기록
async function applyBatchConvert() {
  if (selectedBatch.value.length === 0) {
    alert('일괄 변환할 이미지를 하나 이상 선택해주세요.')
    return
  }

  for (const idx of selectedBatch.value) {
    const imgItem = images.value[idx]
    const originalUrl = imgItem.url
    const originalType = imgItem.type

    const img = new Image()
    img.src = originalUrl
    await img.decode()

    const canvas = document.createElement('canvas')
    canvas.width = img.naturalWidth
    canvas.height = img.naturalHeight
    const ctx = canvas.getContext('2d')!
    ctx.drawImage(img, 0, 0)

    // 해당 idx 히스토리에 원본 URL 저장
    pushToHistory(idx, originalUrl)

    const blob = await new Promise<Blob | null>((resolve) => {
      canvas.toBlob((b) => resolve(b), convertedFormat.value)
    })
    if (blob) {
      // newUrl을 생성하기 전에 기존 URL을 revoke하지 않습니다.
      const newUrl = URL.createObjectURL(blob)
      images.value[idx].url = newUrl
      images.value[idx].type = convertedFormat.value

      // 해당 idx 히스토리에 변환 후 URL 저장
      pushToHistory(idx, newUrl)
    }
  }

  // 첫 번째 선택 이미지를 편집 모드로 전환
  if (selectedBatch.value.length > 0) {
    selectImage(selectedBatch.value[0])
  }
}

// 6) 일괄 Undo
function applyBatchUndo() {
  for (const idx of selectedBatch.value) {
    const currentIdx = historyIndexMap[idx]
    if (currentIdx! > 0) {
      // 현재 URL revoke
      URL.revokeObjectURL(images.value[idx].url)
      // 히스토리 인덱스 감소
      historyIndexMap[idx] = currentIdx! - 1
      // 이전 URL 가져와 복원
      const prevUrl = historyMap[idx]![historyIndexMap[idx]!]
      images.value[idx].url = prevUrl
    }
  }
}

// 7) 일괄 Redo
function applyBatchRedo() {
  for (const idx of selectedBatch.value) {
    const arr = historyMap[idx] || []
    const currentIdx = historyIndexMap[idx]
    if (currentIdx! < arr.length - 1) {
      // 현재 URL revoke
      URL.revokeObjectURL(images.value[idx].url)
      // 히스토리 인덱스 증가
      historyIndexMap[idx] = currentIdx! + 1
      // 다음 URL 가져와 복원
      const nextUrl = historyMap[idx]![historyIndexMap[idx]!]
      images.value[idx].url = nextUrl
    }
  }
}

const batchUndoDisabled = computed(() => {
  return !selectedBatch.value.some(
    (idx) => historyIndexMap[idx]! > 0
  )
})
const batchRedoDisabled = computed(() => {
  return !selectedBatch.value.some(
    (idx) => historyIndexMap[idx]! < (historyMap[idx]!.length - 1)
  )
})

// 8) Cropper 준비 완료
function onCropReady({ detail }: CustomEvent) {
  cropData.width = detail.width
  cropData.height = detail.height
}

// 9) Crop 결과 수신


// 10) Crop된 이미지 다운로드
function downloadCropped() {
  if (!croppedUrl.value || selectedIndex.value === null) return
  const a = document.createElement('a')
  a.href = croppedUrl.value
  const ext = originalFormat.value.split('/')[1] || 'png'
  a.download = `cropped.${ext}`
  a.click()
}
// ─────────────────────────────────────────────────────────────────────────
// (1) 히스토리 초기화: 업로드 시 한 번만 호출해야 함
//    // images.value[idx].url 은 File → createObjectURL(file)로 생성된 원본 URL
//    historyMap[newIdx] = [ images.value[newIdx].url ]
//    historyIndexMap[newIdx] = 0
// ─────────────────────────────────────────────────────────────────────────

// ─────────────────────────────────────────────────────────────────────────
// (2) onCropped: “한 개 이미지 크롭” 후
//    • 절대 기존 URL을 revoke하지 않는다.
//    • 단, 히스토리에 원본 URL(또는 마지막 URL)만 추가하고,
//      새로운 Blob URL을 만들어 바로 덮어쓴 뒤 히스토리에 추가한다.
// ─────────────────────────────────────────────────────────────────────────
function onCropped(blob: Blob) {
  // ① 크롭된 Blob으로 임시 URL 생성
  const newUrl = URL.createObjectURL(blob);
  croppedBlob.value = blob;
  croppedUrl.value = newUrl;

  if (selectedIndex.value === null) return;
  const idx = selectedIndex.value;

  // ② 현재(히스토리 상의) URL을 히스토리에 (한 번) 추가
  //    (historyIndexMap[idx]가 0이고 historyMap[idx]이 [원본URL]이라면,
  //     여기서 그 원본 URL을 또 한 번 쌓는 게 아니라, 이미 ⓐ원본URL이 0번째인 상태이므로
  //     곧바로 새 URL을 추가해도 무방합니다. 하지만 pushToHistory 내부에서
  //     “인덱스 < 내용 길이-1”을 자른 뒤 추가하므로 중복 걱정은 없습니다.)
  pushToHistory(idx, images.value[idx].url);

  // ③ 배열에 덮어쓰기: 실제 화면에 표시될 URL을 ‘크롭된 Blob URL’로 교체
  images.value[idx].url = newUrl;
  images.value[idx].type = images.value[idx].type; // 포맷은 원본 유지

  // ④ 히스토리에 ‘크롭 후 URL’도 저장
  pushToHistory(idx, newUrl);

  // ⚠️ 절대 기존 URL(images.value[idx].url)을 여기서 revokeObjectURL 하지 마세요.
  //    Undo할 때 원본 URL이 유효해야 합니다.
}

// ─────────────────────────────────────────────────────────────────────────
// (3) convertOriginal: “한 개 이미지 포맷 변환” 후
//    • 역시 기존 URL을 revoke하지 않는다.
//    • Canvas → Blob URL 생성, URL만 교체하고 히스토리에 추가한다.
// ─────────────────────────────────────────────────────────────────────────
async function convertOriginal() {
  if (selectedIndex.value === null) return;
  const idx = selectedIndex.value;
  const imgItem = images.value[idx];
  const originalUrl = imgItem.url;

  const img = new Image();
  img.src = originalUrl;
  await img.decode();

  const canvas = document.createElement('canvas');
  canvas.width = img.naturalWidth;
  canvas.height = img.naturalHeight;
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(img, 0, 0);

  // ① 히스토리에 변환 전 URL(현재 URL)만 추가
  pushToHistory(idx, originalUrl);

  // ② Canvas → Blob → 새 URL 생성
  canvas.toBlob((blob) => {
    if (!blob) return;
    const newUrl = URL.createObjectURL(blob);

    // ③ 배열에 덮어쓰기: 화면 표시용 URL만 새로 교체
    images.value[idx].url = newUrl;
    images.value[idx].type = convertedFormat.value;

    // ④ 변환 후 새 URL을 히스토리에 추가
    pushToHistory(idx, newUrl);

    // ⚠️ 기존 originalUrl(히스토리 첫 항목)은 revoke 하지 않는다.
  }, convertedFormat.value);
}

// ─────────────────────────────────────────────────────────────────────────
// (4) 단일 이미지 Undo/Redo
//    • Undo: 현재 URL을 revoke 하지 않고, 히스토리 인덱스를 1 감소시킨 뒤
//      oldUrl = historyMap[idx][newIndex]를 배열에 넣어준다.
//    • Redo: 동일하게 히스토리 인덱스를 1 증가시킨 뒤 해당 URL만 교체.
//    • 절대 images.value.splice 등을 사용하지 않는다! (배열에서 삭제 금지)
// ─────────────────────────────────────────────────────────────────────────
// 11) Undo / Redo (한 이미지)
const undoDisabled = computed(() => {
  if (selectedIndex.value === null) return true;
  return historyIndexMap[selectedIndex.value]! <= 0;
});
const redoDisabled = computed(() => {
  if (selectedIndex.value === null) return true;
  const arr = historyMap[selectedIndex.value] || [];
  return historyIndexMap[selectedIndex.value]! >= arr.length - 1;
});

function undo() {
  if (selectedIndex.value === null) return;
  const idx = selectedIndex.value;
  const currentIdx = historyIndexMap[idx];
  if (currentIdx! > 0) {
    // 1) 현재 URL을 revoke
    URL.revokeObjectURL(images.value[idx].url);

    // 2) 히스토리 인덱스를 1 감소
    historyIndexMap[idx] = currentIdx! - 1;

    // 3) 히스토리에서 이전 URL을 가져와 복원
    const prevUrl = historyMap[idx]![historyIndexMap[idx]!];
    images.value[idx].url = prevUrl;
  }
}

function redo() {
  if (selectedIndex.value === null) return;
  const idx = selectedIndex.value;
  const arr = historyMap[idx] || [];
  const currentIdx = historyIndexMap[idx];
  if (currentIdx! < arr.length - 1) {
    // 1) 현재 URL을 revoke
    URL.revokeObjectURL(images.value[idx].url);

    // 2) 히스토리 인덱스를 1 증가
    historyIndexMap[idx] = currentIdx! + 1;

    // 3) 히스토리에서 다음 URL을 가져와 복원
    const nextUrl = historyMap[idx]![historyIndexMap[idx]!];
    images.value[idx].url = nextUrl;
  }
}

// 12) 필터 스타일 전달
const cropperStyle = computed(() => ({
  filter: `brightness(${brightness.value}) contrast(${contrast.value})`,
}))



// 14) 다운로드 파일명 계산
const downloadCroppedName = computed(() => {
  const ext = originalFormat.value.split('/')[1] || 'png'
  return `cropped.${ext}`
})
const downloadOriginalName = computed(() => {
  const ext = convertedFormat.value.split('/')[1] || 'img'
  return `converted.${ext}`
})

const selectedImage = computed<ImgItem | null>(() => {
  if (selectedIndex.value === null) return null
  return images.value[selectedIndex.value]
})

const originalFormatLabel = computed(() =>
  originalFormat.value ? originalFormat.value.split('/')[1].toUpperCase() : ''
)

// 15) 크롭 박스 크기 적용
function applyDimensions() {
  if (widthInput.value && heightInput.value) {
    desiredWidth.value = widthInput.value
    desiredHeight.value = heightInput.value
  }
}

// 16) 이미지 제거
function removeImage(idx: number) {
  URL.revokeObjectURL(images.value[idx].url)
  images.value.splice(idx, 1)
  croppedBlob.value = null
  croppedUrl.value = ''
  convertedUrl.value = ''
  delete historyMap[idx]
  delete historyIndexMap[idx]

  if (images.value.length === 0) {
    selectedIndex.value = null
    originalFormat.value = ''
    // 히스토리 맵 초기화
    Object.keys(historyMap).forEach((key) => delete historyMap[+key])
    Object.keys(historyIndexMap).forEach((key) => delete historyIndexMap[+key])
    selectedBatch.value = []
  } else {
    selectedIndex.value =
      idx <= images.value.length - 1 ? idx : images.value.length - 1
    originalFormat.value = images.value[selectedIndex.value].type
    brightness.value = 1
    contrast.value = 1
    widthInput.value = null
    heightInput.value = null
    desiredWidth.value = null
    desiredHeight.value = null
    // 선택된 이미지 인덱스에 해당하는 히스토리 유지
    setConvertedFormat()
    selectedBatch.value = selectedBatch.value.filter((i) => i !== idx)
  }
}

// 17) 전체 초기화
function resetAll() {
  images.value.forEach((img) => URL.revokeObjectURL(img.url))
  if (croppedUrl.value) URL.revokeObjectURL(croppedUrl.value)
  if (convertedUrl.value) URL.revokeObjectURL(convertedUrl.value)
  images.value = []
  selectedIndex.value = null
  originalFormat.value = ''
  croppedBlob.value = null
  croppedUrl.value = ''
  convertedUrl.value = ''
  mode.value = 'crop'
  convertedFormat.value = ''
  brightness.value = 1
  contrast.value = 1
  widthInput.value = null
  heightInput.value = null
  desiredWidth.value = null
  desiredHeight.value = null
  selectedBatch.value = []
  // 히스토리 맵 초기화
  Object.keys(historyMap).forEach((key) => delete historyMap[+key])
  Object.keys(historyIndexMap).forEach((key) => delete historyIndexMap[+key])
}

// 18) 변환 대상 포맷 초기 설정
function setConvertedFormat() {
  const fmts = ['image/png', 'image/jpeg', 'image/webp', 'image/jpg']
  convertedFormat.value =
    fmts.find((f) => f !== originalFormat.value) || fmts[0]
}
</script>

<style lang="scss" scoped>
/* 전체 배경 */
#app {
  background: #f5f5f5;
  min-height: 100vh;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    'Helvetica Neue', Arial, sans-serif;
  color: #333;
}

/* 헤더 */
.header {
  background: #ffffff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  justify-content: space-between;

  .header-content {
    max-width: 800px;
    margin: 0 auto;
    padding: 0.75rem 1rem;
    width: 100%;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .logo {
    font-size: 1.25rem;
    font-weight: bold;
    margin: 0;
    color: #409eff;
  }

  .locale-switcher {
    background: #f0f0f0;
    border: none;
    padding: 0.3rem;
    border-radius: 4px;
    font-size: 0.9rem;
    cursor: pointer;
  }
}

/* 메인 컨테이너 */
.container {
  max-width: 800px;
  margin: 1rem auto 2rem;
  padding: 0 1rem;
}

/* 카드 공통 */
.card {
  background: #ffffff;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  margin-bottom: 1.5rem;
  padding: 1.25rem;
}

/* 섹션 제목 */
.section-title {
  margin: 0 0 1rem;
  font-size: 1.2rem;
  font-weight: 500;
  color: #333;
}

/* 버튼 스타일 */
.btn-primary,
.btn-secondary {
  font-size: 0.9rem;
  border-radius: 4px;
  transition: background 0.2s;
}

.btn-primary {
  background: #409eff;
  color: #fff;
  border: none;
  padding: 0.5rem 1rem;
  cursor: pointer;

  &:hover {
    background: #66b1ff;
  }
}

.btn-secondary {
  background: #e0e0e0;
  color: #333;
  border: none;
  padding: 0.4rem 0.8rem;
  cursor: pointer;

  &:hover {
    background: #cacaca;
  }
}

/* 썸네일 목록 + 배치 처리 */
.thumbnails-card {
  .batch-controls {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-bottom: 1rem;

    button {
      flex: none;
    }
  }

  .thumb-list {
    display: flex;
    flex-wrap: nowrap;
    gap: 0.75rem;
    overflow-x: auto;
    padding-bottom: 0.5rem;
    -webkit-overflow-scrolling: touch;
  }

  .thumb-wrapper {
    position: relative;
    flex: 0 0 auto;
    width: 100px;
    cursor: pointer;
    border: 2px solid transparent;
    border-radius: 4px;
    padding: 0.3rem;
    transition: border-color 0.2s;
    text-align: center;

    &.selected {
      border-color: #409eff;
    }

    .batch-checkbox {
      position: absolute;
      top: 4px;
      left: 4px;
      z-index: 10;
      transform: scale(1.2);
    }

    .thumb-img {
      width: 100%;
      height: auto;
      border: 1px solid #ddd;
      border-radius: 4px;
    }

    .thumb-name {
      margin-top: 0.3rem;
      font-size: 0.75rem;
      word-break: break-word;
    }
  }

  .footer-controls {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 1rem;

  }
}

/* 편집 섹션 */
.edit-content {
  display: flex;
  gap: 1.5rem;

  /* 왼쪽: 원본 프리뷰 */
  .original-preview-wrapper {
    flex: 0 0 180px;
    display: flex;
    flex-direction: column;
    align-items: center;

    .original-preview {
      width: 100%;
      border: 1px solid #ddd;
      border-radius: 4px;
      margin-bottom: 0.75rem;
    }

    .format-label {
      font-size: 0.9rem;
      margin-bottom: 0.75rem;
      color: #555;
    }

    .remove-selected {
      width: 100%;
    }
  }

  /* 오른쪽: 탭 & 편집 UI */
  .editor-wrapper {
    flex: 1;
    display: flex;
    flex-direction: column;

    .tabs {
      display: flex;
      gap: 1rem;
      margin-bottom: 1rem;

      .tab-button {
        flex: 1;
        text-align: center;
        padding: 0.5rem;
        background: #f0f0f0;
        border: none;
        border-radius: 4px;
        font-size: 0.95rem;
        cursor: pointer;
        transition: background 0.2s;

        &.active {
          background: #409eff;
          color: #fff;
        }

        &:hover {
          background: #e0e0e0;
        }
      }
    }

    /* 자르기 모드 */
    .crop-mode {
      .sub-title {
        margin-bottom: 0.75rem;
        font-size: 1rem;
        font-weight: 500;
        color: #333;
      }

      /* Undo/Redo 버튼 */
      .history-controls {
        display: flex;
        gap: 0.5rem;
        margin-bottom: 1rem;

        button {
          padding: 0.4rem 0.8rem;
          background: #e0e0e0;
          border: none;
          border-radius: 4px;
          font-size: 0.9rem;
          color: #333;
          cursor: pointer;
          transition: background 0.2s;

          &:disabled {
            background: #f5f5f5;
            color: #aaa;
            cursor: not-allowed;
          }

          &:not(:disabled):hover {
            background: #cacaca;
          }
        }
      }

      .filter-controls {
        display: flex;
        flex-wrap: wrap;
        gap: 1rem;
        margin-bottom: 1rem;

        .filter-group {
          display: flex;
          flex-direction: column;
          align-items: flex-start;
          gap: 0.25rem;
          font-size: 0.9rem;

          input[type='range'] {
            width: 120px;
          }
        }
      }

      /* 픽셀 입력 컨트롤 */
      .dimension-controls {
        display: flex;
        gap: 1rem;
        margin-bottom: 1rem;
        flex-wrap: wrap;

        .dim-group {
          display: flex;
          flex-direction: column;
          gap: 0.25rem;
          font-size: 0.9rem;

          input[type='number'] {
            width: 100px;
            padding: 0.25rem;
            border: 1px solid #ccc;
            border-radius: 4px;
            font-size: 0.9rem;
          }
        }

        .apply-btn {
          align-self: flex-end;
          padding: 0.4rem 0.8rem;
        }
      }

      .cropper-wrapper {
        width: 100%;
        height: 400px;
        margin-bottom: 1rem;
      }

      .crop-result {
        display: flex;
        flex-direction: column;
        align-items: center;

        .sub-title-sm {
          margin-bottom: 0.5rem;
          font-size: 0.95rem;
          font-weight: 500;
          color: #333;
        }

        .preview-img {
          width: 100%;
          max-width: 300px;
          border: 1px solid #ddd;
          border-radius: 4px;
          margin-bottom: 0.75rem;
        }

        .download-btn {
          width: 100%;
        }
      }
    }

    /* 변환 모드 */
    .convert-mode {
      .sub-title {
        margin-bottom: 0.75rem;
        font-size: 1rem;
        font-weight: 500;
        color: #333;
      }

      .convert-controls {
        display: flex;
        flex-wrap: wrap;
        gap: 1rem;
        margin-bottom: 1rem;

        label {
          font-size: 0.9rem;
          display: flex;
          flex-direction: column;
          align-items: flex-start;
          gap: 0.25rem;

          select {
            padding: 0.3rem;
            border: 1px solid #ccc;
            border-radius: 4px;
            font-size: 0.9rem;
          }
        }

        .convert-btn {
          align-self: flex-start;
        }
      }

      .convert-result {
        display: flex;
        flex-direction: column;
        align-items: center;

        .sub-title-sm {
          margin-bottom: 0.5rem;
          font-size: 0.95rem;
          font-weight: 500;
        }

        .preview-img {
          width: 100%;
          max-width: 300px;
          border: 1px solid #ddd;
          border-radius: 4px;
          margin-bottom: 0.75rem;
        }

        .download-btn {
          width: 100%;
        }
      }
    }
  }

  .reset-selected {
    position: absolute;
    top: 1rem;
    right: 1rem;
    padding: 0.4rem 0.8rem;
    font-size: 0.85rem;
  }
}

/* 모바일 대응 (600px 이하) */
@media screen and (max-width: 600px) {
  .container {
    margin: 1rem auto;
    padding: 0 0.5rem;
  }

  /* 썸네일 */
  .thumbnails-card {
    .batch-controls {
      flex-direction: column;
      gap: 0.75rem;
      margin-bottom: 1rem;
    }

    .thumb-wrapper {
      width: 80px;

      .thumb-name {
        font-size: 0.7rem;
      }

      .batch-checkbox {
        transform: scale(1);
      }
    }

    .reset-all {
      font-size: 0.8rem;
    }
  }

  /* 편집 섹션을 세로로 */
  .edit-content {
    flex-direction: column;
    align-items: center;
    gap: 1rem;

    .original-preview-wrapper {
      width: 100%;
      max-width: 180px;

      .original-preview {
        max-width: 100%;
      }
    }

    .editor-wrapper {
      width: 100%;

      .tabs {
        gap: 0.5rem;

        .tab-button {
          padding: 0.4rem 0.6rem;
          font-size: 0.85rem;
        }
      }

      .crop-mode {
        .filter-controls {
          flex-direction: column;
          gap: 0.5rem;

          .filter-group input[type='range'] {
            width: 100px;
          }
        }

        .dimension-controls {
          flex-direction: column;
          gap: 0.75rem;

          .dim-group input[type='number'] {
            width: 100px;
            font-size: 0.85rem;
          }

          .apply-btn {
            width: 100%;
            font-size: 0.85rem;
          }
        }

        .cropper-wrapper {
          height: 300px;
        }

        .crop-result .preview-img {
          max-width: 250px;
        }
      }

      .convert-mode {
        .convert-controls {
          flex-direction: column;
          gap: 0.75rem;

          label select {
            font-size: 0.85rem;
          }

          .convert-btn {
            width: 100%;
            font-size: 0.85rem;
          }
        }

        .convert-result .preview-img {
          max-width: 250px;
        }
      }
    }

    .reset-selected {
      top: 0.5rem;
      right: 0.5rem;
      font-size: 0.75rem;
      padding: 0.3rem 0.6rem;
    }
  }
}
</style>
