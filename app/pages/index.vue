<script setup lang="ts">
const router = useRouter()

const tools = computed(() => {
  return router
    .getRoutes()
    .filter((route) => route.path.startsWith('/tools/'))
    .map((route) => ({
      path: route.path,
      title: (route.meta.title as string) || 'Untitled Tool',
      description: (route.meta.description as string) || '',
      category: (route.meta.category as string) || 'uncategorized',
      tags: (route.meta.tags as string[]) || [],
    }))
    .sort((a, b) => a.title.localeCompare(b.title))
})

const search = ref('')
const selectedCategory = ref<string | null>(null)
const selectedTags = ref<string[]>([])

const PREDEFINED_TAGS = [
  'text',
  'image',
  'media',
  'code',
  'data',
  'math',
  'security',
  'network',
  'color',
  'time',
  'crypto',
  'file',
]

const categories = computed(() => {
  return [...new Set(tools.value.map((t) => t.category))].sort()
})

const filteredTools = computed(() => {
  let result = tools.value

  if (search.value) {
    const q = search.value.toLowerCase()
    result = result.filter(
      (t) =>
        t.title.toLowerCase().includes(q) ||
        t.description.toLowerCase().includes(q),
    )
  }

  if (selectedCategory.value) {
    result = result.filter((t) => t.category === selectedCategory.value)
  }

  if (selectedTags.value.length > 0) {
    result = result.filter((t) =>
      selectedTags.value.every((tag) => t.tags.includes(tag)),
    )
  }

  return result
})

function toggleTag(tag: string) {
  const idx = selectedTags.value.indexOf(tag)
  if (idx === -1) {
    selectedTags.value = [...selectedTags.value, tag]
  } else {
    selectedTags.value = selectedTags.value.filter((t) => t !== tag)
  }
}
</script>

<template>
  <UContainer class="py-8">
    <div class="mb-8 flex items-start justify-between">
      <div>
        <h1 class="text-3xl font-bold tracking-tight mb-1">Tools</h1>
        <p class="text-(--ui-text-muted)">
          Search and browse all available tools.
        </p>
      </div>
      <UButton
        icon="i-lucide-plus"
        color="primary"
        variant="outline"
        to="/add"
        aria-label="Add new tool"
      />
    </div>

    <div class="mb-8 space-y-4">
      <UInput
        v-model="search"
        placeholder="Search tools..."
        icon="i-lucide-search"
        size="lg"
      />

      <div v-if="categories.length > 1" class="flex flex-wrap gap-2">
        <UButton
          v-for="cat in categories"
          :key="cat"
          :label="cat"
          :variant="selectedCategory === cat ? 'solid' : 'outline'"
          :color="selectedCategory === cat ? 'primary' : 'neutral'"
          size="xs"
          @click="selectedCategory = selectedCategory === cat ? null : cat"
        />
      </div>

      <div class="flex flex-wrap gap-1.5">
        <UButton
          v-for="tag in PREDEFINED_TAGS"
          :key="tag"
          :label="tag"
          :variant="selectedTags.includes(tag) ? 'solid' : 'outline'"
          :color="selectedTags.includes(tag) ? 'primary' : 'neutral'"
          size="xs"
          @click="toggleTag(tag)"
        />
      </div>
    </div>

    <div
      v-if="tools.length === 0"
      class="text-center py-16 border border-dashed border-(--ui-border) rounded-xl"
    >
      <UIcon name="i-lucide-wrench" class="size-10 mx-auto mb-4 opacity-40" />
      <p class="text-(--ui-text-muted) text-lg mb-1">No tools yet</p>
      <p class="text-(--ui-text-muted) text-sm">
        Drop a <code class="text-xs bg-(--ui-bg-elevated) px-1 py-0.5 rounded">.vue</code> file into
        <code class="text-xs bg-(--ui-bg-elevated) px-1 py-0.5 rounded">app/pages/tools/</code>
        to get started.
      </p>
    </div>

    <div
      v-else-if="filteredTools.length === 0"
      class="text-center py-12 text-(--ui-text-muted)"
    >
      <UIcon name="i-lucide-search-x" class="size-10 mx-auto mb-3 opacity-50" />
      <p class="text-lg">No tools match your filters.</p>
    </div>

    <div
      v-else
      class="grid gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4"
    >
      <NuxtLink
        v-for="tool in filteredTools"
        :key="tool.path"
        :to="tool.path"
        class="block group"
      >
        <UCard class="h-full transition-shadow group-hover:shadow-md cursor-pointer">
          <template #header>
            <h2 class="font-semibold truncate">{{ tool.title }}</h2>
          </template>

          <p class="text-sm text-(--ui-text-muted) line-clamp-2 min-h-[2.5rem]">
            {{ tool.description || 'No description.' }}
          </p>

          <template #footer>
            <div class="flex flex-wrap gap-1">
              <UBadge
                :label="tool.category"
                color="primary"
                variant="soft"
                size="xs"
              />
              <UBadge
                v-for="tag in tool.tags"
                :key="tag"
                :label="tag"
                color="neutral"
                variant="subtle"
                size="xs"
              />
            </div>
          </template>
        </UCard>
      </NuxtLink>
    </div>
  </UContainer>
</template>
