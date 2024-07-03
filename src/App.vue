<script setup>
import {
  ref,
  shallowRef,
  onMounted,
  onUnmounted,
  nextTick,
  computed,
} from 'vue'

// maximum image size
const MAX_SIZE = 48
// current image data
const imageData = shallowRef([])
// refers to canvas element
let canvasRef = ref(null)
// canvas context 2d
let context
const imgWidth = ref(MAX_SIZE)
const imgHeight = ref(MAX_SIZE)
// customizable charset
const charset = ref('8?!:. ')

onMounted(() => {
  if (canvasRef.value) {
    context = canvasRef.value.getContext('2d', {
      willReadFrequently: true,
    })
  }
})

onUnmounted(() => {
  context = null
})

/**
 * normalize image width and height
 * @param {number} width
 * @param {number} height
 * @returns { width: number, height: number }
 */
function normalizeSize(width, height) {
  const ratio = width / height
  if (ratio > 1) {
    return { width: MAX_SIZE, height: Math.floor(MAX_SIZE / ratio) }
  }

  return { width: Math.floor(MAX_SIZE * ratio), height: MAX_SIZE }
}

/**
 * get char based on gray value
 * @param {number} num
 * @returns {string}
 */
function getCharByNumber(num) {
  const len = charset.value.length
  let index = Math.floor((num / 255) * len)
  if (index > len - 1) index = len - 1
  return charset.value.charAt(index)
}

function handleInputChange(e) {
  if (e.target.files) {
    const file = e.target.files[0]
    const image = new Image()
    image.src = URL.createObjectURL(file)

    image.onload = () => {
      URL.revokeObjectURL(image.src)

      const { width, height } = normalizeSize(image.width, image.height)
      imgWidth.value = width
      imgHeight.value = height

      nextTick(() => {
        context.clearRect(0, 0, width, height)
        context.drawImage(image, 0, 0, width, height)
        saveImageData()
      })
    }
  }
}

/**
 * save image data
 */
function saveImageData() {
  const { data } = context.getImageData(0, 0, imgWidth.value, imgHeight.value)
  imageData.value = data
}

/** gray data from image */
const grayData = computed(() => {
  const result = []
  const rawDatas = imageData.value

  for (let i = 0; i < rawDatas.length; i += 4) {
    const avg = (rawDatas[i] + rawDatas[i + 1] + rawDatas[i + 2]) / 3
    result.push(avg)
  }

  return result
})

/** the ultimate chars */
const outputChars = computed(() => {
  const result = []
  for (let i = 0; i < grayData.value.length; i++) {
    if (i && i % imgWidth.value === 0) {
      result.push('\n')
    }
    result.push(getCharByNumber(grayData.value[i]))
  }
  return result.join('')
})
</script>

<template>
  <main>
    <section>
      <input type="file" @change="handleInputChange" />
      <label>Charset: <input type="text" v-model="charset" /> </label>
    </section>
    <canvas ref="canvasRef" :width="imgWidth" :height="imgHeight" />
    <div class="output">{{ outputChars }}</div>
  </main>
</template>

<style>
.output {
  white-space: pre;
  line-height: 1;
  font-family: monospace;
  font-size: 12px;
}
</style>
