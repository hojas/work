<script setup lang="ts">
import { useIntersectionObserver } from '@vueuse/core'
import { computed, ref } from 'vue'

const props = withDefaults(defineProps<{
  src: string
  poster?: string
  controls?: boolean
  autoplay?: boolean
  loop?: boolean
  muted?: boolean
  rootMargin?: string
}>(), {
  controls: true,
  autoplay: false,
  loop: false,
  muted: false,
  rootMargin: '200px',
})

const el = ref<HTMLElement>()
const isIntersecting = ref(false)
const loaded = ref(false)
const error = ref(false)

const { stop } = useIntersectionObserver(el, ([{ isIntersecting: intersecting }]) => {
  if (intersecting) {
    isIntersecting.value = true
    stop()
  }
}, { rootMargin: props.rootMargin })

const showPlaceholder = computed(() => (!loaded.value && !error.value) || !isIntersecting.value)
const showError = computed(() => error.value)

function onLoaded() {
  loaded.value = true
}

function onError() {
  error.value = true
}
</script>

<template>
  <div ref="el" class="lazy-media">
    <div v-if="showPlaceholder && !error" class="placeholder-bg">
      <svg viewBox="0 0 24 24" class="placeholder-icon" fill="none" stroke="currentColor" stroke-width="1.5">
        <polygon points="23 7 16 12 23 17 23 7" />
        <rect x="1" y="5" width="15" height="14" rx="2" ry="2" />
      </svg>
    </div>
    <div v-else-if="showError" class="placeholder-bg error">
      <svg viewBox="0 0 24 24" class="placeholder-icon" fill="none" stroke="currentColor" stroke-width="1.5">
        <circle cx="12" cy="12" r="10" />
        <line x1="12" y1="8" x2="12" y2="12" />
        <line x1="12" y1="16" x2="12.01" y2="16" />
      </svg>
    </div>
    <video
      v-show="isIntersecting && !error"
      :src="src"
      :poster="poster"
      :controls="controls"
      :autoplay="autoplay"
      :loop="loop"
      :muted="muted"
      class="lazy-video"
      preload="none"
      @loadeddata="onLoaded"
      @error="onError"
    />
  </div>
</template>

<style scoped>
.lazy-media {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.lazy-video {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.placeholder-bg {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  background: #f0f0f0;
  min-height: 200px;
}

.placeholder-bg.error {
  background: #fef2f2;
  color: #f87171;
}

.placeholder-icon {
  width: 48px;
  height: 48px;
  color: #c0c0c0;
}
</style>
