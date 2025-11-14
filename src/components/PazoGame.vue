<template>
  <div class="game-container">
    <!-- Floating UI overlay -->
    <div class="floating-ui">
      <div class="game-title">Pazo</div>
      <div class="game-stats">
        <div class="stat-item">
          <span class="stat-label">Level:</span>
          <span class="stat-value" :class="{ 'level-2': currentLevel >= 2, 'level-5': currentLevel >= 5 }">{{ currentLevel }}</span>
        </div>
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
        <button @click="isMusicPlaying ? stopMusic() : startMusic()" class="music-btn">
          {{ isMusicPlaying ? '🔇' : '🔊' }}
        </button>
      </div>

      <!-- Start Game Button -->
      <div v-if="!gameState.gameStarted" class="start-game-overlay">
        <div class="start-game-content">
          <h1 class="start-game-title">Welcome to Pazo!</h1>
          <p class="start-game-description">Help Pazo the bird collect clues and find the key!</p>
          <button @click="startGame" class="start-game-btn">Start Game</button>
        </div>
      </div>

    <!-- Background music -->
    <audio ref="backgroundMusic" preload="auto">
      <source src="/music.mp3" type="audio/mpeg">
    </audio>

    <!-- Finish sound -->
    <audio ref="finishSound" preload="auto">
      <source src="/finish.mp3" type="audio/mpeg">
    </audio>

    <!-- Game over sound -->
    <audio ref="gameOverSound" preload="auto">
      <source src="/gameover.mp3" type="audio/mpeg">
    </audio>

          <div v-if="gameState.gameWon" class="game-won">
        <h2>🎉 Congratulations! 🎉</h2>
        <p>You've completed Level {{ currentLevel }}!</p>

        <!-- Show trophy for Level 5 completion -->
        <div v-if="currentLevel >= 5" class="trophy-container">
          <div class="trophy">🏆</div>
          <p class="trophy-text">You're a Pazo master! You've completed all levels!</p>
        </div>

        <p v-else-if="currentLevel < 5">Great job! Ready for the next challenge?</p>

        <!-- Kitten(s) appear from behind the door -->
        <div class="kitten-container">
          <img
            v-for="i in Math.min(currentLevel, 5)"
            :key="i"
            src="/kitten.svg"
            :alt="`Kitten ${i}`"
            class="kitten-svg"
            :class="`level-${Math.min(currentLevel, 5)}-kitten`"
          />
        </div>

        <!-- Show different buttons based on level -->
        <div v-if="currentLevel < 5">
          <button @click="nextLevel" class="next-level-btn">Next Level</button>
          <button @click="resetGame" class="reset-btn">Play Again</button>
        </div>
        <div v-else>
          <button @click="resetGame" class="reset-btn">Play Again</button>
        </div>
      </div>

      <!-- Game Over Overlay -->
      <div v-if="gameState.gameOver" class="game-over">
        <h2>😿 Oh no! 😿</h2>
        <p>You've been eaten by the cat!</p>
        <div class="cat-emoji">🐱</div>
        <button @click="restartLevel" class="restart-btn">Try Again</button>
        <button @click="resetGame" class="reset-btn">Main Menu</button>
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
  type: 'player' | 'platform' | 'coin' | 'clue' | 'key' | 'door' | 'cat'
  collected?: boolean
  visible?: boolean
}

interface GameState {
  coins: number
  cluesFound: number
  hasKey: boolean
  gameWon: boolean
  gameStarted: boolean
  gameOver: boolean
}

const gameCanvas = ref<HTMLCanvasElement>()
const backgroundMusic = ref<HTMLAudioElement>()
const finishSound = ref<HTMLAudioElement>()
const gameOverSound = ref<HTMLAudioElement>()
const isMusicPlaying = ref(false)
const hasPlayedFinishSound = ref(false)

const gameState = reactive<GameState>({
  coins: 0,
  cluesFound: 0,
  hasKey: false,
  gameWon: false,
  gameStarted: false,
  gameOver: false
})

const currentLevel = ref(1)
const totalClues = ref(3)
const currentClueIndex = ref(0) // Track which clue should be visible

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

// Cat object
const cat = reactive<GameObject>({
  x: worldWidth / 2, // Start in middle of level
  y: 536, // Ground level - 64 (cat height)
  width: 48,
  height: 48,
  type: 'cat'
})

// Cat AI variables
const catVelocity = reactive({ x: 0, y: 0 })
const catDirection = ref(1) // 1 for right, -1 for left
const catIsMoving = ref(false)
const catOnGround = ref(false)

// Function to generate level objects dynamically
const generateLevelObjects = (level: number): GameObject[] => {
  const objects: GameObject[] = []

  // Ground platform (extends across the entire world)
  objects.push({ x: 0, y: 600, width: worldWidth, height: 3, type: 'platform' })

  // Calculate difficulty scaling
  const platformCount = Math.max(8, 12 - Math.floor(level / 3)) // Fewer platforms as level increases
  const platformWidth = Math.max(60, 120 - level * 2) // Narrower platforms as level increases
  const maxHeight = Math.max(200, 350 - level * 10) // Higher platforms as level increases
  const minGap = Math.max(150, 200 + level * 10) // Larger gaps as level increases

  // Generate platforms with increasing difficulty
  for (let i = 0; i < platformCount; i++) {
    const x = 200 + i * (worldWidth - 400) / platformCount
    const y = 600 - maxHeight + Math.random() * (maxHeight - 100)
    objects.push({
      x: x,
      y: y,
      width: platformWidth,
      height: 20,
      type: 'platform'
    })
  }

  // Generate coins
  const coinCount = Math.max(5, 10 - Math.floor(level / 2))
  const platforms = objects.filter(obj => obj.type === 'platform' && obj.x > 0)

  for (let i = 0; i < coinCount; i++) {
    if (platforms.length > 0) {
      // Pick a random platform for each coin
      const randomPlatformIndex = Math.floor(Math.random() * platforms.length)
      const platform = platforms[randomPlatformIndex]

      objects.push({
        x: platform.x + Math.random() * (platform.width - 20),
        y: platform.y - 30 - Math.random() * 20,
        width: 20,
        height: 20,
        type: 'coin',
        collected: false
      })
    }
  }

  // Generate clues - number increases with level
  const clueCount = Math.min(3 + Math.floor(level / 2), 8) // Max 8 clues
  totalClues.value = clueCount

  for (let i = 0; i < clueCount; i++) {
    if (platforms.length > 0) {
      // Pick a random platform for each clue, ensuring good distribution
      const platformIndex = Math.floor((i / clueCount) * platforms.length)
      const platform = platforms[platformIndex]

      objects.push({
        x: platform.x + Math.random() * (platform.width - 20),
        y: platform.y - 30 - Math.random() * 20,
        width: 20,
        height: 20,
        type: 'clue',
        collected: false
      })
    }
  }

  // Key position - place on a reachable platform
  let keyPlatform: GameObject | undefined
  if (platforms.length > 0) {
    // Find platforms that are reachable (not too high)
    const reachablePlatforms = platforms.filter(p => p.y >= 400)
    if (reachablePlatforms.length > 0) {
      // Pick a platform in the middle area that's reachable
      const midPlatforms = reachablePlatforms.filter(p => p.x > 1000 && p.x < 2000)
      if (midPlatforms.length > 0) {
        keyPlatform = midPlatforms[Math.floor(Math.random() * midPlatforms.length)]
      } else {
        keyPlatform = reachablePlatforms[Math.floor(reachablePlatforms.length / 2)]
      }
    } else {
      // If all platforms are too high, use the lowest one
      keyPlatform = platforms.reduce((lowest, current) =>
        current.y > lowest.y ? current : lowest
      )
    }
  }

  if (keyPlatform) {
    objects.push({
      x: keyPlatform.x + Math.random() * (keyPlatform.width - 20),
      y: keyPlatform.y - 30,
      width: 20,
      height: 20,
      type: 'key',
      collected: false,
      visible: false
    })
  } else {
    // Fallback: place on ground level
    objects.push({
      x: 1500,
      y: 570,
      width: 20,
      height: 20,
      type: 'key',
      collected: false,
      visible: false
    })
  }

  // Exit door
  objects.push({
    x: 2350,
    y: 450,
    width: 50,
    height: 100,
    type: 'door'
  })

  return objects
}

// Current game objects (will be set based on level)
let gameObjects: GameObject[] = generateLevelObjects(1)

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

// Music control functions
const initializeMusic = () => {
  if (backgroundMusic.value) {
    backgroundMusic.value.loop = true
    backgroundMusic.value.volume = 0.3 // Set volume to 30%
  }
  if (finishSound.value) {
    finishSound.value.volume = 0.5 // Set volume to 50% for finish sound
  }
  if (gameOverSound.value) {
    gameOverSound.value.volume = 0.6 // Set volume to 60% for game over sound
  }
}

const startMusic = () => {
  if (backgroundMusic.value && !isMusicPlaying.value) {
    backgroundMusic.value.play().catch(error => {
      console.log('Music autoplay prevented:', error)
    })
    isMusicPlaying.value = true
  }
}

const stopMusic = () => {
  if (backgroundMusic.value && isMusicPlaying.value) {
    backgroundMusic.value.pause()
    backgroundMusic.value.currentTime = 0
    isMusicPlaying.value = false
  }
}

const stopAllSounds = () => {
  // Stop background music
  if (backgroundMusic.value) {
    backgroundMusic.value.pause()
    backgroundMusic.value.currentTime = 0
    isMusicPlaying.value = false
  }

  // Stop finish sound
  if (finishSound.value) {
    finishSound.value.pause()
    finishSound.value.currentTime = 0
  }

  // Stop game over sound
  if (gameOverSound.value) {
    gameOverSound.value.pause()
    gameOverSound.value.currentTime = 0
  }
}

const playFinishSound = () => {
  if (finishSound.value) {
    // Stop background music temporarily
    if (backgroundMusic.value && isMusicPlaying.value) {
      backgroundMusic.value.pause()
    }

    // Play finish sound
    finishSound.value.currentTime = 0
    finishSound.value.play().catch(error => {
      console.log('Finish sound play failed:', error)
    })

    // Don't automatically resume music - let Next Level button handle it
  }
}

const playGameOverSound = () => {
  // Stop background music and finish sound, but don't touch game over sound
  if (backgroundMusic.value && isMusicPlaying.value) {
    backgroundMusic.value.pause()
    backgroundMusic.value.currentTime = 0
    isMusicPlaying.value = false
  }

  if (finishSound.value) {
    finishSound.value.pause()
    finishSound.value.currentTime = 0
  }

  // Wait a moment for sounds to stop, then play game over sound
  setTimeout(() => {
    if (gameOverSound.value) {
      // Only reset if not currently playing
      if (gameOverSound.value.paused) {
        gameOverSound.value.currentTime = 0
      }
      gameOverSound.value.play().catch(error => {
        console.log('Game over sound play failed:', error)
      })
    }
  }, 200)
}

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

  // Update cat AI
  updateCatAI(deltaTime)

  // Check collisions
  checkCollisions()

  // Update key visibility
if (gameState.cluesFound === totalClues.value) {
  const key = gameObjects.find(obj => obj.type === 'key')
  if (key) key.visible = true
}

// Check if player fell off the world (level 2 is harder)
if (player.y > worldHeight + 100) {
  // Player fell off, reset to start of current level
  player.x = 50
  player.y = 500
  playerVelocity.x = 0
  playerVelocity.y = 0
  cameraX.value = 0
}
}

// Update cat AI
const updateCatAI = (deltaTime: number) => {
  // Only start moving if Pazo has started moving
  if (!gameState.gameStarted || !catIsMoving.value) {
    return
  }

  // Stop chasing if Pazo has reached the door (game won)
  if (gameState.gameWon) {
    catVelocity.x = 0
    catVelocity.y = 0
    return
  }

  // Always chase Pazo - determine direction based on Pazo's position
  if (player.x > cat.x) {
    catDirection.value = 1 // Move right towards Pazo
  } else if (player.x < cat.x) {
    catDirection.value = -1 // Move left towards Pazo
  }
  // If Pazo is exactly at cat's position, keep current direction

  // Set cat velocity based on direction
  catVelocity.x = catDirection.value * 80 // Slower than Pazo

  // Make cat jump slowly if Pazo is above it
  const verticalDistance = player.y - cat.y
  const horizontalDistance = Math.abs(player.x - cat.x)

  // Jump if Pazo is significantly above the cat and cat is on ground
  if (catOnGround.value && verticalDistance < -50 && horizontalDistance < 300) {
    catVelocity.y = -400 // Slow jump (slower than Pazo's -300)
  }

  // Apply gravity to cat
  catVelocity.y += 800 * (deltaTime / 1000)

  // Update cat position
  cat.x += catVelocity.x * (deltaTime / 1000)
  cat.y += catVelocity.y * (deltaTime / 1000)

  // Keep cat within world bounds
  if (cat.x < 0) {
    cat.x = 0
    catDirection.value = 1
  }
  if (cat.x + cat.width > worldWidth) {
    cat.x = worldWidth - cat.width
    catDirection.value = -1
  }

  // Keep cat from going below ground level
  if (cat.y > 536) {
    cat.y = 536
    catVelocity.y = 0
  }
}

// Check collisions
const checkCollisions = () => {
  playerOnGround.value = false
  catOnGround.value = false

  // Check cat collision with Pazo
  if (checkCollision(player, cat)) {
    // Game over - show game over screen
    gameState.gameOver = true
    playGameOverSound()
    return
  }

  // Check if cat is on ground level
  if (cat.y >= 536) {
    catOnGround.value = true
  }

  // Check cat collision with platforms
  for (const obj of gameObjects) {
    if (obj.type === 'platform' && checkCollision(cat, obj)) {
      if (catVelocity.y > 0 && cat.y < obj.y) {
        cat.y = obj.y - cat.height
        catVelocity.y = 0
        catOnGround.value = true
      }
    }
  }

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
      // Only collect the current clue
      const clues = gameObjects.filter(o => o.type === 'clue')
      if (clues.length > 0 && currentClueIndex.value < clues.length && obj === clues[currentClueIndex.value]) {
        obj.collected = true
        gameState.cluesFound++
        // Move to next clue
        currentClueIndex.value++
      }
    }

    if (obj.type === 'key' && obj.visible && !obj.collected && checkCollision(player, obj)) {
      obj.collected = true
      gameState.hasKey = true
    }

    if (obj.type === 'door' && gameState.hasKey && checkCollision(player, obj)) {
      gameState.gameWon = true
      if (!hasPlayedFinishSound.value) {
        playFinishSound()
        hasPlayedFinishSound.value = true
      }
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

// Draw level-specific background
const drawLevelBackground = (ctx: CanvasRenderingContext2D) => {
  switch (currentLevel.value) {
    case 1:
      // Level 1: Current sky background (already handled by main canvas clear)
      break

    case 2:
      // Level 2: Forest background
      // Sky gradient
      const forestGradient = ctx.createLinearGradient(0, 0, 0, canvasHeight.value)
      forestGradient.addColorStop(0, '#87CEEB')
      forestGradient.addColorStop(0.7, '#98FB98')
      forestGradient.addColorStop(1, '#228B22')
      ctx.fillStyle = forestGradient
      ctx.fillRect(0, 0, worldWidth, canvasHeight.value)

      // Draw trees in background
      ctx.fillStyle = '#8B4513'
      for (let i = 0; i < 20; i++) {
        const treeX = i * 120 + 50
        const treeHeight = 150 + Math.random() * 100
        const treeY = 600 - treeHeight

        // Tree trunk
        ctx.fillRect(treeX, treeY + treeHeight - 30, 20, 30)

        // Tree foliage
        ctx.fillStyle = '#228B22'
        ctx.beginPath()
        ctx.arc(treeX + 10, treeY + treeHeight - 50, 40, 0, Math.PI * 2)
        ctx.fill()
        ctx.beginPath()
        ctx.arc(treeX + 10, treeY + treeHeight - 80, 35, 0, Math.PI * 2)
        ctx.fill()
        ctx.beginPath()
        ctx.arc(treeX + 10, treeY + treeHeight - 110, 30, 0, Math.PI * 2)
        ctx.fill()

        ctx.fillStyle = '#8B4513'
      }
      break

    case 3:
      // Level 3: Castle background
      // Sky gradient
      const castleGradient = ctx.createLinearGradient(0, 0, 0, canvasHeight.value)
      castleGradient.addColorStop(0, '#4169E1')
      castleGradient.addColorStop(0.8, '#9370DB')
      castleGradient.addColorStop(1, '#8B4513')
      ctx.fillStyle = castleGradient
      ctx.fillRect(0, 0, worldWidth, canvasHeight.value)

      // Draw castle towers in background
      ctx.fillStyle = '#696969'
      for (let i = 0; i < 8; i++) {
        const towerX = i * 300 + 100
        const towerHeight = 200 + Math.random() * 100
        const towerY = 600 - towerHeight

        // Tower base
        ctx.fillRect(towerX, towerY, 60, towerHeight)

        // Tower top (battlements)
        ctx.fillStyle = '#2F4F4F'
        for (let j = 0; j < 4; j++) {
          ctx.fillRect(towerX + j * 15, towerY - 20, 10, 20)
        }

        // Tower flag
        ctx.fillStyle = '#DC143C'
        ctx.fillRect(towerX + 25, towerY - 40, 20, 15)

        ctx.fillStyle = '#696969'
      }
      break

    case 4:
      // Level 4: Underwater background
      // Water gradient
      const waterGradient = ctx.createLinearGradient(0, 0, 0, canvasHeight.value)
      waterGradient.addColorStop(0, '#1E90FF')
      waterGradient.addColorStop(0.5, '#4169E1')
      waterGradient.addColorStop(1, '#000080')
      ctx.fillStyle = waterGradient
      ctx.fillRect(0, 0, worldWidth, canvasHeight.value)

      // Draw bubbles
      ctx.fillStyle = 'rgba(255, 255, 255, 0.3)'
      for (let i = 0; i < 30; i++) {
        const bubbleX = Math.random() * worldWidth
        const bubbleY = Math.random() * canvasHeight.value
        const bubbleSize = 5 + Math.random() * 15

        ctx.beginPath()
        ctx.arc(bubbleX, bubbleY, bubbleSize, 0, Math.PI * 2)
        ctx.fill()
      }

      // Draw seaweed
      ctx.fillStyle = '#228B22'
      for (let i = 0; i < 15; i++) {
        const seaweedX = i * 160 + 80
        const seaweedHeight = 100 + Math.random() * 150
        const seaweedY = 600 - seaweedHeight

        // Draw wavy seaweed
        ctx.beginPath()
        ctx.moveTo(seaweedX, 600)
        for (let j = 0; j < seaweedHeight; j += 10) {
          const wave = Math.sin((j / 20) + (seaweedX / 100)) * 10
          ctx.lineTo(seaweedX + wave, 600 - j)
        }
        ctx.stroke()
      }
      break

    case 5:
      // Level 5: Volcano background
      // Sky gradient (dark and fiery)
      const volcanoGradient = ctx.createLinearGradient(0, 0, 0, canvasHeight.value)
      volcanoGradient.addColorStop(0, '#8B0000')
      volcanoGradient.addColorStop(0.3, '#FF4500')
      volcanoGradient.addColorStop(0.6, '#FF6347')
      volcanoGradient.addColorStop(1, '#2F4F4F')
      ctx.fillStyle = volcanoGradient
      ctx.fillRect(0, 0, worldWidth, canvasHeight.value)

      // Draw volcano in background
      ctx.fillStyle = '#2F4F4F'
      ctx.beginPath()
      ctx.moveTo(0, 600)
      ctx.lineTo(400, 200)
      ctx.lineTo(800, 600)
      ctx.closePath()
      ctx.fill()

      // Draw lava flows
      ctx.fillStyle = '#FF4500'
      for (let i = 0; i < 5; i++) {
        const lavaX = 200 + i * 100
        ctx.fillRect(lavaX, 400, 20, 200)
      }

      // Draw smoke/ash clouds
      ctx.fillStyle = 'rgba(105, 105, 105, 0.6)'
      for (let i = 0; i < 10; i++) {
        const smokeX = 300 + Math.random() * 200
        const smokeY = 150 + Math.random() * 100
        const smokeSize = 30 + Math.random() * 40

        ctx.beginPath()
        ctx.arc(smokeX, smokeY, smokeSize, 0, Math.PI * 2)
        ctx.fill()
      }
      break
  }
}

// Draw cat
const drawCat = (ctx: CanvasRenderingContext2D) => {
  // Cat body (orange)
  ctx.fillStyle = '#FF8C00'
  ctx.fillRect(cat.x, cat.y, cat.width, cat.height)

  // Cat head (smaller circle on top)
  ctx.fillStyle = '#FF8C00'
  ctx.beginPath()
  ctx.arc(cat.x + cat.width/2, cat.y + 10, 16, 0, Math.PI * 2)
  ctx.fill()

  // Cat ears
  ctx.fillStyle = '#FF8C00'
  ctx.beginPath()
  ctx.moveTo(cat.x + 8, cat.y + 5)
  ctx.lineTo(cat.x + 12, cat.y - 5)
  ctx.lineTo(cat.x + 16, cat.y + 5)
  ctx.closePath()
  ctx.fill()

  ctx.beginPath()
  ctx.moveTo(cat.x + cat.width - 16, cat.y + 5)
  ctx.lineTo(cat.x + cat.width - 12, cat.y - 5)
  ctx.lineTo(cat.x + cat.width - 8, cat.y + 5)
  ctx.closePath()
  ctx.fill()

  // Cat eyes
  ctx.fillStyle = '#000'
  ctx.beginPath()
  ctx.arc(cat.x + cat.width/2 - 6, cat.y + 8, 2, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.arc(cat.x + cat.width/2 + 6, cat.y + 8, 2, 0, Math.PI * 2)
  ctx.fill()

  // Cat nose
  ctx.fillStyle = '#FF69B4'
  ctx.beginPath()
  ctx.moveTo(cat.x + cat.width/2, cat.y + 12)
  ctx.lineTo(cat.x + cat.width/2 - 2, cat.y + 15)
  ctx.lineTo(cat.x + cat.width/2 + 2, cat.y + 15)
  ctx.closePath()
  ctx.fill()

  // Cat tail (wavy)
  ctx.strokeStyle = '#FF8C00'
  ctx.lineWidth = 4
  ctx.beginPath()
  ctx.moveTo(cat.x + cat.width, cat.y + cat.height/2)
  for (let i = 0; i < 20; i++) {
    const wave = Math.sin(i * 0.5) * 8
    ctx.lineTo(cat.x + cat.width + i * 2, cat.y + cat.height/2 + wave)
  }
  ctx.stroke()

  // Cat legs
  ctx.fillStyle = '#FF8C00'
  ctx.fillRect(cat.x + 8, cat.y + cat.height - 8, 6, 8)
  ctx.fillRect(cat.x + cat.width - 14, cat.y + cat.height - 8, 6, 8)
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

  // Draw level-specific background
  drawLevelBackground(ctx)

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

  // Draw clues - only show the current clue
  ctx.fillStyle = '#FF6B6B'
  const clues = gameObjects.filter(obj => obj.type === 'clue')
  if (clues.length > 0 && currentClueIndex.value < clues.length) {
    const currentClue = clues[currentClueIndex.value]
    if (!currentClue.collected) {
      ctx.fillRect(currentClue.x, currentClue.y, currentClue.width, currentClue.height)
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

  // Draw cat
  drawCat(ctx)

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

// Next level function
const nextLevel = () => {
  // Don't advance beyond Level 5
  if (currentLevel.value >= 5) {
    return
  }

  // Start background music when advancing to next level
  if (!isMusicPlaying.value) {
    startMusic()
  }

  currentLevel.value++
  gameObjects = generateLevelObjects(currentLevel.value)
  resetGameState()

  // Start cat moving after advancing to next level
  setTimeout(() => {
    catIsMoving.value = true
  }, 1000)
}

// Reset game state (without resetting level)
const resetGameState = () => {
  gameState.coins = 0
  gameState.cluesFound = 0
  gameState.hasKey = false
  gameState.gameWon = false
  gameState.gameStarted = true // Keep game started when resetting
  gameState.gameOver = false // Reset game over state
  hasPlayedFinishSound.value = false // Reset finish sound flag
  currentClueIndex.value = 0 // Reset clue index

  player.x = 50
  player.y = 500
  playerVelocity.x = 0
  playerVelocity.y = 0
  cameraX.value = 0
  consecutiveJumps.value = 0 // Reset jump count on game reset

  // Reset cat to middle of level
  cat.x = worldWidth / 2
  cat.y = 536
  catVelocity.x = 0
  catVelocity.y = 0
  catDirection.value = 1
  catIsMoving.value = false
  catOnGround.value = true

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

// Restart current level (from game over screen)
const restartLevel = () => {
  gameState.gameOver = false

  // Stop game over sound if it's playing
  if (gameOverSound.value) {
    gameOverSound.value.pause()
    gameOverSound.value.currentTime = 0
  }

  resetGameState()

  // Start background music after a short delay
  setTimeout(() => {
    startMusic()
  }, 500)

  // Start cat moving again after restart
  setTimeout(() => {
    catIsMoving.value = true
  }, 1000)
}

// Start game function
const startGame = () => {
  gameState.gameStarted = true
  startMusic()
  // Start cat moving after a short delay
  setTimeout(() => {
    catIsMoving.value = true
  }, 1000)
}

// Reset game
const resetGame = () => {
  // Start background music when resetting game
  if (!isMusicPlaying.value) {
    startMusic()
  }

  currentLevel.value = 1
  gameObjects = generateLevelObjects(1)
  gameState.gameOver = false // Reset game over state
  resetGameState()
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

  // Initialize music (but don't start it yet)
  initializeMusic()

  // Set initial canvas size
  handleResize()
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
  }

  // Stop background music
  stopMusic()

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

.stat-value.level-2 {
  color: #FF6B6B;
  animation: pulse 2s infinite;
}

.stat-value.level-5 {
  color: #FFD700;
  animation: pulse 1s infinite;
  text-shadow: 0 0 10px #FFD700;
}

@keyframes pulse {
  0% { opacity: 1; }
  50% { opacity: 0.7; }
  100% { opacity: 1; }
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
  display: flex;
  gap: 10px;
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

.music-btn {
  background: linear-gradient(45deg, #4CAF50, #45a049);
  color: white;
  border: none;
  padding: 12px 16px;
  border-radius: 25px;
  font-size: 1.2rem;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  box-shadow: 0 4px 15px rgba(0,0,0,0.3);
  min-width: 50px;
}

.music-btn:hover {
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

.trophy-container {
  margin-bottom: 20px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 15px;
}

.trophy {
  font-size: 4rem;
  animation: trophyGlow 2s ease-in-out infinite alternate;
}

.trophy-text {
  font-size: 1.4rem;
  font-weight: bold;
  color: #FFD700;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
  margin: 0;
}

@keyframes trophyGlow {
  0% {
    transform: scale(1);
    filter: drop-shadow(0 0 10px #FFD700);
  }
  100% {
    transform: scale(1.1);
    filter: drop-shadow(0 0 20px #FFD700);
  }
}

.kitten-container {
  margin-bottom: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
}

.kitten-svg {
  fill: none;
  stroke: #8B4513;
  stroke-width: 2;
}

.level-1-kitten {
  width: 100px;
  height: 100px;
}

.level-2-kitten {
  width: 90px;
  height: 90px;
}

.level-3-kitten {
  width: 80px;
  height: 80px;
}

.level-4-kitten {
  width: 70px;
  height: 70px;
}

.level-5-kitten {
  width: 60px;
  height: 60px;
}

.next-level-btn {
  background: linear-gradient(45deg, #4CAF50, #45a049);
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 25px;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  box-shadow: 0 4px 15px rgba(0,0,0,0.3);
  margin-right: 10px;
}

.next-level-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.4);
}

.start-game-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 3000;
  backdrop-filter: blur(10px);
}

.start-game-content {
  text-align: center;
  color: white;
  padding: 40px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 20px;
  box-shadow: 0 20px 40px rgba(0,0,0,0.3);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.start-game-title {
  font-size: 3rem;
  font-weight: bold;
  margin: 0 0 20px 0;
  text-shadow: 3px 3px 6px rgba(0,0,0,0.8);
  background: linear-gradient(45deg, #FFD700, #FFA500);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.start-game-description {
  font-size: 1.3rem;
  margin: 0 0 30px 0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.7);
  line-height: 1.5;
}

.start-game-btn {
  background: linear-gradient(45deg, #4CAF50, #45a049);
  color: white;
  border: none;
  padding: 16px 32px;
  border-radius: 30px;
  font-size: 1.3rem;
  font-weight: bold;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  box-shadow: 0 6px 20px rgba(0,0,0,0.3);
  text-transform: uppercase;
  letter-spacing: 1px;
}

.start-game-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 25px rgba(0,0,0,0.4);
  background: linear-gradient(45deg, #45a049, #4CAF50);
}

.game-over {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: linear-gradient(135deg, #8B0000, #DC143C);
  color: white;
  padding: 40px;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 20px 40px rgba(0,0,0,0.5);
  z-index: 2000;
  backdrop-filter: blur(10px);
  border: 2px solid #FFD700;
}

.game-over h2 {
  margin: 0 0 20px 0;
  font-size: 2.5rem;
  color: #FFD700;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
}

.game-over p {
  font-size: 1.4rem;
  margin: 0 0 20px 0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.7);
}

.cat-emoji {
  font-size: 4rem;
  margin: 20px 0;
  animation: catBounce 1s ease-in-out infinite alternate;
}

@keyframes catBounce {
  0% { transform: scale(1); }
  100% { transform: scale(1.1); }
}

.restart-btn {
  background: linear-gradient(45deg, #FF6B6B, #EE5A24);
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 25px;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  box-shadow: 0 4px 15px rgba(0,0,0,0.3);
  margin-right: 10px;
}

.restart-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.4);
}
</style>
