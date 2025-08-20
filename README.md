# Pazo - A 2D Adventure Game

Welcome to **Pazo**, a charming 2D platformer adventure game built with Vue.js and TypeScript!

## 🎮 Game Overview

**Pazo** is a quest-based platformer where you control **Pazo**, a brave little bird hero on a mission to find clues, collect coins, and ultimately escape through the exit door.

## 🎯 Game Objectives

1. **Find all 3 clues** scattered throughout the level
2. **Collect coins** along the way (optional but fun!)
3. **Discover the key** (only appears after finding all clues)
4. **Reach the exit door** to complete your quest

## 🕹️ Controls

- **W** or **↑** or **Space** - Jump (unlimited soaring!)
- **A** or **←** - Move left
- **D** or **→** - Move right
- **Reset Button** - Start over

## 🎨 Game Features

- **Smooth 2D platformer physics** with gravity and unlimited jumping
- **Bird-like character** with flapping wings animation
- **Collision detection** for platforms and collectibles
- **Progressive gameplay** - the key only appears after finding all clues
- **Beautiful gradient backgrounds** and smooth animations
- **Responsive controls** with multiple input options
- **Game state tracking** showing your progress
- **Fullscreen experience** with floating UI elements

## 🏗️ Technical Details

- Built with **Vue 3** and **TypeScript**
- **Canvas-based rendering** for smooth 2D graphics
- **Game loop** with delta-time based updates
- **Collision detection system** for all game objects
- **Responsive design** with modern CSS
- **Sen font** from Google Fonts throughout

## 🚀 Getting Started

### Prerequisites
- Node.js (version 20.19.0 or higher recommended)
- npm or yarn

### Installation

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd pazo
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Run the development server**
   ```bash
   npm run dev
   ```

4. **Open your browser** and navigate to `http://localhost:5173`

### Building for Production

```bash
npm run build
```

## 🎯 Game Mechanics

### Player (Pazo the Bird)
- **Blue bird character** with cute wings, eyes, beak, and feet
- **Physics-based movement** with realistic gravity
- **Unlimited soaring** - jump as many times as you want to gain height
- **Wing flapping** - wings animate when jumping
- **Platform collision** - can stand on platforms
- **Boundary checking** - stays within the game area

### Game Objects
- **Platforms** (brown) - solid surfaces to jump on
- **Coins** (gold circles) - collectible items
- **Clues** (red squares) - required to progress
- **Key** (gold square with handle) - appears after finding all clues
- **Door** (red/green) - exit point (green when you have the key)

### Level Design
- **Multiple platforms** at different heights
- **Strategic placement** of clues and coins
- **Progressive difficulty** as you move right
- **Logical flow** from start to finish
- **Wide world** (2400px) with camera following

## 🎨 Customization

The game is built to be easily customizable:

- **Game objects** can be modified in the `gameObjects` array
- **Physics values** (gravity, jump force, movement speed) can be adjusted
- **Colors and styling** can be changed in the CSS
- **Level layout** can be redesigned by modifying object positions
- **Kitten SVG** can be replaced with your own design

## 🐛 Troubleshooting

- **Canvas not responding**: Make sure to click on the canvas to give it focus
- **Movement feels sluggish**: Check if your browser supports `requestAnimationFrame`
- **Game won't start**: Ensure all dependencies are installed and the dev server is running

## 🤝 Contributing

Feel free to contribute to this project by:
- Adding new game mechanics
- Improving the graphics
- Adding sound effects
- Creating new levels
- Optimizing performance

## 📝 License

This project is open source and available under the MIT License.

---

**Happy gaming! 🎮✨**

*Help Pazo the Bird find all the clues and escape!*
