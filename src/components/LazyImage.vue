<script setup lang="ts">
import { useIntersectionObserver } from '@vueuse/core'
import { computed, ref } from 'vue'

const props = withDefaults(defineProps<{
  src: string
  alt?: string
  placeholder?: string
  rootMargin?: string
}>(), {
  alt: '',
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

const showPlaceholder = computed(() => !loaded.value && !error.value)
const showError = computed(() => error.value)

function onLoad() {
  loaded.value = true
}

function onError() {
  error.value = true
}
</script>

<template>
  <div ref="el" class="lazy-media">
    <div v-if="showPlaceholder" class="placeholder">
      <img
        v-if="isIntersecting"
        :src="src"
        :alt="alt"
        class="lazy-img"
        loading="lazy"
        @load="onLoad"
        @error="onError"
      >
      <div class="placeholder-bg">
        <svg viewBox="0 0 24 24" class="placeholder-icon" fill="none" stroke="currentColor" stroke-width="1.5">
          <rect x="3" y="3" width="18" height="18" rx="2" ry="2" />
          <circle cx="8.5" cy="8.5" r="1.5" />
          <path d="M21 15l-5-5L5 21" />
        </svg>
      </div>
    </div>
    <div v-else-if="showError" class="placeholder-bg error">
      <svg viewBox="0 0 24 24" class="placeholder-icon" fill="none" stroke="currentColor" stroke-width="1.5">
        <circle cx="12" cy="12" r="10" />
        <line x1="12" y1="8" x2="12" y2="12" />
        <line x1="12" y1="16" x2="12.01" y2="16" />
      </svg>
    </div>
    <img
      v-else
      :src="src"
      :alt="alt"
      class="lazy-img loaded"
    >
  </div>
</template>

<style scoped>
.lazy-media {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.lazy-img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.lazy-img:not(.loaded) {
  position: absolute;
  opacity: 0;
  pointer-events: none;
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
