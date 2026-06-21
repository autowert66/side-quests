<script setup lang="ts">
definePageMeta({
  title: 'Webloc Generator',
  description: 'Convert URLs into macOS .webloc shortcut files with custom names and XML preview.',
  category: 'utility-tools',
  tags: ['file', 'text', 'media'],
})

const url = ref('')
const filename = ref('')
const error = ref('')
const copied = ref(false)

const isValidUrl = computed(() => {
  if (!url.value.trim()) return true
  try {
    new URL(url.value)
    return true
  } catch {
    return false
  }
})

const weblocXml = computed(() => {
  if (!url.value.trim() || !isValidUrl.value) return ''
  
  // Escape XML special characters to ensure valid plist
  const cleanUrl = url.value.trim()
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;')

  return `<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>URL</key>
    <string>${cleanUrl}</string>
</dict>
</plist>`
})

const finalFilename = computed(() => {
  let name = filename.value.trim()
  
  if (!name) {
    if (!url.value.trim() || !isValidUrl.value) {
      return 'shortcut.webloc'
    }
    try {
      const parsed = new URL(url.value)
      name = parsed.hostname.replace(/^www\./, '') || 'shortcut'
    } catch {
      name = 'shortcut'
    }
  }

  if (!name.toLowerCase().endsWith('.webloc')) {
    name += '.webloc'
  }
  
  return name
})

function download() {
  try {
    if (!weblocXml.value) return
    const blob = new Blob([weblocXml.value], { type: 'application/xml' })
    const blobUrl = URL.createObjectURL(blob)
    
    const a = document.createElement('a')
    a.href = blobUrl
    a.download = finalFilename.value
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    
    URL.revokeObjectURL(blobUrl)
    error.value = ''
  } catch (e) {
    error.value = (e as Error).message
  }
}

async function copyToClipboard() {
  try {
    if (!weblocXml.value) return
    await navigator.clipboard.writeText(weblocXml.value)
    copied.value = true
    setTimeout(() => { copied.value = false }, 2000)
    error.value = ''
  } catch (e) {
    error.value = 'Failed to copy to clipboard: ' + (e as Error).message
  }
}
</script>

<template>
  <ClientOnly>
    <UContainer class="py-8 max-w-7xl">
      <div class="mb-6">
        <h1 class="text-3xl font-bold tracking-tight mb-2">macOS Webloc Generator</h1>
        <p class="text-(--ui-text-muted)">Generate macOS `.webloc` shortcut files from any URL. Preview the raw XML property list and set custom file names.</p>
      </div>

      <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <div class="flex flex-col gap-4">
          <UCard>
            <template #header>
              <h2 class="font-semibold">Shortcut Configuration</h2>
            </template>
            
            <div class="space-y-4">
              <UFormField label="Target URL" description="The web address the shortcut should open.">
                <UInput
                  v-model="url"
                  type="url"
                  icon="i-lucide-link"
                  placeholder="https://jumpdesktop.com/downloads/connect/mac"
                  class="w-full"
                />
              </UFormField>

              <UFormField label="Custom Filename (Optional)" description="Leave blank to auto-generate from the domain.">
                <UInput
                  v-model="filename"
                  icon="i-lucide-file-text"
                  placeholder="Connect for macOS"
                  class="w-full"
                >
                  <template #trailing>
                    <span class="text-sm text-(--ui-text-muted) px-2">.webloc</span>
                  </template>
                </UInput>
              </UFormField>
            </div>
          </UCard>

          <UAlert
            v-if="!isValidUrl"
            color="error"
            variant="subtle"
            icon="i-lucide-alert-circle"
            title="Invalid URL"
            description="Please enter a valid URL including the protocol (e.g., https://)."
          />
          
          <UAlert
            v-if="error"
            color="error"
            variant="subtle"
            icon="i-lucide-alert-triangle"
            :title="error"
          />
        </div>

        <UCard class="flex flex-col h-full">
          <template #header>
            <div class="flex items-center justify-between">
              <h2 class="font-semibold">Generated Plist Preview</h2>
              <div class="flex gap-2">
                <UButton 
                  @click="copyToClipboard" 
                  :icon="copied ? 'i-lucide-check' : 'i-lucide-copy'" 
                  variant="outline" 
                  color="neutral" 
                  size="sm"
                  :disabled="!weblocXml"
                >
                  {{ copied ? 'Copied!' : 'Copy XML' }}
                </UButton>
                <UButton 
                  @click="download" 
                  icon="i-lucide-download" 
                  color="primary" 
                  size="sm"
                  :disabled="!weblocXml"
                >
                  Download .webloc
                </UButton>
              </div>
            </div>
          </template>
          
          <div class="space-y-2 flex-1">
            <div class="flex items-center gap-2 mb-2">
              <UBadge color="neutral" variant="subtle" class="font-mono text-xs">
                Output File: {{ finalFilename }}
              </UBadge>
            </div>
            <UTextarea
              :model-value="weblocXml"
              readonly
              class="font-mono text-sm w-full"
              placeholder="XML preview will appear here once you enter a URL..."
              :rows="14"
            />
          </div>
        </UCard>
      </div>
    </UContainer>
    
    <template #fallback>
      <UContainer class="py-8 max-w-7xl">
        <div class="mb-6">
          <USkeleton class="h-10 w-64 mb-2" />
          <USkeleton class="h-6 w-96" />
        </div>
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
          <USkeleton class="h-[300px] w-full" />
          <USkeleton class="h-[300px] w-full" />
        </div>
      </UContainer>
    </template>
  </ClientOnly>
</template>
