<template>
  <div class="uploader">
    <label for="file-input" class="dropzone">
      파일을 드래그하거나 클릭해 업로드 (여러 장 선택 가능)
      <input
        id="file-input"
        type="file"
        accept="image/*"
        multiple
        @change="onFilesChange"
      />
    </label>
  </div>
</template>

<script lang="ts" setup>
// defineEmits 매크로(별도 import 불필요)
const emit = defineEmits<{
  (e: 'files-selected', files: File[]): void;
}>();

function onFilesChange(e: Event) {
  const input = e.target as HTMLInputElement;
  if (!input.files) return;
  // FileList → Array<File>
  const files = Array.from(input.files);
  emit('files-selected', files);
}
</script>

<style lang="scss" scoped>
.uploader {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 1rem 0;

  .dropzone {
    position: relative;
    width: 300px;
    height: 200px;
    border: 2px dashed #bbb;
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    color: #555;
    transition: border-color 0.2s;

    &:hover {
      border-color: #409eff;
      color: #409eff;
    }

    input[type="file"] {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      opacity: 0;
      cursor: pointer;
    }
  }
}
</style>
