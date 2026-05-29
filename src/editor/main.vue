<template>
  <div class="main_canvas_editor" ref="containerRef">
    <canvas
      ref="canvasRef"
      v-on:mousedown="handleMouseDown"
      v-on:mousemove="handleMouseMove"
      v-on:mouseup="handleMouseUp"
      v-on:click="handleCanvasClick"
      v-on:contextmenu.prevent="handleContextMenu"
      v-on:wheel="handleWheel"
    ></canvas>

    <!-- Zoom Controls -->
    <div class="zoom-controls">
      <button @click="zoomOut" class="zoom-btn" title="Уменьшить (Ctrl + -)">
        −
      </button>
      <span class="zoom-level">{{ Math.round(zoom * 100) }}%</span>
      <button @click="zoomIn" class="zoom-btn" title="Увеличить (Ctrl + +)">
        +
      </button>
      <button
        @click="resetZoom"
        class="zoom-btn reset"
        title="Сбросить (Ctrl + 0)"
      >
        ⟲
      </button>
      <button
        @click="zoomToFit"
        class="zoom-btn fit"
        title="Вместить всё (Shift + F)"
      >
        ⊡
      </button>
    </div>

    <!-- Controls for squares -->
    <div class="controls">
      <button @click="addSquare" class="control-btn">
        ➕ Добавить квадрат
      </button>
      <button
        @click="removeSelectedSquare"
        class="control-btn"
        :disabled="!selectedSquareId"
      >
        🗑 Удалить
      </button>
      <button
        @click="clearAllSquares"
        class="control-btn"
        :disabled="squares.length === 0"
      >
        🗑 Всё
      </button>
    </div>

    <!-- Info panel -->
    <div class="info-panel">
      <div>Квадратов: {{ squares.length }}</div>
      <div v-if="selectedSquareId">
        Выбран: #{{ selectedSquareId }} | ({{
          Math.round(getSquareById(selectedSquareId)?.x || 0)
        }}, {{ Math.round(getSquareById(selectedSquareId)?.y || 0) }})
      </div>
      <div>Зум: {{ Math.round(zoom * 100) }}%</div>
      <div>
        Смещение: ({{ Math.round(offsetX) }}, {{ Math.round(offsetY) }})
      </div>
      <div class="hint">
        💡 Перетащите квадрат ЛКМ | Панорамирование: СКМ или Ctrl+ЛКМ
      </div>
      <div class="hint">🔍 Zoom to Fit: Shift+F | Сброс: Ctrl+0</div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

interface Square {
  id: number
  x: number
  y: number
  size: number
  color: string
  isSelected: boolean
}

export default defineComponent({
  name: 'CanvasEditor',

  data() {
    return {
      // Canvas and context
      ctx: null as CanvasRenderingContext2D | null,
      resizeObserver: null as ResizeObserver | null,

      // Zoom and pan
      zoom: 1,
      minZoom: 0.1,
      maxZoom: 3,
      zoomStep: 0.1,
      offsetX: 0,
      offsetY: 0,

      // Panning state
      isPanning: false,
      lastMouseX: 0,
      lastMouseY: 0,

      // Squares array
      squares: [] as Square[],
      nextSquareId: 1,

      // Dragging square
      isDraggingSquare: false,
      dragStartX: 0,
      dragStartY: 0,
      draggedSquareId: null as number | null,
      squareStartX: 0,
      squareStartY: 0,

      // Selection
      selectedSquareId: null as number | null,
    }
  },

  mounted() {
    this.ctx = (this.$refs.canvasRef as HTMLCanvasElement).getContext('2d')
    this.initSquares()
    this.resizeCanvas()
    this.setupResizeObserver()
    this.draw()

    window.addEventListener('resize', this.handleWindowResize)
    window.addEventListener('keydown', this.handleKeyDown)
  },

  beforeUnmount() {
    if (this.resizeObserver) {
      this.resizeObserver.disconnect()
    }
    window.removeEventListener('resize', this.handleWindowResize)
    window.removeEventListener('keydown', this.handleKeyDown)
  },

  methods: {
    initSquares() {
      // Create initial squares at different positions
      this.squares = [
        {
          id: this.nextSquareId++,
          x: -100,
          y: -50,
          size: 80,
          color: '#4CAF50',
          isSelected: false,
        },
        {
          id: this.nextSquareId++,
          x: 150,
          y: -80,
          size: 80,
          color: '#2196F3',
          isSelected: false,
        },
        {
          id: this.nextSquareId++,
          x: -150,
          y: 120,
          size: 80,
          color: '#FF9800',
          isSelected: false,
        },
        {
          id: this.nextSquareId++,
          x: 200,
          y: 150,
          size: 80,
          color: '#9C27B0',
          isSelected: false,
        },
        {
          id: this.nextSquareId++,
          x: 50,
          y: 200,
          size: 80,
          color: '#F44336',
          isSelected: false,
        },
      ]

      // Center view on squares
      // this.zoomToFit();
    },

    zoomToFit() {
      if (this.squares.length === 0) {
        // If no squares, just reset view
        this.resetZoom()
        return
      }

      const canvas = this.$refs.canvasRef as HTMLCanvasElement
      if (!canvas) return

      // Calculate bounding box of all squares
      let minX = Infinity,
        minY = Infinity
      let maxX = -Infinity,
        maxY = -Infinity

      this.squares.forEach((square) => {
        minX = Math.min(minX, square.x)
        minY = Math.min(minY, square.y)
        maxX = Math.max(maxX, square.x + square.size)
        maxY = Math.max(maxY, square.y + square.size)
      })

      // Add padding (10% on each side)
      const padding = 0.1
      const width = maxX - minX
      const height = maxY - minY
      const paddingX = width * padding
      const paddingY = height * padding

      minX -= paddingX
      minY -= paddingY
      maxX += paddingX
      maxY += paddingY

      const contentWidth = maxX - minX
      const contentHeight = maxY - minY

      // Calculate needed zoom to fit content in canvas
      const zoomX = canvas.width / contentWidth
      const zoomY = canvas.height / contentHeight
      let newZoom = Math.min(zoomX, zoomY)

      // Limit zoom to maximum 100% (1.0)
      newZoom = Math.min(newZoom, 1.0)

      // Apply zoom
      this.zoom = newZoom

      // Calculate center of content
      const centerX = (minX + maxX) / 2
      const centerY = (minY + maxY) / 2

      // Set offset to center content in canvas
      this.offsetX = canvas.width / 2 - centerX * this.zoom
      this.offsetY = canvas.height / 2 - centerY * this.zoom

      this.draw()
    },

    centerViewOnSquares() {
      if (this.squares.length === 0) return

      const canvas = this.$refs.canvasRef as HTMLCanvasElement
      if (!canvas) return

      // Calculate center of all squares
      let minX = Infinity,
        minY = Infinity,
        maxX = -Infinity,
        maxY = -Infinity
      this.squares.forEach((square) => {
        minX = Math.min(minX, square.x)
        minY = Math.min(minY, square.y)
        maxX = Math.max(maxX, square.x + square.size)
        maxY = Math.max(maxY, square.y + square.size)
      })

      const centerX = (minX + maxX) / 2
      const centerY = (minY + maxY) / 2

      // Center the view
      this.offsetX = canvas.width / 2 - centerX * this.zoom
      this.offsetY = canvas.height / 2 - centerY * this.zoom

      this.draw()
    },

    addSquare() {
      const canvas = this.$refs.canvasRef as HTMLCanvasElement
      if (!canvas) return

      // Get viewport center in world coordinates
      const centerWorldX = (canvas.width / 2 - this.offsetX) / this.zoom
      const centerWorldY = (canvas.height / 2 - this.offsetY) / this.zoom

      // Random offset from center
      const randomOffsetX = (Math.random() - 0.5) * 200
      const randomOffsetY = (Math.random() - 0.5) * 200

      // Random color
      const hue = Math.random() * 360

      this.squares.push({
        id: this.nextSquareId++,
        x: centerWorldX + randomOffsetX,
        y: centerWorldY + randomOffsetY,
        size: 80,
        color: `hsl(${hue}, 70%, 55%)`,
        isSelected: false,
      })

      this.draw()
    },

    removeSelectedSquare() {
      if (this.selectedSquareId === null) return

      const index = this.squares.findIndex(
        (s) => s.id === this.selectedSquareId,
      )
      if (index !== -1) {
        this.squares.splice(index, 1)
        this.selectedSquareId = null
        this.draw()
      }
    },

    clearAllSquares() {
      this.squares = []
      this.selectedSquareId = null
      this.draw()
    },

    getSquareById(id: number): Square | undefined {
      return this.squares.find((s) => s.id === id)
    },

    handleWheel(event: WheelEvent) {
      if (event.ctrlKey) {
        event.preventDefault()
        const canvas = this.$refs.canvasRef as HTMLCanvasElement
        const rect = canvas.getBoundingClientRect()
        const mouseX = event.clientX - rect.left
        const mouseY = event.clientY - rect.top
        const delta = event.deltaY > 0 ? -this.zoomStep : this.zoomStep
        this.adjustZoom(delta, mouseX, mouseY)
      }
    },

    handleKeyDown(event: KeyboardEvent) {
      if (
        (event.ctrlKey || event.metaKey) &&
        (event.key === '+' || event.key === '=')
      ) {
        event.preventDefault()
        this.zoomIn()
      } else if ((event.ctrlKey || event.metaKey) && event.key === '-') {
        event.preventDefault()
        this.zoomOut()
      } else if ((event.ctrlKey || event.metaKey) && event.key === '0') {
        event.preventDefault()
        this.resetZoom()
      } else if (event.shiftKey && event.key === 'F') {
        event.preventDefault()
        this.zoomToFit()
      } else if (event.key === 'Delete' && this.selectedSquareId !== null) {
        this.removeSelectedSquare()
      }
    },

    zoomIn() {
      this.adjustZoom(this.zoomStep)
    },

    zoomOut() {
      this.adjustZoom(-this.zoomStep)
    },

    adjustZoom(
      delta: number,
      centerX: number | null = null,
      centerY: number | null = null,
    ) {
      const oldZoom = this.zoom
      let newZoom = this.zoom + delta
      newZoom = Math.max(this.minZoom, Math.min(this.maxZoom, newZoom))

      if (newZoom === this.zoom) return

      if (centerX !== null && centerY !== null) {
        // Calculate world coordinates before zoom
        const worldX = (centerX - this.offsetX) / oldZoom
        const worldY = (centerY - this.offsetY) / oldZoom

        this.zoom = newZoom

        // Adjust offset to keep position under mouse
        this.offsetX = centerX - worldX * this.zoom
        this.offsetY = centerY - worldY * this.zoom
      } else {
        this.zoom = newZoom
      }

      this.draw()
    },

    resetZoom() {
      this.zoom = 1
      this.offsetX = 0
      this.offsetY = 0
      this.centerViewOnSquares()
    },

    handleMouseDown(event: MouseEvent) {
      const canvas = this.$refs.canvasRef as HTMLCanvasElement
      const rect = canvas.getBoundingClientRect()

      const screenX = event.clientX - rect.left
      const screenY = event.clientY - rect.top
      const worldX = (screenX - this.offsetX) / this.zoom
      const worldY = (screenY - this.offsetY) / this.zoom

      // Check if clicking on any square
      const squareId = this.isPointInSquare(worldX, worldY)

      if (squareId !== null) {
        // Start dragging square
        this.isDraggingSquare = true
        this.draggedSquareId = squareId
        this.dragStartX = worldX
        this.dragStartY = worldY

        const draggedSquare = this.squares.find((s) => s.id === squareId)
        if (draggedSquare) {
          this.squareStartX = draggedSquare.x
          this.squareStartY = draggedSquare.y

          // Select the square
          this.squares.forEach((s) => (s.isSelected = false))
          draggedSquare.isSelected = true
          this.selectedSquareId = squareId
        }

        event.preventDefault()
        this.draw()
      } else if (event.button === 1 || (event.button === 0 && event.ctrlKey)) {
        // Start canvas panning (middle mouse or Ctrl+left)
        this.isPanning = true
        this.lastMouseX = event.clientX
        this.lastMouseY = event.clientY
        event.preventDefault()
      } else {
        // Deselect all squares
        this.squares.forEach((s) => (s.isSelected = false))
        this.selectedSquareId = null
        this.draw()
      }
    },

    handleMouseMove(event: MouseEvent) {
      const canvas = this.$refs.canvasRef as HTMLCanvasElement
      const rect = canvas.getBoundingClientRect()

      if (this.isDraggingSquare && this.draggedSquareId !== null) {
        const screenX = event.clientX - rect.left
        const screenY = event.clientY - rect.top
        const worldX = (screenX - this.offsetX) / this.zoom
        const worldY = (screenY - this.offsetY) / this.zoom

        const deltaX = worldX - this.dragStartX
        const deltaY = worldY - this.dragStartY

        const draggedSquare = this.squares.find(
          (s) => s.id === this.draggedSquareId,
        )
        if (draggedSquare) {
          draggedSquare.x = this.squareStartX + deltaX
          draggedSquare.y = this.squareStartY + deltaY
        }

        this.draw()
      } else if (this.isPanning) {
        const deltaX = event.clientX - this.lastMouseX
        const deltaY = event.clientY - this.lastMouseY

        this.offsetX += deltaX
        this.offsetY += deltaY

        this.lastMouseX = event.clientX
        this.lastMouseY = event.clientY

        this.draw()
      } else {
        // Change cursor on hover
        const screenX = event.clientX - rect.left
        const screenY = event.clientY - rect.top
        const worldX = (screenX - this.offsetX) / this.zoom
        const worldY = (screenY - this.offsetY) / this.zoom

        const squareId = this.isPointInSquare(worldX, worldY)
        canvas.style.cursor = squareId !== null ? 'move' : 'default'
      }
    },

    handleMouseUp() {
      this.isDraggingSquare = false
      this.isPanning = false
      this.draggedSquareId = null
    },

    handleCanvasClick(event: MouseEvent) {
      const canvas = this.$refs.canvasRef as HTMLCanvasElement
      const rect = canvas.getBoundingClientRect()

      const screenX = event.clientX - rect.left
      const screenY = event.clientY - rect.top
      const worldX = (screenX - this.offsetX) / this.zoom
      const worldY = (screenY - this.offsetY) / this.zoom

      console.log(
        `Clicked at world coordinates: (${Math.round(worldX)}, ${Math.round(worldY)})`,
      )
    },

    handleContextMenu(event: MouseEvent) {
      console.log('Context menu at:', event.clientX, event.clientY)
    },

    isPointInSquare(x: number, y: number): number | null {
      // Check all squares and return ID of the one that was clicked
      for (let square of this.squares) {
        if (
          x >= square.x &&
          x <= square.x + square.size &&
          y >= square.y &&
          y <= square.y + square.size
        ) {
          return square.id
        }
      }
      return null
    },

    draw() {
      const ctx = this.ctx
      const canvas = this.$refs.canvasRef as HTMLCanvasElement
      if (!ctx || !canvas) return

      // Clear canvas
      ctx.clearRect(0, 0, canvas.width, canvas.height)

      // Save context state
      ctx.save()

      // Apply zoom and pan transforms
      ctx.translate(this.offsetX, this.offsetY)
      ctx.scale(this.zoom, this.zoom)

      // Draw grid
      this.drawGrid(ctx, canvas)

      // Draw all squares
      this.drawSquares(ctx)

      // Restore context state
      ctx.restore()
    },

    drawGrid(ctx: CanvasRenderingContext2D, canvas: HTMLCanvasElement) {
      const width = canvas.width / this.zoom
      const height = canvas.height / this.zoom

      const startX = -this.offsetX / this.zoom
      const startY = -this.offsetY / this.zoom
      const endX = startX + width
      const endY = startY + height

      ctx.save()
      ctx.strokeStyle = '#e0e0e0'
      ctx.lineWidth = 1 / this.zoom

      // Draw vertical lines
      const gridSize = 20
      const startGridX = Math.floor(startX / gridSize) * gridSize

      for (let x = startGridX; x <= endX; x += gridSize) {
        ctx.beginPath()
        ctx.moveTo(x, startY)
        ctx.lineTo(x, endY)
        ctx.stroke()
      }

      // Draw horizontal lines
      const startGridY = Math.floor(startY / gridSize) * gridSize

      for (let y = startGridY; y <= endY; y += gridSize) {
        ctx.beginPath()
        ctx.moveTo(startX, y)
        ctx.lineTo(endX, y)
        ctx.stroke()
      }

      // Draw coordinate axes
      ctx.beginPath()
      ctx.strokeStyle = '#ff0000'
      ctx.lineWidth = 2 / this.zoom

      // X axis
      if (startY <= 0 && endY >= 0) {
        ctx.moveTo(startX, 0)
        ctx.lineTo(endX, 0)
        ctx.stroke()
      }

      // Y axis
      if (startX <= 0 && endX >= 0) {
        ctx.moveTo(0, startY)
        ctx.lineTo(0, endY)
        ctx.stroke()
      }

      ctx.restore()
    },

    drawSquares(ctx: CanvasRenderingContext2D) {
      this.squares.forEach((square) => {
        ctx.save()

        // Shadow for selected square
        if (square.isSelected) {
          ctx.shadowColor = 'rgba(0, 0, 0, 0.3)'
          ctx.shadowBlur = 15
          ctx.shadowOffsetX = 3
          ctx.shadowOffsetY = 3
        } else {
          ctx.shadowColor = 'rgba(0, 0, 0, 0.15)'
          ctx.shadowBlur = 8
          ctx.shadowOffsetX = 2
          ctx.shadowOffsetY = 2
        }

        // Draw square
        ctx.fillStyle = square.color
        ctx.fillRect(square.x, square.y, square.size, square.size)

        // Draw border
        ctx.strokeStyle = square.isSelected ? '#FFD700' : '#2E7D32'
        ctx.lineWidth = square.isSelected ? 4 : 2
        ctx.strokeRect(square.x, square.y, square.size, square.size)

        // Draw selection handles for selected square
        if (square.isSelected) {
          ctx.shadowBlur = 0
          const handleSize = 6
          const corners = [
            [square.x, square.y],
            [square.x + square.size, square.y],
            [square.x, square.y + square.size],
            [square.x + square.size, square.y + square.size],
          ]

          corners.forEach((corner) => {
            ctx.beginPath()
            ctx.arc(corner[0], corner[1], handleSize, 0, 2 * Math.PI)
            ctx.fillStyle = '#fff'
            ctx.fill()
            ctx.strokeStyle = square.color
            ctx.lineWidth = 2
            ctx.stroke()
          })

          // Center point
          ctx.beginPath()
          ctx.arc(
            square.x + square.size / 2,
            square.y + square.size / 2,
            4,
            0,
            2 * Math.PI,
          )
          ctx.fillStyle = '#fff'
          ctx.fill()
          ctx.strokeStyle = square.color
          ctx.lineWidth = 2
          ctx.stroke()

          // Draw ID label
          ctx.font = `${16 / this.zoom}px Arial`
          ctx.fillStyle = '#fff'
          ctx.shadowBlur = 0
          ctx.fillText(
            square.id.toString(),
            square.x + square.size / 2 - 8,
            square.y + square.size / 2 + 6,
          )
        }

        ctx.restore()
      })
    },

    resizeCanvas() {
      const container = this.$refs.containerRef as HTMLDivElement
      const canvas = this.$refs.canvasRef as HTMLCanvasElement

      if (!container || !canvas) return

      const containerRect = container.getBoundingClientRect()
      const width = containerRect.width
      const height = containerRect.height

      canvas.width = width
      canvas.height = height
      canvas.style.width = `${width}px`
      canvas.style.height = `${height}px`

      this.draw()
    },

    setupResizeObserver() {
      this.resizeObserver = new ResizeObserver(() => {
        this.resizeCanvas()
      })

      const container = this.$refs.containerRef as HTMLElement
      if (container) {
        this.resizeObserver.observe(container)
      }
    },

    handleWindowResize() {
      this.resizeCanvas()
    },
  },
})
</script>

<style scoped lang="scss">
.main_canvas_editor {
  height: calc(100vh - 340px);
  width: 100%;
  background-color: rgb(241, 251, 198);
  position: relative;
  overflow: hidden;

  canvas {
    display: block;
    background-color: white;
    border: 2px solid #ccc;
    border-radius: 4px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    cursor: default;
    box-sizing: border-box;

    &:active {
      cursor: grabbing;
    }
  }

  .zoom-controls {
    position: absolute;
    bottom: 20px;
    right: 20px;
    display: flex;
    gap: 8px;
    background: white;
    padding: 8px 12px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    z-index: 100;

    .zoom-btn {
      width: 32px;
      height: 32px;
      border: 1px solid #ccc;
      background: white;
      border-radius: 4px;
      cursor: pointer;
      font-size: 18px;
      font-weight: bold;
      display: flex;
      align-items: center;
      justify-content: center;
      transition: all 0.2s;

      &:hover {
        background: #f0f0f0;
        border-color: #999;
      }

      &:active {
        background: #e0e0e0;
        transform: scale(0.95);
      }

      &.reset {
        font-size: 16px;
      }

      &.fit {
        font-size: 18px;
        font-weight: normal;
      }
    }

    .zoom-level {
      min-width: 50px;
      text-align: center;
      font-size: 14px;
      font-weight: 500;
      display: flex;
      align-items: center;
      justify-content: center;
      color: #333;
    }
  }

  .controls {
    position: absolute;
    top: 20px;
    right: 20px;
    display: flex;
    gap: 10px;
    z-index: 100;

    .control-btn {
      padding: 8px 16px;
      background: white;
      border: 1px solid #ccc;
      border-radius: 6px;
      cursor: pointer;
      font-size: 14px;
      font-weight: 500;
      transition: all 0.2s;
      font-family: inherit;

      &:hover:not(:disabled) {
        background: #f0f0f0;
        transform: translateY(-1px);
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
      }

      &:active:not(:disabled) {
        transform: translateY(0);
      }

      &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
      }
    }
  }

  .info-panel {
    position: absolute;
    top: 20px;
    left: 20px;
    background: rgba(255, 255, 255, 0.95);
    padding: 12px 16px;
    border-radius: 8px;
    font-size: 12px;
    font-family: monospace;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    pointer-events: none;
    z-index: 100;
    backdrop-filter: blur(4px);

    div {
      margin: 4px 0;
      color: #333;

      &.hint {
        margin-top: 8px;
        padding-top: 6px;
        border-top: 1px solid #e0e0e0;
        color: #666;
        font-size: 11px;
      }
    }
  }
}
</style>
