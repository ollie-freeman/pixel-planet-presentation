<template>
  <div
    ref="containerRef"
    :style="containerStyle"
    class="canvas-container"
  >
    <canvas ref="canvasRef" class="canvas" />
  </div>
</template>

<script setup>
import {
  defineProps,
  onMounted,
  ref,
  onBeforeUnmount,
  watch,
  computed
} from 'vue'

const props = defineProps({
  fragmentShader: {
    type: String,
    required: true
  },
  fullscreen: {
    type: Boolean,
    default: true
  },
  useTime: {
    type: Boolean,
    default: false
  }
})

const containerRef = ref(null)
const canvasRef = ref(null)

let device, context, format, pipeline
let resolutionBuffer, timeBuffer, bindGroup
let resizeObserver
let visibilityObserver
let isRendering = false
let frameId = 0

const MARGIN = 20  // px

const containerStyle = computed(() => {
  if (props.fullscreen) {
    return {
      position: 'absolute',
      top:    `${MARGIN}px`,
      right:  `${MARGIN}px`,
      bottom: `${MARGIN}px`,
      left:   `${MARGIN}px`,
      display: 'flex',
      justifyContent: 'center',
      alignItems: 'center',
      overflow: 'hidden',
    }
  } else {
    return {
      position: 'absolute',
      top:    `${MARGIN}px`,
      right:  `${MARGIN}px`,
      bottom: `${MARGIN}px`,
      left:   `calc(50% + ${MARGIN}px)`,
      display: 'flex',
      justifyContent: 'center',
      alignItems: 'center',
      overflow: 'hidden',
    }
  }
})

async function initWebGPU() {
  const canvas = canvasRef.value
  if (!canvas || !navigator.gpu) {
    console.error('WebGPU not supported')
    return
  }

  const adapter = await navigator.gpu.requestAdapter()
  if (!adapter) {
    console.error('Failed to get GPU adapter')
    return
  }
  device = await adapter.requestDevice()

  resolutionBuffer = device.createBuffer({
    size: 2 * Float32Array.BYTES_PER_ELEMENT,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
  })

  if (props.useTime) {
    timeBuffer = device.createBuffer({
      size: Float32Array.BYTES_PER_ELEMENT,
      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
    })
  }

  context = canvas.getContext('webgpu')
  format = navigator.gpu.getPreferredCanvasFormat()

  rebuildPipeline()
}

function rebuildPipeline() {
  const fullShader = `
    @vertex
    fn vs_main(@builtin(vertex_index) index: u32) -> @builtin(position) vec4<f32> {
      var positions = array<vec2<f32>, 6>(
        vec2<f32>(-1.0, -1.0), vec2<f32>( 1.0, -1.0), vec2<f32>(-1.0,  1.0),
        vec2<f32>(-1.0,  1.0), vec2<f32>( 1.0, -1.0), vec2<f32>( 1.0,  1.0)
      );
      return vec4<f32>(positions[index], 0.0, 1.0);
    }

    ${props.fragmentShader}
  `

  const shaderModule = device.createShaderModule({ code: fullShader })

  pipeline = device.createRenderPipeline({
    layout: 'auto',
    vertex: { module: shaderModule, entryPoint: 'vs_main' },
    fragment: {
      module: shaderModule,
      entryPoint: 'fs_main',
      targets: [{ format }]
    },
    primitive: { topology: 'triangle-list' }
  })

  const layout0 = pipeline.getBindGroupLayout(0)
  const entries = [
    { binding: 0, resource: { buffer: resolutionBuffer } }
  ]
  if (props.useTime) {
    entries.push({ binding: 1, resource: { buffer: timeBuffer } })
  }

  bindGroup = device.createBindGroup({
    layout: layout0,
    entries
  })
}

function configureContextSize() {
  const canvas = canvasRef.value
  const container = containerRef.value
  if (!canvas || !context || !container) return

  const cw = container.clientWidth
  const ch = container.clientHeight
  const size = Math.min(cw, ch)
  const dpr = window.devicePixelRatio || 1

  canvas.width = size * dpr
  canvas.height = size * dpr
  canvas.style.width = `${size}px`
  canvas.style.height = `${size}px`

  context.configure({
    device,
    format,
    alphaMode: 'premultiplied'
  })

  device.queue.writeBuffer(
    resolutionBuffer,
    0,
    new Float32Array([canvas.width, canvas.height])
  )
}

function frame(startTime = performance.now()) {
  const now = performance.now()
  const elapsed = (now - startTime) / 1000

  if (props.useTime) {
    device.queue.writeBuffer(timeBuffer, 0, new Float32Array([elapsed]))
  }

  configureContextSize()

  const encoder = device.createCommandEncoder()
  const view = context.getCurrentTexture().createView()
  const pass = encoder.beginRenderPass({
    colorAttachments: [{
      view,
      clearValue: { r: 0, g: 0, b: 0, a: 0 },
      loadOp: 'clear',
      storeOp: 'store'
    }]
  })

  pass.setPipeline(pipeline)
  pass.setBindGroup(0, bindGroup)
  pass.draw(6, 1, 0, 0)
  pass.end()
  device.queue.submit([encoder.finish()])

  if (isRendering) {
    frameId = requestAnimationFrame(() => frame(startTime))
  }
}

function startRenderLoop() {
  if (!isRendering) {
    isRendering = true
    frame()
  }
}

function stopRenderLoop() {
  if (isRendering) {
    isRendering = false
    cancelAnimationFrame(frameId)
  }
}

function observeVisibility() {
  visibilityObserver = new IntersectionObserver((entries) => {
    for (const entry of entries) {
      if (entry.isIntersecting) {
        startRenderLoop()
      } else {
        stopRenderLoop()
      }
    }
  }, { root: null, threshold: 0.1 })

  if (containerRef.value) {
    visibilityObserver.observe(containerRef.value)
  }
}

onMounted(() => {
  initWebGPU()
  observeVisibility()

  resizeObserver = new ResizeObserver(configureContextSize)
  if (containerRef.value) {
    resizeObserver.observe(containerRef.value)
  }
})

watch(() => props.fragmentShader, () => {
  if (device) {
    rebuildPipeline()
  }
})

watch(() => props.fullscreen, () => {
  configureContextSize()
})

onBeforeUnmount(() => {
  if (resizeObserver && containerRef.value) {
    resizeObserver.unobserve(containerRef.value)
  }
  if (visibilityObserver && containerRef.value) {
    visibilityObserver.unobserve(containerRef.value)
    visibilityObserver.disconnect()
  }
  stopRenderLoop()
})
</script>

<style scoped>
.canvas {
  display: block;
}
</style>
