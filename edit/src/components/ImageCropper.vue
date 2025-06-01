<template>
  <div class="image-cropper-wrapper">
    <div :style="cropperStyle" class="cropper-container">
      <vue-cropper
        ref="cropper"
        :src="src"
        :aspect-ratio="aspectRatio"
        :view-mode="1"
        :auto-crop-area="1"
        drag-mode="crop"
        :guides="true"
        :background="false"
        :responsive="true"
        :zoomable="false"      
        :zoomOnWheel="false"  
        :movable="false"      
        class="cropper"
        @ready="onReady"
      />
    </div>
    <button class="crop-button" @click="cropImage">{{ $t('cropImageLabel') }}</button>
  </div>
</template>

<script lang="ts" setup>
import { ref, watch, defineProps, defineEmits } from 'vue'
import VueCropper from 'vue-cropperjs'

const props = defineProps<{
  src: string
  aspectRatio: number | null
  cropperStyle?: Record<string, string>
  desiredWidth: number | null
  desiredHeight: number | null
}>()

const emit = defineEmits<{
  (e: 'cropped', blob: Blob): void
}>()

const cropper = ref<InstanceType<typeof VueCropper> | null>(null)

function onReady() {
  applyDesiredBox()
}

watch(
  () => [props.desiredWidth, props.desiredHeight],
  () => {
    applyDesiredBox()
  }
)

function applyDesiredBox() {
  if (!cropper.value) return
  const dw = props.desiredWidth
  const dh = props.desiredHeight
  if (dw && dh) {
    const imageData = cropper.value.getImageData()
    const left = (imageData.naturalWidth - dw) / 2
    const top = (imageData.naturalHeight - dh) / 2
    cropper.value.setCropBoxData({
      left: left < 0 ? 0 : left,
      top: top < 0 ? 0 : top,
      width: dw,
      height: dh,
    })
  }
}

function cropImage() {
  if (!cropper.value) return
  const canvas = cropper.value.getCroppedCanvas()
  canvas.toBlob((blob) => {
    if (blob) emit('cropped', blob)
  }, 'image/jpeg')
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
  }

  .cropper {
    width: 100%;
    height: 100%;
  }

  .crop-button {
    margin-top: 1rem;
    padding: 0.6rem 1.2rem;
    background-color: #409eff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 0.95rem;
    &:hover {
      background-color: #66b1ff;
    }
  }
}
</style>
