<script setup lang="ts">
defineProps<{ msg: string }>()
</script>

<template>
  <div class="greeting">
    <h1>{{ msg }}</h1>
  </div>
</template>

<style scoped>
.greeting {
  text-align: center;
  margin-top: 2rem;
}
</style>
