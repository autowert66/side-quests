<script setup lang="ts">
definePageMeta({
  title: 'JSON Formatter',
  description: 'Format and validate messy JSON strings.',
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
    <UContainer class="py-8">
      <h1 class="text-2xl font-bold tracking-tight mb-1">JSON Formatter</h1>
      <p class="text-(--ui-text-muted) mb-6">Format, validate, and minify JSON.</p>

      <UForm @submit.prevent="format" class="space-y-4">
        <UFormField label="Input">
          <UTextarea
            v-model="input"
            placeholder='{"hello": "world"}'
            :rows="10"
          />
        </UFormField>

        <div class="flex gap-2">
          <UButton type="submit" icon="i-lucide-indent" color="primary">
            Format
          </UButton>
          <UButton @click="minify" icon="i-lucide-shrink" variant="outline">
            Minify
          </UButton>
        </div>

        <UFormField v-if="error" label="Error">
          <UAlert color="error" variant="subtle" :title="error" />
        </UFormField>

        <UFormField v-if="output" label="Output">
          <UTextarea
            :model-value="output"
            readonly
            :rows="10"
          />
        </UFormField>
      </UForm>
    </UContainer>
  </ClientOnly>
</template>
