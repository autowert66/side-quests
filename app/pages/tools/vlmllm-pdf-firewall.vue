<script setup lang="ts">
definePageMeta({
  title: 'Zero-Trust PDF Firewall',
  description: 'Client-side Rasterization, OCR, and Heuristic Diffing Engine.',
  category: 'security',
  tags: ['file', 'security', 'data'],
})

// Inject required external scripts dynamically since standard imports are forbidden.
useHead({
  script: [
    { src: 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.min.js', defer: true },
    { src: 'https://cdn.jsdelivr.net/npm/tesseract.js@5.0.5/dist/tesseract.min.js', defer: true },
    { src: 'https://cdnjs.cloudflare.com/ajax/libs/diff_match_patch/20121119/diff_match_patch.js', defer: true }
  ]
})

const fileInputRef = ref<HTMLInputElement | null>(null)
const viewerContainerRef = ref<HTMLElement | null>(null)

const isScanning = ref(false)
const progress = ref(0)
const progressTitle = ref('')
const progressText = ref('')
const riskScore = ref(0)

const logs = ref<{ message: string; severity: 'high' | 'medium' | 'low' | 'info'; snippet?: string }[]>([])
const diffResults = ref<[number, string][]>([])

const stats = ref({
  pages: 0,
  rawWords: 0,
  ocrWords: 0,
  anomalies: 0,
})

const tabs = [
  { label: 'Detection Engine', slot: 'logs', icon: 'i-lucide-shield-alert' },
  { label: 'Visual Diff (OCR vs Raw)', slot: 'diff', icon: 'i-lucide-file-diff' },
  { label: 'PDF Viewer', slot: 'viewer', icon: 'i-lucide-file-image' }
]

const verdict = computed(() => {
  if (riskScore.value === 0) return { text: 'Clean', subtext: 'OCR and Programmatic text align well.', color: 'success' }
  if (riskScore.value < 40) return { text: 'Suspicious', subtext: 'Unusual programmatic structures found.', color: 'warning' }
  return { text: 'Malicious', subtext: 'Definite Prompt Injection / Honeypot detected.', color: 'error' }
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

function updateProgress(percent: number, title: string) {
  progress.value = percent
  progressText.value = `${Math.round(percent)}%`
  progressTitle.value = title
}

async function startAnalysis(file: File) {
  if (file.type !== 'application/pdf') {
    addLog('Invalid file type. Please upload a PDF.', 'high')
    return
  }

  // Ensure scripts are loaded
  if (!(window as any).pdfjsLib || !(window as any).Tesseract || !(window as any).diff_match_patch) {
    alert("Analysis engines are still downloading. Please wait a few seconds and try again.")
    return
  }

  const pdfjsLib = (window as any).pdfjsLib
  const Tesseract = (window as any).Tesseract
  const DiffMatchPatch = (window as any).diff_match_patch

  pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.worker.min.js'
  const dmp = new DiffMatchPatch()

  // Reset State[cite: 1]
  isScanning.value = true
  logs.value = []
  diffResults.value = []
  riskScore.value = 0
  stats.value = { pages: 0, rawWords: 0, ocrWords: 0, anomalies: 0 }
  if (viewerContainerRef.value) viewerContainerRef.value.innerHTML = ''
  
  let globalRawText = ""
  let globalOcrText = ""

  try {
    updateProgress(5, "Initializing Tesseract OCR Worker...[cite: 1]")
    const worker = await Tesseract.createWorker('eng')

    updateProgress(15, "Parsing PDF Structure...[cite: 1]")
    const arrayBuffer = await file.arrayBuffer()
    const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise
    
    stats.value.pages = pdf.numPages

    // Step 1: Metadata Extraction[cite: 1]
    const metadata = await pdf.getMetadata()
    const producer = metadata?.info?.Producer || ''
    const creator = metadata?.info?.Creator || ''
    if (/(pypdf|pdf-lib|ghostscript)/i.test(producer) || /(pypdf|pdf-lib|ghostscript)/i.test(creator)) {
      riskScore.value += 10
      addLog(`Programmatic PDF generation detected.[cite: 1]`, 'low', `Producer: ${producer}`)
    }

    // Step 2: Page by Page Analysis[cite: 1]
    for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
      let pageProgressBase = 15 + ((pageNum - 1) / pdf.numPages) * 70
      updateProgress(pageProgressBase, `Analyzing Page ${pageNum}/${pdf.numPages}...[cite: 1]`)

      const page = await pdf.getPage(pageNum)
      const viewport = page.getViewport({ scale: 1.5 })

      // Render to Canvas[cite: 1]
      const canvas = document.createElement('canvas')
      const ctx = canvas.getContext('2d')
      if (!ctx) continue
      
      canvas.width = viewport.width
      canvas.height = viewport.height
      canvas.className = "max-w-full h-auto mb-4 border border-(--ui-border)"
      await page.render({ canvasContext: ctx, viewport: viewport }).promise
      
      if (viewerContainerRef.value) viewerContainerRef.value.appendChild(canvas)

      // Programmatic Text Extraction & Heuristics[cite: 1]
      const textContent = await page.getTextContent()
      const opList = await page.getOperatorList()
      let pageRawText = ""

      textContent.items.forEach((item: any) => {
        const str = item.str
        if (!str.trim()) return
        pageRawText += str + " "

        // Font Size Check[cite: 1]
        const fontSize = Math.hypot(item.transform[0], item.transform[1])
        if (fontSize > 0 && fontSize < 1) {
          riskScore.value += 30
          addLog(`Page ${pageNum}: Micro-text (< 1pt) detected.[cite: 1]`, 'high', `Text: "${str}"`)
        }

        // Zero-Width Check[cite: 1]
        if (/[\u200B-\u200D\uFEFF\u202E]/.test(str)) {
          riskScore.value += 40
          addLog(`Page ${pageNum}: Invisible Zero-Width characters detected.[cite: 1]`, 'high', str.replace(/[\u200B-\u200D\uFEFF\u202E]/g, '[ZERO-WIDTH]'))
        }

        // Out of Bounds Check[cite: 1]
        const x = item.transform[4]
        const y = item.transform[5]
        if (x < -100 || x > viewport.width + 100 || y < -100 || y > viewport.height + 100) {
          riskScore.value += 20
          addLog(`Page ${pageNum}: Out-of-bounds text detected.[cite: 1]`, 'medium', `Text: "${str}"`)
        }
      })
      globalRawText += pageRawText + "\n"

      // Operator Check (Tr=3)[cite: 1]
      for (let i = 0; i < opList.fnArray.length; i++) {
        if (opList.fnArray[i] === pdfjsLib.OPS.setTextRenderingMode && opList.argsArray[i][0] === 3) {
          riskScore.value += 30
          addLog(`Page ${pageNum}: Invisible render mode (Tr=3) detected.[cite: 1]`, 'high')
          break
        }
      }

      // OCR Visual Extraction[cite: 1]
      updateProgress(pageProgressBase + (35 / pdf.numPages), `Running OCR on Page ${pageNum}...[cite: 1]`)
      const ocrResult = await worker.recognize(canvas)
      globalOcrText += ocrResult.data.text + "\n"
    }

    // Step 3: Diff Engine[cite: 1]
    updateProgress(90, "Diffing Visual vs Programmatic Text...[cite: 1]")
    const cleanRaw = globalRawText.replace(/\s+/g, ' ').trim()
    const cleanOcr = globalOcrText.replace(/\s+/g, ' ').trim()

    stats.value.rawWords = cleanRaw.split(/\s+/).length
    stats.value.ocrWords = cleanOcr.split(/\s+/).length

    const diffs = dmp.diff_main(cleanOcr, cleanRaw)
    dmp.diff_cleanupSemantic(diffs)
    diffResults.value = diffs

    let hasSubstantialDiff = false
    diffs.forEach((part: [number, string]) => {
      // 1 signifies insertion in Raw (present in code, hidden from OCR)[cite: 1]
      if (part[0] === 1 && part[1].length > 15) { 
        hasSubstantialDiff = true
      }
    })

    if (hasSubstantialDiff) {
      riskScore.value += 50
      addLog(`Substantial Text Mismatch![cite: 1]`, 'high', `The programmatic text contains significant instructions visually hidden from the human reader. Check the 'Visual Diff' tab.[cite: 1]`)
    } else {
      addLog(`OCR and Programmatic text align well. No hidden textual overlays detected.[cite: 1]`, 'info')
    }

    updateProgress(100, "Analysis Complete.[cite: 1]")
    await worker.terminate()
    riskScore.value = Math.min(riskScore.value, 100)

  } catch (error) {
    addLog(`System Error: ${(error as Error).message}`, 'high')
  } finally {
    setTimeout(() => {
      isScanning.value = false
    }, 1500)
  }
}
</script>

<template>
  <ClientOnly>
    <UContainer class="py-8 max-w-7xl">
      <div class="mb-6">
        <h1 class="text-3xl font-bold tracking-tight mb-2">Zero-Trust PDF Firewall</h1>
        <p class="text-(--ui-text-muted)">Client-side Rasterization, OCR, and Heuristic Diffing Engine.[cite: 1]</p>
      </div>

      <div class="grid grid-cols-1 xl:grid-cols-3 gap-8">
        
        <!-- Left Column: Input and Status -->
        <div class="xl:col-span-1 space-y-6">
          <UCard v-if="!isScanning && !hasScanned">
            <div 
              class="border-2 border-dashed border-(--ui-border) rounded-xl p-8 text-center cursor-pointer bg-(--ui-bg-elevated) hover:opacity-80 transition-opacity"
              @click="triggerFileInput"
              @dragover.prevent
              @drop.prevent="onDrop"
            >
              <UIcon name="i-lucide-file-up" class="mx-auto size-10 text-(--ui-text-muted) mb-3" />
              <p class="text-sm font-medium mb-1">Select PDF Document</p>
              <p class="text-xs text-(--ui-text-muted)">No file size limit. Processing is local.[cite: 1]</p>
              <input ref="fileInputRef" type="file" class="hidden" accept="application/pdf" @change="onFileChange">
            </div>
          </UCard>

          <UCard v-if="isScanning">
            <div class="space-y-4">
              <div class="flex justify-between items-center text-sm font-semibold">
                <span class="truncate pr-4">{{ progressTitle }}</span>
                <span>{{ progressText }}</span>
              </div>
              <UProgress :value="progress" />
            </div>
          </UCard>

          <UCard v-if="hasScanned && !isScanning">
            <template #header>
              <div class="flex items-center justify-between">
                <h2 class="font-semibold text-lg">Analysis Verdict[cite: 1]</h2>
                <UButton @click="logs = []; diffResults = []; hasScanned = false" color="neutral" variant="ghost" icon="i-lucide-refresh-cw" size="sm">
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
                <span class="text-(--ui-text-muted) block text-xs">Pages</span>
                <span class="font-bold">{{ stats.pages }}</span>
              </div>
              <div class="bg-(--ui-bg-elevated) p-3 rounded-lg border border-(--ui-border)">
                <span class="text-(--ui-text-muted) block text-xs">Anomalies</span>
                <span class="font-bold text-[var(--ui-error)]">{{ stats.anomalies }}</span>
              </div>
              <div class="bg-(--ui-bg-elevated) p-3 rounded-lg border border-(--ui-border)">
                <span class="text-(--ui-text-muted) block text-xs">OCR Words</span>
                <span class="font-bold">{{ stats.ocrWords }}</span>
              </div>
              <div class="bg-(--ui-bg-elevated) p-3 rounded-lg border border-(--ui-border)">
                <span class="text-(--ui-text-muted) block text-xs">Raw Words</span>
                <span class="font-bold">{{ stats.rawWords }}</span>
              </div>
            </div>
          </UCard>
        </div>

        <!-- Right Column: Tabs (Logs, Diff, Viewer) -->
        <UCard class="xl:col-span-2 flex flex-col h-[800px]">
          <UTabs :items="tabs" class="flex-1 flex flex-col overflow-hidden">
            
            <!-- Detection Engine Logs[cite: 1] -->
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

            <!-- Visual Diff Tab[cite: 1] -->
            <template #diff>
              <div class="h-full flex flex-col p-4 overflow-hidden">
                <UAlert color="info" variant="subtle" icon="i-lucide-info" class="mb-4 shrink-0">
                  <template #title>
                    <strong>How this works:</strong> We compare the text an AI sees (Raw Extract) against the text a human sees (OCR). Text enclosed in dashed boxes is hidden in the PDF but will be read by an LLM (Prompt Injection).[cite: 1]
                  </template>
                </UAlert>
                
                <div class="flex-1 overflow-y-auto bg-(--ui-bg-elevated) p-4 border border-(--ui-border) rounded-lg font-mono text-sm whitespace-pre-wrap leading-relaxed">
                  <span v-if="diffResults.length === 0" class="text-(--ui-text-muted)">No diff generated yet.[cite: 1]</span>
                  <template v-else v-for="(part, index) in diffResults" :key="index">
                    <span 
                      v-if="part[0] === 1 && part[1].length > 15" 
                      class="bg-red-500/10 text-red-600 font-bold border border-red-500 border-dashed px-1 mx-1 rounded"
                      title="This text is hidden visually but readable by AI.[cite: 1]"
                    >
                      {{ part[1] }}
                    </span>
                    <span v-else-if="part[0] === 0" class="text-(--ui-text-muted)">
                      {{ part[1] }}
                    </span>
                  </template>
                </div>
              </div>
            </template>

            <!-- PDF Viewer Canvas Dump[cite: 1] -->
            <template #viewer>
              <div class="h-full overflow-y-auto p-4 bg-(--ui-bg-elevated) rounded-lg border border-(--ui-border)">
                <div v-if="!hasScanned" class="h-full flex items-center justify-center text-(--ui-text-muted) text-sm italic">
                  Canvas will render here...
                </div>
                <div ref="viewerContainerRef" class="flex flex-col items-center">
                  <!-- dynamically appended by pdf.js renderer -->
                </div>
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
