<script setup lang="ts">
definePageMeta({
  title: 'JSON Formatter',
  description: 'Format, validate, and minify messy JSON strings.',
  category: 'developer-tools',
  tags: ['code', 'data'],
})

const input = ref('')
const output = ref('')
const error = ref('')

function format() {
  try {
    output.value = JSON.stringify(JSON.parse(input.value), null, 2)
    error.value = ''
  } catch (e) {
    output.value = ''
    error.value = (e as Error).message
  }
}

function minify() {
  try {
    output.value = JSON.stringify(JSON.parse(input.value))
    error.value = ''
  } catch (e) {
    output.value = ''
    error.value = (e as Error).message
  }
}
</script>

<template>
  <ClientOnly>
    <UContainer class="py-8 max-w-7xl">
      <div class="mb-6">
        <h1 class="text-3xl font-bold tracking-tight mb-2">JSON Formatter</h1>
        <p class="text-(--ui-text-muted)">Format, validate, and minify JSON.</p>
      </div>

      <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <UCard>
          <template #header>
            <div class="flex items-center justify-between">
              <h2 class="font-semibold">Input</h2>
              <div class="flex gap-2">
                <UButton @click="format" icon="i-lucide-indent" color="primary" size="sm">
                  Format
                </UButton>
                <UButton @click="minify" icon="i-lucide-shrink" variant="outline" color="neutral" size="sm">
                  Minify
                </UButton>
              </div>
            </div>
          </template>
          <UTextarea
            v-model="input"
            class="font-mono text-sm w-full"
            placeholder='{"hello": "world"}'
            :rows="16"
          />
        </UCard>

        <div class="flex flex-col gap-4">
          <UAlert
            v-if="error"
            color="error"
            variant="subtle"
            icon="i-lucide-alert-circle"
            :title="error"
          />
          <UCard class="flex-1">
            <template #header>
              <h2 class="font-semibold">Output</h2>
            </template>
            <UTextarea
              :model-value="output"
              readonly
              class="font-mono text-sm w-full"
              placeholder="Result will appear here..."
              :rows="16"
            />
          </UCard>
        </div>
      </div>
    </UContainer>
  </ClientOnly>
</template>
