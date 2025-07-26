<script setup lang="ts">
import {nextTick, onMounted, reactive, ref, watch} from 'vue'

interface ImagePosition {
  x: number
  y: number
  scale: number
}

interface DragStart {
  x: number
  y: number
  pinchDistance?: number | null
  initialScale?: number | null
}

// Refs
const canvasRef = ref<HTMLCanvasElement | null>(null)
const fileInputRef = ref<HTMLInputElement | null>(null)

// Reactive state
const profileImage = ref<HTMLImageElement | null>(null)
const frameImage = ref<HTMLImageElement | null>(null)
const imagePosition = reactive<ImagePosition>({x: 0, y: 0, scale: 1.2})
const isDragging = ref<boolean>(false)
const dragStart = reactive<DragStart>({x: 0, y: 0, pinchDistance: null, initialScale: null})
const isDragOver = ref<boolean>(false)
const isLoading = ref<boolean>(true)
const isDownloading = ref<boolean>(false)
const renderKey = ref<number>(0)

const CANVAS_SIZE = 400 // Reduced for better mobile performance

// Force re-render function
const forceRender = (): void => {
  renderKey.value += 1
  nextTick(() => {
    drawCanvas()
  })
}

// Load the frame with multiple fallback attempts
onMounted(async () => {
  // console.log("Component mounted, loading frame...")

  try {
    const frame = new Image()

    await new Promise<void>((resolve, reject) => {
      frame.onload = () => {
        // console.log("Frame loaded successfully:", frame.width, frame.height)
        resolve()
      }
      frame.onerror = (error: string | Event) => {
        console.error("Frame load error:", error)
        reject(new Error(typeof error === 'string' ? error : 'Frame load failed'))
      }

      frame.src = "/frame.png"

      setTimeout(() => reject(new Error("Frame load timeout")), 10000)
    })

    frameImage.value = frame
  } catch (error) {
    console.error("Failed to load frame:", error)
    // Continue without frame
  } finally {
    isLoading.value = false

    // Initial draw
    setTimeout(() => {
      drawCanvas()
    }, 100)
  }
})

// Enhanced draw function with better error handling
const drawCanvas = (): void => {
  const canvas = canvasRef.value
  if (!canvas) {
    // console.log("Canvas element not found")
    return
  }

  const ctx = canvas.getContext("2d")
  if (!ctx) {
    // console.log("Canvas context not available")
    return
  }

  try {
    // Set canvas size
    canvas.width = CANVAS_SIZE
    canvas.height = CANVAS_SIZE

    // Clear canvas with white background
    ctx.fillStyle = "#ffffff"
    ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)

    // Draw subtle border
    ctx.strokeStyle = "#f3f4f6"
    ctx.lineWidth = 1
    ctx.strokeRect(0.5, 0.5, CANVAS_SIZE - 1, CANVAS_SIZE - 1)

    // Draw the circular frame area with better styling
    const centerX = CANVAS_SIZE / 2
    const centerY = CANVAS_SIZE / 2
    const radius = CANVAS_SIZE * 0.42

    // Draw profile image if exists
    if (profileImage.value && profileImage.value.complete && profileImage.value.naturalWidth > 0) {
      // console.log("Drawing profile image", {
      //   width: profileImage.value.naturalWidth,
      //   height: profileImage.value.naturalHeight,
      //   position: imagePosition
      // })

      ctx.save()

      // Create circular clipping path
      ctx.beginPath()
      ctx.arc(centerX, centerY, radius, 0, 2 * Math.PI)
      ctx.clip()

      const imageAspect = profileImage.value.naturalWidth / profileImage.value.naturalHeight
      let drawWidth = CANVAS_SIZE * imagePosition.scale
      let drawHeight = CANVAS_SIZE * imagePosition.scale

      if (imageAspect > 1) {
        drawHeight = drawWidth / imageAspect
      } else {
        drawWidth = drawHeight * imageAspect
      }

      const drawX = (CANVAS_SIZE - drawWidth) / 2 + imagePosition.x
      const drawY = (CANVAS_SIZE - drawHeight) / 2 + imagePosition.y

      // console.log("Drawing at:", {drawX, drawY, drawWidth, drawHeight})

      // Use simpler drawImage call
      ctx.globalAlpha = 1
      ctx.imageSmoothingEnabled = true
      ctx.drawImage(profileImage.value, drawX, drawY, drawWidth, drawHeight)
      ctx.restore()
      // console.log("Profile image drawn successfully")
    } else if (profileImage.value) {
      // console.log("Profile image not ready:", {
      //   complete: profileImage.value.complete,
      //   naturalWidth: profileImage.value.naturalWidth,
      //   src: profileImage.value.src?.substring(0, 50) + '...'
      // })
    }

    // Draw circular border
    ctx.strokeStyle = "#e5e7eb"
    ctx.lineWidth = 2
    ctx.beginPath()
    ctx.arc(centerX, centerY, radius, 0, 2 * Math.PI)
    ctx.stroke()

    // Draw frame on top if available
    if (frameImage.value && frameImage.value.complete && frameImage.value.naturalWidth > 0) {
      // console.log("Drawing frame image")
      ctx.globalAlpha = 1
      ctx.drawImage(frameImage.value, 0, 0, CANVAS_SIZE, CANVAS_SIZE)
      // console.log("Frame drawn successfully")
    } else if (frameImage.value) {
      // console.log("Frame not ready:", {
      //   complete: frameImage.value.complete,
      //   naturalWidth: frameImage.value.naturalWidth
      // })
    }

    // Add subtle shadow effect
    const gradient = ctx.createRadialGradient(
        centerX,
        centerY,
        radius * 0.8,
        centerX,
        centerY,
        radius * 1.1,
    )
    gradient.addColorStop(0, "rgba(0,0,0,0)")
    gradient.addColorStop(1, "rgba(0,0,0,0.1)")
    ctx.fillStyle = gradient
    ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)

    // Debug: Draw a test rectangle to verify canvas is working
    ctx.fillStyle = "rgba(255, 0, 0, 0.1)"
    ctx.fillRect(10, 10, 50, 20)
    ctx.fillStyle = "#000"
    ctx.font = "12px Arial"
    ctx.fillText("", 15, 25)

  } catch (error: unknown) {
    console.error("Canvas drawing error:", error)

    // Fallback: draw error indicator
    ctx.fillStyle = "#fef2f2"
    ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)
    ctx.fillStyle = "#dc2626"
    ctx.font = "14px system-ui, sans-serif"
    ctx.textAlign = "center"
    ctx.fillText("Canvas Error", CANVAS_SIZE / 2, CANVAS_SIZE / 2)
    const errorMessage = error instanceof Error ? error.message : 'Unknown error'
    ctx.fillText(errorMessage, CANVAS_SIZE / 2, CANVAS_SIZE / 2 + 20)
  }
}

// Watch for changes and redraw with immediate effect
watch([profileImage, frameImage, imagePosition, renderKey], () => {
  nextTick(() => {
    drawCanvas()
  })
}, {deep: true, immediate: true})

// Handle file upload
const handleFileUpload = (file: File): void => {
  if (!file.type.startsWith("image/")) {
    // console.log("Invalid file type:", file.type)
    return
  }

  // console.log("Starting file upload:", file.name, file.size)

  const reader = new FileReader()
  reader.onload = (e) => {
    const result = e.target?.result as string
    if (!result) {
      console.error("Failed to read file")
      return
    }

    // console.log("File read successfully, creating image...")

    const img = new Image()

    img.onload = () => {
      // console.log("Profile image loaded successfully:", {
      //   width: img.naturalWidth,
      //   height: img.naturalHeight,
      //   complete: img.complete
      // })

      profileImage.value = img
      Object.assign(imagePosition, {x: 0, y: 0, scale: 1.2})

      // Force multiple re-renders to ensure image displays
      setTimeout(() => forceRender(), 50)
      setTimeout(() => forceRender(), 100)
      setTimeout(() => forceRender(), 200)
    }

    img.onerror = (error: string | Event) => {
      console.error("Failed to load uploaded image:", error)
    }

    img.onabort = () => {
      console.error("Image loading aborted")
    }

    // Don't set crossOrigin for data URLs
    img.src = result
    // console.log("Image src set, waiting for load...")
  }

  reader.onerror = (error: ProgressEvent<FileReader>) => {
    console.error("FileReader error:", error)
  }

  reader.readAsDataURL(file)
}

// Handle drag and drop
const handleDragOver = (e: DragEvent): void => {
  e.preventDefault()
  isDragOver.value = true
}

const handleDragLeave = (e: DragEvent): void => {
  e.preventDefault()
  isDragOver.value = false
}

const handleDrop = (e: DragEvent): void => {
  e.preventDefault()
  isDragOver.value = false

  const files = Array.from(e.dataTransfer?.files || [])
  if (files.length > 0 && files[0].type.startsWith("image/")) {
    handleFileUpload(files[0])
  }
}

// Touch and mouse event handling
const getEventPosition = (e: MouseEvent | TouchEvent) => {
  if ('touches' in e) {
    return {x: e.touches[0].clientX, y: e.touches[0].clientY}
  }
  return {x: e.clientX, y: e.clientY}
}

const handleStart = (e: MouseEvent | TouchEvent): void => {
  if (!profileImage.value) {
    triggerFileInput()
    return
  }

  e.preventDefault()
  isDragging.value = true
  const rect = canvasRef.value?.getBoundingClientRect()
  const pos = getEventPosition(e)

  if (rect) {
    const scaleRatio = CANVAS_SIZE / rect.width
    Object.assign(dragStart, {
      x: (pos.x - rect.left) * scaleRatio - imagePosition.x,
      y: (pos.y - rect.top) * scaleRatio - imagePosition.y,
    })
  }
}

const handleMove = (e: MouseEvent | TouchEvent): void => {
  if (!isDragging.value || !profileImage.value) return

  e.preventDefault()
  const rect = canvasRef.value?.getBoundingClientRect()
  const pos = getEventPosition(e)

  if (rect) {
    const scaleRatio = CANVAS_SIZE / rect.width
    Object.assign(imagePosition, {
      ...imagePosition,
      x: (pos.x - rect.left) * scaleRatio - dragStart.x,
      y: (pos.y - rect.top) * scaleRatio - dragStart.y,
    })
  }
}

const handleEnd = (): void => {
  isDragging.value = false
}

// Handle wheel for scaling
const handleWheel = (e: WheelEvent): void => {
  if (!profileImage.value) return

  e.preventDefault()
  const scaleChange = e.deltaY > 0 ? -0.1 : 0.1
  imagePosition.scale = Math.max(0.3, Math.min(4, imagePosition.scale + scaleChange))
}

// Handle pinch to zoom for mobile
const handleTouchStart = (e: TouchEvent): void => {
  if (e.touches.length === 2) {
    e.preventDefault()
    const touch1 = e.touches[0]
    const touch2 = e.touches[1]
    const distance = Math.sqrt(
        Math.pow(touch2.clientX - touch1.clientX, 2) +
        Math.pow(touch2.clientY - touch1.clientY, 2)
    )
    dragStart.pinchDistance = distance
    dragStart.initialScale = imagePosition.scale
  } else {
    handleStart(e)
  }
}

const handleTouchMove = (e: TouchEvent): void => {
  if (e.touches.length === 2 && profileImage.value) {
    e.preventDefault()
    const touch1 = e.touches[0]
    const touch2 = e.touches[1]
    const distance = Math.sqrt(
        Math.pow(touch2.clientX - touch1.clientX, 2) +
        Math.pow(touch2.clientY - touch1.clientY, 2)
    )

    if (dragStart.pinchDistance && dragStart.initialScale !== null && dragStart.initialScale !== undefined) {
      const scale = (distance / dragStart.pinchDistance) * dragStart.initialScale
      imagePosition.scale = Math.max(0.3, Math.min(4, scale))
    }
  } else if (e.touches.length === 1) {
    handleMove(e)
  }
}

const handleTouchEnd = (e: TouchEvent): void => {
  if (e.touches.length === 0) {
    dragStart.pinchDistance = null
    dragStart.initialScale = null
    handleEnd()
  }
}

// Reset position
const resetPosition = (): void => {
  Object.assign(imagePosition, {x: 0, y: 0, scale: 1.2})
  setTimeout(() => {
    forceRender()
  }, 50)
}

// Trigger file input
const triggerFileInput = (): void => {
  fileInputRef.value?.click()
}

// Handle file input change
const handleFileInputChange = (e: Event): void => {
  const target = e.target as HTMLInputElement
  const file = target.files?.[0]
  if (file) handleFileUpload(file)
}

// Download function
const downloadImage = async (): Promise<void> => {
  const canvas = canvasRef.value
  if (!canvas) return

  isDownloading.value = true

  try {
    const exportCanvas = document.createElement("canvas")
    const exportCtx = exportCanvas.getContext("2d")
    const exportSize = 1080

    exportCanvas.width = exportSize
    exportCanvas.height = exportSize

    if (exportCtx) {
      exportCtx.fillStyle = "#ffffff"
      exportCtx.fillRect(0, 0, exportSize, exportSize)

      if (profileImage.value) {
        exportCtx.save()
        exportCtx.beginPath()
        exportCtx.arc(exportSize / 2, exportSize / 2, exportSize * 0.42, 0, 2 * Math.PI)
        exportCtx.clip()

        const scale = exportSize / CANVAS_SIZE
        const imageAspect = profileImage.value.naturalWidth / profileImage.value.naturalHeight
        let drawWidth = exportSize * imagePosition.scale
        let drawHeight = exportSize * imagePosition.scale

        if (imageAspect > 1) {
          drawHeight = drawWidth / imageAspect
        } else {
          drawWidth = drawHeight * imageAspect
        }

        const drawX = (exportSize - drawWidth) / 2 + imagePosition.x * scale
        const drawY = (exportSize - drawHeight) / 2 + imagePosition.y * scale

        exportCtx.drawImage(profileImage.value, drawX, drawY, drawWidth, drawHeight)
        exportCtx.restore()
      }

      if (frameImage.value) {
        exportCtx.drawImage(frameImage.value, 0, 0, exportSize, exportSize)
      }

      exportCanvas.toBlob(
          (blob) => {
            if (blob) {
              const url = URL.createObjectURL(blob)
              const link = document.createElement("a")
              link.href = url
              link.download = `cambodiawantpeace-frame-${Date.now()}.png`
              document.body.appendChild(link)
              link.click()
              document.body.removeChild(link)
              URL.revokeObjectURL(url)
            }
          },
          "image/png",
          1.0,
      )
    }
  } catch (error: unknown) {
    console.error("Download failed:", error)
  } finally {
    isDownloading.value = false
  }
}
</script>

<template>
  <div class="min-h-screen bg-gray-50">
    <!-- Loading State -->
    <div v-if="isLoading" class="flex items-center justify-center min-h-[80vh] px-4">
      <div class="text-center p-8 bg-white rounded-xl shadow-sm border max-w-sm w-full">
        <div class="w-8 h-8 border-3 border-gray-200 border-t-blue-600 rounded-full animate-spin mx-auto mb-4"></div>
        <p class="text-gray-600 font-medium">Loading frame...</p>
      </div>
    </div>

    <!-- Main Content -->
    <div v-else class="max-w-5xl mx-auto p-4 sm:p-6">
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <!-- Preview Section -->
        <div class="lg:col-span-2">
          <div class="bg-white rounded-xl shadow-sm border p-6">
            <div class="flex items-center justify-between mb-4">
              <h2 class="text-lg font-semibold text-gray-900">Preview</h2>
              <div v-if="profileImage" class="text-sm text-gray-500">
                Size: {{ Math.round(imagePosition.scale * 100) }}%
              </div>
            </div>

            <div class="flex justify-center">
              <div class="relative w-full max-w-md">
                <canvas
                    ref="canvasRef"
                    :width="CANVAS_SIZE"
                    :height="CANVAS_SIZE"
                    class="w-full h-auto block touch-none rounded-lg shadow-sm border border-gray-200 bg-white"
                    :style="{
                      cursor: isDragging ? 'grabbing' : profileImage ? 'grab' : 'pointer',
                      aspectRatio: '1 / 1',
                    }"
                    @mousedown="handleStart"
                    @mousemove="handleMove"
                    @mouseup="handleEnd"
                    @mouseleave="handleEnd"
                    @wheel="handleWheel"
                    @touchstart="handleTouchStart"
                    @touchmove="handleTouchMove"
                    @touchend="handleTouchEnd"
                    @dragover="handleDragOver"
                    @dragleave="handleDragLeave"
                    @drop="handleDrop"
                />

                <!-- Instructions overlay -->
                <div v-if="profileImage"
                     class="absolute top-3 left-3 bg-black/75 text-white px-2 py-1 rounded text-xs font-medium">
                  <span class="hidden sm:inline">Drag • Scroll to zoom</span>
                  <span class="sm:hidden">Drag • Pinch zoom</span>
                </div>

                <!-- Upload prompt for empty state -->
                <div v-if="!profileImage"
                     class="absolute inset-0 flex items-center justify-center bg-gray-50/90 rounded-lg border-2 border-dashed border-gray-300">
                  <div class="text-center p-6">
                    <svg class="w-12 h-12 text-gray-400 mx-auto mb-3" fill="none" stroke="currentColor"
                         viewBox="0 0 24 24">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5"
                            d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"/>
                    </svg>
                    <p class="text-gray-600 font-medium mb-1">Click to upload</p>
                    <p class="text-gray-500 text-sm">or drag & drop</p>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Controls Section -->
        <div class="space-y-4">
          <!-- Upload Section -->
          <div class="bg-white rounded-xl shadow-sm border p-6">
            <h3 class="text-lg font-semibold text-gray-900 mb-4">
              {{ profileImage ? 'Change Image' : 'Upload Image' }}
            </h3>
            <button
                type="button"
                @click="triggerFileInput"
                class="w-full h-12 bg-blue-600 hover:bg-blue-700 active:bg-blue-800 text-white font-medium rounded-lg transition-colors flex items-center justify-center gap-2"
            >
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path v-if="profileImage" stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/>
                <path v-else stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"/>
              </svg>
              {{ profileImage ? 'Change Photo' : 'Upload Photo' }}
            </button>
          </div>

          <!-- Position Controls -->
          <div v-if="profileImage" class="bg-white rounded-xl shadow-sm border p-6">
            <h3 class="text-lg font-semibold text-gray-900 mb-4">Position & Scale</h3>
            <div class="space-y-4">
              <div class="text-center">
                <div class="text-sm text-gray-600 mb-2">Current scale</div>
                <div class="text-2xl font-bold text-blue-600">{{ Math.round(imagePosition.scale * 100) }}%</div>
              </div>
              <button
                  type="button"
                  @click="resetPosition"
                  class="w-full h-10 bg-gray-100 hover:bg-gray-200 text-gray-700 font-medium rounded-lg transition-colors flex items-center justify-center gap-2"
              >
                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                        d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/>
                </svg>
                Reset Position
              </button>
            </div>
          </div>

          <!-- Download Section -->
          <div v-if="profileImage" class="bg-white rounded-xl shadow-sm border p-6">
            <h3 class="text-lg font-semibold text-gray-900 mb-4">Export</h3>
            <button
                type="button"
                @click="downloadImage"
                :disabled="isDownloading"
                class="w-full h-12 bg-green-600 hover:bg-green-700 active:bg-green-800 disabled:bg-green-400 text-white font-medium rounded-lg transition-colors flex items-center justify-center gap-2 disabled:cursor-not-allowed"
            >
              <div v-if="isDownloading"
                   class="w-5 h-5 border-2 border-white border-t-transparent rounded-full animate-spin"></div>
              <template v-else>
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                        d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
                </svg>
                Download HD Image
              </template>
            </button>
            <p class="text-gray-500 text-center mt-3 text-sm">1080×1080 PNG format</p>
          </div>
        </div>
      </div>
    </div>

    <!-- Hidden file input -->
    <input
        ref="fileInputRef"
        type="file"
        accept="image/*"
        class="hidden"
        @change="handleFileInputChange"
    />
  </div>
</template>

<style scoped>
.animate-spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* Ensure canvas maintains aspect ratio on all screens */
canvas {
  width: 100% !important;
  height: auto !important;
  max-width: 100%;
}

/* Prevent zoom on double tap for iOS */
canvas {
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

/* Clean border styling */
.border-3 {
  border-width: 3px;
}

/* Smooth transitions */
button {
  transition: all 0.2s ease;
}

/* Mobile optimizations */
@media (max-width: 640px) {
  .grid {
    gap: 1rem;
  }
}

/* Ensure proper spacing on larger screens */
@media (min-width: 1024px) {
  .lg\\:col-span-2 {
    grid-column: span 2 / span 2;
  }
}
</style>