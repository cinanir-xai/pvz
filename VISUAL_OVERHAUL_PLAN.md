# Visual Overhaul Plan - Plants vs Zombies v1.1

## Overview
Transform the basic shape-based game into a visually stunning, full-screen experience that captures the charm and art style of the original Plants vs. Zombies. All graphics will be procedurally drawn using the Canvas API - no external image assets.

---

## 1. Full-Screen Responsive Layout

### Layout Philosophy
The game must fill the entire browser window while maintaining the 5:9 aspect ratio of the lawn grid. We'll use dynamic scaling to ensure the game looks good on all screen sizes.

### Viewport Configuration
```javascript
const LAYOUT_CONFIG = {
  // Base design resolution (internal coordinate system)
  DESIGN_WIDTH: 1024,
  DESIGN_HEIGHT: 600,
  
  // Margins for UI elements
  UI_TOP_MARGIN: 80,      // Space for seed packets
  UI_BOTTOM_MARGIN: 20,   // Small bottom padding
  UI_LEFT_MARGIN: 180,    // Space for house/lawn mowers
  UI_RIGHT_MARGIN: 100,   // Space for zombies to spawn
  
  // The lawn grid (5 lanes × 9 columns)
  LANES: 5,
  COLUMNS: 9,
  CELL_WIDTH: 81,   // (1024 - 180 - 100) / 9 ≈ 82
  CELL_HEIGHT: 100, // (600 - 80 - 20) / 5 = 100
};
```

### Responsive Scaling Strategy
```javascript
function calculateScale() {
  const windowWidth = window.innerWidth;
  const windowHeight = window.innerHeight;
  
  // Calculate scale to fit while maintaining aspect ratio
  const scaleX = windowWidth / DESIGN_WIDTH;
  const scaleY = windowHeight / DESIGN_HEIGHT;
  const scale = Math.min(scaleX, scaleY);
  
  // Center the game in the window
  const offsetX = (windowWidth - DESIGN_WIDTH * scale) / 2;
  const offsetY = (windowHeight - DESIGN_HEIGHT * scale) / 2;
  
  return { scale, offsetX, offsetY };
}
```

### Canvas Setup
```javascript
// Full window canvas
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

// Use ctx.scale() for all drawing
const { scale, offsetX, offsetY } = calculateScale();
ctx.save();
ctx.translate(offsetX, offsetY);
ctx.scale(scale, scale);
// All drawing happens in design coordinates (0-1024, 0-600)
ctx.restore();
```

### Why This Approach?
- **Consistent aspect ratio**: Lawn grid never stretches/distorts
- **Pixel-perfect at any size**: Vector-like scaling via Canvas API
- **Centered gameplay**: Black bars on sides if window is too wide/tall
- **Touch-friendly**: Full-screen immersion

---

## 2. Background & Environment Art

### Sky Gradient
```javascript
const SKY_COLORS = {
  top: '#4fc3f7',      // Light blue
  middle: '#81d4fa',   // Lighter blue
  bottom: '#b3e5fc'    // Very light blue near horizon
};

function drawSky(ctx) {
  const gradient = ctx.createLinearGradient(0, 0, 0, DESIGN_HEIGHT);
  gradient.addColorStop(0, SKY_COLORS.top);
  gradient.addColorStop(0.6, SKY_COLORS.middle);
  gradient.addColorStop(1, SKY_COLORS.bottom);
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, DESIGN_WIDTH, DESIGN_HEIGHT);
}
```

### The House (Detailed)
```javascript
const HOUSE_COLORS = {
  wall: '#8d6e63',      // Brown siding
  wallShadow: '#6d4c41', // Darker brown for depth
  roof: '#5d4037',      // Dark brown roof
  roofHighlight: '#795548',
  window: '#424242',    // Dark window
  windowLight: '#ffeb3b', // Lit from inside
  door: '#4e342e',      // Dark door
  doorKnob: '#ffc107',  // Gold knob
  trim: '#d7ccc8'       // Light trim
};

function drawHouse(ctx) {
  const houseX = 20;
  const houseY = 120;
  const houseWidth = 140;
  const houseHeight = 360;
  
  // Main wall with horizontal siding lines
  ctx.fillStyle = HOUSE_COLORS.wall;
  ctx.fillRect(houseX, houseY, houseWidth, houseHeight);
  
  // Siding lines (horizontal grooves)
  ctx.strokeStyle = HOUSE_COLORS.wallShadow;
  ctx.lineWidth = 1;
  for (let i = 1; i < 12; i++) {
    const y = houseY + i * 30;
    ctx.beginPath();
    ctx.moveTo(houseX, y);
    ctx.lineTo(houseX + houseWidth, y);
    ctx.stroke();
  }
  
  // Roof (triangle)
  ctx.fillStyle = HOUSE_COLORS.roof;
  ctx.beginPath();
  ctx.moveTo(houseX - 10, houseY);
  ctx.lineTo(houseX + houseWidth / 2, houseY - 60);
  ctx.lineTo(houseX + houseWidth + 10, houseY);
  ctx.closePath();
  ctx.fill();
  
  // Roof highlight
  ctx.fillStyle = HOUSE_COLORS.roofHighlight;
  ctx.beginPath();
  ctx.moveTo(houseX + 20, houseY - 10);
  ctx.lineTo(houseX + houseWidth / 2, houseY - 50);
  ctx.lineTo(houseX + houseWidth - 20, houseY - 10);
  ctx.closePath();
  ctx.fill();
  
  // Windows (one per lane)
  for (let lane = 0; lane < 5; lane++) {
    const winY = houseY + 30 + lane * 70;
    
    // Window frame
    ctx.fillStyle = HOUSE_COLORS.trim;
    ctx.fillRect(houseX + 20, winY, 40, 50);
    
    // Window glass (slight gradient for depth)
    const winGrad = ctx.createLinearGradient(houseX + 25, winY + 5, houseX + 55, winY + 45);
    winGrad.addColorStop(0, HOUSE_COLORS.windowLight);
    winGrad.addColorStop(1, HOUSE_COLORS.window);
    ctx.fillStyle = winGrad;
    ctx.fillRect(houseX + 25, winY + 5, 30, 40);
    
    // Window crossbars
    ctx.strokeStyle = HOUSE_COLORS.trim;
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(houseX + 40, winY + 5);
    ctx.lineTo(houseX + 40, winY + 45);
    ctx.moveTo(houseX + 25, winY + 25);
    ctx.lineTo(houseX + 55, winY + 25);
    ctx.stroke();
  }
  
  // Door at bottom
  ctx.fillStyle = HOUSE_COLORS.door;
  ctx.fillRect(houseX + 80, houseY + 280, 50, 80);
  
  // Door panels
  ctx.strokeStyle = HOUSE_COLORS.wallShadow;
  ctx.lineWidth = 2;
  ctx.strokeRect(houseX + 85, houseY + 285, 18, 30);
  ctx.strokeRect(houseX + 107, houseY + 285, 18, 30);
  ctx.strokeRect(houseX + 85, houseY + 320, 18, 35);
  ctx.strokeRect(houseX + 107, houseY + 320, 18, 35);
  
  // Door knob
  ctx.fillStyle = HOUSE_COLORS.doorKnob;
  ctx.beginPath();
  ctx.arc(houseX + 123, houseY + 320, 4, 0, Math.PI * 2);
  ctx.fill();
}
```

### The Lawn (Detailed Grass)
```javascript
const LAWN_COLORS = {
  grassLight: '#8bc34a',
  grassDark: '#689f38',
  grassHighlight: '#aed581',
  grassShadow: '#558b2f',
  dirt: '#5d4037'
};

function drawLawn(ctx) {
  const lawnX = UI_LEFT_MARGIN;
  const lawnY = UI_TOP_MARGIN;
  const lawnWidth = COLUMNS * CELL_WIDTH;
  const lawnHeight = LANES * CELL_HEIGHT;
  
  // Base grass with subtle gradient
  const grassGrad = ctx.createLinearGradient(0, lawnY, 0, lawnY + lawnHeight);
  grassGrad.addColorStop(0, LAWN_COLORS.grassLight);
  grassGrad.addColorStop(1, LAWN_COLORS.grassDark);
  ctx.fillStyle = grassGrad;
  ctx.fillRect(lawnX, lawnY, lawnWidth, lawnHeight);
  
  // Draw individual grass blades for texture
  ctx.save();
  ctx.beginPath();
  ctx.rect(lawnX, lawnY, lawnWidth, lawnHeight);
  ctx.clip();
  
  // Random grass blades
  ctx.strokeStyle = LAWN_COLORS.grassHighlight;
  ctx.lineWidth = 2;
  for (let i = 0; i < 200; i++) {
    const x = lawnX + Math.random() * lawnWidth;
    const y = lawnY + Math.random() * lawnHeight;
    const height = 5 + Math.random() * 10;
    const lean = (Math.random() - 0.5) * 4;
    
    ctx.beginPath();
    ctx.moveTo(x, y);
    ctx.quadraticCurveTo(x + lean, y - height / 2, x + lean * 2, y - height);
    ctx.stroke();
  }
  ctx.restore();
  
  // Grid lines (subtle)
  ctx.strokeStyle = 'rgba(0, 0, 0, 0.08)';
  ctx.lineWidth = 1;
  
  // Vertical grid lines
  for (let i = 0; i <= COLUMNS; i++) {
    const x = lawnX + i * CELL_WIDTH;
    ctx.beginPath();
    ctx.moveTo(x, lawnY);
    ctx.lineTo(x, lawnY + lawnHeight);
    ctx.stroke();
  }
  
  // Horizontal grid lines
  for (let i = 0; i <= LANES; i++) {
    const y = lawnY + i * CELL_HEIGHT;
    ctx.beginPath();
    ctx.moveTo(lawnX, y);
    ctx.lineTo(lawnX + lawnWidth, y);
    ctx.stroke();
  }
  
  // Mower tracks (subtle darker lines along lanes)
  ctx.strokeStyle = 'rgba(0, 0, 0, 0.03)';
  ctx.lineWidth = 3;
  for (let lane = 0; lane < LANES; lane++) {
    const y = lawnY + lane * CELL_HEIGHT + CELL_HEIGHT / 2;
    ctx.beginPath();
    ctx.moveTo(lawnX, y);
    ctx.lineTo(lawnX + lawnWidth, y);
    ctx.stroke();
  }
}
```

### Fence (Background Detail)
```javascript
function drawFence(ctx) {
  const fenceY = UI_TOP_MARGIN - 20;
  const fenceX = UI_LEFT_MARGIN - 50;
  const fenceWidth = COLUMNS * CELL_WIDTH + 100;
  
  // Fence posts
  ctx.fillStyle = '#a1887f';
  const postWidth = 12;
  const postSpacing = 40;
  
  for (let x = fenceX; x < fenceX + fenceWidth; x += postSpacing) {
    // Post
    ctx.fillRect(x, fenceY - 30, postWidth, 50);
    
    // Post cap (pointed)
    ctx.beginPath();
    ctx.moveTo(x - 2, fenceY - 30);
    ctx.lineTo(x + postWidth / 2, fenceY - 45);
    ctx.lineTo(x + postWidth + 2, fenceY - 30);
    ctx.closePath();
    ctx.fill();
  }
  
  // Horizontal rails
  ctx.fillStyle = '#8d6e63';
  ctx.fillRect(fenceX, fenceY - 10, fenceWidth, 8);
  ctx.fillRect(fenceX, fenceY + 10, fenceWidth, 8);
}
```

---

## 3. Peashooter Sprite Design

### Visual Style
The peashooter should look like a small, cute cannon with a leafy base. Think "friendly vegetable soldier" aesthetic.

### Color Palette
```javascript
const PEASHOOTER_COLORS = {
  bodyMain: '#66bb6a',      // Main green
  bodyDark: '#43a047',      // Shadow side
  bodyLight: '#81c784',     // Highlight
  head: '#4caf50',          // Slightly darker head
  mouth: '#2e7d32',         // Dark green inside mouth
  mouthDark: '#1b5e20',     // Deepest shadow
  leaf: '#388e3c',          // Dark green leaves
  leafLight: '#4caf50',     // Leaf highlight
  stem: '#2e7d32',          // Stem color
  eyeWhite: '#ffffff',
  eyePupil: '#000000',
  cheek: '#f8bbd0'          // Subtle pink cheeks
};
```

### Sprite Structure (Procedural Drawing)
```javascript
function drawPeashooter(ctx, x, y, scale = 1, animationFrame = 0) {
  ctx.save();
  ctx.translate(x, y);
  ctx.scale(scale, scale);
  
  // Idle bob animation
  const bob = Math.sin(animationFrame * 0.1) * 2;
  ctx.translate(0, bob);
  
  // Draw from back to front
  drawPeashooterBackLeaves(ctx);
  drawPeashooterStem(ctx);
  drawPeashooterHeadBack(ctx);
  drawPeashooterMouthInside(ctx);
  drawPeashooterHeadFront(ctx);
  drawPeashooterFace(ctx);
  drawPeashooterFrontLeaves(ctx);
  
  ctx.restore();
}
```

### Component Breakdown

#### 1. Stem (Curved neck)
```javascript
function drawPeashooterStem(ctx) {
  ctx.strokeStyle = PEASHOOTER_COLORS.stem;
  ctx.lineWidth = 12;
  ctx.lineCap = 'round';
  
  ctx.beginPath();
  ctx.moveTo(0, 20);  // Base
  ctx.quadraticCurveTo(-5, 0, 0, -25);  // Curved neck
  ctx.stroke();
  
  // Stem highlight
  ctx.strokeStyle = PEASHOOTER_COLORS.bodyLight;
  ctx.lineWidth = 4;
  ctx.beginPath();
  ctx.moveTo(-3, 18);
  ctx.quadraticCurveTo(-7, 0, -3, -23);
  ctx.stroke();
}
```

#### 2. Head (Tube shape)
```javascript
function drawPeashooterHeadBack(ctx) {
  // Back of the cylindrical head
  ctx.fillStyle = PEASHOOTER_COLORS.bodyDark;
  ctx.beginPath();
  ctx.ellipse(0, -35, 18, 22, 0, 0, Math.PI * 2);
  ctx.fill();
}

function drawPeashooterHeadFront(ctx) {
  // Main head gradient for 3D effect
  const headGrad = ctx.createLinearGradient(-15, -50, 15, -20);
  headGrad.addColorStop(0, PEASHOOTER_COLORS.bodyDark);
  headGrad.addColorStop(0.5, PEASHOOTER_COLORS.bodyMain);
  headGrad.addColorStop(1, PEASHOOTER_COLORS.bodyLight);
  
  ctx.fillStyle = headGrad;
  ctx.beginPath();
  ctx.ellipse(0, -35, 20, 25, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Outline
  ctx.strokeStyle = PEASHOOTER_COLORS.bodyDark;
  ctx.lineWidth = 2;
  ctx.stroke();
}
```

#### 3. Mouth (The cannon opening)
```javascript
function drawPeashooterMouthInside(ctx) {
  // Dark interior
  ctx.fillStyle = PEASHOOTER_COLORS.mouthDark;
  ctx.beginPath();
  ctx.ellipse(12, -35, 10, 15, 0, 0, Math.PI * 2);
  ctx.fill();
}

function drawPeashooterMouthLip(ctx) {
  // The lip around the mouth
  ctx.fillStyle = PEASHOOTER_COLORS.head;
  ctx.beginPath();
  ctx.ellipse(12, -35, 12, 17, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Inner hole
  ctx.fillStyle = PEASHOOTER_COLORS.mouth;
  ctx.beginPath();
  ctx.ellipse(14, -35, 8, 12, 0, 0, Math.PI * 2);
  ctx.fill();
}
```

#### 4. Face (Eyes and cheeks)
```javascript
function drawPeashooterFace(ctx) {
  // Left eye
  ctx.fillStyle = PEASHOOTER_COLORS.eyeWhite;
  ctx.beginPath();
  ctx.ellipse(-5, -40, 6, 8, -0.2, 0, Math.PI * 2);
  ctx.fill();
  
  // Right eye
  ctx.beginPath();
  ctx.ellipse(5, -42, 6, 8, 0.2, 0, Math.PI * 2);
  ctx.fill();
  
  // Pupils (looking right toward zombies)
  ctx.fillStyle = PEASHOOTER_COLORS.eyePupil;
  ctx.beginPath();
  ctx.arc(-3, -40, 3, 0, Math.PI * 2);
  ctx.arc(7, -42, 3, 0, Math.PI * 2);
  ctx.fill();
  
  // Eye shine
  ctx.fillStyle = '#fff';
  ctx.beginPath();
  ctx.arc(-4, -42, 1.5, 0, Math.PI * 2);
  ctx.arc(6, -44, 1.5, 0, Math.PI * 2);
  ctx.fill();
  
  // Cheeks (subtle pink)
  ctx.fillStyle = PEASHOOTER_COLORS.cheek;
  ctx.globalAlpha = 0.4;
  ctx.beginPath();
  ctx.arc(-12, -30, 5, 0, Math.PI * 2);
  ctx.arc(12, -32, 5, 0, Math.PI * 2);
  ctx.fill();
  ctx.globalAlpha = 1;
  
  // Eyebrows (expression)
  ctx.strokeStyle = PEASHOOTER_COLORS.bodyDark;
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(-10, -48);
  ctx.lineTo(-2, -46);
  ctx.moveTo(2, -48);
  ctx.lineTo(10, -50);
  ctx.stroke();
}
```

#### 5. Leaves (Base)
```javascript
function drawPeashooterBackLeaves(ctx) {
  // Back leaves (darker)
  ctx.fillStyle = PEASHOOTER_COLORS.leaf;
  
  // Left back leaf
  ctx.beginPath();
  ctx.ellipse(-15, 20, 12, 18, -0.5, 0, Math.PI * 2);
  ctx.fill();
  
  // Right back leaf
  ctx.beginPath();
  ctx.ellipse(15, 18, 10, 15, 0.5, 0, Math.PI * 2);
  ctx.fill();
}

function drawPeashooterFrontLeaves(ctx) {
  // Front leaves (lighter, with veins)
  ctx.fillStyle = PEASHOOTER_COLORS.leafLight;
  
  // Center front leaf
  ctx.beginPath();
  ctx.ellipse(0, 22, 14, 20, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Leaf vein
  ctx.strokeStyle = PEASHOOTER_COLORS.leaf;
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(0, 8);
  ctx.lineTo(0, 38);
  ctx.stroke();
  
  // Side veins
  ctx.beginPath();
  ctx.moveTo(0, 20);
  ctx.lineTo(-8, 15);
  ctx.moveTo(0, 20);
  ctx.lineTo(8, 15);
  ctx.moveTo(0, 28);
  ctx.lineTo(-8, 32);
  ctx.moveTo(0, 28);
  ctx.lineTo(8, 32);
  ctx.stroke();
}
```

### Peashooter Animations

#### Idle Animation
```javascript
function getPeashooterIdleOffset(time) {
  // Gentle bobbing
  return Math.sin(time * 0.003) * 2;
}
```

#### Shooting Animation
```javascript
function getPeashooterShootScale(time, lastShotTime) {
  const elapsed = time - lastShotTime;
  if (elapsed > 200) return 1;
  
  // Quick recoil: squash then stretch
  if (elapsed < 100) {
    return 0.9; // Recoil back
  } else {
    return 1.1; // Stretch forward
  }
}
```

#### Taking Damage Flash
```javascript
function getPeashooterDamageFlash(entity) {
  if (entity.render.flashTime > 0) {
    return 'rgba(255, 255, 255, 0.7)';
  }
  return null;
}
```

---

## 4. Zombie Sprite Design

### Visual Style
Cartoonish undead with oversized head, tattered suit, and lurching posture. Should look both comical and slightly menacing.

### Color Palette
```javascript
const ZOMBIE_COLORS = {
  skin: '#c5e1a5',          // Sickly green
  skinShadow: '#9ccc65',    // Darker green for shadows
  skinHighlight: '#dcedc8', // Lighter for highlights
  suit: '#616161',          // Grey suit
  suitDark: '#424242',      // Shadowed suit
  suitLight: '#757575',     // Highlight
  shirt: '#eeeeee',         // White shirt
  tie: '#d32f2f',           // Red tie
  tieDark: '#b71c1c',       // Dark red
  pants: '#424242',         // Dark grey pants
  eyeWhite: '#ffeb3b',      // Yellowed eyes
  eyePupil: '#000000',
  mouth: '#3e2723',         // Dark mouth
  teeth: '#eeeeee',         // White teeth
  bone: '#e0e0e0'           // Exposed bone
};
```

### Sprite Structure
```javascript
function drawZombie(ctx, x, y, scale = 1, state = 'walking', animationFrame = 0) {
  ctx.save();
  ctx.translate(x, y);
  ctx.scale(scale, scale);
  
  // State-based positioning
  if (state === 'eating') {
    // Lean forward
    ctx.rotate(0.1);
    ctx.translate(5, 0);
  }
  
  // Walking bob
  if (state === 'walking') {
    const bob = Math.abs(Math.sin(animationFrame * 0.15)) * 3;
    ctx.translate(0, bob);
  }
  
  // Draw back to front
  drawZombieBackArm(ctx, animationFrame);
  drawZombieLegs(ctx, animationFrame);
  drawZombieBody(ctx);
  drawZombieTie(ctx);
  drawZombieHead(ctx, animationFrame);
  drawZombieFrontArm(ctx, animationFrame, state);
  
  ctx.restore();
}
```

### Component Breakdown

#### 1. Legs (Walking animation)
```javascript
function drawZombieLegs(ctx, animationFrame) {
  const walkCycle = Math.sin(animationFrame * 0.15);
  
  // Left leg
  ctx.fillStyle = ZOMBIE_COLORS.pants;
  ctx.save();
  ctx.translate(-10, 30);
  ctx.rotate(walkCycle * 0.2);
  ctx.fillRect(-8, 0, 16, 40);
  // Shoe
  ctx.fillStyle = '#212121';
  ctx.fillRect(-10, 35, 22, 10);
  ctx.restore();
  
  // Right leg
  ctx.fillStyle = ZOMBIE_COLORS.pants;
  ctx.save();
  ctx.translate(10, 30);
  ctx.rotate(-walkCycle * 0.2);
  ctx.fillRect(-8, 0, 16, 40);
  // Shoe
  ctx.fillStyle = '#212121';
  ctx.fillRect(-10, 35, 22, 10);
  ctx.restore();
}
```

#### 2. Body (Torso)
```javascript
function drawZombieBody(ctx) {
  // Jacket body
  const bodyGrad = ctx.createLinearGradient(-20, -20, 20, 20);
  bodyGrad.addColorStop(0, ZOMBIE_COLORS.suitDark);
  bodyGrad.addColorStop(0.5, ZOMBIE_COLORS.suit);
  bodyGrad.addColorStop(1, ZOMBIE_COLORS.suitLight);
  
  ctx.fillStyle = bodyGrad;
  ctx.beginPath();
  ctx.roundRect(-22, -10, 44, 45, 5);
  ctx.fill();
  
  // Shirt triangle
  ctx.fillStyle = ZOMBIE_COLORS.shirt;
  ctx.beginPath();
  ctx.moveTo(-10, -10);
  ctx.lineTo(10, -10);
  ctx.lineTo(0, 15);
  ctx.closePath();
  ctx.fill();
  
  // Jacket lapels
  ctx.fillStyle = ZOMBIE_COLORS.suitDark;
  ctx.beginPath();
  ctx.moveTo(-22, -10);
  ctx.lineTo(-5, 15);
  ctx.lineTo(-22, 20);
  ctx.closePath();
  ctx.fill();
  
  ctx.beginPath();
  ctx.moveTo(22, -10);
  ctx.lineTo(5, 15);
  ctx.lineTo(22, 20);
  ctx.closePath();
  ctx.fill();
}
```

#### 3. Tie
```javascript
function drawZombieTie(ctx) {
  // Tie
  ctx.fillStyle = ZOMBIE_COLORS.tie;
  ctx.beginPath();
  ctx.moveTo(-5, -8);
  ctx.lineTo(5, -8);
  ctx.lineTo(3, 5);
  ctx.lineTo(0, 25);
  ctx.lineTo(-3, 5);
  ctx.closePath();
  ctx.fill();
  
  // Tie shadow
  ctx.fillStyle = ZOMBIE_COLORS.tieDark;
  ctx.beginPath();
  ctx.moveTo(0, -8);
  ctx.lineTo(3, 5);
  ctx.lineTo(0, 25);
  ctx.lineTo(-1, 5);
  ctx.closePath();
  ctx.fill();
}
```

#### 4. Head (The main feature)
```javascript
function drawZombieHead(ctx, animationFrame) {
  // Head bob
  const headBob = Math.sin(animationFrame * 0.1) * 2;
  ctx.save();
  ctx.translate(0, -35 + headBob);
  
  // Head shape (slightly oval)
  const headGrad = ctx.createRadialGradient(-10, -15, 5, 0, 0, 35);
  headGrad.addColorStop(0, ZOMBIE_COLORS.skinHighlight);
  headGrad.addColorStop(0.5, ZOMBIE_COLORS.skin);
  headGrad.addColorStop(1, ZOMBIE_COLORS.skinShadow);
  
  ctx.fillStyle = headGrad;
  ctx.beginPath();
  ctx.ellipse(0, 0, 28, 32, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Outline
  ctx.strokeStyle = ZOMBIE_COLORS.skinShadow;
  ctx.lineWidth = 1;
  ctx.stroke();
  
  // Hair (messy)
  ctx.fillStyle = '#424242';
  for (let i = 0; i < 8; i++) {
    const angle = -Math.PI / 2 + (i - 4) * 0.3;
    const x = Math.cos(angle) * 25;
    const y = Math.sin(angle) * 28;
    ctx.beginPath();
    ctx.arc(x, y, 6, 0, Math.PI * 2);
    ctx.fill();
  }
  
  // Eyes (yellowed, looking left)
  drawZombieEyes(ctx);
  
  // Nose (simple bump)
  ctx.fillStyle = ZOMBIE_COLORS.skinShadow;
  ctx.beginPath();
  ctx.ellipse(-5, 8, 4, 6, 0.3, 0, Math.PI * 2);
  ctx.fill();
  
  // Mouth
  drawZombieMouth(ctx, animationFrame);
  
  ctx.restore();
}

function drawZombieEyes(ctx) {
  // Eye sockets (dark circles)
  ctx.fillStyle = 'rgba(0, 0, 0, 0.2)';
  ctx.beginPath();
  ctx.ellipse(-10, -5, 10, 12, 0, 0, Math.PI * 2);
  ctx.ellipse(10, -5, 10, 12, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Eyeballs (yellowed)
  ctx.fillStyle = ZOMBIE_COLORS.eyeWhite;
  ctx.beginPath();
  ctx.ellipse(-10, -5, 7, 9, 0, 0, Math.PI * 2);
  ctx.ellipse(10, -5, 7, 9, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Pupils (looking left at plants)
  ctx.fillStyle = ZOMBIE_COLORS.eyePupil;
  ctx.beginPath();
  ctx.arc(-13, -5, 3, 0, Math.PI * 2);
  ctx.arc(7, -5, 3, 0, Math.PI * 2);
  ctx.fill();
  
  // Red veins
  ctx.strokeStyle = 'rgba(211, 47, 47, 0.4)';
  ctx.lineWidth = 1;
  for (let i = 0; i < 5; i++) {
    ctx.beginPath();
    ctx.moveTo(-10 + (Math.random() - 0.5) * 10, -10 + Math.random() * 10);
    ctx.lineTo(-10 + (Math.random() - 0.5) * 10, -5 + Math.random() * 8);
    ctx.stroke();
  }
}

function drawZombieMouth(ctx, animationFrame) {
  // Open mouth (slightly agape)
  ctx.fillStyle = ZOMBIE_COLORS.mouth;
  ctx.beginPath();
  ctx.ellipse(-5, 20, 12, 8, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Teeth
  ctx.fillStyle = ZOMBIE_COLORS.teeth;
  for (let i = 0; i < 4; i++) {
    ctx.fillRect(-12 + i * 6, 15, 4, 5);
    ctx.fillRect(-12 + i * 6, 22, 4, 4);
  }
  
  // Tongue (dark red)
  ctx.fillStyle = '#5d1010';
  ctx.beginPath();
  ctx.ellipse(-5, 22, 6, 3, 0, 0, Math.PI * 2);
  ctx.fill();
}
```

#### 5. Arms
```javascript
function drawZombieBackArm(ctx, animationFrame) {
  const swing = Math.sin(animationFrame * 0.15 + Math.PI) * 0.3;
  
  ctx.save();
  ctx.translate(-20, 0);
  ctx.rotate(swing);
  
  // Arm
  ctx.fillStyle = ZOMBIE_COLORS.suit;
  ctx.fillRect(-8, 0, 16, 35);
  
  // Hand
  ctx.fillStyle = ZOMBIE_COLORS.skin;
  ctx.beginPath();
  ctx.ellipse(0, 38, 8, 10, 0, 0, Math.PI * 2);
  ctx.fill();
  
  // Fingers
  ctx.fillStyle = ZOMBIE_COLORS.skinShadow;
  for (let i = 0; i < 4; i++) {
    ctx.fillRect(-6 + i * 4, 42, 3, 8);
  }
  
  ctx.restore();
}

function drawZombieFrontArm(ctx, animationFrame, state) {
  let rotation = Math.sin(animationFrame * 0.15) * 0.3;
  
  if (state === 'eating') {
    // Arm raised forward
    rotation = -0.8;
  }
  
  ctx.save();
  ctx.translate(20, 0);
  ctx.rotate(rotation);
  
  // Sleeve
  ctx.fillStyle = ZOMBIE_COLORS.suit;
  ctx.fillRect(-8, 0, 16, 20);
  
  // Forearm
  ctx.fillStyle = ZOMBIE_COLORS.suitDark;
  ctx.fillRect(-7, 18, 14, 18);
  
  // Hand (reaching)
  ctx.fillStyle = ZOMBIE_COLORS.skin;
  ctx.beginPath();
  ctx.ellipse(0, 40, 9, 11, 0, 0, Math.PI * 2);
  ctx.fill();
  
  if (state === 'eating') {
    // Claw-like fingers
    ctx.fillStyle = ZOMBIE_COLORS.skinShadow;
    for (let i = 0; i < 4; i++) {
      ctx.save();
      ctx.translate(-6 + i * 4, 45);
      ctx.rotate(-0.5 + i * 0.3);
      ctx.fillRect(0, 0, 4, 12);
      ctx.restore();
    }
  }
  
  ctx.restore();
}
```

---

## 5. Projectile (Pea) Visual Design

### Style
Bright, glowing green sphere with motion trail. Should look energetic and fast.

```javascript
const PEA_COLORS = {
  core: '#76ff03',        // Bright center
  main: '#64dd17',        // Main green
  shadow: '#33691e',      // Dark edge
  glow: 'rgba(118, 255, 3, 0.3)'  // Outer glow
};

function drawPea(ctx, x, y, velocityX) {
  ctx.save();
  ctx.translate(x, y);
  
  // Motion blur trail (behind)
  if (velocityX > 0) {
    const trailGrad = ctx.createLinearGradient(-20, 0, 0, 0);
    trailGrad.addColorStop(0, 'rgba(118, 255, 3, 0)');
    trailGrad.addColorStop(1, 'rgba(118, 255,  3, 0.5)');
    ctx.fillStyle = trailGrad;
    ctx.beginPath();
    ctx.ellipse(-10, 0, 15, 6, 0, 0, Math.PI * 2);
    ctx.fill();
  }
  
  // Outer glow
  ctx.fillStyle = PEA_COLORS.glow;
  ctx.beginPath();
  ctx.arc(0, 0, 12, 0, Math.PI * 2);
  ctx.fill();
  
  // Main sphere
  const peaGrad = ctx.createRadialGradient(-3, -3, 2, 0, 0, 8);
  peaGrad.addColorStop(0, PEA_COLORS.core);
  peaGrad.addColorStop(0.7, PEA_COLORS.main);
  peaGrad.addColorStop(1, PEA_COLORS.shadow);
  
  ctx.fillStyle = peaGrad;
  ctx.beginPath();
  ctx.arc(0, 0, 8, 0, Math.PI * 2);
  ctx.fill();
  
  // Highlight
  ctx.fillStyle = 'rgba(255, 255, 255, 0.6)';
  ctx.beginPath();
  ctx.arc(-3, -3, 3, 0, Math.PI * 2);
  ctx.fill();
  
  ctx.restore();
}
```

---

## 6. Plant Selection UI

### Layout
Horizontal bar at the top with seed packets. Each packet shows:
- Plant icon
- Name
- Cost (sun amount)
- Cooldown overlay when used

### Seed Packet Design
```javascript
const SEED_PACKET = {
  width: 70,
  height: 90,
  padding: 10,
  colors: {
    bg: '#8d6e63',        // Brown background
    bgSelected: '#5d4037', // Darker when selected
    border: '#3e2723',    // Dark border
    borderHighlight: '#d7ccc8', // Light highlight
    text: '#ffffff',
    cost: '#ffeb3b'       // Yellow sun cost
  }
};

function drawSeedPacket(ctx, x, y, plantType, isSelected, cooldownPercent = 0) {
  const { width, height, colors } = SEED_PACKET;
  
  // Background
  ctx.fillStyle = isSelected ? colors.bgSelected : colors.bg;
  ctx.beginPath();
  ctx.roundRect(x, y, width, height, 5);
  ctx.fill();
  
  // 3D bevel effect
  ctx.strokeStyle = colors.borderHighlight;
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(x + 2, y + height - 2);
  ctx.lineTo(x + 2, y + 2);
  ctx.lineTo(x + width - 2, y + 2);
  ctx.stroke();
  
  ctx.strokeStyle = colors.border;
  ctx.beginPath();
  ctx.moveTo(x + width - 2, y + 2);
  ctx.lineTo(x + width - 2, y + height - 2);
  ctx.lineTo(x + 2, y + height - 2);
  ctx.stroke();
  
  // Plant icon (mini version)
  ctx.save();
  ctx.translate(x + width / 2, y + 35);
  ctx.scale(0.6, 0.6);
  drawPeashooterIcon(ctx); // Simplified version
  ctx.restore();
  
  // Name
  ctx.fillStyle = colors.text;
  ctx.font = 'bold 11px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText(plantType.name, x + width / 2, y + 65);
  
  // Sun cost
  ctx.fillStyle = colors.cost;
  ctx.font = 'bold 14px sans-serif';
  ctx.fillText(plantType.cost, x + width / 2, y + 82);
  
  // Cooldown overlay (darkens the packet)
  if (cooldownPercent > 0) {
    ctx.fillStyle = `rgba(0, 0, 0, ${0.6 * cooldownPercent})`;
    ctx.beginPath();
    ctx.roundRect(x, y, width, height, 5);
    ctx.fill();
  }
  
  // Selection border
  if (isSelected) {
    ctx.strokeStyle = '#ffeb3b';
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.roundRect(x - 2, y - 2, width + 4, height + 4, 7);
    ctx.stroke();
  }
}
```

### Sun Counter
```javascript
function drawSunCounter(ctx, x, y, sunAmount) {
  const width = 100;
  const height = 50;
  
  // Background panel
  ctx.fillStyle = '#5d4037';
  ctx.beginPath();
  ctx.roundRect(x, y, width, height, 8);
  ctx.fill();
  
  ctx.strokeStyle = '#8d6e63';
  ctx.lineWidth = 3;
  ctx.stroke();
  
  // Sun icon
  ctx.fillStyle = '#ffeb3b';
  ctx.beginPath();
  ctx.arc(x + 25, y + 25, 15, 0, Math.PI * 2);
  ctx.fill();
  
  // Sun rays
  ctx.strokeStyle = '#ffeb3b';
  ctx.lineWidth = 3;
  for (let i = 0; i < 8; i++) {
    const angle = (i / 8) * Math.PI * 2;
    ctx.beginPath();
    ctx.moveTo(x + 25 + Math.cos(angle) * 18, y + 25 + Math.sin(angle) * 18);
    ctx.lineTo(x + 25 + Math.cos(angle) * 22, y + 25 + Math.sin(angle) * 22);
    ctx.stroke();
  }
  
  // Sun amount
  ctx.fillStyle = '#ffeb3b';
  ctx.font = 'bold 24px sans-serif';
  ctx.textAlign = 'left';
  ctx.fillText(sunAmount.toString(), x + 50, y + 32);
}
```

---

## 7. Animation System Architecture

### Animation State Management
```javascript
const AnimationSystem = {
  // Store animation state per entity
  states: new Map(),
  
  getState(entityId) {
    if (!this.states.has(entityId)) {
      this.states.set(entityId, {
        frame: 0,
        lastUpdate: 0,
        currentAnim: 'idle'
      });
    }
    return this.states.get(entityId);
  },
  
  update(deltaTime) {
    for (const state of this.states.values()) {
      state.frame += deltaTime * 0.06; // Convert to ~60fps frames
    }
  },
  
  trigger(entityId, animationName) {
    const state = this.getState(entityId);
    state.currentAnim = animationName;
    state.frame = 0;
  }
};
```

### Entity Render with Animation
```javascript
function renderEntity(ctx, entity) {
  const animState = AnimationSystem.getState(entity.id);
  const time = performance.now();
  
  switch (entity.type) {
    case 'plant':
      drawPeashooter(ctx, entity.transform.x, entity.transform.y, 1, animState.frame);
      break;
    case 'zombie':
      drawZombie(ctx, entity.transform.x, entity.transform.y, 1, entity.zombie.state, animState.frame);
      break;
    case 'projectile':
      drawPea(ctx, entity.transform.x, entity.transform.y, entity.projectile.speed);
      break;
  }
}
```

---

## 8. Visual Effects

### Damage Flash
```javascript
function applyDamageFlash(ctx, entity, drawFn) {
  if (entity.render.flashTime > 0) {
    ctx.save();
    
    // Draw normally first
    drawFn();
    
    // Overlay white with fade
    ctx.globalCompositeOperation = 'source-atop';
    ctx.fillStyle = `rgba(255, 255, 255, ${entity.render.flashTime / 100})`;
    ctx.fillRect(entity.transform.x - 50, entity.transform.y - 50, 100, 100);
    
    ctx.restore();
  } else {
    drawFn();
  }
}
```

### Shadow Under Entities
```javascript
function drawShadow(ctx, x, y, width) {
  ctx.fillStyle = 'rgba(0, 0, 0, 0.2)';
  ctx.beginPath();
  ctx.ellipse(x, y, width / 2, width / 6, 0, 0, Math.PI * 2);
  ctx.fill();
}
```

### Particle Effects (Simple)
```javascript
const particles = [];

function createHitParticles(x, y, color, count = 5) {
  for (let i = 0; i < count; i++) {
    particles.push({
      x, y,
      vx: (Math.random() - 0.5) * 100,
      vy: (Math.random() - 0.5) * 100 - 50,
      life: 1,
      color,
      size: 3 + Math.random() * 4
    });
  }
}

function updateAndDrawParticles(ctx, deltaTime) {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx * deltaTime / 1000;
    p.y += p.vy * deltaTime / 1000;
    p.vy += 200 * deltaTime / 1000; // Gravity
    p.life -= deltaTime / 500;
    
    if (p.life <= 0) {
      particles.splice(i, 1);
      continue;
    }
    
    ctx.globalAlpha = p.life;
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.size * p.life, 0, Math.PI * 2);
    ctx.fill();
  }
  ctx.globalAlpha = 1;
}
```

---

## 9. Implementation Checklist

### Phase 1: Layout & Background
- [ ] Full-screen canvas with responsive scaling
- [ ] Sky gradient background
- [ ] Detailed house with windows, door, roof, siding
- [ ] Detailed lawn with grass texture and grid
- [ ] Fence in background

### Phase 2: Entity Sprites
- [ ] Peashooter with all components (leaves, stem, head, mouth, eyes, cheeks)
- [ ] Zombie with all components (legs, body, tie, head, eyes, mouth, arms)
- [ ] Pea projectile with glow and trail

### Phase 3: Animations
- [ ] Peashooter idle bob
- [ ] Peashooter shoot recoil
- [ ] Zombie walking cycle (legs, arms, head bob)
- [ ] Zombie eating pose
- [ ] Damage flash effect

### Phase 4: UI
- [ ] Seed packet design with selection state
- [ ] Sun counter display
- [ ] Plant placement ghost preview
- [ ] Game over / victory screens

### Phase 5: Polish
- [ ] Shadows under entities
- [ ] Particle effects on hit
- [ ] Health bars styling
- [ ] Stats display styling

---

## Summary

This visual overhaul transforms the basic shape-based game into a rich, detailed experience:

1. **Full-screen responsive layout** with proper scaling
2. **Detailed environment** - house with siding/windows, textured lawn, fence
3. **Complex sprites** - multi-component peashooters and zombies with shading
4. **Smooth animations** - idle bobs, walking cycles, shooting recoils
5. **Polished UI** - 3D seed packets, sun counter, placement preview
6. **Visual effects** - damage flashes, shadows, particles

All achieved through Canvas API drawing commands - no external images needed.
