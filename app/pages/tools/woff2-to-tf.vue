<script setup lang="ts">
definePageMeta({
  title: 'WOFF2 to TTF Converter',
  description: 'Convert WOFF2 web fonts into TrueType (TTF) desktop fonts locally in your browser.',
  category: 'developer-tools',
  tags: ['file', 'data'],
})

const file = ref<File | null>(null)
const ttfData = ref<Uint8Array | null>(null)
const isConverting = ref(false)
const error = ref('')
const isReady = ref(false)
const previewText = ref('The quick brown fox jumps over the lazy dog.\n0123456789\n?@#$%&*()')

onMounted(() => {
  // Inject a global style for the font preview to avoid inline style attributes
  if (!document.getElementById('woff2-preview-style')) {
    const style = document.createElement('style')
    style.id = 'woff2-preview-style'
    style.innerHTML = '.font-preview textarea { font-family: "CustomFontPreview", sans-serif !important; }'
    document.head.appendChild(style)
  }

  // Load the conversion engine (WebAssembly wrapper) dynamically using native ES modules
  // This avoids build-time import errors and injects the function globally
  if ((window as any).woff2Decompress) {
    isReady.value = true
    return
  }

  const script = document.createElement('script')
  script.type = 'module'
  script.innerHTML = `
    import { decompress } from 'https://cdn.jsdelivr.net/npm/woff2-encoder@2.0.0/+esm';
    window.woff2Decompress = decompress;
    window.dispatchEvent(new Event('woff2-ready'));
  `
  document.head.appendChild(script)

  window.addEventListener('woff2-ready', () => {
    isReady.value = true
  }, { once: true })
})

function handleFileChange(e: Event) {
  const target = e.target as HTMLInputElement
  if (target && target.files && target.files.length > 0) {
    file.value = target.files[0]
    ttfData.value = null
    error.value = ''
  }
}

async function convert() {
  if (!file.value || !isReady.value) return
  
  isConverting.value = true
  error.value = ''
  ttfData.value = null

  try {
    const arrayBuffer = await file.value.arrayBuffer()
    const outputBuffer = await (window as any).woff2Decompress(new Uint8Array(arrayBuffer))
    
    ttfData.value = outputBuffer

    // Dynamically load the converted font for preview via the FontFace API
    const font = new FontFace('CustomFontPreview', outputBuffer)
    await font.load()
    document.fonts.add(font)
  } catch (e: any) {
    error.value = e.message || 'Failed to convert or parse the font. The file might be corrupted.'
  } finally {
    isConverting.value = false
  }
}

function download() {
  if (!ttfData.value || !file.value) return
  
  const blob = new Blob([ttfData.value], { type: 'font/ttf' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  
  a.href = url
  a.download = file.value.name.replace(/\.woff2$/i, '.ttf')
  a.click()
  
  URL.revokeObjectURL(url)
}
</script>

<template>
  <ClientOnly>
    <UContainer class="py-8 max-w-7xl">
      <div class="mb-6">
        <h1 class="text-3xl font-bold tracking-tight mb-2">WOFF2 to TTF</h1>
        <p class="text-(--ui-text-muted)">Convert web fonts into installable desktop fonts entirely in your browser.</p>
      </div>

      <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <UCard>
          <template #header>
            <div class="flex items-center justify-between">
              <h2 class="font-semibold">Input</h2>
              <UButton
                @click="convert"
                :loading="isConverting"
                :disabled="!file || !isReady"
                icon="i-lucide-arrow-right"
                color="primary"
                size="sm"
              >
                Convert
              </UButton>
            </div>
          </template>

          <div class="space-y-6">
            <UFormField label="Select WOFF2 File" description="Your file is processed locally and never uploaded to a server.">
              <UInput
                type="file"
                accept=".woff2"
                @change="handleFileChange"
                icon="i-lucide-file"
                class="w-full"
                placeholder="Upload a .woff2 file..."
              />
            </UFormField>

            <div v-if="!isReady" class="flex items-center gap-2 text-sm text-(--ui-text-muted)">
              <UIcon name="i-lucide-loader" class="size-4" />
              <span>Initializing WebAssembly conversion engine...</span>
            </div>

            <UAlert
              v-if="error"
              color="error"
              variant="subtle"
              icon="i-lucide-alert-circle"
              :title="error"
            />
          </div>
        </UCard>

        <UCard class="flex-1">
          <template #header>
            <div class="flex items-center justify-between">
              <h2 class="font-semibold">Output & Preview</h2>
              <UButton
                v-if="ttfData"
                @click="download"
                icon="i-lucide-download"
                color="success"
                size="sm"
              >
                Download TTF
              </UButton>
            </div>
          </template>

          <div v-if="ttfData" class="space-y-6">
            <UAlert
              color="success"
              variant="subtle"
              icon="i-lucide-check-circle"
              title="Conversion Successful!"
              description="Your TTF file is ready to download. You can preview the font below."
            />

            <UFormField label="Font Preview" description="Type in the text area to test the converted font.">
              <UTextarea
                v-model="previewText"
                :rows="10"
                class="w-full text-xl font-preview"
                placeholder="Type here to test..."
              />
            </UFormField>
          </div>

          <div v-else class="flex flex-col items-center justify-center min-h-[16rem] text-(--ui-text-muted)">
            <UIcon name="i-lucide-file-type-2" class="size-12 mb-4 opacity-50" />
            <p class="text-center">Upload and convert a WOFF2 file to generate a TTF.<br>The preview will appear here.</p>
          </div>
        </UCard>
      </div>
    </UContainer>

    <template #fallback>
      <UContainer class="py-8 max-w-7xl">
        <div class="mb-6 space-y-2">
          <USkeleton class="h-10 max-w-sm" />
          <USkeleton class="h-6 max-w-md" />
        </div>
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
          <USkeleton class="min-h-[20rem]" />
          <USkeleton class="min-h-[20rem]" />
        </div>
      </UContainer>
    </template>
  </ClientOnly>
</template>
