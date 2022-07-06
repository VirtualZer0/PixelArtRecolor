<template>
  <main>
    <h1>PixelArt Recolor</h1>

    <div class="panel">
      <h2>Current palette:</h2>
      <div class="palette" v-if="colors.length">
        <div class="color" v-for="color in colors" :key="color" :style="{backgroundColor: color}"></div>
      </div>
      <div class="empty" v-else>None</div>

      <div class="buttons">
        <div class="input">

          <label class="file-label" for="input">
            <svg viewBox="0 0 24 24" :class="{spin: inputProgress}">

              <path v-if="!inputProgress" fill="currentColor"
                d="M17.5,12A1.5,1.5 0 0,1 16,10.5A1.5,1.5 0 0,1 17.5,9A1.5,1.5 0 0,1 19,10.5A1.5,1.5 0 0,1 17.5,12M14.5,8A1.5,1.5 0 0,1 13,6.5A1.5,1.5 0 0,1 14.5,5A1.5,1.5 0 0,1 16,6.5A1.5,1.5 0 0,1 14.5,8M9.5,8A1.5,1.5 0 0,1 8,6.5A1.5,1.5 0 0,1 9.5,5A1.5,1.5 0 0,1 11,6.5A1.5,1.5 0 0,1 9.5,8M6.5,12A1.5,1.5 0 0,1 5,10.5A1.5,1.5 0 0,1 6.5,9A1.5,1.5 0 0,1 8,10.5A1.5,1.5 0 0,1 6.5,12M12,3A9,9 0 0,0 3,12A9,9 0 0,0 12,21A1.5,1.5 0 0,0 13.5,19.5C13.5,19.11 13.35,18.76 13.11,18.5C12.88,18.23 12.73,17.88 12.73,17.5A1.5,1.5 0 0,1 14.23,16H16A5,5 0 0,0 21,11C21,6.58 16.97,3 12,3Z" />

              <path v-else fill="currentColor" d="M12,4V2A10,10 0 0,0 2,12H4A8,8 0 0,1 12,4Z" />
            </svg>

            {{inputProgress ? 'Extracting...' : 'Add source'}}
          </label>
          <input type="file" id="input" @change="setInput" multiple accept="image/png, image/gif, image/jpeg" />
        </div>

        <button class="round-button" @click="colors.length = 0">
          <svg viewBox="0 0 24 24">
            <path fill="currentColor"
              d="M3.27,1L2,2.27L6.22,6.5L3,9L4.63,10.27L12,16L14.1,14.37L15.53,15.8L12,18.54L4.63,12.81L3,14.07L12,21.07L16.95,17.22L20.73,21L22,19.73L3.27,1M19.36,10.27L21,9L12,2L9.09,4.27L16.96,12.15L19.36,10.27M19.81,15L21,14.07L19.57,12.64L18.38,13.56L19.81,15Z" />
          </svg>
          Clear
        </button>
      </div>

    </div>

    <div class="panel">
      <h2>Converted images:</h2>

      <div class="images" v-if="images.length">
        <img v-for="(image, index) in images" class="image" :key="index" :src="image" @click="downloadImage(image)"/>
      </div>

      <div class="empty" v-else>None</div>

      <div class="buttons">
        <div class="output">
          <label class="file-label" for="output">

            <svg viewBox="0 0 24 24" :class="{spin: outputProgress}">

              <path v-if="!outputProgress" fill="currentColor"
                d="M22,16V4A2,2 0 0,0 20,2H8A2,2 0 0,0 6,4V16A2,2 0 0,0 8,18H20A2,2 0 0,0 22,16M11,12L13.03,14.71L16,11L20,16H8M2,6V20A2,2 0 0,0 4,22H18V20H4V6" />

              <path v-else fill="currentColor" d="M12,4V2A10,10 0 0,0 2,12H4A8,8 0 0,1 12,4Z" />
            </svg>

            {{outputProgress ? 'Converting...' : 'Add images'}}
          </label>
          <input type="file" id="output" @change="setOutput" multiple accept="image/png, image/gif, image/jpeg" />
        </div>

        <button class="round-button" @click="images.length = 0">
          <svg viewBox="0 0 24 24">
            <path fill="currentColor"
              d="M15,16H19V18H15V16M15,8H22V10H15V8M15,12H21V14H15V12M3,18A2,2 0 0,0 5,20H11A2,2 0 0,0 13,18V8H3V18M14,5H11L10,4H6L5,5H2V7H14V5Z" />
          </svg>
          Clear
        </button>
      </div>
    </div>

  </main>
</template>

<script lang="ts">
import { defineComponent, nextTick, ref } from 'vue';
import rgbHex from 'rgb-hex';
import hexRgb, { RgbaObject } from 'hex-rgb';


export default defineComponent({
  name: 'App',
  setup() {

    const outputImg = ref(null);
    const colors = ref<string[]>([]);
    const images = ref<string[]>([]);

    const inputProgress = ref(false);
    const outputProgress = ref(false);

    let cache = new Map<string,RgbaObject>();

    // eslint-disable-next-line @typescript-eslint/no-var-requires
    let nearestHumanColor = require('nearest-human-color').from({});

    // Extract palette from image
    const setInput = (ev: any) => {
      inputProgress.value = true;
      setTimeout(() => {
        Array.prototype.forEach.call(ev.target.files, (file: File) => {
          const reader = new FileReader();
          reader.onload = (ev: any) => {

            const img = new Image();
            img.src = ev.target.result;

            if (img.onload === undefined) {
              return;
            }

            img.onload = (e) => {
              const canvas = document.createElement('canvas');
              canvas.width = img.width;
              canvas.height = img.height;
              const ctx = canvas.getContext('2d');

              if (!ctx) {
                return;
              }

              ctx.drawImage(img, 0, 0);
              const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
              const data = imageData.data;
              const len = data.length;
              for (let i = 0; i < len; i += 4) {
                const r = data[i];
                const g = data[i + 1];
                const b = data[i + 2];
                const a = data[i + 3];
                const color = '#'+rgbHex(r, g, b);
                if (!colors.value.includes(color)) {
                  colors.value.push(color);
                }
              }

              // eslint-disable-next-line @typescript-eslint/no-var-requires
              nearestHumanColor = require('nearest-human-color').from(colors.value);
              cache = new Map<string,RgbaObject>();
            };

          };
          reader.readAsDataURL(file);
        });

        setTimeout(() => {
          inputProgress.value = false;
        }, 150);
      }, 150)
    };

    // Convert images to the current palette
    const setOutput = (ev: any) => {
      outputProgress.value = true;
      setTimeout(() => {

        Array.prototype.forEach.call(ev.target.files, (file: File) => {

          const reader = new FileReader();
          reader.onload = (ev: any) => {
            const img = new Image();
            img.src = ev.target.result;
            img.onload = () => {
              const canvas = document.createElement('canvas');
              canvas.width = img.width;
              canvas.height = img.height;
              const ctx = canvas.getContext('2d');

              if (!ctx) {
                return;
              }

              ctx.drawImage(img, 0, 0);
              const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
              const data = imageData.data;
              const len = data.length;
              for (let i = 0; i < len; i += 4) {
                const r = data[i];
                const g = data[i + 1];
                const b = data[i + 2];
                const a = data[i + 3];
                const color = '#'+rgbHex(r, g, b);

                let nearest = cache.get(color);
                if (!nearest) {
                  nearest = hexRgb(nearestHumanColor(color) as string);
                  cache.set(color, nearest);
                }

                data[i] = nearest.red;
                data[i + 1] = nearest.green;
                data[i + 2] = nearest.blue;
              }

              ctx.putImageData(imageData, 0, 0);
              const res = canvas.toDataURL('image/png', 1);
              images.value.push(res);
            }
          };
          reader.readAsDataURL(file);

        });

        setTimeout(() => {
          outputProgress.value = false;
        }, 150);
      }, 150)
    };

    const downloadImage = (img: string) => {
      const link = document.createElement('a');
      link.href = img;
      link.download = 'image.png';
      link.click();
    };

    return {
      outputImg,
      colors,
      images,
      inputProgress,
      outputProgress,
      setInput,
      setOutput,
      downloadImage
    };

  }
});

</script>

<style>
html, body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  image-rendering: pixelated;
  text-align: center;
  color: #2c3e50;
  margin: 0;
  padding: 0;
}

h1 {
  position: relative;
  padding: 16px;
  margin: 0;
  cursor: default;
  background: linear-gradient(to right, #FF0080, #FF8C00, #40E0D0, #FF0080);
  background-size: 500px 100%;
  color: #fff;
  display: inline-block;
  border-radius: 12px;
  animation: title-gradient 25s linear infinite;
}

h2 {
  margin: 0;
}

.panel {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 12px;
  margin: 0 32px;
  border-radius: 12px;
  margin-bottom: 24px;
}

.palette {
  display: flex;
  flex-wrap: wrap;
  padding: 22px 0;
}

.empty {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 21px;
  height: 64px;
  color: #888;
  margin-bottom: 12px;
}

.color {
  width: 32px;
  height: 32px;
}

input[type=file] {
  opacity: 0;
  display: none;
}

.buttons {
  display: flex;
  gap: 16px;
}

.round-button, .file-label {
  width: 160px;
  padding: 12px 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 3px solid #2c3e50;
  background: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: bold;
  transition: all .2s ease-in-out;
  cursor: pointer;
}

.round-button svg, .file-label svg {
  width: 24px;
  height: 24px;
  fill: #2c3e50;
  margin-right: 6px;
}

.file-label:hover, .round-button:hover {
  background: #2c3e50;
  color: #fff;
}

.spin {
  animation: spin 1s linear infinite;
}

.images {
  margin-top: 24px;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  gap: 16px;
  margin-bottom: 24px;
}

.image {
  object-fit: fill;
  border: 2px solid #2c3e50;
  border-radius: 6px;
  max-width: 350px;
  max-height: 350px;
  cursor: pointer;
  transition: transform .2s ease-in-out;
  background: #fff;
}

.image:hover {
  transform: scale(2.2)
}

@keyframes title-gradient {
  0% {
    background-position-x: 0px;
  }

  100% {
    background-position-x: 500px;
  }
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(359deg);
  }
}
</style>
