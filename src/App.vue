<script setup>
import CatVideoGame from './components/CatVideoGame.vue'
</script>

<template>
  <div class="app-shell">
    <CatVideoGame />
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}
</style>
