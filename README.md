# AnyIMG

**이미지 편집 및 포맷 변환기**

웹 브라우저에서 간편하게 이미지 **자르기(Crop)** 와 **포맷 변환(Convert)** 기능을 제공하는 Vue 3 기반 SPA 프로젝트입니다.

---

## 💻데모

---

## 🔍 주요 기능

- **이미지 업로드**  
  - 드래그&드롭 또는 파일 선택  
- **자르기(Crop) 모드**  
  - 자유 비율 크롭  
  - 실시간 미리보기  
  - 크롭된 영역의 픽셀 크기 표시  
  - 자른 결과를 원본 포맷(`png`, `jpeg`, `webp`)으로 다운로드  
- **포맷 변환(Convert) 모드**  
  - 자동 원본 포맷 감지  
  - `PNG`, `JPEG`, `WEBP` 중 원본과 다른 포맷 선택  
  - HTML5 Canvas 기반 변환  
  - 변환 결과 다운로드  

---

## 🛠️ 기술 스택
- **프레임워크: Vue 3 (Composition API, <script setup>)**  
- **빌드 도구: Vite**  
- **언어: TypeScript**  
- **스타일링: SCSS**  
- **라이브러리: Cropper.js**  
- **웹 API: HTML5 Canvas (이미지 변환)**
  
---


## 🚀 설치 및 실행

```bash
# 저장소 복제
git clone https://github.com/USERNAME/AnyIMG.git
cd AnyIMG

# 의존성 설치
npm install

# 개발 서버 실행
npm run dev

---
