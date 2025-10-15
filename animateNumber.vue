<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

interface DigitComponent {
  offset: number
  target: number
}

const props = defineProps<{
  value: number
  mode?: 'smooth' | 'roll'
  mountDuration?: number
  updateDuration?: number
}>()

const mode = computed(() => props.mode ?? 'smooth')
const mountDuration = computed(() => props.mountDuration ?? 1000)
const updateDuration = computed(() => props.updateDuration ?? mountDuration.value)

const currentValue = ref(0)
const digitComponents = ref<DigitComponent[]>([])
const prevValue = ref(0)

const formattedValue = computed(() => {
  return Math.round(currentValue.value).toLocaleString()
})

const animateSmooth = (from: number, to: number, duration: number): void => {
  const startTime = performance.now()

  const animate = (currentTime: number): void => {
    const elapsed = currentTime - startTime
    const progress = Math.min(elapsed / duration, 1)
    const easeOutQuad = 1 - (1 - progress) * (1 - progress)

    currentValue.value = from + (to - from) * easeOutQuad

    if (progress < 1) {
      requestAnimationFrame(animate)
    } else {
      currentValue.value = to
    }
  }

  requestAnimationFrame(animate)
}

const prepareDigits = (value: number): number[] => {
  const str = String(Math.round(value))
  return str.split('').map(d => parseInt(d))
}

const animateRoll = (from: number, to: number, duration: number, isMount = false): void => {
  const fromDigits = prepareDigits(from)
  const toDigits = prepareDigits(to)

  const maxLength = Math.max(fromDigits.length, toDigits.length)
  while (fromDigits.length < maxLength) fromDigits.unshift(0)
  while (toDigits.length < maxLength) toDigits.unshift(0)

  digitComponents.value = fromDigits.map(d => ({ offset: d, target: d }))

  const startTime = performance.now()

  const animate = (currentTime: number): void => {
    const elapsed = currentTime - startTime
    const progress = Math.min(elapsed / duration, 1)
    const easeOutQuad = 1 - (1 - progress) * (1 - progress)

    if (isMount) {
      digitComponents.value = toDigits.map((target, index) => {
        const rotations = Math.max(1, maxLength - index)
        const current = (rotations * 10 * easeOutQuad + target) % 10
        return { offset: current, target }
      })
    } else {
      digitComponents.value = fromDigits.map((start, index) => {
        const end = toDigits[index]
        if (start !== end) {
          let diff = end - start
          if (diff < 0) diff += 10
          const currentOffset = start + diff * easeOutQuad
          return { offset: currentOffset % 10, target: end }
        }
        return { offset: start, target: end }
      })
    }

    if (progress < 1) {
      requestAnimationFrame(animate)
    } else {
      digitComponents.value = toDigits.map(d => ({ offset: d, target: d }))
    }
  }

  requestAnimationFrame(animate)
}

watch(
    () => props.value,
    (newVal) => {
      if (mode.value === 'smooth') {
        animateSmooth(currentValue.value, newVal, updateDuration.value)
      } else {
        animateRoll(prevValue.value, newVal, updateDuration.value, false)
      }
      prevValue.value = newVal
    }
)

onMounted(() => {
  if (mode.value === 'smooth') {
    animateSmooth(0, props.value, mountDuration.value)
  } else {
    animateRoll(0, props.value, mountDuration.value, true)
  }
  prevValue.value = props.value
})
</script>


<template>
  <span class="animated-number">
    <!-- smooth mode -->
    <span v-if="mode === 'smooth'">
      {{ formattedValue }}
    </span>

    <!-- roll mode -->
    <span v-else class="roll-container">
      <span
          v-for="(digit, index) in digitComponents"
          :key="index"
          class="digit-wrapper"
      >
        <span
            class="digit-roll"
            :style="{ transform: `translate3d(0, -${digit.offset * 10}%, 0)` }"
        >
          <span v-for="n in 10" :key="n" class="digit">{{ n - 1 }}</span>
        </span>
      </span>
    </span>
  </span>
</template>


<style scoped>
.animated-number {
  display: inline-block;
  font-variant-numeric: tabular-nums;
  font-feature-settings: "tnum";
}

.roll-container {
  display: inline-flex;
  gap: 2px;
}

.digit-wrapper {
  display: inline-block;
  height: 1.2em;
  overflow: hidden;
  position: relative;
  width: 0.6em;
  text-align: center;
  perspective: 1000px;
}

.digit-roll {
  display: flex;
  flex-direction: column;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  will-change: transform;
  transform: translateZ(0);
  backface-visibility: hidden;
  transition: transform 0.05s linear;
}

.digit {
  display: block;
  height: 1.2em;
  line-height: 1.2em;
  font-weight: 500;
}
</style>
