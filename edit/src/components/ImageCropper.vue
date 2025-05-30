<template>
  <div class="image-cropper-wrapper">
    <!-- 픽셀 단위 입력 폼 -->
    <div class="dimensions-inputs">
      <label>
        Width(px):
        <input type="number" v-model.number="widthPx" min="1" />
      </label>
      <label>
        Height(px):
        <input type="number" v-model.number="heightPx" min="1" />
      </label>
      <label>
        Format:
        <select v-model="outputFormat">
          <option value="image/jpeg">JPEG</option>
          <option value="image/png">PNG</option>
        </select>
      </label>
      <button class="apply-button" @click="applyDimensions">적용</button>
    </div>

    <!-- Cropper.js 컴포넌트 (자유 비율) -->
    <vue-cropper
      ref="cropper"
      :src="src"
      view-mode="1"
      :auto-crop-area="1"
      drag-mode="crop"
      :guides="true"
      :background="false"
      :responsive="true"
      class="cropper"
    />

    <!-- 이미지 자르기 버튼 -->
    <button class="crop-button" @click="cropImage">이미지 자르기</button>
  </div>
</template>

<script lang="ts" setup>
import { ref, defineProps, defineEmits } from 'vue';
import VueCropper from 'vue-cropperjs';

// 부모로부터 이미지 URL만 받음
const props = defineProps<{ src: string }>();
const emit = defineEmits<{ (e: 'cropped', blob: Blob, format: string): void }>();

// Cropper 인스턴스
const cropper = ref<InstanceType<typeof VueCropper> | null>(null);

// 원하는 픽셀 단위 크기
const widthPx = ref<number>(100);
const heightPx = ref<number>(100);
// 출력 포맷 결정
const outputFormat = ref<string>('image/jpeg');

// 크롭 박스 크기 설정 (픽셀 단위)
function applyDimensions() {
  if (!cropper.value) return;
  cropper.value.setCropBoxData({ width: widthPx.value, height: heightPx.value });
}

// 실제 이미지 자르기 및 포맷 변환
function cropImage() {
  if (!cropper.value) return;
  const canvas = cropper.value.getCroppedCanvas({ width: widthPx.value, height: heightPx.value });
  canvas.toBlob((blob) => {
    if (blob) emit('cropped', blob, outputFormat.value);
  }, outputFormat.value);
}
</script>

<style lang="scss" scoped>
.image-cropper-wrapper {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1rem;

  .dimensions-inputs {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-bottom: 1rem;

    label {
      font-size: 0.9rem;
      display: flex;
      align-items: center;

      input,
      select {
        margin-left: 0.5rem;
        padding: 0.2rem;
        border: 1px solid #ccc;
        border-radius: 4px;
      }
    }

    .apply-button {
      position: relative;
      z-index: 10;
      padding: 0.4rem 1rem;
      font-size: 0.9rem;
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

  .cropper {
    width: 100%;
    max-width: 500px;
    height: 400px;
  }

  .crop-button {
    position: relative;
    z-index: 10;
    margin-top: 1rem;
    padding: 0.5rem 1rem;
    font-size: 0.9rem;
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
</style>
