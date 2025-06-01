<template>
  <div class="uploader-wrapper">
    <!-- 숨겨진 실제 파일 입력 -->
    <input
      ref="fileInput"
      type="file"
      accept="image/png, image/jpeg, image/webp, image/jpg"
      multiple
      @change="handleFilesChange"
      class="hidden-file-input"
    />
    <!-- 사용자에게 보일 버튼 -->
    <button class="upload-button" @click="triggerFileInput">
      이미지 선택하기
    </button>
    <p class="note">PNG/JPEG/WEBP/JPG 형식만 업로드 가능합니다 (최대 10MB)</p>
  </div>
</template>

<script lang="ts" setup>
import { ref } from "vue";
import { defineEmits } from "vue";
import imageCompression from "browser-image-compression";

const MAX_SIZE_BYTES = 10 * 1024 * 1024;
const COMPRESS_THRESHOLD = 2 * 1024 * 1024;

const emit = defineEmits<{
  (e: "files-selected", files: File[]): void;
}>();

const fileInput = ref<HTMLInputElement | null>(null);

async function processFile(file: File): Promise<File | null> {
  if (file.size > MAX_SIZE_BYTES) {
    alert(`"${file.name}" 파일 크기가 너무 큽니다. (최대 10MB)`);
    return null;
  }

  const allowedTypes = ["image/png", "image/jpeg", "image/webp", "image/jpg"];
  if (!allowedTypes.includes(file.type)) {
    alert(`"${file.name}" 은(는) 지원하지 않는 포맷입니다.`);
    return null;
  }

  const headerArray = new Uint8Array(8);
  await new Promise<void>((resolve) => {
    const reader = new FileReader();
    reader.onload = () => {
      const result = reader.result as ArrayBuffer;
      const arr = new Uint8Array(result.slice(0, 8));
      headerArray.set(arr);
      resolve();
    };
    reader.readAsArrayBuffer(file.slice(0, 8));
  });

  const isPng =
    headerArray[0] === 0x89 &&
    headerArray[1] === 0x50 &&
    headerArray[2] === 0x4e &&
    headerArray[3] === 0x47;
  const isJpeg = headerArray[0] === 0xff && headerArray[1] === 0xd8 && headerArray[2] === 0xff;
  const isWebp =
    headerArray[0] === 0x52 &&
    headerArray[1] === 0x49 &&
    headerArray[2] === 0x46 &&
    headerArray[3] === 0x46 &&
    headerArray[8 - 4] === 0x57 &&
    headerArray[8 - 3] === 0x45 &&
    headerArray[8 - 2] === 0x42 &&
    headerArray[8 - 1] === 0x50;
  if (!isPng && !isJpeg && !isWebp) {
    alert(`"${file.name}" 의 파일 헤더를 확인할 수 없습니다. 지원되지 않는 포맷입니다.`);
    return null;
  }

  if (file.size > COMPRESS_THRESHOLD) {
    try {
      const options = {
        maxSizeMB: 1,
        maxWidthOrHeight: 1920,
        useWebWorker: true,
      };
      const compressedFile = await imageCompression(file, options);
      return compressedFile;
    } catch (err) {
      console.error("압축 중 오류:", err);
      return file;
    }
  }

  return file;
}

async function handleFilesChange(e: Event) {
  const input = e.target as HTMLInputElement;
  if (!input.files) return;

  const fileList = Array.from(input.files);
  const processed: File[] = [];

  for (const f of fileList) {
    const result = await processFile(f);
    if (result) processed.push(result);
  }

  if (processed.length > 0) {
    emit("files-selected", processed);
  }
  input.value = "";
}

function triggerFileInput() {
  fileInput.value?.click();
}
</script>

<style scoped lang="scss">
.uploader-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 2rem 0;
}

.hidden-file-input {
  display: none;
}

.upload-button {
  background-color: #409eff;
  color: #fff;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 6px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s, box-shadow 0.2s;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);

  &:hover {
    background-color: #66b1ff;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
  }

  &:active {
    background-color: #338ad1;
  }
}

.note {
  margin-top: 0.5rem;
  font-size: 0.85rem;
  color: #666;
  text-align: center;
}
</style>
