<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Fruit Ninja - Hand Controlled</title>
  <style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: sans-serif;
  background: #000;
  color: white;
  text-align: center;
  overflow: hidden;
}

h1 {
  margin-top: 10px;
}

#webcam, #gameCanvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  z-index: 0;
}

#gameCanvas {
  z-index: 1;
}

#score {
  position: absolute;
  top: 20px;
  right: 40px;
  font-size: 2rem;
  color: #fff;
  z-index: 2;
  background: rgba(0,0,0,0.5);
  padding: 8px 20px;
  border-radius: 10px;
}

#gameOver {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0,0,0,0.85);
  padding: 40px 60px;
  border-radius: 20px;
  z-index: 3;
  display: flex;
  flex-direction: column;
  align-items: center;
}

#gameOverText {
  font-size: 2.5rem;
  color: #ff5252;
  margin-bottom: 20px;
}

#restartBtn {
  font-size: 1.2rem;
  padding: 10px 30px;
  border: none;
  border-radius: 8px;
  background: #ff9800;
  color: #fff;
  cursor: pointer;
  transition: background 0.2s;
}
#restartBtn:hover {
  background: #ffb74d;
}
  </style>
</head>
<body>
  <h1>🖐 Slice the Fruits!</h1>
  <div id="score">Score: 0</div>
  <div id="gameOver" style="display:none;">
    <div id="gameOverText">Game Over!</div>
    <button id="restartBtn">Restart</button>
  </div>
  <video id="webcam" autoplay muted></video>
  <canvas id="gameCanvas"></canvas>

  <!-- Load MediaPipe Hands -->
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/core@0.1"></script> 
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script> 
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script> 

  <script>
// Setup canvas and context
const video = document.getElementById('webcam');
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

let width = window.innerWidth;
let height = window.innerHeight;
canvas.width = width;
canvas.height = height;

// Fruit array
let fruits = [];

// Blade position
let bladeX = 0, bladeY = 0;
let isBladeActive = false;

// Load fruit images
const fruitImages = [
  { src: 'https://i.imgur.com/ZBQ9fE6.png',  name: 'apple' },
  { src: 'https://i.imgur.com/LuT2JFw.png',  name: 'orange' },
  { src: 'https://i.imgur.com/7kxBEVt.png',  name: 'banana' },
];

let score = 0;
let gameOver = false;
let imageLoadCount = 0;
const totalImages = fruitImages.length;

// Responsive canvas
function resizeCanvas() {
  width = window.innerWidth;
  height = window.innerHeight;
  canvas.width = width;
  canvas.height = height;
}
window.addEventListener('resize', resizeCanvas);
resizeCanvas();

// Preload images
const loadedFruitImages = {};
fruitImages.forEach(f => {
  const img = new Image();
  img.src = f.src;
  img.onload = () => {
    imageLoadCount++;
    if (imageLoadCount === totalImages) startGame();
  };
  loadedFruitImages[f.name] = img;
});

function startGame() {
  score = 0;
  gameOver = false;
  fruits = [];
  document.getElementById('score').textContent = 'Score: 0';
  document.getElementById('gameOver').style.display = 'none';
  if (spawnInterval) clearInterval(spawnInterval);
  spawnInterval = setInterval(() => {
    if (!gameOver) spawnFruit();
  }, 1000);
  requestAnimationFrame(gameLoop);
}

let spawnInterval;

function spawnFruit() {
  const fruitNames = Object.keys(loadedFruitImages);
  const name = fruitNames[Math.floor(Math.random() * fruitNames.length)];
  const img = loadedFruitImages[name];
  const size = Math.random() * 50 + 40;
  fruits.push({
    x: Math.random() * (width - size),
    y: height,
    vx: (Math.random() - 0.5) * 2,
    vy: -(Math.random() * 3 + 2),
    size,
    img,
    sliced: false,
    angle: 0,
    rotationSpeed: Math.random() * 0.1 - 0.05
  });
}

// Sliced pieces for animation
let slicedPieces = [];

function gameLoop() {
  ctx.clearRect(0, 0, width, height);

  // Update and draw fruits
  for (let i = fruits.length - 1; i >= 0; i--) {
    const f = fruits[i];
    if (f.sliced) continue;

    f.x += f.vx;
    f.y += f.vy;
    f.angle += f.rotationSpeed;

    ctx.save();
    ctx.translate(f.x + f.size / 2, f.y + f.size / 2);
    ctx.rotate(f.angle);
    ctx.drawImage(f.img, -f.size / 2, -f.size / 2, f.size, f.size);
    ctx.restore();

    // Check if out of bounds (game over if not sliced)
    if (f.y > height) {
      if (!f.sliced) {
        endGame();
        return;
      }
      fruits.splice(i, 1);
    }

    // Check collision with blade
    if (isBladeActive && !f.sliced) {
      const dx = f.x + f.size / 2 - bladeX;
      const dy = f.y + f.size / 2 - bladeY;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < f.size * 0.7) {
        f.sliced = true;
        score++;
        document.getElementById('score').textContent = 'Score: ' + score;
        sliceEffect(f.x, f.y, f.size, f.img, f.vx, f.vy);
        fruits.splice(i, 1);
      }
    }
  }

  // Animate sliced pieces
  for (let i = slicedPieces.length - 1; i >= 0; i--) {
    const p = slicedPieces[i];
    p.x += p.vx;
    p.y += p.vy;
    p.vy += 0.2; // gravity
    p.angle += p.rotationSpeed;
    ctx.save();
    ctx.translate(p.x + p.size / 2, p.y + p.size / 2);
    ctx.rotate(p.angle);
    ctx.drawImage(p.img, -p.size / 2, -p.size / 2, p.size, p.size);
    ctx.restore();
    if (p.y > height) slicedPieces.splice(i, 1);
  }

  // Draw blade
  if (isBladeActive) {
    ctx.beginPath();
    ctx.arc(bladeX, bladeY, 10, 0, Math.PI * 2);
    ctx.fillStyle = 'red';
    ctx.fill();
  }

  if (!gameOver) requestAnimationFrame(gameLoop);
}

function endGame() {
  gameOver = true;
  document.getElementById('gameOver').style.display = 'flex';
  document.getElementById('gameOverText').textContent = 'Game Over!\nScore: ' + score;
  clearInterval(spawnInterval);
}

document.getElementById('restartBtn').onclick = () => {
  startGame();
};

// Slicing effect with animation
function sliceEffect(x, y, size, img, vx, vy) {
  for (let i = 0; i < 2; i++) {
    slicedPieces.push({
      x: x,
      y: y,
      vx: vx + (i === 0 ? -2 : 2),
      vy: vy - 2,
      size: size / 1.2,
      img: img,
      angle: 0,
      rotationSpeed: (Math.random() - 0.5) * 0.2
    });
  }
}

// ----------------------------
// MediaPipe Setup
// ----------------------------

const hands = new Hands({ 
  locateFile: file => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` 
});
hands.setOptions({
  maxNumHands: 1,
  modelComplexity: 1,
  minDetectionConfidence: 0.5,
  minTrackingConfidence: 0.5
});

hands.onResults(onResults);

const camera = new Camera(video, {
  onFrame: async () => {
    await hands.send({ image: video });
  },
  width: width,
  height: height
});
camera.start();

function onResults(results) {
  if (results.multiHandLandmarks.length > 0) {
    const indexTip = results.multiHandLandmarks[0][8]; // Index fingertip
    bladeX = indexTip.x * width;
    bladeY = indexTip.y * height;
    isBladeActive = true;
  } else {
    isBladeActive = false;
  }
}
  </script>
</body>
</html>
