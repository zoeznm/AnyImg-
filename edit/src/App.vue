<template>
  <div id="app">
    <!-- 헤더 -->
    <header class="header">
      <div class="header-content">
        <h1 class="logo">AnyIMG 편집기</h1>
      </div>
    </header>

    <!-- 메인 컨테이너 -->
    <main class="container">
      <!-- 1. 업로드 섹션 -->
      <section v-if="images.length === 0" class="card upload-card">
        <h2 class="section-title">이미지를 업로드하세요</h2>
        <ImageUploader @files-selected="onFilesSelected" />
      </section>

      <!-- 2. 썸네일 목록 -->
      <section v-else class="card thumbnails-card">
        <h2 class="section-title">
          업로드된 이미지 (총 {{ images.length }}장)
        </h2>
        <div class="thumb-list">
          <div
            v-for="(img, idx) in images"
            :key="img.id"
            class="thumb-wrapper"
            :class="{ selected: idx === selectedIndex }"
            @click="selectImage(idx)"
          >
            <img :src="img.url" alt="Thumbnail" class="thumb-img" />
            <p class="thumb-name">{{ img.name }}</p>
          </div>
        </div>
        <button class="btn-secondary reset-all" @click="resetAll">
          전체 초기화
        </button>
      </section>

      <!-- 3. 편집 섹션 -->
      <section v-if="selectedImage" class="card edit-card">
        <h2 class="section-title">편집: {{ selectedImage.name }}</h2>
        <div class="edit-content">
          <!-- 왼쪽: 원본 프리뷰 & 제거 버튼 -->
          <div class="original-preview-wrapper">
            <img
              :src="selectedImage.url"
              alt="Original Preview"
              class="original-preview"
            />
            <p class="format-label">원본 포맷: {{ originalFormatLabel }}</p>
            <button
              class="btn-secondary remove-selected"
              @click="removeImage(selectedIndex)"
            >
              해당 이미지 제거
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
                자르기
              </button>
              <button
                :class="['tab-button', { active: mode === 'convert' }]"
                @click="mode = 'convert'"
              >
                포맷 변환
              </button>
            </div>

            <!-- 자르기 모드 -->
            <div v-if="mode === 'crop'" class="crop-mode">
              <h3 class="sub-title">필터 & 픽셀 지정 크롭</h3>

              <!-- 필터 컨트롤 -->
              <div class="filter-controls">
                <div class="filter-group">
                  <label>밝기: {{ (brightness * 100).toFixed(0) }}%</label>
                  <input
                    type="range"
                    min="0.5"
                    max="1.5"
                    step="0.01"
                    v-model.number="brightness"
                  />
                </div>
                <div class="filter-group">
                  <label>대비: {{ (contrast * 100).toFixed(0) }}%</label>
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
                  <label>가로 픽셀:</label>
                  <input
                    type="number"
                    min="1"
                    v-model.number="widthInput"
                    placeholder="ex) 200"
                  />
                </div>
                <div class="dim-group">
                  <label>세로 픽셀:</label>
                  <input
                    type="number"
                    min="1"
                    v-model.number="heightInput"
                    placeholder="ex) 100"
                  />
                </div>
                <button class="btn-primary apply-btn" @click="applyDimensions">
                  적용
                </button>
              </div>

              <!-- Cropper 영역 (desiredWidth/Height prop 전달) -->
              <div class="cropper-wrapper">
                <ImageCropper
                  :key="selectedImage.url"
                  :src="selectedImage.url"
                  :cropperStyle="cropperStyle"
                  :aspectRatio="null"
                  :desiredWidth="desiredWidth"
                  :desiredHeight="desiredHeight"
                  @ready="onCropReady"
                  @cropped="onCropped"
                />
              </div>

              <!-- 크롭 결과 -->
              <div v-if="croppedBlob" class="crop-result">
                <h4 class="sub-title-sm">크롭된 결과</h4>
                <img
                  :src="croppedUrl"
                  alt="Cropped Preview"
                  class="preview-img"
                />
                <button
                  class="btn-primary download-btn"
                  @click="downloadCropped"
                >
                  크롭 다운로드
                </button>
              </div>
            </div>

            <!-- 포맷 변환 모드 -->
            <div v-if="mode === 'convert'" class="convert-mode">
              <h3 class="sub-title">이미지 포맷 변환</h3>
              <div class="convert-controls">
                <label
                  >변환 포맷:
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
                <button
                  class="btn-primary convert-btn"
                  @click="convertOriginal"
                >
                  변환하기
                </button>
              </div>
              <div v-if="convertedUrl" class="convert-result">
                <button class="sub-title-sm">변환된 결과</button>
                <img
                  :src="convertedUrl"
                  alt="Converted Preview"
                  class="preview-img"
                />
                <a :href="convertedUrl" :download="downloadOriginalName">
                  <button class="btn-primary download-btn">다운로드</button>
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
import { ref, reactive, computed } from "vue";
import ImageUploader from "./components/ImageUploader.vue";
import ImageCropper from "./components/ImageCropper.vue";

// --- 상태 정의 ---
interface ImgItem {
  id: number;
  url: string;
  name: string;
  type: string;
}
const images = ref<ImgItem[]>([]);
const selectedIndex = ref<number | null>(null);

const originalFormat = ref<string>("");

const mode = ref<"crop" | "convert">("crop");
const croppedBlob = ref<Blob | null>(null);
const croppedUrl = ref<string>("");
const cropData = reactive({ width: 0, height: 0 });

const brightness = ref<number>(1);
const contrast = ref<number>(1);

const widthInput = ref<number | null>(null);
const heightInput = ref<number | null>(null);
const desiredWidth = ref<number | null>(null);
const desiredHeight = ref<number | null>(null);

const convertedFormat = ref<string>("");
const convertedUrl = ref<string>("");

// 1) 다중 파일 업로드 핸들러
function onFilesSelected(files: File[]) {
  files.forEach((file, idx) => {
    const id = Date.now() + idx;
    images.value.push({
      id,
      url: URL.createObjectURL(file),
      name: file.name,
      type: file.type,
    });
  });
  if (selectedIndex.value === null && images.value.length > 0) {
    selectImage(0);
  }
}

// 2) 썸네일 클릭 시 선택 (편집 상태 초기화)
function selectImage(idx: number) {
  selectedIndex.value = idx;
  originalFormat.value = images.value[idx].type;

  mode.value = "crop";
  croppedBlob.value = null;
  croppedUrl.value = "";
  convertedUrl.value = "";
  brightness.value = 1;
  contrast.value = 1;
  widthInput.value = null;
  heightInput.value = null;
  desiredWidth.value = null;
  desiredHeight.value = null;
  setConvertedFormat();
}

// 3) Cropper 준비 완료
function onCropReady({ detail }: CustomEvent) {
  cropData.width = detail.width;
  cropData.height = detail.height;
}

// 4) Crop 결과 수신
function onCropped(blob: Blob) {
  if (croppedUrl.value) URL.revokeObjectURL(croppedUrl.value);
  croppedBlob.value = blob;
  croppedUrl.value = URL.createObjectURL(blob);
}

// 5) Crop된 이미지 다운로드
function downloadCropped() {
  if (!croppedUrl.value || selectedIndex.value === null) return;
  const a = document.createElement("a");
  a.href = croppedUrl.value;
  const ext = originalFormat.value.split("/")[1] || "png";
  a.download = `cropped.${ext}`;
  a.click();
}

// 6) 필터 스타일 전달
const cropperStyle = computed(() => ({
  filter: `brightness(${brightness.value}) contrast(${contrast.value})`,
}));

// 7) 변환 가능한 포맷 목록 (원본 제외)
const availableFormats = computed(() =>
  ["image/png", "image/jpeg", "image/webp", "image/jpg"].filter(
    (f) => f !== originalFormat.value
  )
);
function formatLabel(fmt: string) {
  return fmt.split("/")[1].toUpperCase();
}

// 8) 변환 대상 포맷 초기 설정
function setConvertedFormat() {
  const fmts = ["image/png", "image/jpeg", "image/webp", "image/jpg"];
  convertedFormat.value =
    fmts.find((f) => f !== originalFormat.value) || fmts[0];
}

// 9) 포맷 변환
async function convertOriginal() {
  if (selectedIndex.value === null) return;
  const imgItem = images.value[selectedIndex.value];
  const img = new Image();
  img.src = imgItem.url;
  await img.decode();

  const canvas = document.createElement("canvas");
  canvas.width = img.naturalWidth;
  canvas.height = img.naturalHeight;
  const ctx = canvas.getContext("2d")!;
  // 필터 적용하려면 아래 주석 해제
  // ctx.filter = `brightness(${brightness.value}) contrast(${contrast.value})`
  ctx.drawImage(img, 0, 0);

  canvas.toBlob((blob) => {
    if (blob) {
      if (convertedUrl.value) URL.revokeObjectURL(convertedUrl.value);
      convertedUrl.value = URL.createObjectURL(blob);
    }
  }, convertedFormat.value);
}

// 10) 다운로드 파일명 계산
const downloadCroppedName = computed(() => {
  const ext = originalFormat.value.split("/")[1] || "png";
  return `cropped.${ext}`;
});
const downloadOriginalName = computed(() => {
  const ext = convertedFormat.value.split("/")[1] || "img";
  return `converted.${ext}`;
});

const selectedImage = computed<ImgItem | null>(() => {
  if (selectedIndex.value === null) return null;
  return images.value[selectedIndex.value];
});

const originalFormatLabel = computed(() =>
  originalFormat.value ? originalFormat.value.split("/")[1].toUpperCase() : ""
);

// 11) 크롭 박스 크기 적용
function applyDimensions() {
  if (widthInput.value && heightInput.value) {
    desiredWidth.value = widthInput.value;
    desiredHeight.value = heightInput.value;
  }
}

// 12) 이미지 제거
function removeImage(idx: number) {
  URL.revokeObjectURL(images.value[idx].url);
  images.value.splice(idx, 1);
  croppedBlob.value = null;
  croppedUrl.value = "";
  convertedUrl.value = "";
  if (images.value.length === 0) {
    selectedIndex.value = null;
    originalFormat.value = "";
  } else {
    selectedIndex.value =
      idx <= images.value.length - 1 ? idx : images.value.length - 1;
    originalFormat.value = images.value[selectedIndex.value].type;
    brightness.value = 1;
    contrast.value = 1;
    widthInput.value = null;
    heightInput.value = null;
    desiredWidth.value = null;
    desiredHeight.value = null;
    setConvertedFormat();
  }
}

// 13) 전체 초기화
function resetAll() {
  images.value.forEach((img) => URL.revokeObjectURL(img.url));
  if (croppedUrl.value) URL.revokeObjectURL(croppedUrl.value);
  if (convertedUrl.value) URL.revokeObjectURL(convertedUrl.value);
  images.value = [];
  selectedIndex.value = null;
  originalFormat.value = "";
  croppedBlob.value = null;
  croppedUrl.value = "";
  convertedUrl.value = "";
  mode.value = "crop";
  convertedFormat.value = "";
  brightness.value = 1;
  contrast.value = 1;
  widthInput.value = null;
  heightInput.value = null;
  desiredWidth.value = null;
  desiredHeight.value = null;
}
</script>

<style lang="scss" scoped>
/* 전체 배경 */
#app {
  background: #f5f5f5;
  min-height: 100vh;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    "Helvetica Neue", Arial, sans-serif;
  color: #333;
}

/* 헤더 */
.header {
  background: #ffffff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
  position: sticky;
  top: 0;
  z-index: 10;

  .header-content {
    max-width: 800px;
    margin: 0 auto;
    padding: 0.75rem 1rem;
    display: flex;
    align-items: center;
  }

  .logo {
    font-size: 1.25rem;
    font-weight: bold;
    margin: 0;
    color: #409eff;
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

/* 썸네일 목록 */
.thumbnails-card {
  .thumb-list {
    display: flex;
    flex-wrap: nowrap;
    gap: 0.75rem;
    overflow-x: auto;
    padding-bottom: 0.5rem;
    -webkit-overflow-scrolling: touch;
  }

  .thumb-wrapper {
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

  .reset-all {
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

          input[type="range"] {
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

          input[type="number"] {
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
          background: #fff540;
          color: #333;
          border: none;
          border-radius: 4px;
          padding: 0.5rem 1rem;
          margin-top: 4rem;
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
    .thumb-wrapper {
      width: 80px;

      .thumb-name {
        font-size: 0.7rem;
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

          .filter-group input[type="range"] {
            width: 100px;
          }
        }

        .dimension-controls {
          flex-direction: column;
          gap: 0.75rem;

          .dim-group input[type="number"] {
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
