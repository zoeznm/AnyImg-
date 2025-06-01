<template>
  <div class="uploader-wrapper">
    <input
      type="file"
      accept="image/png, image/jpeg, image/webp"
      multiple
      @change="handleFilesChange"
    />
  </div>
</template>

<script lang="ts" setup>
import { defineEmits } from "vue";
import imageCompression from "browser-image-compression";

// 업로드 가능한 최대 용량 (바이트). 예: 10MB
const MAX_SIZE_BYTES = 10 * 1024 * 1024;
// 압축 기준 용량 (바이트). 예: 2MB 이상이면 압축
const COMPRESS_THRESHOLD = 2 * 1024 * 1024;

// Emit: 부모(App.vue) 쪽으로 선택된(압축/검사 통과된) File 배열을 보냄
const emit = defineEmits<{
  (e: "files-selected", files: File[]): void;
}>();

/**
 * 실제로 파일을 처리해서 return할 비동기 함수
 * ・용량 제한 체크
 * ・MIME 타입 검사
 * ・압축 후 리턴
 */
async function processFile(file: File): Promise<File | null> {
  // 1) 용량 제한 검사
  if (file.size > MAX_SIZE_BYTES) {
    alert(`"${file.name}" 파일 크기가 너무 큽니다. (최대 10MB)`);
    return null;
  }

  // 2) MIME 타입 검사 (Input accept에 의해 어느 정도 필터링됨)
  const allowedTypes = ["image/png", "image/jpeg", "image/webp"];
  if (!allowedTypes.includes(file.type)) {
    alert(`"${file.name}" 은(는) 지원하지 않는 포맷입니다.`);
    return null;
  }

  // 3) 파일 헤더 검사 (간단히 FileReader로 첫 바이트를 읽어 MIME 헤더가 일치하는지 확인)
  //    ※ 간단 예시이므로, JPEG/PNG/WebP 정도만 체크
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
  // PNG (89 50 4E 47 0D 0A 1A 0A), JPEG (FF D8 FF), WebP ("RIFF" + "WEBP")
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
    headerArray[8 - 4] === 0x57 && // 'W'
    headerArray[8 - 3] === 0x45 && // 'E'
    headerArray[8 - 2] === 0x42 && // 'B'
    headerArray[8 - 1] === 0x50; // 'P'
  if (!isPng && !isJpeg && !isWebp) {
    alert(`"${file.name}" 의 파일 헤더를 확인할 수 없습니다. 지원되지 않는 포맷입니다.`);
    return null;
  }

  // 4) 압축이 필요한 크기일 경우
  if (file.size > COMPRESS_THRESHOLD) {
    try {
      const options = {
        maxSizeMB: 1, // 최대 1MB 이내로 압축
        maxWidthOrHeight: 1920,
        useWebWorker: true,
      };
      const compressedFile = await imageCompression(file, options);
      return compressedFile;
    } catch (err) {
      console.error("압축 중 오류:", err);
      // 압축 실패해도 원본 파일을 계속 사용하도록 함
      return file;
    }
  }

  // 5) 압축 필요 없는 작은 파일은 그대로 반환
  return file;
}

/**
 * input[type=file] 변화 핸들러
 * - multiple로 여러 파일 선택 가능
 * - 각 파일에 대해 processFile() 비동기 호출 → 통과 파일만 모아서 emit
 */
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
  // input 초기화 (동일 파일 재업로드 감지할 때 필요)
  input.value = "";
}
</script>

<style scoped>
.uploader-wrapper {
  display: flex;
  justify-content: center;
  margin: 2rem 0;
}

.uploader-wrapper input[type="file"] {
  cursor: pointer;
}
</style>
