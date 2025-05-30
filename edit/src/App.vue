<template>
  <div id="app">
    <h1>이미지 편집기</h1>

    <!-- 1. 파일 업로드 -->
    <ImageUploader @file-selected="onFileSelected" />

    <!-- 2. 크롭퍼 표시: src가 설정된 경우에만 -->
    <div v-if="imageUrl">
      <ImageCropper
        :src="imageUrl"
        :aspectRatio="1"
        @cropped="onCropped"
      />
    </div>

    <!-- 3. 자른 이미지 미리보기 & 다운로드 -->
    <div v-if="croppedUrl" class="result">
      <h2>자른 이미지</h2>
      <img :src="croppedUrl" alt="Cropped Image" />
      <a :href="croppedUrl" download="cropped.jpg">
        <button>다운로드</button>
      </a>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref } from 'vue';
import ImageUploader from './components/ImageUploader.vue';
import ImageCropper from './components/ImageCropper.vue';

// 업로드된 파일의 URL
const imageUrl = ref<string>('');
// 크롭 후 Blob URL
const croppedUrl = ref<string>('');

// 파일 선택 시 호출
function onFileSelected(file: File) {
  // 기존 URL 해제
  if (imageUrl.value) URL.revokeObjectURL(imageUrl.value);
  // Object URL 생성
  imageUrl.value = URL.createObjectURL(file);
  // 이전 크롭 결과 초기화
  croppedUrl.value = '';
}

// Cropper에서 자른 후 Blob 받으면 URL 생성
function onCropped(blob: Blob) {
  // 기존 Blob URL 해제
  if (croppedUrl.value) URL.revokeObjectURL(croppedUrl.value);
  const url = URL.createObjectURL(blob);
  croppedUrl.value = url;
}
</script>

<style lang="scss">
#app {
  max-width: 600px;
  margin: 2rem auto;
  text-align: center;

  h1 {
    margin-bottom: 1rem;
  }

  .result {
    margin-top: 2rem;

    img {
      max-width: 100%;
      height: auto;
      border: 1px solid #ddd;
      border-radius: 4px;
    }

    button {
      margin-top: 0.5rem;
      padding: 0.5rem 1rem;
      background-color: #409eff;
      color: #fff;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      &:hover {
        background-color: #66b1ff;
      }
    }
  }
}
</style>
