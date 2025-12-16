<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { AfterimagePass } from 'three/addons/postprocessing/AfterimagePass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

const sceneContainer = ref(null);
const timeSinceLastBlock = ref('0s');
const difficulty = ref(100);
const isMining = ref(false);
const selectedOption = ref(null);
const showResultOverlay = ref(false);
const showBettingModal = ref(false);
const betAmount = ref(1000);
const pendingOption = ref(null);
const resultData = ref({
  bet: '',
  amount: 0,
  prevDiff: 0,
  newDiff: 0,
  won: false
});

const handleOptionClick = (type) => {
  if (isMining.value) return;
  pendingOption.value = type;
  showBettingModal.value = true;
};

const submitBet = () => {
  if (betAmount.value < 1000) {
    alert('Minimum bet is 1000 sats');
    return;
  }
  
  showBettingModal.value = false;
  isMining.value = true;
  selectedOption.value = pendingOption.value;

  // Simulate waiting for result
  setTimeout(() => {
    // Randomly determine the next difficulty change
    // 50% chance to increase, 50% to decrease
    let change = Math.random() > 0.5 ? 10 : -10;
    
    // Boundary check: If difficulty is 0, force increase
    if (difficulty.value === 0 && change < 0) {
        change = 10;
    }

    const newDiff = difficulty.value + change;
    const won = (change > 0 && selectedOption.value === 'increase') || (change < 0 && selectedOption.value === 'decrease');

    resultData.value = {
      bet: selectedOption.value === 'increase' ? 'Long (INCREASE)' : 'Short (DECREASE)',
      amount: betAmount.value,
      prevDiff: difficulty.value,
      newDiff: newDiff,
      won: won
    };
    
    showResultOverlay.value = true;
  }, 5000);
};

const cancelBet = () => {
  showBettingModal.value = false;
  pendingOption.value = null;
};

const confirmResult = () => {
  showResultOverlay.value = false;
  
  // Apply the calculated difficulty
  difficulty.value = resultData.value.newDiff;

  // Execute 3D transition
  transitionToNextBlock();

  // Reset UI state
  selectedOption.value = null;
  isMining.value = false;
};

const transitionToNextBlock = () => {
  // 1. Turn current cube into a "closed" background cube
  if (cube) {
    // Change material to dark grey wireframe
    // We need to replace the LineSegments with a Mesh if we want it to look exactly like bg cubes
    // Or just change the material of the LineSegments. 
    cube.material = new THREE.LineBasicMaterial({ 
      color: 0x333333, 
      transparent: true, 
      opacity: 0.3 
    });
    
    // Add to backgroundCubes list so it rotates slowly
    backgroundCubes.push(cube);
  }

  // 2. Create NEW active cube
  const geometry = new THREE.BoxGeometry(2.5, 2.5, 2.5);
  const edges = new THREE.EdgesGeometry(geometry);
  const material = new THREE.LineBasicMaterial({ color: 0xffffff }); 
  const newCube = new THREE.LineSegments(edges, material);
  
  // Position it to the right of the old cube
  const oldPos = cube ? cube.position : new THREE.Vector3(0,0,0);
  // Move right by 8 units, and maybe slight random Y/Z variation
  newCube.position.set(
    oldPos.x + 8, 
    (Math.random() - 0.5) * 5, 
    (Math.random() - 0.5) * 5
  );
  
  scene.add(newCube);
  
  // 3. Update Chain Line
  // We need to add the new point to the line
  if (chainLine) {
    // Get current points
    const points = [];
    const positionAttribute = chainLine.geometry.getAttribute('position');
    for (let i = 0; i < positionAttribute.count; i++) {
      points.push(new THREE.Vector3().fromBufferAttribute(positionAttribute, i));
    }
    // Add new point
    points.push(newCube.position);
    // Update geometry
    chainLine.geometry.setFromPoints(points);
  }

  // 4. Move Camera Target
  targetCameraPos.copy(newCube.position);
  targetCameraPos.z += 5; // Keep distance
  // Adjust X slightly to keep cube on the left side of screen
  const aspect = window.innerWidth / window.innerHeight;
  const offset = 2.5 * (aspect > 1 ? 1 : 0.5);
  targetCameraPos.x += offset;

  // Update reference
  cube = newCube;
  
  // Reset timer
  startTime = Date.now(); 
  timeSinceLastBlock.value = '0s';
};

// Mock data updates
let startTime = Date.now();
setInterval(() => {
  const diff = Math.floor((Date.now() - startTime) / 1000);
  timeSinceLastBlock.value = `${diff}s`;
}, 1000);

let renderer, scene, camera, cube, particles, composer;
let backgroundCubes = [];
let chainLine;
let animationId;
let targetCameraPos = new THREE.Vector3(0, 0, 5);

const initThreeJS = () => {
  // Scene setup
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x050505);
  scene.fog = new THREE.FogExp2(0x000000, 0.03);

  // Camera
  const width = window.innerWidth;
  const height = window.innerHeight;
  camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
  camera.position.z = 5;

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(window.devicePixelRatio);
  sceneContainer.value.appendChild(renderer.domElement);

  // --- Cube Setup ---
  // Using EdgesGeometry for clean lines
  const geometry = new THREE.BoxGeometry(2.5, 2.5, 2.5);
  const edges = new THREE.EdgesGeometry(geometry);
  // LineBasicMaterial doesn't support thickness on all platforms well, 
  // but we use color white as requested. 
  // To simulate "thickness" and "glow", we will use Bloom post-processing.
  const material = new THREE.LineBasicMaterial({ color: 0xffffff }); 
  cube = new THREE.LineSegments(edges, material);
  
  // Position cube to the left side of the screen
  // We need to calculate roughly where "left" is based on aspect ratio
  const aspect = width / height;
  cube.position.x = -2.5 * (aspect > 1 ? 1 : 0.5); 
  
  scene.add(cube);

  // --- Particles Setup ---
  const particlesGeometry = new THREE.BufferGeometry();
  const particlesCount = 2000; // Increased count
  const posArray = new Float32Array(particlesCount * 3);

  for(let i = 0; i < particlesCount * 3; i++) {
    // Spread particles across a wider area to cover the whole screen
    posArray[i] = (Math.random() - 0.5) * 30; 
  }

  particlesGeometry.setAttribute('position', new THREE.BufferAttribute(posArray, 3));
  const particlesMaterial = new THREE.PointsMaterial({
    size: 0.03,
    color: 0x42b883, // Vue Green tint for particles
    transparent: true,
    opacity: 0.6,
    blending: THREE.AdditiveBlending
  });

  particles = new THREE.Points(particlesGeometry, particlesMaterial);
  scene.add(particles);

  // --- Background Cubes (Closed Blocks) ---
  const bgCubeGeometry = new THREE.BoxGeometry(1, 1, 1);
  const bgCubeMaterial = new THREE.MeshBasicMaterial({ 
    color: 0x666666, 
    wireframe: true,
    transparent: true,
    opacity: 0.3
  });
  
  const bgCount = 20;
  const bgPoints = [];

  for (let i = 0; i < bgCount; i++) {
    const bgCube = new THREE.Mesh(bgCubeGeometry, bgCubeMaterial);
    
    // Distribute evenly across the width (x), with random height (y) and depth (z)
    // This creates a "chain" that spans across the background
    const x = -40 + (i / bgCount) * 80 + (Math.random() - 0.5) * 5;
    const y = (Math.random() - 0.5) * 30;
    const z = -15 - Math.random() * 20; // Behind the main scene

    bgCube.position.set(x, y, z);
    
    bgCube.rotation.x = Math.random() * Math.PI;
    bgCube.rotation.y = Math.random() * Math.PI;
    
    // Random scale
    const scale = Math.random() * 1.5 + 0.5;
    bgCube.scale.set(scale, scale, scale);
    
    scene.add(bgCube);
    backgroundCubes.push(bgCube);
    bgPoints.push(new THREE.Vector3(x, y, z));
  }

  // --- Chain Line ---
  // Connect the cubes with a line
  const lineGeometry = new THREE.BufferGeometry().setFromPoints(bgPoints);
  const lineMaterial = new THREE.LineBasicMaterial({ 
    color: 0x888888, 
    transparent: true, 
    opacity: 0.6 
  });
  chainLine = new THREE.Line(lineGeometry, lineMaterial);
  scene.add(chainLine);

  // Initialize target camera position
  targetCameraPos.copy(camera.position);

  // --- Post Processing ---
  composer = new EffectComposer(renderer);
  const renderPass = new RenderPass(scene, camera);
  composer.addPass(renderPass);

  // 1. Afterimage (Trails)
  const afterimagePass = new AfterimagePass();
  afterimagePass.uniforms['damp'].value = 0.7; // Lower value = darker/shorter trails
  composer.addPass(afterimagePass);

  // 2. Bloom (Glow/Thickness feel)
  const bloomPass = new UnrealBloomPass(new THREE.Vector2(width, height), 1.5, 0.4, 0.85);
  bloomPass.threshold = 0;
  bloomPass.strength = 1.2; // Intensity of glow
  bloomPass.radius = 0.5;
  composer.addPass(bloomPass);

  // Handle resize
  window.addEventListener('resize', onWindowResize);
};

const onWindowResize = () => {
  if (!sceneContainer.value) return;
  const width = window.innerWidth;
  const height = window.innerHeight;
  
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
  composer.setSize(width, height);

  // Re-adjust cube position on resize
  const aspect = width / height;
  if (cube) {
     cube.position.x = -2.5 * (aspect > 1 ? 1 : 0.5);
  }
};

const animate = () => {
  animationId = requestAnimationFrame(animate);

  // Rotate cube
  if (cube) {
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.015;
    cube.rotation.z += 0.005;
  }

  // Rotate particles
  if (particles) {
    particles.rotation.y += 0.001;
    particles.rotation.x += 0.0005;
  }

  // Rotate background cubes
  backgroundCubes.forEach(bgCube => {
    bgCube.rotation.x += 0.002;
    bgCube.rotation.y += 0.003;
  });

  // Move camera smoothly
  if (camera) {
    camera.position.lerp(targetCameraPos, 0.02);
  }

  // Use composer for rendering instead of renderer
  composer.render();
};

onMounted(() => {
  initThreeJS();
  animate();
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize);
  cancelAnimationFrame(animationId);
  if (renderer) {
    renderer.dispose();
  }
});

</script>

<template>
  <div class="app-container">
    <!-- Navigation Bar -->
    <nav class="navbar">
      <div class="nav-left">
        <span class="brand-name">DiffHedge</span>
      </div>
      <div class="nav-right">
        <div class="user-icon">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
            <circle cx="12" cy="7" r="4"></circle>
          </svg>
        </div>
      </div>
    </nav>

    <!-- Background Canvas -->
    <div class="canvas-container" ref="sceneContainer"></div>

    <!-- Content Overlay -->
    <div class="content-overlay">
      <div class="left-spacer"></div>
      <div class="right-panel">
        <div class="info-card">
          <div class="card-header">
            <div class="status-dot"></div>
            <h2>BLOCK STATUS</h2>
          </div>
          
          <div class="card-body">
            <div class="info-row">
              <span class="label">DIFFICULTY</span>
              <span class="value glow-text">{{ difficulty }}</span>
            </div>
            <div class="info-row">
              <span class="label">PREV DIFFICULTY</span>
              <span class="value">90</span>
            </div>
            <div class="info-row">
              <span class="label">LAST BLOCK</span>
              <span class="value time-value">{{ timeSinceLastBlock }}</span>
            </div>
          </div>

          <div class="card-footer">
            <div class="progress-bar">
              <div class="progress-fill"></div>
            </div>
            <span class="system-status">SYSTEM ONLINE</span>
          </div>
        </div>

        <div class="control-buttons">
          <template v-if="!selectedOption">
            <button class="btn btn-increase" @click="handleOptionClick('increase')">
              <span class="btn-icon">▲</span> INCREASE
            </button>
            <button class="btn btn-decrease" @click="handleOptionClick('decrease')">
              <span class="btn-icon">▼</span> DECREASE
            </button>
          </template>
          <template v-else>
            <button 
              class="btn btn-selected" 
              :class="selectedOption === 'increase' ? 'btn-increase' : 'btn-decrease'"
              disabled
            >
              <span class="btn-icon">{{ selectedOption === 'increase' ? '▲' : '▼' }}</span> 
              {{ selectedOption === 'increase' ? 'INCREASE' : 'DECREASE' }}
            </button>
          </template>
        </div>
      </div>
    </div>

    <!-- Betting Modal -->
    <div v-if="showBettingModal" class="overlay">
      <div class="overlay-content betting-modal">
        <h3 class="modal-title">Place Your Bet</h3>
        <div class="bet-info">
          <span class="bet-direction" :class="pendingOption">
            {{ pendingOption === 'increase' ? '▲ Long' : '▼ Short' }}
          </span>
        </div>
        
        <div class="input-group">
          <label>Amount (Sats)</label>
          <input type="number" v-model="betAmount" min="1000" step="100" class="bet-input">
          <span class="min-hint">Min: 1000 sats</span>
        </div>

        <div class="modal-actions">
          <button class="btn btn-cancel" @click="cancelBet">Cancel</button>
          <button class="btn btn-confirm" @click="submitBet">Confirm Bet</button>
        </div>
      </div>
    </div>

    <!-- Result Overlay -->
    <div v-if="showResultOverlay" class="overlay">
      <div class="overlay-content" :class="{ 'won': resultData.won, 'lost': !resultData.won }">
        <h3 class="result-title">{{ resultData.won ? 'You Won!' : 'You Lost...' }}</h3>
        
        <div class="result-details">
          <div class="result-item">
            <span class="result-label">Your Bet</span>
            <span class="result-value">{{ resultData.bet }}</span>
          </div>
          <div class="result-item">
            <span class="result-label">Amount</span>
            <span class="result-value">{{ resultData.amount }} sats</span>
          </div>
          <div class="result-item">
            <span class="result-label">Prev Difficulty</span>
            <span class="result-value">{{ resultData.prevDiff }}</span>
          </div>
          <div class="result-item">
            <span class="result-label">Current Difficulty</span>
            <span class="result-value highlight">{{ resultData.newDiff }}</span>
          </div>
        </div>

        <button class="btn btn-confirm" @click="confirmResult">Confirm</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* --- Betting Modal Styles --- */
.betting-modal {
  min-width: 350px;
}

.modal-title {
  font-size: 1.8rem;
  margin-bottom: 20px;
  color: #fff;
}

.bet-info {
  margin-bottom: 20px;
}

.bet-direction {
  font-size: 1.5rem;
  font-weight: bold;
  padding: 5px 15px;
  border-radius: 8px;
}

.bet-direction.increase {
  color: #42b883;
  background: rgba(66, 184, 131, 0.1);
  border: 1px solid rgba(66, 184, 131, 0.3);
}

.bet-direction.decrease {
  color: #eb4d4b;
  background: rgba(235, 77, 75, 0.1);
  border: 1px solid rgba(235, 77, 75, 0.3);
}

.input-group {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  margin-bottom: 30px;
  width: 100%;
}

.input-group label {
  color: #aaa;
  margin-bottom: 8px;
  font-size: 0.9rem;
}

.bet-input {
  width: 100%;
  padding: 12px;
  background: rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  color: #fff;
  font-size: 1.2rem;
  font-family: 'Courier New', monospace;
  outline: none;
  transition: border-color 0.3s;
}

.bet-input:focus {
  border-color: #42b883;
}

.min-hint {
  font-size: 0.8rem;
  color: #666;
  margin-top: 5px;
}

.modal-actions {
  display: flex;
  gap: 15px;
  width: 100%;
}

.btn-cancel {
  background: transparent;
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: #aaa;
  padding: 12px 20px;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s;
  flex: 1;
}

.btn-cancel:hover {
  background: rgba(255, 255, 255, 0.1);
  color: #fff;
}

.btn:disabled {
  opacity: 1; /* Keep opacity for selected state */
  cursor: default;
  transform: none !important;
  box-shadow: 0 0 20px rgba(255, 255, 255, 0.2) !important;
}

.btn-selected {
  width: 100%;
  font-size: 1.1rem;
  letter-spacing: 2px;
}

/* --- Overlay --- */
.overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(5px);
  z-index: 100;
  display: flex;
  justify-content: center;
  align-items: center;
}

.overlay-content {
  background: rgba(30, 30, 30, 0.95);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 20px;
  padding: 40px;
  text-align: center;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.8);
  animation: popIn 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  min-width: 400px;
}

.overlay-content.won {
  border-color: #42b883;
  box-shadow: 0 0 30px rgba(66, 184, 131, 0.2);
}

.overlay-content.lost {
  border-color: #eb4d4b;
  box-shadow: 0 0 30px rgba(235, 77, 75, 0.2);
}

.result-title {
  font-size: 2.5rem;
  margin-bottom: 30px;
  color: #fff;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

.won .result-title {
  color: #42b883;
  text-shadow: 0 0 15px rgba(66, 184, 131, 0.5);
}

.lost .result-title {
  color: #eb4d4b;
  text-shadow: 0 0 15px rgba(235, 77, 75, 0.5);
}

.result-details {
  margin-bottom: 30px;
  background: rgba(0, 0, 0, 0.3);
  padding: 20px;
  border-radius: 12px;
}

.result-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  padding-bottom: 15px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.result-item:last-child {
  margin-bottom: 0;
  padding-bottom: 0;
  border-bottom: none;
}

.result-label {
  color: #aaa;
  font-size: 1rem;
}

.result-value {
  color: #fff;
  font-size: 1.2rem;
  font-weight: bold;
  font-family: 'Courier New', monospace;
}

.result-value.highlight {
  font-size: 1.4rem;
  color: #fff;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

.btn-confirm {
  background: #42b883;
  color: #fff;
  padding: 12px 40px;
  font-size: 1.2rem;
  border-radius: 8px;
  border: none;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-confirm:hover {
  background: #3aa876;
  transform: scale(1.05);
  box-shadow: 0 0 20px rgba(66, 184, 131, 0.4);
}

@keyframes popIn {
  0% { opacity: 0; transform: scale(0.8); }
  100% { opacity: 1; transform: scale(1); }
}

.app-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background-color: #000;
}

.canvas-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
}

/* --- Navbar --- */
.navbar {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 80px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 40px;
  z-index: 10; /* Above canvas and content */
  background: linear-gradient(to bottom, rgba(0,0,0,0.8) 0%, rgba(0,0,0,0) 100%);
  pointer-events: none; /* Let clicks pass through empty areas */
}

.nav-left, .nav-right {
  pointer-events: auto; /* Re-enable clicks for buttons */
}

.brand-name {
  font-family: 'Inter', sans-serif;
  font-size: 1.5rem;
  font-weight: 700;
  color: #fff;
  letter-spacing: 1px;
  text-transform: uppercase;
  background: linear-gradient(45deg, #fff, #42b883);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.user-icon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  transition: all 0.3s ease;
  color: #fff;
}

.user-icon:hover {
  background: rgba(66, 184, 131, 0.2);
  border-color: #42b883;
  transform: scale(1.1);
  box-shadow: 0 0 15px rgba(66, 184, 131, 0.3);
}

.content-overlay {
  position: relative;
  z-index: 2;
  display: flex;
  width: 100%;
  height: 100%;
}

.left-spacer {
  flex: 1;
  /* Keeps the left side clear for the cube */
}

.right-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 20px;
  /* Removed background, now transparent */
}

/* --- Cool Card Design --- */
.info-card {
  width: 380px;
  background: rgba(71, 71, 71, 0.65);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 24px;
  padding: 30px;
  box-shadow: 
    0 20px 50px rgba(0, 0, 0, 0.5),
    0 0 0 1px rgba(255, 255, 255, 0.05) inset;
  color: #fff;
  font-family: 'Inter', sans-serif;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.info-card:hover {
  transform: translateY(-5px);
  box-shadow: 
    0 30px 60px rgba(0, 0, 0, 0.6),
    0 0 0 1px rgba(66, 184, 131, 0.3) inset;
  border-color: rgba(66, 184, 131, 0.4);
}

.card-header {
  display: flex;
  align-items: center;
  margin-bottom: 30px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  padding-bottom: 15px;
}

.status-dot {
  width: 10px;
  height: 10px;
  background-color: #42b883;
  border-radius: 50%;
  margin-right: 12px;
  box-shadow: 0 0 10px #42b883;
  animation: pulse 2s infinite;
}

h2 {
  font-size: 1.1rem;
  letter-spacing: 2px;
  font-weight: 600;
  color: #e0e0e0;
  margin: 0;
}

.info-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.label {
  font-size: 0.9rem;
  color: #888;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.value {
  font-size: 1.4rem;
  font-weight: 700;
  font-family: 'Courier New', monospace;
}

.glow-text {
  color: #fff;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

.time-value {
  color: #42b883;
}

.card-footer {
  margin-top: 30px;
}

.progress-bar {
  width: 100%;
  height: 4px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 2px;
  margin-bottom: 10px;
  overflow: hidden;
}

.progress-fill {
  width: 60%;
  height: 100%;
  background: #42b883;
  box-shadow: 0 0 10px #42b883;
  animation: loading 3s ease-in-out infinite;
}

.system-status {
  font-size: 0.7rem;
  color: #555;
  letter-spacing: 1px;
  display: block;
  text-align: right;
}

@keyframes pulse {
  0% { opacity: 1; }
  50% { opacity: 0.5; }
  100% { opacity: 1; }
}

@keyframes loading {
  0% { width: 0%; transform: translateX(-100%); }
  50% { width: 100%; transform: translateX(0); }
  100% { width: 0%; transform: translateX(100%); }
}

/* --- Control Buttons --- */
.control-buttons {
  display: flex;
  gap: 20px;
  width: 380px; /* Match card width */
}

.btn {
  flex: 1;
  padding: 15px;
  border: none;
  border-radius: 12px;
  font-family: 'Inter', sans-serif;
  font-weight: 600;
  font-size: 0.9rem;
  letter-spacing: 1px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 8px;
  backdrop-filter: blur(10px);
  color: #fff;
  text-transform: uppercase;
}

.btn-icon {
  font-size: 1.2rem;
}

.btn-increase {
  background: rgba(66, 184, 131, 0.2);
  border: 1px solid rgba(66, 184, 131, 0.4);
  box-shadow: 0 0 15px rgba(66, 184, 131, 0.1);
}

.btn-increase:hover {
  background: rgba(66, 184, 131, 0.4);
  box-shadow: 0 0 25px rgba(66, 184, 131, 0.3);
  transform: translateY(-2px);
}

.btn-increase:active {
  transform: translateY(0);
}

.btn-decrease {
  background: rgba(235, 77, 75, 0.2);
  border: 1px solid rgba(235, 77, 75, 0.4);
  box-shadow: 0 0 15px rgba(235, 77, 75, 0.1);
}

.btn-decrease:hover {
  background: rgba(235, 77, 75, 0.4);
  box-shadow: 0 0 25px rgba(235, 77, 75, 0.3);
  transform: translateY(-2px);
}

.btn-decrease:active {
  transform: translateY(0);
}
</style>
