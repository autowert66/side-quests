<script setup lang="ts">
const filename = ref('')
const code = ref('')

const normalized = computed(() => {
  let slug = filename.value.trim()
  if (slug.endsWith('.vue')) {
    slug = slug.slice(0, -4)
  }
  slug = slug
    .toLowerCase()
    .replace(/[\s_]+/g, '-')
    .replace(/[^a-z0-9-]/g, '')
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '')
  return slug
})

const valid = computed(() => normalized.value.length > 0)

const buttonLabel = computed(() =>
  valid.value && code.value ? 'Open Issue on GitHub' : 'Fill out on GitHub',
)

function openIssue() {
  const params = new URLSearchParams()
  params.set('template', 'add-tool.yml')
  if (valid.value) {
    params.set('title', `Add Tool: ${normalized.value}`)
    params.set('page_slug', normalized.value)
  }
  if (code.value) {
    params.set('page_code', code.value)
  }

  const url = `https://github.com/autowert66/side-quests/issues/new?${params.toString()}`
  window.open(url, '_blank')
}

onMounted(() => {
  const hash = window.location.hash.slice(1)
  if (!hash) return

  try {
    const params = new URLSearchParams(hash)
    const slug = params.get('page_slug')
    const codeB64 = params.get('page_code')

    if (slug) {
      filename.value = slug
    }
    if (codeB64) {
      const normalizedB64 = codeB64.replace(/-/g, '+').replace(/_/g, '/')
      code.value = atob(normalizedB64)
    }
  } catch {
    // ignore malformed hash
  }
})
</script>

<template>
  <ClientOnly>
    <UContainer class="py-8 max-w-2xl">
      <div class="mb-8">
        <h1 class="text-3xl font-bold tracking-tight mb-1">Add Tool</h1>
        <p class="text-(--ui-text-muted)">
          Fill out the form to open a pre-filled GitHub issue. Submit the issue
          to trigger the tool creation workflow.
        </p>
      </div>

      <div class="space-y-6">
        <UFormField
          label="Filename"
          :help="`Resulting file: ${normalized || '...'}.vue`"
        >
          <UInput
            v-model="filename"
            placeholder="my-new-tool"
            size="lg"
            class="w-full max-w-md"
          />
        </UFormField>

        <UFormField label="Vue Component Code">
          <UTextarea
            v-model="code"
            placeholder="<script setup lang=&#34;ts&#34;&gt;
definePageMeta({
  title: 'My Tool',
  description: 'Does something useful.',
  category: 'developer-tools',
  tags: ['code'],
})
&lt;/script&gt;

&lt;template&gt;
  &lt;ClientOnly&gt;
    &lt;div&gt;...&lt;/div&gt;
  &lt;/ClientOnly&gt;
&lt;/template&gt;"
            :rows="16"
            autoresize
            class="w-full font-mono text-sm"
          />
        </UFormField>

        <UButton
          :label="buttonLabel"
          icon="i-lucide-external-link"
          :color="valid && code ? 'primary' : 'neutral'"
          :variant="valid && code ? 'solid' : 'outline'"
          size="lg"
          block
          @click="openIssue"
        />
      </div>
    </UContainer>
  </ClientOnly>
</template>
