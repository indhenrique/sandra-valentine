<template>
  <div class="container" :class="{ 'party-mode': partyMode, falling, rising }" :style="gridStyle">
    <div class="cell" v-for="i in totalCells" :key="i" :class="{
      rotated: rotatedCells.has(i),
      special: i === specialCell,
      locked: i === specialCell && flowerClickCount < 10,
      party: i === partyCell,
      memory: MEMORY_FLOWER_POSITIONS.includes(i) && !discoveredMemories.has(MEMORY_FLOWER_POSITIONS.indexOf(i) + 1),
      clue: CLUE_FLOWER_POSITIONS.includes(i) && !discoveredClues.has(CLUE_FLOWER_POSITIONS.indexOf(i) + 1),
      menuEgg: i === MENU_EGG_POSITION && !menuEggDiscovered,
      bloomed: bloomedFlowers.has(i)
    }" :style="(falling || rising) ? { animationDelay: `${Math.random() * 0.8}s` } : {}" @click="onCellClick(i)">
      <img :src="shuffled[i - 1]" alt="flor" />
    </div>
  </div>

  <!-- Progress Trackers -->
  <div class="progress-container" v-if="!showCountdown">
    <div class="progress-item" v-if="flowerClickCount < 10">
      🌸 Bloom Progress: {{ flowerClickCount }}/10
    </div>
    <div class="progress-item" v-if="discoveredMemories.size > 0">
      💌 Memories found: {{ discoveredMemories.size }}/{{ MEMORY_FLOWER_POSITIONS.length }}
    </div>
    <div class="progress-item" v-if="discoveredClues.size > 0">
      🍽️ Dinner Clues: {{ discoveredClues.size }}/{{ CLUE_FLOWER_POSITIONS.length }}
    </div>
  </div>

  <!-- Countdown Timer -->
  <div class="countdown-timer" v-if="showCountdown">
    <div class="countdown-time">✨ Dinner in: {{ dinnerCountdown }} ✨</div>
    <div class="countdown-message">{{ COUNTDOWN_MESSAGES[countdownMessageIndex] }}</div>
  </div>

  <!-- Discovered Clue Cards -->
  <div class="clue-cards" v-if="discoveredClues.size > 0">
    <div class="clue-card" v-if="discoveredClues.has(1)">
      🍽️ {{ DINNER_CLUES.restaurant }}
    </div>
    <div class="clue-card" v-if="discoveredClues.has(2)">
      🕗 {{ DINNER_CLUES.time }}
    </div>
    <div class="clue-card" v-if="discoveredClues.has(3)">
      👗 {{ DINNER_CLUES.dressCode }}
    </div>
  </div>

  <!-- Heart Particles -->
  <div class="heart-particles" v-if="showHeartParticles">
    <div class="heart" v-for="n in (isMobile ? 10 : 15)" :key="n" :style="{ left: `${Math.random() * 100}%`, animationDelay: `${Math.random() * 2}s` }">
      ❤️
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

    <template v-if="dialogStep === 'memory'">
      <div style="color: white; font-size: 1.6rem; line-height: 1.5;">
        {{ MEMORY_MESSAGES[currentMemoryId] }}
      </div>
      <div class="dialog-buttons">
        <button class="fall-button" @click="closeDialog">💕</button>
      </div>
    </template>

    <template v-if="dialogStep === 'clue'">
      <div style="color: white; font-size: 1.8rem;">
        <template v-if="currentClueId === 1">🍽️ Restaurant: {{ DINNER_CLUES.restaurant }}</template>
        <template v-if="currentClueId === 2">🕗 Time: {{ DINNER_CLUES.time }}</template>
        <template v-if="currentClueId === 3">👗 Dress Code: {{ DINNER_CLUES.dressCode }}</template>
      </div>
      <div class="dialog-buttons">
        <button class="fall-button" @click="closeDialog">Got it!</button>
      </div>
    </template>

    <template v-if="dialogStep === 'menu'">
      <div style="color: white; font-size: 1.8rem;">
        🍷 {{ MENU_TEASER }}
      </div>
      <div class="dialog-buttons">
        <button class="fall-button" @click="closeDialog">Yum! 😋</button>
      </div>
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

// ⚠️ EASILY CUSTOMIZABLE DATA - Replace with your personal details!
const DINNER_TIME = "2026-02-14T20:00:00" // ⚠️ SET YOUR DINNER DATE/TIME
const MEMORY_MESSAGES: Record<number, string> = {
  1: "You light up my world in ways I never knew possible",
  2: "Every moment with you is my favorite moment",
  3: "Your smile is the reason I look forward to every day",
  4: "I still remember the first time I saw you and knew you were special",
  5: "Thank you for being exactly who you are"
}
const DINNER_CLUES = {
  restaurant: "Your Favorite Place", // ⚠️ REPLACE WITH RESTAURANT NAME
  time: "8:00 PM",                   // ⚠️ REPLACE WITH TIME
  dressCode: "Dress to Impress"      // ⚠️ REPLACE WITH DRESS CODE
}
const MENU_TEASER = "Save room for something special... 🎂" // ⚠️ CUSTOMIZE HINT

const MEMORY_FLOWER_POSITIONS = [4, 12, 19, 27, 35] // Cell positions for memory flowers
const CLUE_FLOWER_POSITIONS = [8, 23, 31] // Cell positions for dinner clue flowers
const MENU_EGG_POSITION = 15 // Cell position for menu easter egg
const COUNTDOWN_MESSAGES = [
  "Can't wait to see you tonight",
  "Get ready for something special",
  "You're going to love this"
]

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
  if (!musicStarted.value) {
    audio.play().catch(() => {
      // Browser blocked autoplay, will try on first user interaction
    })
    musicStarted.value = true
  }
}

onMounted(() => {
  window.addEventListener('resize', onResize)
  // Show welcome dialog
  dialogRef.value?.showModal()
  // Try to start music immediately
  startMusic()
  // Also try on first click anywhere
  const startOnClick = () => {
    startMusic()
    document.removeEventListener('click', startOnClick)
  }
  document.addEventListener('click', startOnClick)
})
onUnmounted(() => {
  window.removeEventListener('resize', onResize)
  audio.pause()
  if (countdownInterval.value) {
    clearInterval(countdownInterval.value)
  }
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

// New state for enhancements
const discoveredMemories = ref<Set<number>>(new Set())
const currentMemoryId = ref<number>(0)
const discoveredClues = ref<Set<number>>(new Set())
const currentClueId = ref<number>(0)
const menuEggDiscovered = ref(false)
const bloomedFlowers = ref<Set<number>>(new Set())
const flowerClickCount = ref(0)
const showHeartParticles = ref(false)
const dinnerCountdown = ref("")
const countdownInterval = ref<number | null>(null)
const countdownMessageIndex = ref(0)
const showCountdown = ref(false)

function toggleRotation(i: number) {
  if (rotatedCells.value.has(i)) {
    rotatedCells.value.delete(i)
  } else {
    rotatedCells.value.add(i)
  }
  rotatedCells.value = new Set(rotatedCells.value)
}

function onCellClick(i: number) {
  // Track bloom animation for first click
  if (!bloomedFlowers.value.has(i)) {
    bloomedFlowers.value.add(i)
    bloomedFlowers.value = new Set(bloomedFlowers.value)
  }

  // Check for memory flower
  const memoryIndex = MEMORY_FLOWER_POSITIONS.indexOf(i)
  if (memoryIndex !== -1 && !discoveredMemories.value.has(memoryIndex + 1)) {
    discoveredMemories.value.add(memoryIndex + 1)
    discoveredMemories.value = new Set(discoveredMemories.value)
    currentMemoryId.value = memoryIndex + 1
    dialogStep.value = 'memory'
    dialogRef.value?.showModal()
    return
  }

  // Check for clue flower
  const clueIndex = CLUE_FLOWER_POSITIONS.indexOf(i)
  if (clueIndex !== -1 && !discoveredClues.value.has(clueIndex + 1)) {
    discoveredClues.value.add(clueIndex + 1)
    discoveredClues.value = new Set(discoveredClues.value)
    currentClueId.value = clueIndex + 1
    dialogStep.value = 'clue'
    dialogRef.value?.showModal()
    return
  }

  // Check for menu easter egg
  if (i === MENU_EGG_POSITION && !menuEggDiscovered.value) {
    menuEggDiscovered.value = true
    dialogStep.value = 'menu'
    dialogRef.value?.showModal()
    return
  }

  // Check for special cell (Valentine's question) - only if unlocked
  if (i === specialCell.value && flowerClickCount.value >= 10) {
    dialogStep.value = 'ask'
    dialogRef.value?.showModal()
    return
  }

  // Check for party cell
  if (i === partyCell.value) {
    partyMode.value = !partyMode.value
    return
  }

  // Regular flower click - increment counter and toggle rotation
  flowerClickCount.value++
  toggleRotation(i)
}

const dialogStep = ref<'welcome' | 'ask' | 'sure' | 'party' | 'memory' | 'clue' | 'menu'>('welcome')

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
  showHeartParticles.value = true
  showCountdown.value = true
  startCountdown()
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

// Countdown timer functions
function startCountdown() {
  updateCountdown()
  if (countdownInterval.value) {
    clearInterval(countdownInterval.value)
  }
  countdownInterval.value = window.setInterval(() => {
    updateCountdown()
  }, 1000)

  // Rotate countdown messages every 10 seconds
  setInterval(() => {
    countdownMessageIndex.value = (countdownMessageIndex.value + 1) % COUNTDOWN_MESSAGES.length
  }, 10000)
}

function updateCountdown() {
  const now = new Date().getTime()
  const dinnerTime = new Date(DINNER_TIME).getTime()
  const distance = dinnerTime - now

  if (distance < 0) {
    dinnerCountdown.value = "It's time! 🎉"
    if (countdownInterval.value) {
      clearInterval(countdownInterval.value)
    }
    return
  }

  const hours = Math.floor(distance / (1000 * 60 * 60))
  const minutes = Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60))

  dinnerCountdown.value = `${hours}h ${minutes}m`
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

@keyframes goldenGlow {
  0%, 100% {
    box-shadow: 0 0 15px rgba(255, 215, 0, 0.4);
  }
  50% {
    box-shadow: 0 0 30px rgba(255, 215, 0, 0.8);
  }
}

@keyframes sparkle {
  0%, 100% {
    opacity: 0.6;
    transform: translate(-50%, -50%) scale(1);
  }
  50% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1.2);
  }
}

@keyframes bounce {
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-8px);
  }
}

@keyframes bloom {
  0% {
    transform: scale(0.8) rotate(0deg);
    opacity: 0.8;
  }
  50% {
    transform: scale(1.2) rotate(180deg);
  }
  100% {
    transform: scale(1) rotate(360deg);
    opacity: 1;
  }
}

@keyframes floatUp {
  0% {
    transform: translateY(0) translateX(0);
    opacity: 1;
  }
  100% {
    transform: translateY(-100vh) translateX(20px);
    opacity: 0;
  }
}

@keyframes heartbeat {
  0%, 100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
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

  &.memory {
    position: relative;

    &::after {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border-radius: 8px;
      box-shadow: 0 0 20px rgba(255, 215, 0, 0.6);
      animation: goldenGlow 2s ease-in-out infinite;
      pointer-events: none;
    }
  }

  &.clue {
    position: relative;

    &::after {
      content: '⭐';
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      font-size: 2rem;
      animation: sparkle 1.5s ease-in-out infinite;
      pointer-events: none;

      @media (max-width: 767px) {
        font-size: 1.5rem;
      }
    }
  }

  &.menuEgg {
    cursor: url('@/assets/cursors/hover_32.png'), pointer;

    &::after {
      content: '🍴';
      position: absolute;
      top: 8px;
      right: 8px;
      font-size: 1.5rem;
      animation: bounce 1s ease-in-out infinite;

      @media (max-width: 767px) {
        font-size: 1.2rem;
      }
    }
  }

  &.locked {
    filter: grayscale(1) brightness(0.6);
    cursor: default !important;

    &::after {
      display: none;
    }
  }

  &.bloomed img {
    animation: bloom 0.6s ease-out;
    filter: brightness(1.1);
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

.progress-container {
  position: fixed;
  top: 1rem;
  left: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  z-index: 100;

  @media (max-width: 767px) {
    top: 0.5rem;
    left: 0.5rem;
  }
}

.progress-item {
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  padding: 0.5rem 1rem;
  border-radius: 12px;
  color: white;
  font-family: 'Poppins', sans-serif;
  font-size: 0.9rem;
  font-weight: 500;
  border: 1px solid rgba(255, 255, 255, 0.3);

  @media (max-width: 767px) {
    font-size: 0.75rem;
    padding: 0.4rem 0.8rem;
  }
}

.countdown-timer {
  position: fixed;
  top: 1rem;
  right: 1rem;
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  padding: 1rem 1.5rem;
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.3);
  z-index: 100;
  text-align: center;
  animation: heartbeat 2s ease-in-out infinite;

  @media (max-width: 767px) {
    top: 0.5rem;
    left: 0.5rem;
    right: auto;
    padding: 0.8rem 1.2rem;
  }
}

.countdown-time {
  color: white;
  font-family: 'Poppins', sans-serif;
  font-size: 1.2rem;
  font-weight: 600;
  margin-bottom: 0.3rem;

  @media (max-width: 767px) {
    font-size: 1rem;
  }
}

.countdown-message {
  color: rgba(255, 255, 255, 0.9);
  font-family: 'Poppins', sans-serif;
  font-size: 0.85rem;
  font-weight: 400;
  font-style: italic;

  @media (max-width: 767px) {
    font-size: 0.7rem;
  }
}

.clue-cards {
  position: fixed;
  top: 1rem;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 1rem;
  z-index: 100;

  @media (max-width: 767px) {
    flex-direction: column;
    top: auto;
    bottom: 1rem;
    gap: 0.5rem;
  }
}

.clue-card {
  background: rgba(255, 255, 255, 0.25);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  padding: 0.6rem 1.2rem;
  border-radius: 12px;
  color: white;
  font-family: 'Poppins', sans-serif;
  font-size: 0.95rem;
  font-weight: 500;
  border: 1px solid rgba(255, 255, 255, 0.3);
  white-space: nowrap;

  @media (max-width: 767px) {
    font-size: 0.8rem;
    padding: 0.5rem 1rem;
  }
}

.heart-particles {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 50;
}

.heart {
  position: absolute;
  bottom: -50px;
  font-size: 2rem;
  animation: floatUp 4s ease-in infinite;

  @media (max-width: 767px) {
    font-size: 1.5rem;
  }
}
</style>
