<template>
  <div class="game-container">
    <!-- Floating UI overlay -->
    <div class="floating-ui">
      <div class="game-title">Pazo</div>
      <div class="game-stats">
        <div class="stat-item">
          <span class="stat-label">Coins:</span>
          <span class="stat-value">{{ gameState.coins }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Clues:</span>
          <span class="stat-value">{{ gameState.cluesFound }}/{{ totalClues }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Key:</span>
          <span class="stat-value">{{ gameState.hasKey ? 'Found' : 'Missing' }}</span>
        </div>
      </div>
      <!-- <div class="instructions">
        <div>WASD or Arrow Keys to move</div>
        <div>Space to jump</div>
        <div>Find all clues to get the key!</div>
      </div> -->
    </div>

    <canvas
      ref="gameCanvas"
      :width="canvasWidth"
      :height="canvasHeight"
      @keydown="handleKeyDown"
      @keyup="handleKeyUp"
      tabindex="0"
      class="game-canvas"
    ></canvas>

    <div class="game-controls">
      <button @click="resetGame" class="reset-btn">Reset Game</button>
    </div>

    <div v-if="gameState.gameWon" class="game-won">
      <h2>🎉 Congratulations! 🎉</h2>
      <p>You've completed Pazo!</p>

      <!-- Kitten appears from behind the door -->
      <div class="kitten-container">
        <img src="/kitten.svg" alt="Cute kitten" class="kitten-svg" />
        <p class="kitten-text">A friendly kitten has appeared! 🐱</p>
      </div>

      <button @click="resetGame" class="reset-btn">Play Again</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, reactive } from 'vue'

interface GameObject {
  x: number
  y: number
  width: number
  height: number
  type: 'player' | 'platform' | 'coin' | 'clue' | 'key' | 'door'
  collected?: boolean
  visible?: boolean
}

interface GameState {
  coins: number
  cluesFound: number
  hasKey: boolean
  gameWon: boolean
}

const gameCanvas = ref<HTMLCanvasElement>()
const gameState = reactive<GameState>({
  coins: 0,
  cluesFound: 0,
  hasKey: false,
  gameWon: false
})

const totalClues = 3

// Canvas and viewport dimensions
const canvasWidth = ref(800)
const canvasHeight = ref(600)
const worldWidth = 2400 // Much wider world
const worldHeight = 600

// Camera position
const cameraX = ref(0)

// Static grass pattern (generated once)
const grassPattern: { x: number; y: number }[] = []
for (let x = 0; x < worldWidth; x += 40) {
  for (let y = 600; y < 1200; y += 30) { // Extended height to cover tall screens
    grassPattern.push({
      x: x + Math.random() * 20,
      y: y + Math.random() * 15
    })
  }
}

// Static cloud pattern (generated once)
const cloudPattern: { x: number; y: number; size: number }[] = []
for (let i = 0; i < 15; i++) {
  cloudPattern.push({
    x: Math.random() * worldWidth,
    y: 50 + Math.random() * 200,
    size: 30 + Math.random() * 40
  })
}

// Game objects
const player = reactive<GameObject>({
  x: 50,
  y: 500,
  width: 64,
  height: 64,
  type: 'player'
})

const gameObjects: GameObject[] = [
  // Ground platform (extends across the entire world)
  { x: 0, y: 600, width: worldWidth, height: 3, type: 'platform' },

  // Platforms extending to the right - adjusted heights for jumpability
  { x: 200, y: 480, width: 100, height: 20, type: 'platform' },
  { x: 400, y: 410, width: 100, height: 20, type: 'platform' },
  { x: 600, y: 340, width: 100, height: 20, type: 'platform' },
  { x: 800, y: 470, width: 120, height: 20, type: 'platform' },
  { x: 1000, y: 400, width: 100, height: 20, type: 'platform' },
  { x: 1200, y: 330, width: 150, height: 20, type: 'platform' },
  { x: 1400, y: 460, width: 100, height: 20, type: 'platform' },
  { x: 1600, y: 390, width: 120, height: 20, type: 'platform' },
  { x: 1800, y: 320, width: 100, height: 20, type: 'platform' },
  { x: 2000, y: 450, width: 150, height: 20, type: 'platform' },
  { x: 2200, y: 380, width: 100, height: 20, type: 'platform' },

  // Coins scattered throughout the world - adjusted positions to match platforms
  { x: 250, y: 430, width: 20, height: 20, type: 'coin', collected: false },
  { x: 450, y: 360, width: 20, height: 20, type: 'coin', collected: false },
  { x: 650, y: 290, width: 20, height: 20, type: 'coin', collected: false },
  { x: 850, y: 420, width: 20, height: 20, type: 'coin', collected: false },
  { x: 1050, y: 350, width: 20, height: 20, type: 'coin', collected: false },
  { x: 1250, y: 280, width: 20, height: 20, type: 'coin', collected: false },
  { x: 1450, y: 410, width: 20, height: 20, type: 'coin', collected: false },
  { x: 1650, y: 340, width: 20, height: 20, type: 'coin', collected: false },
  { x: 1850, y: 270, width: 20, height: 20, type: 'coin', collected: false },
  { x: 2050, y: 400, width: 20, height: 20, type: 'coin', collected: false },
  { x: 2250, y: 330, width: 20, height: 20, type: 'coin', collected: false },

  // Clues scattered throughout the world - adjusted positions to match platforms
  { x: 220, y: 450, width: 20, height: 20, type: 'clue', collected: false },
  { x: 420, y: 380, width: 20, height: 20, type: 'clue', collected: false },
  { x: 620, y: 310, width: 20, height: 20, type: 'clue', collected: false },

  // Key (hidden until all clues are found) - adjusted position
  { x: 2300, y: 330, width: 20, height: 20, type: 'key', collected: false, visible: false },

  // Exit door (at the very end of the world)
  { x: 2350, y: 450, width: 50, height: 100, type: 'door' }
]

// Player physics
const playerVelocity = reactive({
  x: 0,
  y: 0
})

const playerOnGround = ref(false)
const keysPressed = new Set<string>()
const consecutiveJumps = ref(0)
const wingFlapAngle = ref(0)
const isFlapping = ref(false)
const flapCount = ref(0)
const flapTimer = ref(0)

let animationFrameId: number
let lastTime = 0

// Update camera to follow player
const updateCamera = () => {
  const targetCameraX = player.x - canvasWidth.value / 2

  // Keep camera within world bounds
  if (targetCameraX < 0) {
    cameraX.value = 0
  } else if (targetCameraX > worldWidth - canvasWidth.value) {
    cameraX.value = worldWidth - canvasWidth.value
  } else {
    cameraX.value = targetCameraX
  }
}

// Game loop
const gameLoop = (currentTime: number) => {
  const deltaTime = currentTime - lastTime
  lastTime = currentTime

  updateGame(deltaTime)
  updateCamera()
  renderGame()

  animationFrameId = requestAnimationFrame(gameLoop)
}

// Update game state
const updateGame = (deltaTime: number) => {
  // Handle input
  if (keysPressed.has('a') || keysPressed.has('ArrowLeft')) {
    playerVelocity.x = -150
  } else if (keysPressed.has('d') || keysPressed.has('ArrowRight')) {
    playerVelocity.x = 150
  } else {
    playerVelocity.x = 0
  }

  if ((keysPressed.has('w') || keysPressed.has('ArrowUp') || keysPressed.has(' '))) {
    playerVelocity.y = -300
    consecutiveJumps.value++
    playerOnGround.value = false
    // Trigger 3 wing flaps
    isFlapping.value = true
    flapCount.value = 0
    flapTimer.value = 0
    wingFlapAngle.value = 0
  }

  // Apply gravity
  playerVelocity.y += 800 * (deltaTime / 1000)

  // Update player position
  player.x += playerVelocity.x * (deltaTime / 1000)
  player.y += playerVelocity.y * (deltaTime / 1000)

  // Update wing flap animation
  if (isFlapping.value) {
    flapTimer.value += deltaTime

    if (flapTimer.value >= 1000) { // 1 second per flap
      flapCount.value++
      flapTimer.value = 0

      if (flapCount.value >= 3) { // 3 flaps total
        isFlapping.value = false
        flapCount.value = 0
        wingFlapAngle.value = 0
      }
    }

    // Animate wing angle during each flap
    const flapProgress = (flapTimer.value / 1000) * 2 * Math.PI // 0 to 2π over 1 second
    wingFlapAngle.value = Math.sin(flapProgress) * 30 // 30 degrees max rotation
  }

  // Check collisions
  checkCollisions()

  // Update key visibility
  if (gameState.cluesFound === totalClues) {
    const key = gameObjects.find(obj => obj.type === 'key')
    if (key) key.visible = true
  }
}

// Check collisions
const checkCollisions = () => {
  playerOnGround.value = false

  for (const obj of gameObjects) {
    if (obj.type === 'platform' && checkCollision(player, obj)) {
      if (playerVelocity.y > 0 && player.y < obj.y) {
        player.y = obj.y - player.height
        playerVelocity.y = 0
        playerOnGround.value = true
        consecutiveJumps.value = 0 // Reset jump count when landing
      }
    }

    if (obj.type === 'coin' && !obj.collected && checkCollision(player, obj)) {
      obj.collected = true
      gameState.coins++
    }

    if (obj.type === 'clue' && !obj.collected && checkCollision(player, obj)) {
      obj.collected = true
      gameState.cluesFound++
    }

    if (obj.type === 'key' && obj.visible && !obj.collected && checkCollision(player, obj)) {
      obj.collected = true
      gameState.hasKey = true
    }

    if (obj.type === 'door' && gameState.hasKey && checkCollision(player, obj)) {
      gameState.gameWon = true
    }
  }

  // Keep player in world bounds
  if (player.x < 0) player.x = 0
  if (player.x + player.width > worldWidth) player.x = worldWidth - player.width
  if (player.y < 0) player.y = 0
  if (player.y + player.height > worldHeight) player.y = worldHeight - player.height
}

// Collision detection
const checkCollision = (obj1: GameObject, obj2: GameObject): boolean => {
  return obj1.x < obj2.x + obj2.width &&
         obj1.x + obj1.width > obj2.x &&
         obj1.y < obj2.y + obj2.height &&
         obj1.y + obj1.height > obj2.y
}

// Render game
const renderGame = () => {
  if (!gameCanvas.value) return

  const ctx = gameCanvas.value.getContext('2d')
  if (!ctx) return

  // Set font for any text that might be drawn
  ctx.font = '16px Sen, sans-serif'

  // Clear canvas
  ctx.fillStyle = '#87CEEB'
  ctx.fillRect(0, 0, canvasWidth.value, canvasHeight.value)

  // Save context for camera transformation
  ctx.save()
  ctx.translate(-cameraX.value, 0)

  // Draw clouds (before other objects so they appear behind)
  ctx.fillStyle = '#FFFFFF' // White clouds
  for (const cloud of cloudPattern) {
    ctx.beginPath()
    ctx.arc(cloud.x, cloud.y, cloud.size, 0, Math.PI * 2)
    ctx.fill()
  }

  // Draw grass background below ground level
  ctx.fillStyle = '#228B22'
  ctx.fillRect(0, 600, worldWidth, canvasHeight.value - 600)

  // Draw grass texture (simple pattern)
  ctx.fillStyle = '#32CD32'
  for (const grass of grassPattern) {
    // Only draw grass blades that are within the visible area
    if (grass.y < canvasHeight.value) {
      ctx.fillRect(grass.x, grass.y, 3, 8)
    }
  }

  // Draw platforms
  ctx.fillStyle = '#8B4513'
  for (const obj of gameObjects) {
    if (obj.type === 'platform') {
      ctx.fillRect(obj.x, obj.y, obj.width, obj.height)
    }
  }

  // Draw coins
  ctx.fillStyle = '#FFD700'
  for (const obj of gameObjects) {
    if (obj.type === 'coin' && !obj.collected) {
      ctx.beginPath()
      ctx.arc(obj.x + obj.width/2, obj.y + obj.height/2, obj.width/2, 0, Math.PI * 2)
      ctx.fill()
    }
  }

  // Draw clues
  ctx.fillStyle = '#FF6B6B'
  for (const obj of gameObjects) {
    if (obj.type === 'clue' && !obj.collected) {
      ctx.fillRect(obj.x, obj.y, obj.width, obj.height)
    }
  }

  // Draw key
  for (const obj of gameObjects) {
    if (obj.type === 'key' && obj.visible && !obj.collected) {
      ctx.fillStyle = '#FFD700'
      ctx.fillRect(obj.x, obj.y, obj.width, obj.height)
      ctx.fillStyle = '#000'
      ctx.fillRect(obj.x + 8, obj.y + 5, 4, 10)
    }
  }

  // Draw door
  for (const obj of gameObjects) {
    if (obj.type === 'door') {
      ctx.fillStyle = gameState.hasKey ? '#228B22' : '#8B0000'
      ctx.fillRect(obj.x, obj.y, obj.width, obj.height)
      ctx.fillStyle = '#000'
      ctx.fillRect(obj.x + 20, obj.y + 30, 10, 40)
    }
  }

  // Draw player (Pazo the Bird)
  // Bird body (round)
  ctx.fillStyle = '#4169E1'
  ctx.beginPath()
  ctx.arc(player.x + player.width/2, player.y + player.height/2, player.width/2, 0, Math.PI * 2)
  ctx.fill()

  // Bird wings (triangular and flapping)
  ctx.fillStyle = '#4169E1' // Same color as body

  // Left wing (triangular, attached to body)
  ctx.save()
  ctx.translate(player.x + 4, player.y + player.height/2)
  ctx.rotate((wingFlapAngle.value * Math.PI) / 180)
  ctx.beginPath()
  ctx.moveTo(0, 0)
  ctx.lineTo(-24, -12)
  ctx.lineTo(-24, 12)
  ctx.closePath()
  ctx.fill()
  ctx.restore()

  // Right wing (triangular, attached to body)
  ctx.save()
  ctx.translate(player.x + player.width - 4, player.y + player.height/2)
  ctx.rotate((-wingFlapAngle.value * Math.PI) / 180)
  ctx.beginPath()
  ctx.moveTo(0, 0)
  ctx.lineTo(24, -12)
  ctx.lineTo(24, 12)
  ctx.closePath()
  ctx.fill()
  ctx.restore()

  // Bird eyes (white with black pupils)
  ctx.fillStyle = '#FFFFFF'
  ctx.beginPath()
  ctx.arc(player.x + 24, player.y + 20, 8, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.arc(player.x + 40, player.y + 20, 8, 0, Math.PI * 2)
  ctx.fill()

  // Black pupils
  ctx.fillStyle = '#000'
  ctx.beginPath()
  ctx.arc(player.x + 24, player.y + 20, 4, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.arc(player.x + 40, player.y + 20, 4, 0, Math.PI * 2)
  ctx.fill()

  // Bird beak (small triangle)
  ctx.fillStyle = '#FFA500'
  ctx.beginPath()
  ctx.moveTo(player.x + player.width/2, player.y + 36)
  ctx.lineTo(player.x + player.width/2 - 6, player.y + 44)
  ctx.lineTo(player.x + player.width/2 + 6, player.y + 44)
  ctx.closePath()
  ctx.fill()

  // Bird feet (tiny)
  ctx.fillStyle = '#FFA500'
  ctx.fillRect(player.x + 20, player.y + player.height - 4, 6, 8)
  ctx.fillRect(player.x + player.width - 26, player.y + player.height - 4, 6, 8)

  // Restore context
  ctx.restore()
}

// Input handling
const handleKeyDown = (event: KeyboardEvent) => {
  // Handle arrow keys and other keys properly
  if (event.key === 'ArrowLeft' || event.key === 'ArrowRight' || event.key === 'ArrowUp' || event.key === 'ArrowDown') {
    keysPressed.add(event.key)
  } else {
    keysPressed.add(event.key.toLowerCase())
  }
}

const handleKeyUp = (event: KeyboardEvent) => {
  // Handle arrow keys and other keys properly
  if (event.key === 'ArrowLeft' || event.key === 'ArrowRight' || event.key === 'ArrowUp' || event.key === 'ArrowDown') {
    keysPressed.delete(event.key)
  } else {
    keysPressed.delete(event.key.toLowerCase())
  }
}

// Reset game
const resetGame = () => {
  gameState.coins = 0
  gameState.cluesFound = 0
  gameState.hasKey = false
  gameState.gameWon = false

  player.x = 50
  player.y = 500
  playerVelocity.x = 0
  playerVelocity.y = 0
  cameraX.value = 0
  consecutiveJumps.value = 0 // Reset jump count on game reset

  for (const obj of gameObjects) {
    if (obj.type === 'coin' || obj.type === 'clue') {
      obj.collected = false
    }
    if (obj.type === 'key') {
      obj.collected = false
      obj.visible = false
    }
  }
}

// Handle window resize
const handleResize = () => {
  canvasWidth.value = window.innerWidth
  canvasHeight.value = window.innerHeight
}

// Lifecycle
onMounted(() => {
  if (gameCanvas.value) {
    gameCanvas.value.focus()
    lastTime = performance.now()
    animationFrameId = requestAnimationFrame(gameLoop)
  }

  // Set initial canvas size
  handleResize()
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
  }
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped>
.game-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  font-family: 'Arial', sans-serif;
}

.floating-ui {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 1000;
  color: white;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.7);
}

.game-title {
  font-size: 2.5rem;
  font-weight: bold;
  margin-bottom: 20px;
  text-shadow: 3px 3px 6px rgba(0,0,0,0.8);
}

.game-stats {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
}

.stat-item {
  display: flex;
  gap: 10px;
  font-size: 1.1rem;
}

.stat-label {
  font-weight: bold;
  min-width: 60px;
}

.stat-value {
  color: #FFD700;
  font-weight: bold;
}

.instructions {
  background: rgba(0,0,0,0.3);
  padding: 15px;
  border-radius: 10px;
  font-size: 0.9rem;
  line-height: 1.4;
  backdrop-filter: blur(5px);
}

.game-canvas {
  display: block;
  width: 100%;
  height: 100%;
  cursor: crosshair;
}

.game-controls {
  position: absolute;
  bottom: 20px;
  right: 20px;
  z-index: 1000;
}

.reset-btn {
  background: linear-gradient(45deg, #ff6b6b, #ee5a24);
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 25px;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  box-shadow: 0 4px 15px rgba(0,0,0,0.3);
}

.reset-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.4);
}

.game-won {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  padding: 40px;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 20px 40px rgba(0,0,0,0.5);
  z-index: 2000;
  backdrop-filter: blur(10px);
}

.game-won h2 {
  margin: 0 0 20px 0;
  font-size: 2.5rem;
}

.game-won p {
  font-size: 1.3rem;
  margin: 0 0 30px 0;
}

.kitten-container {
  margin-bottom: 20px;
}

.kitten-svg {
  display: block;
  margin: 0 auto 10px auto;
  fill: none;
  stroke: #8B4513;
  stroke-width: 2;
}

.kitten-text {
  font-size: 1.2rem;
  font-weight: bold;
  color: #FFB6C1;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.5);
}
</style>
