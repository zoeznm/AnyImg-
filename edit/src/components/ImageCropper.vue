<template>
  <div class="image-cropper-wrapper">
    <!-- cropperStyle prop을 상위에서 받아서 스타일로 바인딩 -->
    <div :style="cropperStyle" class="cropper-container">
      <vue-cropper
        ref="cropper"
        :src="src"
        :aspect-ratio="aspectRatio"
        :view-mode="1"
        :auto-crop-area="1"
        :drag-mode="'crop'"
        :guides="true"
        :background="false"
        :responsive="true"
        class="cropper"
      />
    </div>
    <button class="crop-button" @click="cropImage">이미지 자르기</button>
  </div>
</template>

<script lang="ts" setup>
import { ref, defineProps, defineEmits } from "vue";
import VueCropper from "vue-cropperjs";

// Props: src, aspectRatio, 그리고 새로 추가된 cropperStyle
const props = defineProps<{
  src: string;
  aspectRatio: number | null;
  cropperStyle?: Record<string, string>;
}>();

const emit = defineEmits<{
  (e: "cropped", blob: Blob): void;
}>();

const cropper = ref<InstanceType<typeof VueCropper> | null>(null);

function cropImage() {
  if (!cropper.value) return;
  const canvas = cropper.value.getCroppedCanvas();
  canvas.toBlob((blob) => {
    if (blob) emit("cropped", blob);
  }, "image/jpeg");
}

function onReady(e: CustomEvent) {
  // 필요 시 ready 이벤트 활용 (이미 App.vue에서 handle)
}
</script>

<style lang="scss" scoped>
.image-cropper-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1rem;

  .cropper-container {
    width: 100%;
    max-width: 700px;
    height: 400px;
    /* 이제 App.vue에서 전달한 filter(cropperStyle)가 적용됩니다 */
  }

  .cropper {
    width: 100%;
    height: 100%;
  }

  .crop-button {
    margin-top: 1rem;
    padding: 0.5rem 1rem;
    background-color: #409eff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    &:hover {
      background-color: #66b1ff;
    }
  }
}
</style>
