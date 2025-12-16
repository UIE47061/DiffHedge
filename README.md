# DiffHedge

DiffHedge is an immersive 3D blockchain visualization and gamified difficulty hedging platform. Built with Vue 3 and Three.js, it offers a futuristic interface to interact with blockchain concepts visually.

![DiffHedge Icon](./public/icon.svg)

## 🌟 Features

- **Immersive 3D Environment**: Real-time rendering of blockchain blocks using Three.js with bloom and afterimage effects.
- **Interactive Mining**: Gamified experience where users can bet on the difficulty adjustment of the next block.
- **Visual Feedback**: Dynamic camera movements, particle effects, and chain visualization that responds to user actions.
- **Modern UI**: Glassmorphism design with responsive interactive elements.

## 🛠 Tech Stack

- **Frontend Framework**: Vue 3 (Composition API)
- **Build Tool**: Vite
- **3D Graphics**: Three.js
- **Post-Processing**: Three.js EffectComposer (UnrealBloom, Afterimage)
- **Styling**: CSS3 (Variables, Flexbox, Backdrop-filter)

## 🚀 Project Setup

### Prerequisites

- Node.js (v16.0.0 or higher)
- npm or yarn

### Installation

```sh
# Install dependencies
npm install
```

### Development

```sh
# Start the development server
npm run dev
```

### Production Build

```sh
# Build for production
npm run build

# Preview production build
npm run preview
```

---

## 📡 API Documentation (Mock)

Although DiffHedge currently runs with client-side mock data, the following API specification outlines how the frontend would communicate with a real backend service.

### Base URL
`https://api.diffhedge.com/v1`

### 1. Get System Status
Check if the mining system is online.

- **Endpoint**: `GET /status`
- **Response**:
  ```json
  {
    "status": "online",
    "serverTime": "2025-12-16T10:00:00Z",
    "activeMiners": 1240
  }
  ```

### 2. Get Current Block Information
Retrieve details about the current active block.

- **Endpoint**: `GET /block/current`
- **Response**:
  ```json
  {
    "blockHeight": 84021,
    "difficulty": 100,
    "prevDifficulty": 90,
    "timestamp": 1734345600,
    "hash": "0x7f83b1..."
  }
  ```

### 3. Place Difficulty Bet (Mine)
Submit a prediction for the next block's difficulty adjustment.

- **Endpoint**: `POST /game/bet`
- **Headers**: `Authorization: Bearer <token>`
- **Body**:
  ```json
  {
    "direction": "increase", // or "decrease"
    "amount": 10 // Optional: bet amount
  }
  ```
- **Response (Success)**:
  ```json
  {
    "success": true,
    "won": true,
    "result": {
      "bet": "Long (INCREASE)",
      "prevDifficulty": 100,
      "newDifficulty": 110,
      "payout": 20
    },
    "nextBlock": {
      "hash": "0x8a92c3...",
      "difficulty": 110
    }
  }
  ```
- **Response (Error)**:
  ```json
  {
    "error": "Mining in progress",
    "code": "MINING_LOCKED"
  }
  ```

### 4. Get User History
Retrieve the betting history for the current user.

- **Endpoint**: `GET /user/history`
- **Query Params**:
  - `limit`: number (default 10)
  - `offset`: number (default 0)
- **Response**:
  ```json
  {
    "totalGames": 50,
    "winRate": 0.54,
    "history": [
      {
        "id": "game_123",
        "timestamp": 1734345000,
        "bet": "increase",
        "result": "won",
        "diffChange": 10
      }
    ]
  }
  ```

## 📂 Project Structure

```
DiffHedge/
├── public/             # Static assets (icons, favicon)
├── src/
│   ├── assets/         # CSS and images
│   ├── components/     # Vue components
│   ├── App.vue         # Main application component (3D Logic & UI)
│   └── main.js         # App entry point
├── index.html          # HTML entry point
├── package.json        # Dependencies and scripts
└── vite.config.js      # Vite configuration
```
