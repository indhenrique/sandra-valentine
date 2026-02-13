<template>
  <div class="container" :class="{ 'party-mode': partyMode, falling, rising }" :style="gridStyle">
    <div class="cell" v-for="i in totalCells" :key="i" :class="{ rotated: rotatedCells.has(i), special: i === specialCell, party: i === partyCell }" :style="(falling || rising) ? { animationDelay: `${Math.random() * 0.8}s` } : {}" @click="onCellClick(i)">
      <img :src="shuffled[i - 1]" alt="flor" />
    </div>
  </div>

  <dialog ref="dialogRef" class="flower-dialog" @cancel.prevent>
    <button class="close-button" @click="closeDialog">&times;</button>

    <template v-if="dialogStep === 'welcome'">
      <div style="color: white">Encontra a flor correta</div>
      <div class="dialog-buttons">
        <button class="fall-button" @click="startExperience">Começar</button>
      </div>
    </template>

    <template v-if="dialogStep === 'ask'">
      <div style="color: white">Sandra, queres ser a minha "valentina"?</div>
      <div class="dialog-buttons">
        <button class="fall-button" @click="answerYes">Sim</button>
        <button class="fall-button" @click="answerNo">Não</button>
      </div>
    </template>

    <template v-if="dialogStep === 'sure'">
      <div style="color: white">Tens a certeza?</div>
      <div class="dialog-buttons">
        <button ref="runawayRef" class="fall-button runaway" :style="{ transform: `translate(${runawayOffset.x}px, ${runawayOffset.y}px)` }" @mouseover="moveRunaway" @click="sureYes">Sim</button>
        <button class="fall-button" @click="sureNo">Não</button>
      </div>
    </template>

    <template v-if="dialogStep === 'party'">
      <div style="color: white">siiiiiiimmmmm!</div>
    </template>
  </dialog>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

import flower1 from '@/assets/images/flower_1.png'
import flower2 from '@/assets/images/flower_2.png'
import flower3 from '@/assets/images/flower_3.png'
import flower4 from '@/assets/images/flower_4.png'
import flower5 from '@/assets/images/flower_5.png'
import flower6 from '@/assets/images/flower_6.png'
import romanticMusic from '@/assets/music/amalvinay-all-of-me-piano-version-168903.mp3'

const images = [flower1, flower2, flower3, flower4, flower5, flower6]
const audio = new Audio(romanticMusic)
audio.loop = true
audio.volume = 0.5

const width = ref(window.innerWidth)
const height = ref(window.innerHeight)
const musicStarted = ref(false)

function onResize() {
  width.value = window.innerWidth
  height.value = window.innerHeight
}

function startMusic() {
  if (!musicStarted.value && audio.paused) {
    audio.play()
      .then(() => {
        musicStarted.value = true
      })
      .catch((error) => {
        console.log('Autoplay blocked:', error)
        // Will retry on next user interaction
      })
  }
}

function ensureMusicPlays() {
  if (!musicStarted.value) {
    audio.play()
      .then(() => {
        musicStarted.value = true
      })
      .catch(() => {
        // Last resort fallback
      })
  }
}

onMounted(() => {
  window.addEventListener('resize', onResize)
  // Show welcome dialog
  dialogRef.value?.showModal()

  // Try autoplay (will fail on most browsers)
  startMusic()

  // Listen for ANY user interaction
  const handlers = ['click', 'touchstart', 'keydown']
  const ensureMusic = () => {
    ensureMusicPlays()
    // Clean up after first successful play
    handlers.forEach(event => document.removeEventListener(event, ensureMusic))
  }
  handlers.forEach(event => document.addEventListener(event, ensureMusic))
})
onUnmounted(() => {
  window.removeEventListener('resize', onResize)
  audio.pause()
})

const isMobile = computed(() => width.value < 768)
const targetSize = computed(() => isMobile.value ? 75 : 150)
const minSize = computed(() => isMobile.value ? 60 : 120)
const maxSize = computed(() => isMobile.value ? 90 : 180)
const gap = 20

const rotatedCells = ref(new Set<number>())
const partyMode = ref(false)
const falling = ref(false)

const dialogRef = ref<HTMLDialogElement>()

function toggleRotation(i: number) {
  if (rotatedCells.value.has(i)) {
    rotatedCells.value.delete(i)
  } else {
    rotatedCells.value.add(i)
  }
  rotatedCells.value = new Set(rotatedCells.value)
}

function onCellClick(i: number) {
  if (i === specialCell.value) {
    dialogStep.value = 'ask'
    dialogRef.value?.showModal()
  } else if (i === partyCell.value) {
    partyMode.value = !partyMode.value
    // Music keeps playing in the background
  } else {
    toggleRotation(i)
  }
}

const dialogStep = ref<'welcome' | 'ask' | 'sure' | 'party'>('welcome')

function closeDialog() {
  dialogRef.value?.close()
  dialogStep.value = 'ask'
  if (partyMode.value) {
    stopParty()
  }
}

function startExperience() {
  dialogRef.value?.close()
  dialogStep.value = 'ask'
}

const rising = ref(false)

function startParty() {
  partyMode.value = true
  // Music is already playing in the background
}

function stopParty() {
  partyMode.value = false
  // Music keeps playing in the background
}

function dropFlowers() {
  rising.value = false
  falling.value = true
}

function riseFlowers() {
  falling.value = false
  rising.value = true
}

function answerYes() {
  dialogStep.value = 'party'
  startParty()
}

function answerNo() {
  dialogStep.value = 'sure'
  dropFlowers()
}

function sureYes() {
  closeDialog()
}

const runawayOffset = ref({ x: 0, y: 0 })
const runawayRef = ref<HTMLButtonElement>()

function moveRunaway() {
  const dialog = dialogRef.value
  const btn = runawayRef.value
  if (!dialog || !btn) return

  const dialogRect = dialog.getBoundingClientRect()
  const btnRect = btn.getBoundingClientRect()

  const padding = 16
  const minX = dialogRect.left + padding - (btnRect.left - runawayOffset.value.x)
  const maxX = dialogRect.right - padding - btnRect.width - (btnRect.left - runawayOffset.value.x)
  const minY = dialogRect.top + padding - (btnRect.top - runawayOffset.value.y)
  const maxY = dialogRect.bottom - padding - btnRect.height - (btnRect.top - runawayOffset.value.y)

  const x = minX + Math.random() * (maxX - minX)
  const y = minY + Math.random() * (maxY - minY)
  runawayOffset.value = { x, y }
}

function sureNo() {
  dialogStep.value = 'ask'
  runawayOffset.value = { x: 0, y: 0 }
  riseFlowers()
}

const cols = computed(() => {
  let c = Math.round(width.value / targetSize.value)
  const size = (width.value - (c - 1) * gap) / c
  if (size < minSize.value) c--
  if (size > maxSize.value) c++
  return c
})

const cellSize = computed(() => (width.value - (cols.value - 1) * gap) / cols.value)
const rows = computed(() => Math.ceil((height.value + gap) / (cellSize.value + gap)))
const totalCells = computed(() => cols.value * rows.value)
const specialCell = computed(() => Math.floor(Math.random() * totalCells.value) + 1)
const partyCell = computed(() => {
  let cell
  do {
    cell = Math.floor(Math.random() * totalCells.value) + 1
  } while (cell === specialCell.value)
  return cell
})

const shuffled = computed(() => {
  return Array.from({ length: totalCells.value }, () =>
    images[Math.floor(Math.random() * images.length)]
  )
})

const gridStyle = computed(() => ({
  gridTemplateColumns: `repeat(${cols.value}, ${cellSize.value}px)`,
  gridTemplateRows: `repeat(${rows.value}, ${cellSize.value}px)`,
  gap: `${gap}px`,
}))
</script>

<style scoped lang="scss">
.container {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  display: grid;
  transition: background-color 5s ease;

  &.party-mode {
    animation: strobe 0.3s infinite;
  }

  &.falling {
    background-color: #1a1a1a;
  }
}

@keyframes strobe {
  0% { background-color: black; }
  50% { background-color: white; }
  100% { background-color: black; }
}

@keyframes pulse {
  0%, 100% { opacity: 0.5; transform: scale(1); }
  50% { opacity: 1; transform: scale(1.3); }
}

@keyframes glow {
  0%, 100% { box-shadow: 0 0 8px rgba(255, 100, 150, 0.4); }
  50% { box-shadow: 0 0 20px rgba(255, 100, 150, 0.8); }
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@keyframes fall {
  0% {
    transform: translateY(0) rotate(0deg);
    opacity: 1;
  }
  70% {
    opacity: 1;
  }
  100% {
    transform: translateY(100vh) rotate(45deg);
    opacity: 0;
  }
}

@keyframes rise {
  0% {
    transform: translateY(100vh) rotate(45deg);
    opacity: 0;
  }
  30% {
    opacity: 1;
  }
  100% {
    transform: translateY(0) rotate(0deg);
    opacity: 1;
  }
}

.cell {
  cursor: url('@/assets/cursors/hover_32.png'), pointer;

  &.special {
    cursor: url('@/assets/cursors/heart_32.png'), pointer;
    position: relative;

    &::after {
      @media (max-width: 767px) {
        content: '';
        position: absolute;
        top: 4px;
        right: 4px;
        width: 14px;
        height: 14px;
        border-radius: 50%;
        background: rgba(255, 100, 150, 0.9);
        animation: pulse 1.5s ease-in-out infinite;
      }
    }
  }

  &.party {
    cursor: url('@/assets/cursors/disco_32.png'), pointer;
  }

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: transform 1.5s ease;
  }

  &:hover img {
    transform: rotate(360deg);
  }

  &.rotated img {
    transform: rotate(360deg);
  }

  .party-mode & img {
    animation: spin 1s linear infinite;
  }

  .falling & {
    animation: fall 1.5s ease-in forwards;
  }

  .rising & {
    opacity: 0;
    transform: translateY(100vh);
    animation: rise 1.5s ease-out forwards;
  }
}

.flower-dialog {
  border: 1px solid rgba(255, 255, 255, 0.3);
  height: 300px;
  width: 500px;
  border-radius: 24px;
  padding: 3rem 4rem;
  font-family: 'Poppins', sans-serif;
  font-weight: 600;
  font-size: 2.2rem;
  letter-spacing: 0.02em;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  box-shadow:
    0 8px 32px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.4);
  outline: none;
  position: relative;

  @media (max-width: 767px) {
    width: 85vw;
    height: 250px;
    padding: 2rem 1.5rem;
    font-size: 1.5rem;
    border-radius: 18px;
  }
}

.close-button {
  position: absolute;
  top: 1rem;
  right: 1rem;
  background: none;
  border: none;
  color: #fff;
  font-size: 1.8rem;
  cursor: url('@/assets/cursors/hover_32.png'), pointer;
  line-height: 1;
  padding: 0;
  opacity: 0.7;
  transition: opacity 0.2s;

  &:hover {
    opacity: 1;
  }

  &:focus {
    outline: none;
  }
}

.dialog-buttons {
  display: flex;
  gap: 1rem;
  position: absolute;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
}

.fall-button {
  font-family: 'Poppins', sans-serif;
  font-weight: 400;
  font-size: 1rem;
  padding: 0.7rem 1.5rem;

  @media (max-width: 767px) {
    font-size: 0.85rem;
    padding: 0.6rem 1.2rem;
  }
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.3);
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  color: #fff;
  cursor: url('@/assets/cursors/hover_32.png'), pointer;
  transition: background 0.3s;

  &:hover {
    background: rgba(255, 255, 255, 0.3);
  }

  &:focus {
    outline: none;
  }

  &.runaway {
    position: relative;
    transition: transform 0.2s ease;
  }
}
</style>
