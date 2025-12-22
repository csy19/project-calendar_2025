<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { ObjectDetector, FilesetResolver } from '@mediapipe/tasks-vision';

const imageFile = ref(null);
const imageUrl = ref('');
const predictions = ref([]);
const detector = ref(null);
const isLoading = ref(false);
const isDetecting = ref(false);
const isAddingLegs = ref(false); // 足を生やす処理中
const canvasRef = ref(null);
const numLegs = ref(10);
const excludedIndices = ref(new Set());
const selectedLegPattern = ref('01');
const errorMessage = ref(''); // エラーメッセージ

// 画像サイズ制限（10MB）
const MAX_FILE_SIZE = 10 * 1024 * 1024;
const MAX_IMAGE_DIMENSION = 4096;

// 足パターンの定義（重み付き）
const legPatterns = [
  { id: '01', label: '足1', files: [{ name: '01', weight: 1 }] },
  { id: '02', label: '足2', files: [{ name: '02', weight: 1 }] },
  { id: '03', label: '足3', files: [{ name: '03', weight: 1 }] },
  { id: '04', label: '足4', files: [{ name: '04', weight: 1 }] },
  { id: '05', label: '足5', files: [{ name: '05', weight: 1 }] },
  {
    id: '6-8',
    label: '🎲 6-8',
    files: [
      { name: '06', weight: 1 },
      { name: '07', weight: 1 },
      { name: '08', weight: 0.2 },
    ],
  },
  {
    id: '9-10',
    label: '🎲 9-10',
    files: [
      { name: '09', weight: 1 },
      { name: '10', weight: 0.2 },
    ],
  },
  {
    id: 'all',
    label: '🎲 全部',
    files: [
      { name: '01', weight: 1 },
      { name: '02', weight: 1 },
      { name: '03', weight: 1 },
      { name: '04', weight: 1 },
      { name: '05', weight: 1 },
      { name: '06', weight: 1 },
      { name: '07', weight: 1 },
      { name: '08', weight: 0.2 },
      { name: '09', weight: 1 },
      { name: '10', weight: 0.2 },
    ],
  },
];

onMounted(async () => {
  isLoading.value = true;
  errorMessage.value = '';

  try {
    const vision = await FilesetResolver.forVisionTasks(
      'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm'
    );

    detector.value = await ObjectDetector.createFromOptions(vision, {
      baseOptions: {
        modelAssetPath:
          'https://storage.googleapis.com/mediapipe-models/object_detector/efficientdet_lite0/float16/1/efficientdet_lite0.tflite',
      },
      scoreThreshold: 0.5,
      runningMode: 'IMAGE',
    });
  } catch (error) {
    console.error('Error loading MediaPipe:', error);
    errorMessage.value =
      '物体検出モデルの読み込みに失敗しました。ページを再読み込みしてください。';
  } finally {
    isLoading.value = false;
  }
});

// クリーンアップ
onUnmounted(() => {
  if (imageUrl.value) {
    URL.revokeObjectURL(imageUrl.value);
  }
});

const handleImageUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  errorMessage.value = '';

  // ファイルサイズチェック
  if (file.size > MAX_FILE_SIZE) {
    errorMessage.value = '画像サイズが大きすぎます（最大10MB）';
    return;
  }

  // 画像の解像度チェック
  const img = new Image();
  const objectUrl = URL.createObjectURL(file);

  img.onload = () => {
    if (img.width > MAX_IMAGE_DIMENSION || img.height > MAX_IMAGE_DIMENSION) {
      errorMessage.value = `画像サイズが大きすぎます（最大${MAX_IMAGE_DIMENSION}×${MAX_IMAGE_DIMENSION}px）`;
      URL.revokeObjectURL(objectUrl);
      return;
    }

    // 古いURLをクリーンアップ
    if (imageUrl.value) {
      URL.revokeObjectURL(imageUrl.value);
    }

    imageFile.value = file;
    imageUrl.value = objectUrl;
    predictions.value = [];
    excludedIndices.value.clear();
  };

  img.onerror = () => {
    errorMessage.value = '画像の読み込みに失敗しました';
    URL.revokeObjectURL(objectUrl);
  };

  img.src = objectUrl;
};

const detectObjects = async () => {
  if (!imageUrl.value || !detector.value || isDetecting.value) return;

  isDetecting.value = true;
  errorMessage.value = '';

  try {
    const img = document.getElementById('uploaded-image');

    if (!img.complete || img.naturalWidth === 0) {
      isDetecting.value = false;
      return;
    }

    const result = detector.value.detect(img);
    predictions.value = result.detections;

    if (predictions.value.length === 0) {
      errorMessage.value =
        '物体が検出されませんでした。別の画像を試してみてください。';
    }

    drawImageWithButtons();
  } catch (error) {
    console.error('Error in detectObjects:', error);
    errorMessage.value = '物体検出に失敗しました';
  } finally {
    isDetecting.value = false;
  }
};

const drawImageWithButtons = () => {
  const canvas = canvasRef.value;
  const img = document.getElementById('uploaded-image');
  const ctx = canvas.getContext('2d');

  canvas.width = img.naturalWidth;
  canvas.height = img.naturalHeight;
  ctx.drawImage(img, 0, 0);

  predictions.value.forEach((detection, index) => {
    const box = detection.boundingBox;

    if (excludedIndices.value.has(index)) {
      ctx.strokeStyle = '#ccc';
      ctx.lineWidth = 2;
      ctx.setLineDash([5, 5]);
    } else {
      ctx.strokeStyle = 'red';
      ctx.lineWidth = 3;
      ctx.setLineDash([]);
    }

    ctx.strokeRect(box.originX, box.originY, box.width, box.height);

    const btnSize = 40;
    const btnX = box.originX + box.width - btnSize - 5;
    const btnY = box.originY + 5;

    ctx.fillStyle = excludedIndices.value.has(index) ? '#ccc' : 'red';
    ctx.fillRect(btnX, btnY, btnSize, btnSize);

    ctx.strokeStyle = 'white';
    ctx.lineWidth = 3;
    ctx.setLineDash([]);
    ctx.beginPath();
    ctx.moveTo(btnX + 8, btnY + 8);
    ctx.lineTo(btnX + btnSize - 8, btnY + btnSize - 8);
    ctx.moveTo(btnX + btnSize - 8, btnY + 8);
    ctx.lineTo(btnX + 8, btnY + btnSize - 8);
    ctx.stroke();
  });
};

const handleCanvasClick = (event) => {
  const canvas = canvasRef.value;
  const rect = canvas.getBoundingClientRect();

  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  const x = (event.clientX - rect.left) * scaleX;
  const y = (event.clientY - rect.top) * scaleY;

  predictions.value.forEach((detection, index) => {
    const box = detection.boundingBox;
    const btnSize = 40;
    const btnX = box.originX + box.width - btnSize - 5;
    const btnY = box.originY + 5;

    if (x >= btnX && x <= btnX + btnSize && y >= btnY && y <= btnY + btnSize) {
      if (excludedIndices.value.has(index)) {
        excludedIndices.value.delete(index);
      } else {
        excludedIndices.value.add(index);
      }
      drawImageWithButtons();
    }
  });
};

const getRandomLegImage = () => {
  const pattern = legPatterns.find((p) => p.id === selectedLegPattern.value);
  const totalWeight = pattern.files.reduce((sum, file) => sum + file.weight, 0);
  let random = Math.random() * totalWeight;

  for (const file of pattern.files) {
    random -= file.weight;
    if (random <= 0) {
      return `images/ashi/ashi${file.name}.png`;
    }
  }

  return `images/ashi/ashi${pattern.files[0].name}.png`;
};

const addLegs = async () => {
  if (isAddingLegs.value) return;

  isAddingLegs.value = true;
  errorMessage.value = '';

  try {
    const canvas = canvasRef.value;
    const img = document.getElementById('uploaded-image');
    const ctx = canvas.getContext('2d');

    ctx.drawImage(img, 0, 0);

    // 並列で足画像を読み込み
    const legImagePromises = [];
    const legData = [];

    for (const [index, detection] of predictions.value.entries()) {
      if (excludedIndices.value.has(index)) continue;

      const box = detection.boundingBox;
      const bottomY = box.originY + box.height - 10;
      const legWidth = box.width / numLegs.value;

      for (let i = 0; i < numLegs.value; i++) {
        const x = box.originX + i * legWidth;
        const legImgSrc = getRandomLegImage();

        const promise = new Promise((resolve, reject) => {
          const legImg = new Image();
          legImg.onload = () => {
            const aspectRatio = legImg.height / legImg.width;
            const legHeight = legWidth * aspectRatio;
            const shouldFlip = Math.random() > 0.5;

            resolve({ legImg, x, bottomY, legWidth, legHeight, shouldFlip });
          };
          legImg.onerror = () =>
            reject(new Error('足画像の読み込みに失敗しました'));
          legImg.src = legImgSrc;
        });

        legImagePromises.push(promise);
      }
    }

    // 全ての画像を並列で読み込み
    const allLegs = await Promise.all(legImagePromises);

    // 描画
    allLegs.forEach(
      ({ legImg, x, bottomY, legWidth, legHeight, shouldFlip }) => {
        if (shouldFlip) {
          ctx.save();
          ctx.translate(x + legWidth, bottomY);
          ctx.scale(-1, 1);
          ctx.drawImage(legImg, 0, 0, legWidth, legHeight);
          ctx.restore();
        } else {
          ctx.drawImage(legImg, x, bottomY, legWidth, legHeight);
        }
      }
    );
  } catch (error) {
    console.error('Error adding legs:', error);
    errorMessage.value = '足を生やす処理に失敗しました';
  } finally {
    isAddingLegs.value = false;
  }
};

const downloadImage = () => {
  try {
    const canvas = canvasRef.value;
    const link = document.createElement('a');
    link.download = 'legs-image.png';
    link.href = canvas.toDataURL();
    link.click();
  } catch (error) {
    console.error('Error downloading image:', error);
    errorMessage.value = '画像のダウンロードに失敗しました';
  }
};
</script>

<template>
  <div class="app">
    <div v-if="isLoading">
      <div class="loading-text">よみこみちゅう</div>
      <div class="loading-image">
        <img src="/images/loading.png" alt="loading" />
      </div>
    </div>

    <!-- エラーメッセージ -->
    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>

    <section class="pick-image" v-if="!imageUrl && !isLoading">
      <h1 class="top-image">
        <img src="/images/top.png" alt="あなたの画像に足を生やす あしはや" />
      </h1>

      <div class="upload-section">
        <label class="file-upload">
          <input type="file" accept="image/*" @change="handleImageUpload" />
        </label>
      </div>
    </section>

    <section v-if="imageUrl" class="settings">
      <div class="image-section">
        <div class="upload-image-wrap">
          <div class="upload-image">
            <img
              id="uploaded-image"
              :src="imageUrl"
              @load="detectObjects"
              style="display: none"
            />

            <!-- 検出中ローディング -->
            <div v-if="isDetecting" class="detecting-overlay">
              <div class="loading-text">物体を検出中...</div>
              <div class="loading-image">
                <img src="/images/loading.png" alt="loading" />
              </div>
            </div>

            <canvas
              ref="canvasRef"
              @click="handleCanvasClick"
              style="max-width: 100%; cursor: pointer"
            />
          </div>
          <div class="sampling-items" v-if="predictions.length > 0">
            <p>検出された物体:</p>
            <ul>
              <li v-for="(pred, index) in predictions" :key="index">
                {{ pred.categories[0].categoryName }}
                (信頼度: {{ (pred.categories[0].score * 100).toFixed(1) }}%)
                <span v-if="excludedIndices.has(index)" style="color: #999">
                  - 除外中</span
                >
              </li>
            </ul>
          </div>
        </div>

        <div v-if="predictions.length > 0" class="predictions">
          <h3>あしが生えそうな物体: {{ predictions.length }}個</h3>
          <p style="font-size: 12px; color: #666">
            ※あしを生やしたくないものは✗ボタンをクリックすると除外できるよ
          </p>

          <div class="leg-pattern-selector">
            <h4>好みのあしを選ぼう</h4>
            <div class="pattern-params">
              <button
                v-for="pattern in legPatterns"
                :key="pattern.id"
                :class="{ active: selectedLegPattern === pattern.id }"
                @click="selectedLegPattern = pattern.id"
                class="pattern-btn"
              >
                <img
                  v-if="pattern.files.length === 1"
                  :src="`images/ashi/ashi${pattern.files[0].name}.png`"
                  :alt="pattern.label"
                />
                <span v-else>{{ pattern.label }}</span>
              </button>
              <div class="leg-control">
                <label>
                  あしの本数: {{ numLegs }}本
                  <input
                    type="range"
                    v-model.number="numLegs"
                    min="2"
                    max="100"
                  />
                </label>
              </div>
            </div>
          </div>

          <div class="add-legs-button">
            <button @click="addLegs" :disabled="isAddingLegs">
              <img
                v-if="!isAddingLegs"
                src="/images/button_01.svg"
                alt="あしを生やすボタン"
              />
              <div v-else class="loading-text">あしを生やしています...</div>
            </button>
          </div>
          <div class="dl-button">
            <button @click="downloadImage" v-if="predictions.length > 0">
              <img src="/images/dl_button.png" alt="ダウンロード" />
            </button>
          </div>
        </div>
      </div>
    </section>
    <p class="copyright">&copy; 2025 csy</p>
  </div>
</template>

<style scoped>
.app {
  position: relative;
}

h3,
h4 {
  font-family: 'Yusei Magic', sans-serif;
}

.error-message {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: #ff6b6b;
  color: white;
  padding: 1rem 2rem;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  z-index: 1000;
  max-width: 90%;
  text-align: center;
}

.detecting-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.9);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

.upload-section {
  margin: 2rem 0;
}

.pick-image {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
}

.top-image {
  width: calc(727 / 1440 * 100%);
  max-width: 727px;
  min-width: 500px;
  img {
    width: 100%;
  }
}

.file-upload {
  display: inline-block;
  width: 450px;
  height: 100px;
  cursor: pointer;
  background-image: url('/images/button_01.svg');
  background-repeat: no-repeat;
  background-size: contain;
  &:hover {
    opacity: 0.7;
  }
  input {
    display: none;
  }
}

.settings {
  width: calc(1280 / 1440 * 100%);
  max-width: 1280px;
  margin: 60px auto;
  padding: 50px;
  border: 2px solid #3c3c3c;
  border-radius: 40px;
}

.image-section {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 0 60px;
}

.upload-image {
  position: relative;
  grid-area: 0 / 1;
  overflow: hidden;
  border-radius: 20px;
}

.leg-pattern-selector {
  margin-top: 1rem;
}

.leg-pattern-selector h4 {
  margin: 0 0 0.5rem 0;
}

.pattern-params {
  display: grid;
  grid-template-columns: repeat(5, auto);
  grid-template-rows: repeat(3, auto);
  gap: 0.5rem;
  justify-content: center;
  padding: 30px;
  border-radius: 10px;
  background: #f0f0f0;
}

.pattern-btn {
  width: 60px;
  height: 60px;
  padding: 0.25rem;
  border: 2px solid #ddd;
  background: white;
  cursor: pointer;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.pattern-btn img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.pattern-btn.active {
  border-color: red;
}

.leg-control {
  margin-top: 1rem;
  grid-area: 3 / 1 / 4 / 6;
}

.leg-control label {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.add-legs-button {
  width: 90%;
  margin: 30px auto 0;
  img {
    width: 100%;
  }
}

.dl-button {
  width: 169px;
  margin: 30px auto 0;
  img {
    width: 100%;
  }
}

button {
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background-color: transparent;
}

button:hover {
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.predictions {
  max-width: 400px;
}

.sampling-items {
  font-size: 12px;
  ul {
    list-style: none;
    padding: 0;
  }

  li {
    padding: 6px 0 0;
  }
}

input[type='range'] {
  width: 100%;
}

.loading-image {
  width: 100px;
  height: 100px;
  animation: loading 1s linear infinite;
  img {
    display: block;
    width: 100%;
  }
}

.copyright {
  text-align: center;
}

@keyframes loading {
  from {
    rotate: 0deg;
  }
  to {
    rotate: -360deg;
  }
}

/* モバイル対応 */
@media (max-width: 768px) {
  .settings {
    width: 95%;
    padding: 20px;
  }

  .image-section {
    grid-template-columns: 1fr;
  }

  .top-image {
    min-width: 300px;
  }

  .file-upload {
    width: 300px;
    height: 70px;
  }

  .pattern-params {
    grid-template-columns: repeat(3, auto);
    padding: 15px;
  }

  .leg-control {
    grid-area: auto;
  }
}
</style>
