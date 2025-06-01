<template>
  <div class="image-cropper-wrapper">
    <!-- Cropper 컨테이너에 cropperStyle을 적용하고, ref로 인스턴스를 잡아둠 -->
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
        @ready="onReady"
      />
    </div>
    <button class="crop-button" @click="cropImage">이미지 자르기</button>
  </div>
</template>

<script lang="ts" setup>
import { ref, watch, defineProps, defineEmits } from 'vue'
import VueCropper from 'vue-cropperjs'

// 부모로부터 받을 props 선언
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

// Cropper.js 인스턴스를 잡을 ref
const cropper = ref<InstanceType<typeof VueCropper> | null>(null)

/** ready 이벤트 핸들러:
 *  - 초기 로딩 시 점점 크롭 박스가 잡히면, 부모가 원하는 크기가 있다면 반영
 */
function onReady() {
  applyDesiredBox()
}

/** 부모로부터 받은 desiredWidth/Height가 바뀔 때마다 호출 */
watch(
  () => [props.desiredWidth, props.desiredHeight],
  () => {
    applyDesiredBox()
  }
)

/** 실제로 Cropper.js 인스턴스에 setCropBoxData를 호출하여 크롭 박스 조정 */
function applyDesiredBox() {
  if (!cropper.value) return
  const dw = props.desiredWidth
  const dh = props.desiredHeight
  if (dw && dh) {
    // 현재 이미지의 natural 크기를 가져옴
    const imageData = cropper.value.getImageData()
    // 중앙에 박스를 위치시키기 위해 left/top 계산
    const left = (imageData.naturalWidth - dw) / 2
    const top = (imageData.naturalHeight - dh) / 2

    // Cropper.js API 호출: setCropBoxData
    cropper.value.setCropBoxData({
      left: left < 0 ? 0 : left,
      top: top < 0 ? 0 : top,
      width: dw,
      height: dh,
    })
  }
}

/** “이미지 자르기” 버튼 클릭 시, getCroppedCanvas → Blob 생성하여 부모에 emit */
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
    /* App.vue에서 전달한 filter가 적용됨 */
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
