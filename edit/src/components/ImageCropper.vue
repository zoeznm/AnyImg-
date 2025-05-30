<template>
  <div class="cropper-container">
    <vue-cropper
      ref="cropper"
      :src="src"
      :aspect-ratio="aspectRatio"
      :view-mode="1"
      :auto-crop-area="1"
      :background="false"
      :responsive="true"
      class="cropper"
    />
    <button @click="cropImage">이미지 자르기</button>
  </div>
</template>

<script lang="ts" setup>
import { ref, defineProps, defineEmits } from 'vue';
import VueCropper from 'vue-cropperjs';
import 'cropperjs/dist/cropper.css';

// 부모로부터 받아올 프로퍼티
const props = defineProps<{
  src: string;
  aspectRatio?: number;
}>();

// 'cropped' 이벤트로 Blob을 상위 컴포넌트에 전달
const emit = defineEmits<{
  (e: 'cropped', blob: Blob): void;
}>();

// Cropper 인스턴스를 참조하기 위한 ref
const cropper = ref<InstanceType<typeof VueCropper> | null>(null);
const aspectRatio = props.aspectRatio ?? 1;

function cropImage() {
  if (!cropper.value) return;
  const canvas = cropper.value.getCroppedCanvas();
  canvas.toBlob((blob) => {
    if (blob) emit('cropped', blob);
  }, 'image/jpeg');
}
</script>

<style lang="scss" scoped>
.cropper-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1rem;

  .cropper {
    width: 100%;
    max-width: 500px;
    height: 400px;
  }

  button {
    margin-top: 1rem;
    padding: 0.5rem 1rem;
    border: none;
    background-color: #409eff;
    color: white;
    border-radius: 4px;
    cursor: pointer;
    &:hover {
      background-color: #66b1ff;
    }
  }
}
</style>
