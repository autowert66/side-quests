<script setup lang="ts">
const route = useRoute()
const colorMode = useColorMode()

const isHome = computed(() => route.path === '/')

function toggleColorMode() {
  colorMode.preference = colorMode.preference === 'dark' ? 'light' : 'dark'
}
</script>

<template>
  <div class="min-h-screen flex flex-col">
    <header class="border-b border-(--ui-border) bg-(--ui-bg) sticky top-0 z-50">
      <UContainer class="flex items-center justify-between h-14">
        <div class="flex items-center gap-3">
          <UButton
            v-if="!isHome"
            icon="i-lucide-arrow-left"
            to="/"
            variant="ghost"
            color="neutral"
            size="sm"
            aria-label="Back to tools"
          />
          <NuxtLink to="/" class="font-semibold text-lg tracking-tight select-none">
            Side Quests
          </NuxtLink>
        </div>
        <UButton
          :icon="colorMode.preference === 'dark' ? 'i-lucide-sun' : 'i-lucide-moon'"
          variant="ghost"
          color="neutral"
          size="sm"
          @click="toggleColorMode"
          aria-label="Toggle color mode"
        />
      </UContainer>
    </header>

    <main class="flex-1">
      <slot />
    </main>
  </div>
</template>
