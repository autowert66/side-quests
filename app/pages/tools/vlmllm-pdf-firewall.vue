<script setup lang="ts">
definePageMeta({
  title: 'Zero-Trust PDF Firewall',
  description: 'Client-side heuristic engine to detect malicious PDF structures, invisible text, and prompt injections.[cite: 1]',
  category: 'security',
  tags: ['file', 'security', 'data'],
})

const fileInputRef = ref<HTMLInputElement | null>(null)
const isScanning = ref(false)
const progress = ref(0)
const progressText = ref('')
const riskScore = ref(0)
const logs = ref<{ message: string; severity: 'high' | 'medium' | 'low' | 'info'; snippet?: string }[]>([])
const rawExtractedText = ref('')

const stats = ref({
  pages: 0,
  rawWords: 0,
  anomalies: 0,
})

const tabs = [
  { label: 'Detection Engine', slot: 'logs', icon: 'i-lucide-shield-alert' },
  { label: 'Raw String Dump', slot: 'raw', icon: 'i-lucide-file-code' }
]

const verdict = computed(() => {
  if (riskScore.value === 0) return { text: 'Clean', subtext: 'No anomalies detected.', color: 'success' }
  if (riskScore.value < 40) return { text: 'Suspicious', subtext: 'Some unusual structures found.', color: 'warning' }
  return { text: 'Malicious', subtext: 'Definite Prompt Injection / Honeypot detected.[cite: 1]', color: 'error' }
})

const hasScanned = computed(() => logs.value.length > 0)

function addLog(message: string, severity: 'high' | 'medium' | 'low' | 'info', snippet?: string) {
  logs.value.push({ message, severity, snippet })
  if (severity === 'high' || severity === 'medium') stats.value.anomalies++
}

function getAlertColor(severity: string) {
  switch (severity) {
    case 'high': return 'error'
    case 'medium': return 'warning'
    case 'low': return 'info'
    default: return 'neutral'
  }
}

function triggerFileInput() {
  fileInputRef.value?.click()
}

function onFileChange(event: Event) {
  const target = event.target as HTMLInputElement
  if (target.files?.length) {
    startAnalysis(target.files[0])
  }
}

function onDrop(event: DragEvent) {
  if (event.dataTransfer?.files.length) {
    startAnalysis(event.dataTransfer.files[0])
  }
}

async function startAnalysis(file: File) {
  if (file.type !== 'application/pdf') {
    addLog('Invalid file type. Please upload a PDF.', 'high')
    return
  }

  isScanning.value = true
  progress.value = 5
  progressText.value = 'Initializing native scanning engine...[cite: 1]'
  logs.value = []
  riskScore.value = 0
  stats.value = { pages: 0, rawWords: 0, anomalies: 0 }
  rawExtractedText.value = ''

  try {
    const arrayBuffer = await file.arrayBuffer()
    progress.value = 25
    progressText.value = 'Parsing PDF Binary Structure...'

    // Decode binary as raw string to find heuristic markers without external libs
    const decoder = new TextDecoder('latin1')
    const rawString = decoder.decode(arrayBuffer)
    rawExtractedText.value = rawString.substring(0, 15000) + (rawString.length > 15000 ? '\n...[TRUNCATED]' : '')
    
    stats.value.rawWords = rawString.split(/\s+/).length
    
    // Estimate pages based on PDF structural tags
    const pageMatches = rawString.match(/\/Type\s*\/Page\b/g)
    stats.value.pages = pageMatches ? pageMatches.length : 1

    progress.value = 50
    progressText.value = 'Running Heuristic Checks...[cite: 1]'

    // 1. JavaScript / Active Content Check
    if (rawString.includes('/JS') || rawString.includes('/JavaScript')) {
      riskScore.value += 40
      addLog('Active Content: JavaScript embedded in PDF detected.', 'high', '/JS or /JavaScript dictionary found.')
    }

    // 2. Auto-execution (OpenAction)
    if (rawString.includes('/OpenAction') || rawString.includes('/AA')) {
      riskScore.value += 30
      addLog('Auto-execution: OpenAction or Additional Actions found.', 'high', '/OpenAction flag triggers actions on load.')
    }

    // 3. Invisible Text Render Mode (Tr 3)
    if (rawString.includes('3 Tr') || rawString.includes('3 Tz')) {
      riskScore.value += 30
      addLog('Invisible Text: Invisible render mode (Tr=3) detected.[cite: 1]', 'high', '3 Tr')
    }

    // 4. Zero-Width Characters
    if (/[\u200B-\u200D\uFEFF\u202E]/.test(rawString)) {
      riskScore.value += 40
      addLog('Obfuscation: Invisible Zero-Width characters detected.[cite: 1]', 'high', 'Contains unicode zero-width spaces/joiners.')
    }

    // 5. Common Prompt Injection Phrases in Raw Text
    const promptInjectionPhrases = ['ignore previous', 'system prompt', 'you are now', 'bypass']
    const lowerRaw = rawString.toLowerCase()
    for (const phrase of promptInjectionPhrases) {
      if (lowerRaw.includes(phrase)) {
        riskScore.value += 50
        addLog(`Prompt Injection: Suspicious instruction phrase found.`, 'high', `Found: "${phrase}"`)
      }
    }

    progress.value = 100
    progressText.value = 'Analysis Complete.'

    if (riskScore.value === 0) {
      addLog('Heuristic scan clear. No hidden textual overlays or malicious objects detected.[cite: 1]', 'info')
    }

    // Cap score at 100
    riskScore.value = Math.min(riskScore.value, 100)

  } catch (error) {
    addLog(`System Error: ${(error as Error).message}[cite: 1]`, 'high')
  } finally {
    setTimeout(() => {
      isScanning.value = false
    }, 1000)
  }
}
</script>

<template>
  <ClientOnly>
    <UContainer class="py-8 max-w-7xl">
      <div class="mb-6">
        <h1 class="text-3xl font-bold tracking-tight mb-2">Zero-Trust PDF Firewall</h1>
        <p class="text-(--ui-text-muted)">100% Client-Side Processing. No external servers.[cite: 1]</p>
      </div>

      <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
        
        <!-- Left Column: Input and Status -->
        <div class="space-y-6">
          <UCard v-if="!isScanning && !hasScanned">
            <div 
              class="border-2 border-dashed border-(--ui-border) rounded-xl p-8 text-center cursor-pointer bg-(--ui-bg-elevated) hover:opacity-80 transition-opacity"
              @click="triggerFileInput"
              @dragover.prevent
              @drop.prevent="onDrop"
            >
              <UIcon name="i-lucide-file-up" class="mx-auto size-10 text-(--ui-text-muted) mb-3" />
              <p class="text-sm font-medium mb-1">Select PDF Document[cite: 1]</p>
              <p class="text-xs text-(--ui-text-muted)">No file size limit. Processing is local.[cite: 1]</p>
              <input ref="fileInputRef" type="file" class="hidden" accept="application/pdf" @change="onFileChange">
            </div>
          </UCard>

          <UCard v-if="isScanning">
            <div class="space-y-4">
              <div class="flex justify-between items-center text-sm font-semibold">
                <span>{{ progressText }}</span>
                <span>{{ Math.round(progress) }}%</span>
              </div>
              <UProgress :value="progress" />
            </div>
          </UCard>

          <UCard v-if="hasScanned && !isScanning">
            <template #header>
              <div class="flex items-center justify-between">
                <h2 class="font-semibold text-lg">Analysis Verdict[cite: 1]</h2>
                <UButton @click="logs = []; rawExtractedText = ''" color="neutral" variant="ghost" icon="i-lucide-refresh-cw" size="sm">
                  Reset
                </UButton>
              </div>
            </template>
            
            <div class="flex items-center gap-6 mb-6">
              <UMeter :value="riskScore" :max="100" :color="verdict.color" indicator class="w-24 h-24" size="xl" />
              <div>
                <p class="text-2xl font-bold" :class="`text-var(--ui-${verdict.color})`">
                  {{ verdict.text }}
                </p>
                <p class="text-sm text-(--ui-text-muted) mt-1 leading-tight">
                  {{ verdict.subtext }}
                </p>
              </div>
            </div>

            <div class="grid grid-cols-2 gap-3 text-sm">
              <div class="bg-(--ui-bg-elevated) p-3 rounded-lg border border-(--ui-border)">
                <span class="text-(--ui-text-muted) block text-xs">Pages Found</span>
                <span class="font-bold">{{ stats.pages }}</span>
              </div>
              <div class="bg-(--ui-bg-elevated) p-3 rounded-lg border border-(--ui-border)">
                <span class="text-(--ui-text-muted) block text-xs">Anomalies</span>
                <span class="font-bold text-red-500">{{ stats.anomalies }}[cite: 1]</span>
              </div>
            </div>
          </UCard>
        </div>

        <!-- Right Column: Tabs and Logs -->
        <UCard class="lg:col-span-2 flex flex-col h-[700px]">
          <UTabs :items="tabs" class="flex-1 flex flex-col overflow-hidden">
            <template #logs>
              <div class="h-full overflow-y-auto space-y-3 p-4">
                <div v-if="!hasScanned" class="h-full flex items-center justify-center text-(--ui-text-muted) text-sm italic">
                  Awaiting document ingestion...[cite: 1]
                </div>
                <div v-else>
                  <UAlert
                    v-for="(log, idx) in logs"
                    :key="idx"
                    :color="getAlertColor(log.severity) as any"
                    :title="`[${log.severity.toUpperCase()}] ${log.message}`"
                    :icon="log.severity === 'high' ? 'i-lucide-alert-triangle' : 'i-lucide-info'"
                    class="mb-3"
                    variant="subtle"
                  >
                    <template #description v-if="log.snippet">
                      <div class="mt-2 bg-(--ui-bg) p-2 rounded text-xs font-mono border border-(--ui-border) break-all">
                        {{ log.snippet }}
                      </div>
                    </template>
                  </UAlert>
                </div>
              </div>
            </template>

            <template #raw>
              <div class="h-full flex flex-col p-4">
                <UAlert color="info" variant="subtle" icon="i-lucide-file-binary" class="mb-4">
                  <template #title>
                    <strong>How this works:</strong> We scan the raw binary string for heuristic indicators of malicious intent or hidden elements.
                  </template>
                </UAlert>
                
                <UTextarea
                  :model-value="rawExtractedText"
                  readonly
                  class="font-mono text-sm flex-1"
                  placeholder="Raw binary dump will appear here..."
                  :rows="20"
                />
              </div>
            </template>
          </UTabs>
        </UCard>

      </div>
    </UContainer>
    <template #fallback>
      <USkeleton class="h-[800px] w-full max-w-7xl mx-auto mt-8" />
    </template>
  </ClientOnly>
</template>
