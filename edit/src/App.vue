<template>
  <div id="app">
    <h1>AnyIMG (다중 업로드 + 필터 + 자르기/변환)</h1>

    <!-- 1. 업로드 섹션 -->
    <div v-if="images.length === 0" class="upload-section">
      <ImageUploader @files-selected="onFilesSelected" />
    </div>

    <!-- 2. 이미지 목록 (썸네일) 및 선택 -->
    <div v-else class="thumbnails">
      <h2>업로드된 이미지 (총 {{ images.length }}장)</h2>
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
      <button class="reset-all" @click="resetAll">모두 초기화</button>
    </div>

    <!-- 3. 선택된 이미지가 있을 때만 편집 UI 표시 -->
    <div v-if="selectedImage" class="edit-section">
      <h2>선택된 이미지</h2>
      <img
        :src="selectedImage.url"
        alt="Original Preview"
        class="original-preview"
      />
      <p>원본 포맷: {{ originalFormatLabel }}</p>

      <!-- 모드 탭: 자르기 / 변환 -->
      <div class="tabs">
        <button :class="{ active: mode === 'crop' }" @click="mode = 'crop'">
          자르기
        </button>
        <button
          :class="{ active: mode === 'convert' }"
          @click="mode = 'convert'"
        >
          포맷 변환
        </button>
      </div>

      <!-- 3-1. 자르기 모드 -->
      <div v-if="mode === 'crop'" class="crop-mode">
        <h3>이미지 자르기</h3>
        <!-- 필터 슬라이더 (Brightness / Contrast) -->
        <div class="filter-controls">
          <label>밝기: {{ (brightness * 100).toFixed(0) }}%</label>
          <input
            type="range"
            min="0.5"
            max="1.5"
            step="0.01"
            v-model.number="brightness"
          />
          <label>대비: {{ (contrast * 100).toFixed(0) }}%</label>
          <input
            type="range"
            min="0.5"
            max="1.5"
            step="0.01"
            v-model.number="contrast"
          />
        </div>

        <!-- Cropper.js 래퍼에 key 추가 -->
        <div class="cropper-wrapper">
          <ImageCropper
            :key="selectedImage.url"               
            :src="selectedImage.url"
            :cropperStyle="cropperStyle"
            :aspectRatio="null"
            @ready="onCropReady"
            @cropped="onCropped"
          />
        </div>

        <div v-if="croppedBlob" class="controls">
          <h4>크롭된 미리보기 (필터 적용)</h4>
          <img :src="croppedUrl" alt="Cropped Preview" class="preview" />
          <button @click="downloadCropped">크롭 다운로드</button>
        </div>
      </div>

      <!-- 3-2. 포맷 변환 모드 -->
      <div v-if="mode === 'convert'" class="convert-mode">
        <h3>포맷 변환</h3>
        <p>원본 포맷: {{ originalFormatLabel }}</p>
        <div class="controls">
          <label>변환 포맷:
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
          <h4>변환된 이미지 미리보기</h4>
          <img :src="convertedUrl" alt="Converted Preview" class="preview" />
          <a :href="convertedUrl" :download="downloadOriginalName">
            <button>다운로드</button>
          </a>
        </div>
      </div>

      <!-- 3-3. 이미지 선택 초기화 -->
      <button class="reset-selected" @click="removeImage(selectedIndex)">
        해당 이미지 제거
      </button>
    </div>
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
  // 첫 번째 업로드 시 자동 선택
  if (selectedIndex.value === null && images.value.length > 0) {
    selectImage(0);
  }
}

// 2) 썸네일 클릭 시 선택 (밝기/대비도 리셋)
function selectImage(idx: number) {
  selectedIndex.value = idx;
  originalFormat.value = images.value[idx].type;

  // 모드 및 편집 상태 초기화
  mode.value = "crop";
  croppedBlob.value = null;
  croppedUrl.value = "";
  convertedUrl.value = "";
  brightness.value = 1;    // 밝기 기본값으로 리셋
  contrast.value = 1;      // 대비 기본값으로 리셋
  setConvertedFormat();
}

// 3) Cropper 준비 완료
function onCropReady({ detail }: CustomEvent) {
  cropData.width = detail.width;
  cropData.height = detail.height;
}

// 4) Crop 결과 Blob 수신
function onCropped(blob: Blob) {
  if (croppedUrl.value) URL.revokeObjectURL(croppedUrl.value);
  croppedBlob.value = blob;
  croppedUrl.value = URL.createObjectURL(blob);
}

// 5) Crop된 이미지 다운로드 (원본 포맷 유지)
function downloadCropped() {
  if (!croppedUrl.value || selectedIndex.value === null) return;
  const a = document.createElement("a");
  a.href = croppedUrl.value;
  const ext = originalFormat.value.split("/")[1] || "png";
  a.download = `cropped.${ext}`;
  a.click();
}

// 6) 필터 적용용 스타일 (Cropper에 전달)
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

// 9) 원본 이미지 변환
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
  // 필터를 변환 결과에 적용하려면 아래 주석 해제
  // ctx.filter = `brightness(${brightness.value}) contrast(${contrast.value})`;
  ctx.drawImage(img, 0, 0);

  canvas.toBlob(
    (blob) => {
      if (blob) {
        if (convertedUrl.value) URL.revokeObjectURL(convertedUrl.value);
        convertedUrl.value = URL.createObjectURL(blob);
      }
    },
    convertedFormat.value
  );
}

// 10) Crop 모드용 다운로드 파일명
const downloadCroppedName = computed(() => {
  const ext = originalFormat.value.split("/")[1] || "png";
  return `cropped.${ext}`;
});
// 11) Convert 모드용 다운로드 파일명
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

// 14) 개별 이미지 제거
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
    setConvertedFormat();
  }
}

// 15) 전체 초기화
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
}
</script>

<style lang="scss" scoped>
#app {
  max-width: 800px;
  margin: 2rem auto;
  text-align: center;

  h1 {
    margin-bottom: 1rem;
  }

  /* 1. 업로드 섹션 */
  .upload-section {
    margin: 2rem 0;
  }

  /* 2. 썸네일 목록 */
  .thumbnails {
    margin-bottom: 2rem;

    h2 {
      margin-bottom: 1rem;
    }

    .thumb-list {
      display: flex;
      flex-wrap: wrap;
      gap: 1rem;
      justify-content: center;
    }

    .thumb-wrapper {
      cursor: pointer;
      border: 2px solid transparent;
      border-radius: 4px;
      padding: 0.5rem;
      transition: border-color 0.2s;
      width: 120px;

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
        margin-top: 0.5rem;
        font-size: 0.85rem;
        word-break: break-word;
      }
    }

    .reset-all {
      margin-top: 1rem;
      padding: 0.5rem 1rem;
      background: #ddd;
      border: none;
      border-radius: 4px;
      cursor: pointer;

      &:hover {
        background: #ccc;
      }
    }
  }

  /* 3. 편집 섹션 */
  .edit-section {
    margin-bottom: 2rem;
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

    /* 자르기 모드 */
    .crop-mode {
      margin-bottom: 1.5rem;

      .filter-controls {
        display: flex;
        justify-content: center;
        align-items: center;
        gap: 1rem;
        margin-bottom: 1rem;

        label {
          display: flex;
          align-items: center;
          gap: 0.5rem;
          font-size: 0.9rem;

          input[type="range"] {
            width: 120px;
          }
        }
      }

      .cropper-wrapper {
        max-width: 700px;
        margin: 0 auto;
        height: 400px;
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
    }

    /* 변환 모드 */
    .convert-mode {
      margin-bottom: 1.5rem;

      .controls {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 0.5rem;
        margin-bottom: 1rem;

        label {
          font-size: 0.9rem;
          display: flex;
          align-items: center;
          gap: 0.5rem;

          select {
            padding: 0.3rem;
            border: 1px solid #ccc;
            border-radius: 4px;
          }
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

      .result {
        .preview {
          max-width: 300px;
          border: 1px solid #ddd;
        }

        button {
          margin-top: 0.5rem;
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
    }

    .reset-selected {
      position: absolute;
      top: 0.5rem;
      right: 0.5rem;
      padding: 0.3rem 0.6rem;
      background: #ddd;
      border: none;
      cursor: pointer;

      &:hover {
        background: #ccc;
      }
    }
  }
}
</style>
