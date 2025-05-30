<template>
  <div class="uploader">
    <label for="file-input" class="dropzone">
      파일을 드래그하거나 클릭해 업로드
      <input
        id="file-input"
        type="file"
        accept="image/*"
        @change="onFileChange"
      />
    </label>
  </div>
</template>

<script lang="ts" setup>

const emit = defineEmits<{
  (e: 'file-selected', file: File): void
}>();

function onFileChange(e: Event) {
  const input = e.target as HTMLInputElement;
  const file = input.files?.[0];
  if (file) {
    emit('file-selected', file);
  }
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
