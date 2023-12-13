<template>
  <!-- All correct animation -->
  <div v-if="allCorrect" class="all-correct"></div>

  <!-- Correct size border -->
  <div v-if="windowInfo.correctSize" class="correct-size">
  </div>

  <!-- Correct location check mark -->
  <div v-if="windowInfo.correctLocation" class="correct-location">
    <img :src="pathToIcon" width="200" height="200">
  </div>

  <!-- Info dialogue -->
  <div class="info">
    <button @click="showInfo = !showInfo">Toggle Info</button>
    <button @click="reset()">Reset</button>
    <div v-if="showInfo">
      <div>Window ID: {{ windowInfo.id }}</div>
      <div>Window Width: {{ windowInfo.width }}</div>
      <div>Window Height: {{ windowInfo.height }}</div>
      <div>Window X: {{ windowInfo.x }}</div>
      <div>Window Y: {{ windowInfo.y }}</div>
      <div>Number of windows: {{ numberOfWindows }}</div>
      <div>Intended Width: {{ windowInfo.intendedWidth }}</div>
      <div>Intended Height: {{ windowInfo.intendedHeight }}</div>
      <div>Intended X: {{ windowInfo.intendedX }}</div>
      <div>Intended Y: {{ windowInfo.intendedY }}</div>
      <div>Correct Size: {{ windowInfo.correctSize }}</div>
      <div>Correct Location: {{ windowInfo.correctLocation }}</div>
    </div>
  </div>

  <!-- Video stream -->
  <div>
    <video :srcObject="videoStream" :width="windowInfo.screenWidth" :height="windowInfo.screenHeight" autoplay
      :style="{ transform: 'translate(' + windowInfo.intendedX + 'px,' + windowInfo.intendedY + 'px)' }"></video>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, onBeforeMount } from 'vue';

// Path to checkmark icon (have to do it this way for github pages)
let pathToIcon = new URL('/src/assets/checkmark.png', import.meta.url).href

// State
let timerID = 0;
const showInfo = ref(false);
const allCorrect = ref(false);
const videoStream: any = ref(null);
const numberOfWindows = ref(0);
const closeEnoughThreshold = 30;

const windowInfo = ref({
  id: "window-0",
  screenWidth: window.screen.availWidth,
  screenHeight: window.screen.availHeight,
  width: window.outerWidth,
  height: window.innerHeight,
  x: window.screenX,
  y: window.screenY,
  intendedWidth: 0,
  intendedHeight: 0,
  intendedX: 0,
  intendedY: 0,
  correctSize: false,
  correctLocation: false,
});

onBeforeMount(() => {
  windowInfo.value.width = window.screen.availWidth;
  windowInfo.value.height = window.screen.availHeight;
  windowInfo.value.id = "window-" + getWindowID();
  setLocalStorage()
  initCamera()
});

onMounted(() => {
  timerID = setInterval(update, 10);
});

onBeforeUnmount(() => {
  window.clearInterval(timerID);
  if (videoStream.value) {
    const tracks = videoStream.value.getTracks();
    tracks.forEach((track: any) => track.stop());
  }
});

const initCamera = async () => {
  try {
    await navigator.mediaDevices.getUserMedia({ video: true }).then((stream) => {
      videoStream.value = stream;
    })
  } catch (error) {
    console.error('Error accessing webcam:', error);
  }
};

function getWindowID() {
  const existingScreens = Object.keys(window.localStorage)
    .filter((key) => key.startsWith('window-'))
    .map((key) => parseInt(key.replace('window-', '')))
    .sort((a, b) => a - b);
  return existingScreens.at(-1) + 1 || 1;
}

function getNumberOfWindows() {
  return Object.keys(window.localStorage).filter((key) => key.startsWith('window-')).length
}

function update() {
  // If the number of windows open changes, we need to recalculate the intended values
  // Since numberOfWindows.value is 0 to start, this always happens the first iteration
  if (numberOfWindows.value != getNumberOfWindows()) {
    numberOfWindows.value = getNumberOfWindows()
    setIntendedValues()
  }

  setScreenDetails()
  setLocalStorage()
  checkIfCorrect()
  allCorrect.value = checkIfAllCorrect()
}

function setLocalStorage() {
  window.localStorage.setItem(windowInfo.value.id, JSON.stringify(windowInfo.value))
}

function setScreenDetails() {
  windowInfo.value.x = -window.screenX
  windowInfo.value.y = -window.screenY
  windowInfo.value.width = window.outerWidth
  windowInfo.value.height = window.outerHeight
}

function findClosestFactors(number: number) {
  let factor1 = Math.floor(Math.sqrt(number));
  while (number % factor1 !== 0) {
    factor1 = factor1 - 1;
  }

  return [factor1, number / factor1];
}

function setIntendedValues() {
  let closestFactors: number[] = findClosestFactors(numberOfWindows.value)
  let numRows = closestFactors[0]
  let numCols = closestFactors[1]

  setIntendedSize(numRows, numCols)
  setIntendedLocation(numCols, windowInfo.value.intendedWidth, windowInfo.value.intendedHeight)
}

function setIntendedLocation(numCols: number, windowWidth: number, windowHeight: number) {
  let id = parseInt(windowInfo.value.id.replace('window-', ''))
  let row = Math.floor((id - 1) / numCols)
  let col = (id - 1) % numCols
  windowInfo.value.intendedY = -row * windowHeight
  windowInfo.value.intendedX = -col * windowWidth
}

function setIntendedSize(numRows: number, numCols: number) {
  windowInfo.value.intendedWidth = window.screen.availWidth / numCols
  windowInfo.value.intendedHeight = window.screen.availHeight / numRows
}

function checkIfCorrect() {
  if (isCloseEnough(windowInfo.value.width, windowInfo.value.intendedWidth) &&
    isCloseEnough(windowInfo.value.height, windowInfo.value.intendedHeight)) {
    windowInfo.value.correctSize = true
  } else {
    windowInfo.value.correctSize = false
  }
  if (isCloseEnough(windowInfo.value.x, windowInfo.value.intendedX) &&
    isCloseEnough(windowInfo.value.y, windowInfo.value.intendedY)) {
    windowInfo.value.correctLocation = true
  } else {
    windowInfo.value.correctLocation = false
  }
}

function checkIfAllCorrect() {
  const existingScreens = Object.keys(window.localStorage)
    .filter((key) => key.startsWith('window-'))

  for (let i = 0; i < existingScreens.length; i++) {
    let screen = JSON.parse(window.localStorage.getItem(existingScreens[i])!)
    if (!screen.correctSize || !screen.correctLocation) {
      return false
    }
  }

  return true
}

function isCloseEnough(a: number, b: number) {
  return Math.abs(a - b) < closeEnoughThreshold;
}

function reset() {
  window.clearInterval(timerID);
  window.localStorage.clear();
  setTimeout(() => window.location.reload(), Math.random() * 1000);
}

</script>

<style scoped lang="sass">

.info
  position: fixed
  top: 10px
  left: 10px
  background: rgba(0, 0, 0, 0.5)
  color: white
  padding: 10px
  font-size: 12px
  font-family: monospace
  z-index: 3

.correct-size
  position: absolute
  top: 0
  left: 0
  right: 0
  bottom: 0
  z-index: 1
  border: 10px solid green
  
.correct-location
  position: absolute
  top: 0
  left: 0
  right: 0
  bottom: 0
  z-index: 1
  display: flex
  align-items: center
  justify-content: center

.all-correct
  opacity: 0.5
  --angle: 0deg
  position: absolute
  top: 0
  left: 0
  right: 0
  bottom: 0
  z-index: 2
  border: 10vmin solid
  border-image: conic-gradient(from var(--angle), red, yellow, lime, aqua, blue, magenta, red) 1
  border-image-slice: 1 fill
  animation: 10s rotate linear infinite

@keyframes rotate
  to
    --angle: 360deg

@property --angle
  syntax: '<angle>'
  initial-value: 0deg
  inherits: false

</style>