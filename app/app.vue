<template>
  <div>
    <NuxtRouteAnnouncer />
    <NuxtPage />
    <Toaster rich-colors position="top-center" />
  </div>
</template>

<script setup lang="ts">
import { Toaster } from 'vue-sonner'
</script>
