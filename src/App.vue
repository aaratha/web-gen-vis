<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import CircularWaveform from './components/CircularWaveform.vue'

// Reactive variables
const isDragging = ref(false)
const mousePosition = ref({ x: 0, y: 0 })
const windowCenter = ref({ x: 0, y: 0 })
const restPosition = ref({ x: 0, y: 0 })
const containerCanvas = ref<HTMLCanvasElement | null>(null)
const currentRopeEnd = ref({ x: 0, y: 0 })
let animationFrameId: number | null = null

// Audio variables
let audioContext: AudioContext | null = null
let oscillator: OscillatorNode | null = null
let gainNode: GainNode | null = null
let lfo: OscillatorNode | null = null
let lfoGain: GainNode | null = null
let lowpassFilter: BiquadFilterNode | null = null
const isAudioInitialized = ref(false)

const PARAMETERS = {
  oscillatorType: 'square' as OscillatorType,
  lfoType: 'sine' as OscillatorType,
  minLFOFreq: 0.1,
  maxLFOFreq: 15,
  lfoGain: 3,
  minOscillatorGain: 150,
  maxOscillatorGain: 500,
}

// Pentatonic scale frequencies (A2 to A5)
const PENTATONIC_FREQUENCIES = [
  110.0, // A2
  130.81, // C3
  146.83, // D3
  164.81, // E3
  196.0, // G3
  220.0, // A3
  261.63, // C4
  293.66, // D4
  329.63, // E4
  392.0, // G4
  440.0, // A4
  523.25, // C5
  587.33, // D5
  659.25, // E5
  783.99, // G5
  880.0, // A5
]

// Initialize audio context
function initAudio() {
  if (!isAudioInitialized.value) {
    audioContext = new AudioContext()
    oscillator = audioContext.createOscillator()
    gainNode = audioContext.createGain()

    // Create the lowpass filter
    lowpassFilter = audioContext.createBiquadFilter()
    lowpassFilter.type = 'lowpass'
    lowpassFilter.frequency.setValueAtTime(1000, audioContext.currentTime) // Set initial cutoff frequency
    lowpassFilter.Q.setValueAtTime(10, audioContext.currentTime) // Set quality factor (resonance)

    // Create LFO for frequency modulation
    lfo = audioContext.createOscillator()
    lfoGain = audioContext.createGain()

    oscillator.type = PARAMETERS.oscillatorType
    oscillator.connect(lowpassFilter) // Connect oscillator to lowpass filter
    lowpassFilter.connect(gainNode) // Connect filter to gain
    gainNode.connect(audioContext.destination) // Connect gain to destination

    lfo.type = PARAMETERS.lfoType // Square wave LFO
    lfo.frequency.setValueAtTime(1, audioContext.currentTime) // LFO frequency (1 Hz for now)
    lfo.connect(lfoGain)
    lfoGain.connect(oscillator.frequency)

    // LFO modulation depth
    lfoGain.gain.setValueAtTime(PARAMETERS.lfoGain, audioContext.currentTime) // Depth of modulation (adjust for effect)

    // Start the oscillator and LFO
    oscillator.start()
    lfo.start()

    // Start with no volume
    gainNode.gain.value = 0

    isAudioInitialized.value = true
  }
}

function findClosestPentatonicFrequency(freq: number): number {
  return PENTATONIC_FREQUENCIES.reduce((prev, curr) =>
    Math.abs(curr - freq) < Math.abs(prev - freq) ? curr : prev,
  )
}

onMounted(() => {
  const windowDiv = document.querySelector('.window') as HTMLDivElement
  const canvas = containerCanvas.value

  if (windowDiv && canvas) {
    canvas.width = document.documentElement.clientWidth
    canvas.height = document.documentElement.clientHeight
    updateWindowCenter()

    // Initialize rope end outside window, bottom
    const rect = windowDiv.getBoundingClientRect()
    currentRopeEnd.value = {
      x: rect.left + rect.width / 2,
      y: rect.bottom + 20, // Slightly below window bottom edge
    }
    restPosition.value = { ...currentRopeEnd.value }
  }

  animate()
  window.addEventListener('mousedown', handleMouseDown)
  window.addEventListener('mousemove', handleMouseMove)
  window.addEventListener('mouseup', handleMouseUp)
  window.addEventListener('scroll', updateWindowCenter)
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  if (animationFrameId !== null) {
    cancelAnimationFrame(animationFrameId)
  }
  if (oscillator) {
    oscillator.stop()
  }
  if (audioContext) {
    audioContext.close()
  }
  if (lfo) {
    lfo.stop()
  }
  lowpassFilter = null
  window.removeEventListener('mousedown', handleMouseDown)
  window.removeEventListener('mousemove', handleMouseMove)
  window.removeEventListener('mouseup', handleMouseUp)
  window.removeEventListener('scroll', updateWindowCenter)
  window.removeEventListener('resize', handleResize)
})

function updateWindowCenter() {
  const windowDiv = document.querySelector('.window') as HTMLDivElement
  const rect = windowDiv.getBoundingClientRect()

  windowCenter.value = {
    x: rect.left + rect.width / 2,
    y: rect.top + rect.height / 2,
  }
}

function calculateRestPosition(mouseX: number, mouseY: number) {
  const windowDiv = document.querySelector('.window') as HTMLDivElement
  const rect = windowDiv.getBoundingClientRect()
  const radius = 200 / 2 + 20 // Adding small offset from window edge
  // const radius = rect.width / 2 + 20 // Adding small offset from window edge

  // Calculate vector from center to mouse
  const dx = mouseX - windowCenter.value.x
  const dy = mouseY - windowCenter.value.y
  const distance = Math.sqrt(dx * dx + dy * dy)

  // Normalize vector and multiply by radius to get point on circle
  const normalizedX = dx / distance
  const normalizedY = dy / distance

  return {
    x: windowCenter.value.x + normalizedX * radius,
    y: windowCenter.value.y + normalizedY * radius,
  }
}

function handleResize() {
  if (containerCanvas.value) {
    containerCanvas.value.width = document.documentElement.clientWidth
    containerCanvas.value.height = document.documentElement.clientHeight
    updateWindowCenter()
  }
}

function handleMouseDown(event: MouseEvent) {
  const { x, y } = mousePosition.value
  const dx = event.clientX - currentRopeEnd.value.x
  const dy = event.clientY - currentRopeEnd.value.y
  const distance = Math.sqrt(dx * dx + dy * dy)

  if (distance <= 18) {
    // Check if click is on rope end
    initAudio()
    isDragging.value = true
    updateMousePosition(event)
  }
}

function handleMouseMove(event: MouseEvent) {
  if (isDragging.value) {
    updateMousePosition(event)
  }
}

function handleMouseUp() {
  isDragging.value = false
  if (gainNode) {
    gainNode.gain.setTargetAtTime(0, audioContext!.currentTime, 0.1)
  }
  // Calculate new rest position on the edge of the window circle
  restPosition.value = calculateRestPosition(mousePosition.value.x, mousePosition.value.y)
}

function updateMousePosition(event: MouseEvent) {
  mousePosition.value = {
    x: event.clientX,
    y: event.clientY,
  }
}

function lerp(start: number, end: number, t: number): number {
  return start + (end - start) * t
}

function calculateRopeLength(x1: number, y1: number, x2: number, y2: number): number {
  return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2))
}

function calculateAngle(x1: number, y1: number, x2: number, y2: number): number {
  // Calculate the angle between the rope and the vertical axis
  const dx = x2 - x1
  const dy = y2 - y1
  return Math.atan2(dx, dy) // Returns angle in radians
}

function updateAudio(ropeLength: number) {
  if (oscillator && gainNode && audioContext && lfo && lfoGain) {
    const angle = calculateAngle(
      windowCenter.value.x,
      windowCenter.value.y,
      currentRopeEnd.value.x,
      currentRopeEnd.value.y,
    )

    if (lowpassFilter) {
      const minCutoff = 0
      const maxCutoff = 5000
      const normalizedLength = Math.min((ropeLength - 150) / (500 - 150), 1)
      const cutoffFrequency = minCutoff + normalizedLength * (maxCutoff - minCutoff)
      lowpassFilter.frequency.setValueAtTime(cutoffFrequency, audioContext.currentTime)
    }

    // Map angle to frequency range (A2 to A5)
    const normalizedAngle = (angle + Math.PI) / (2 * Math.PI) // Normalize to 0-1
    const baseFrequency = 110 + normalizedAngle * 770 // Map to frequency range A2-A5

    // Snap to closest pentatonic frequency
    const snappedFrequency = findClosestPentatonicFrequency(baseFrequency)

    // Reduce LFO effect for clearer notes
    const minLFOFreq = PARAMETERS.minLFOFreq
    const maxLFOFreq = PARAMETERS.maxLFOFreq
    const normalizedRopeLength = Math.min((ropeLength - 150) / (500 - 150), 1)

    const lfoFrequency = minLFOFreq + normalizedRopeLength * (maxLFOFreq - minLFOFreq)
    lfo.frequency.setValueAtTime(lfoFrequency, audioContext.currentTime)

    // Apply snapped frequency with reduced LFO modulation
    //lfoGain.gain.value = 0
    oscillator.frequency.setTargetAtTime(
      snappedFrequency + lfoGain.gain.value, // Reduced LFO influence
      audioContext.currentTime,
      0.1,
    )

    // Gain handling remains the same
    const minGainDistance = PARAMETERS.minOscillatorGain
    const maxGainDistance = PARAMETERS.maxOscillatorGain
    let targetGain = 0

    if (ropeLength > minGainDistance) {
      const normalizedLength = Math.min(
        (ropeLength - minGainDistance) / (maxGainDistance - minGainDistance),
        1,
      )
      targetGain = normalizedLength * 0.4
    }

    gainNode.gain.setTargetAtTime(targetGain, audioContext.currentTime, 0.1)
  }
}

function animate() {
  const canvas = containerCanvas.value
  const windowDiv = document.querySelector('.window') as HTMLDivElement
  if (!canvas || !windowDiv) {
    animationFrameId = requestAnimationFrame(animate)
    return
  }

  const context = canvas.getContext('2d')!
  context.clearRect(0, 0, canvas.width, canvas.height)

  const lerpFactor = isDragging.value ? 0.2 : 0.01

  const targetX = isDragging.value ? mousePosition.value.x : restPosition.value.x
  const targetY = isDragging.value ? mousePosition.value.y : restPosition.value.y

  currentRopeEnd.value = {
    x: lerp(currentRopeEnd.value.x, targetX, lerpFactor),
    y: lerp(currentRopeEnd.value.y, targetY, lerpFactor),
  }

  const ropeLength = calculateRopeLength(
    windowCenter.value.x,
    windowCenter.value.y,
    currentRopeEnd.value.x,
    currentRopeEnd.value.y,
  )
  updateAudio(ropeLength)

  // Dynamically adjust window size based on rope length
  const baseSize = 200 // Minimum size
  const maxSize = 500 // Maximum size
  const normalizedLength = Math.min((ropeLength - 150) / (500 - 150), 1)
  const newSize = baseSize + normalizedLength * (maxSize - baseSize)

  windowDiv.style.width = `${newSize}px`
  windowDiv.style.height = `${newSize}px`

  // Draw rope
  context.beginPath()
  context.moveTo(windowCenter.value.x, windowCenter.value.y)
  context.lineTo(currentRopeEnd.value.x, currentRopeEnd.value.y)
  context.strokeStyle = 'rgba(0,0,0,0.5)'
  context.lineWidth = 2
  context.stroke()

  // Draw rope end
  context.beginPath()
  context.arc(currentRopeEnd.value.x, currentRopeEnd.value.y, 18, 0, Math.PI * 2)
  context.fillStyle = 'rgba(100,100,100,1)'
  context.fill()

  animationFrameId = requestAnimationFrame(animate)
}
</script>

<template>
  <div class="container">
    <canvas ref="containerCanvas" class="container-canvas"></canvas>
    <div class="window">
      <div class="content">
        <CircularWaveform
          v-if="isAudioInitialized && audioContext && oscillator"
          :audioContext="audioContext"
          :oscillator="oscillator"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  width: 100%;
  height: 100vh;
  position: relative;
  background-color: #f4f4f4;
}

.container-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  pointer-events: none;
}

.window {
  background-color: #f4f4f4;
  width: 200px;
  height: 200px;
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  box-shadow: 0px 0px 50px rgba(0, 0, 0, 0.2);
  border-radius: 50%;
  padding: 20px;
  z-index: 2;
  transition:
    width 0.1s ease,
    height 0.1s ease;
}

.content {
  width: 100%;
  height: 100%;
  position: relative;
  background-color: #f4f4f4;
  border: 1px solid rgba(0, 0, 0, 0.4);
  border-radius: 50%;
}
</style>
