# Plants vs. Zombies - Complete Game Design Document

## Project Overview
A complete browser-based clone of Plants vs. Zombies with focus on performance, visual polish, and quality of life features.

---

## 1. Architecture & Module Structure

### Core Philosophy
- **Entity-Component-System (ECS)** approach for flexibility and performance
- **Object Pooling** to eliminate garbage collection stutters
- **Spatial Hashing** for efficient collision detection
- **Canvas-based Rendering** with dirty-rectangle optimization
- **Event-driven Architecture** for loose coupling

### Module Hierarchy
```
GameEngine/
├── Core/
│   ├── GameLoop.js          # RAF-based loop with delta time
│   ├── StateManager.js      # Game states: MENU, PLAYING, PAUSED, GAMEOVER
│   ├── EventBus.js          # Pub/sub for decoupled communication
│   └── AssetManager.js      # Sprite/animation management
├── ECS/
│   ├── EntityManager.js     # ID generation, component storage
│   ├── Component.js         # Base component class
│   └── System.js            # Base system class
├── Systems/
│   ├── RenderSystem.js      # Canvas rendering with layers
│   ├── MovementSystem.js    # Position updates
│   ├── CombatSystem.js      # Damage, health, death
│   ├── CollisionSystem.js   # Spatial grid collisions
│   ├── AnimationSystem.js   # Sprite animation updates
│   └── EconomySystem.js     # Sun generation/collection
├── Entities/
│   ├── Plant.js             # Base plant class
│   ├── Zombie.js            # Base zombie class
│   ├── Projectile.js        # Peas, etc
│   ├── Sun.js               # Falling sun particles
│   └── LawnMower.js         # Lane clearing defense
├── Grid/
│   ├── Lawn.js              # 5x9 tile grid management
│   └── Tile.js              # Individual cell logic
├── UI/
│   ├── SeedPacket.js        # Plant selection cards
│   ├── SunCounter.js        # Economy display
│   ├── Shovel.js            # Remove tool
│   └── ProgressBar.js       # Wave indicator
└── Utils/
    ├── ObjectPool.js        # Reusable object management
    ├── SpatialGrid.js       # 2D spatial hashing
    └── BezierEasing.js      # Smooth animation curves
```

---

## 2. Entity Component System Design

### Why ECS?
- **Performance**: Systems process homogeneous data sequentially (cache-friendly)
- **Flexibility**: Mix and match behaviors via components
- **Scalability**: Add new entity types without changing existing code

### Component Types

#### TransformComponent
```javascript
{
  x, y,              // Position in pixels
  gridX, gridY,      // Grid coordinates (if applicable)
  width, height,     // Bounding box
  layer              // Render layer (0=back, 10=front)
}
```

#### RenderComponent
```javascript
{
  sprite,            // Current animation frame
  color,             // Fallback shape color
  shape,             // 'circle', 'rect', 'custom'
  scale,             // For squash/stretch effects
  alpha,             // Fade in/out
  flipX              // Face left/right
}
```

#### HealthComponent
```javascript
{
  current,           // Current HP
  max,               // Maximum HP
  armor,             // Damage reduction
  invulnerable,      // Frames of invulnerability
  showDamageFlash    // Visual feedback on hit
}
```

#### CombatComponent
```javascript
{
  damage,            // Attack damage
  range,             // Attack range in pixels
  cooldown,          // Frames between attacks
  currentCooldown,   // Current cooldown timer
  projectileType,    // null for melee
  attackSound,       // Reference (muted by design)
  targetLayer        // Which entities to target
}
```

#### MovementComponent
```javascript
{
  velocityX,         // Horizontal speed
  velocityY,         // Vertical speed
  speed,             // Base movement speed
  frozen,            // Slow effect duration
  stunned            // Stop duration
}
```

#### AnimationComponent
```javascript
{
  state,             // 'idle', 'attack', 'walk', 'death'
  frame,             // Current frame index
  frameTimer,        // Time until next frame
  frameDuration,     // ms per frame
  animations: {       // State -> frames mapping
    idle: { frames, loop },
    attack: { frames, loop },
    ...
  }
}
```

#### PlantComponent
```javascript
{
  type,              // 'sunflower', 'peashooter', etc
  cost,              // Sun cost
  rechargeTime,      // Seconds until reusable
  producing,         // For sun generators
  armor              // For defensive plants
}
```

#### ZombieComponent
```javascript
{
  type,              // 'basic', 'cone', 'bucket', 'flag'
  lane,              // Which of 5 lanes
  eating,            // Currently eating a plant
  eatTimer,          // Damage interval while eating
  reachedHouse       // Game over trigger
}
```

### Entity Factory Pattern
```javascript
// Plants
EntityFactory.createSunflower(gridX, gridY)
EntityFactory.createPeashooter(gridX, gridY, variant='snow')
EntityFactory.createWallnut(gridX, gridY, stage='full|cracked|destroyed')
EntityFactory.createCherryBomb(gridX, gridY)

// Zombies
EntityFactory.createZombie(lane, type='basic', armor=0)
EntityFactory.createZombieWave(waveConfig)

// Projectiles
EntityFactory.createProjectile(type, x, y, lane, damage)

// Effects
EntityFactory.createExplosion(x, y, radius, damage)
EntityFactory.createDamageNumber(x, y, amount)
```

---

## 3. Rendering System Architecture

### Layer Structure (Back to Front)
1. **Layer 0**: Background (lawn, sky, fence)
2. **Layer 1**: Lawn mowers (when idle at left)
3. **Layer 2**: Plants (behind zombies when appropriate)
4. **Layer 3**: Zombies
5. **Layer 4**: Plants (in front of zombies)
6. **Layer 5**: Projectiles
7. **Layer 6**: Effects (explosions, particles)
8. **Layer 7**: Sun particles
9. **Layer 8**: UI elements on lawn (selection highlight)
10. **Layer 9**: Full UI overlay

### Rendering Optimizations

#### 1. Dirty Rectangle Tracking
- Only redraw regions that changed
- Merge overlapping dirty regions
- Use `ctx.clip()` for efficiency

#### 2. Off-screen Caching
- Pre-render static background to offscreen canvas
- Cache plant/zombie sprites at different animation frames
- Reuse cached renders for identical entities

#### 3. Spatial Culling
- Don't render entities outside viewport
- Quick AABB check before detailed render

#### 4. Batched Drawing
- Group same-shape renders (all circles, then all rects)
- Minimize context state changes

### Visual Style (Procedural Graphics)
Since no external assets:

#### Color Palette
```javascript
const PALETTE = {
  // Lawn
  grassLight: '#7cb342',
  grassDark: '#689f38',
  grassHighlight: '#8bc34a',
  
  // Plants
  sunflower: { petal: '#ffeb3b', center: '#5d4037', stem: '#4caf50' },
  peashooter: { body: '#4caf50', head: '#66bb6a', mouth: '#2e7d32' },
  wallnut: { light: '#d7ccc8', medium: '#a1887f', dark: '#5d4037' },
  cherry: { red: '#f44336', stem: '#4caf50' },
  
  // Zombies
  zombie: { skin: '#c5e1a5', suit: '#424242', tie: '#d32f2f', bone: '#eeeeee' },
  
  // Effects
  sun: { core: '#fff176', glow: '#ffeb3b', outer: '#fff9c4' },
  explosion: ['#ff5722', '#ff9800', '#ffc107', '#ffeb3b'],
  
  // UI
  ui: { panel: '#5d4037', highlight: '#8d6e63', text: '#ffffff' }
}
```

#### Shape Primitives
- **Circles** with gradients for 3D effect
- **Rounded rectangles** for organic feel
- **Bezier curves** for leaves, stems
- **Simple eyes**: white circle + black pupil (directional)

#### Animation Techniques
- **Squash and stretch**: Scale Y up/down when walking
- **Bobbing**: Sine wave offset for idle animations
- **Wobble**: Rotation oscillation for plants
- **Particle systems**: For explosions, sun collection

---

## 4. Game Mechanics & Systems

### The Lawn Grid
```javascript
const GRID = {
  lanes: 5,
  columns: 9,
  cellWidth: 80,
  cellHeight: 100,
  offsetX: 250,  // Space for lawn mowers
  offsetY: 100   // Space for UI
}
```

Each tile tracks:
- Occupying plant (if any)
- Occupying zombie count
- Special properties (pool, roof, etc - basic version: all grass)

### Economy System (Sun)

#### Sun Sources
1. **Natural sun**: Falls from sky periodically (25 sun each)
2. **Sunflowers**: Generate sun periodically (25 sun each)
3. **Level start**: Initial sun amount

#### Sun Mechanics
- Sun entities fall with gravity
- Click/tap to collect (or auto-collect after timeout)
- Float animation while falling
- Disappear after 10 seconds if not collected
- Collection animates sun to counter

### Combat System

#### Damage Calculation
```javascript
damageTaken = max(1, incomingDamage - armor)
```

#### Attack Types
- **Single target**: Peashooter, zombie bites
- **Area of Effect**: Cherry Bomb (3x3 grid)
- **Lane-wide**: Lawn mowers, instant-kill plants

#### Status Effects
- **Frozen**: 50% speed reduction (snow pea)
- **Stunned**: Brief stop (explosion)
- **Invulnerable**: Brief period after hit

### Zombie Wave System

#### Wave Structure
```javascript
const WAVE_CONFIG = {
  initialDelay: 30,      // Seconds before first zombie
  waveInterval: 45,      // Seconds between waves
  warningTime: 5,        // Seconds of "Huge Wave" warning
  
  waves: [
    { count: 3, types: ['basic'], lanes: 'random' },
    { count: 5, types: ['basic', 'basic', 'cone'], lanes: 'all' },
    { count: 10, types: ['mixed'], lanes: 'all', huge: true },
    // ... final wave
  ]
}
```

#### Spawn Logic
- Zombies spawn off-screen to the right
- Stagger spawns within a wave (0.5-2s intervals)
- Flag zombie leads huge waves
- Progress bar shows wave advancement

### Lawn Mower Defense
- One per lane, starts at left edge
- Triggered when zombie reaches leftmost column
- Moves right, killing all zombies in lane
- Single use per lane
- Visual: Red riding mower with animation

### Plant Types (MVP)

#### 1. Sunflower (50 sun)
- **Role**: Economy
- **Health**: 300
- **Behavior**: Generates 25 sun every 24 seconds
- **Visual**: Yellow petals, brown center, happy face

#### 2. Peashooter (100 sun)
- **Role**: Basic attacker
- **Health**: 300
- **Damage**: 20 per pea
- **Fire rate**: 1.4 seconds
- **Visual**: Green tube, shoots green peas

#### 3. Snow Pea (175 sun)
- **Role**: Slowing attacker
- **Same as Peashooter but**: Blue tint, slows zombies 50%
- **Visual**: Icy blue, snow particles

#### 4. Wall-nut (50 sun)
- **Role**: Defender
- **Health**: 4000 (takes ~13 zombie bites)
- **Visual**: Brown nut, cracking animation at damage thresholds

#### 5. Cherry Bomb (150 sun)
- **Role**: Instant use, area damage
- **Damage**: 1800 (kills most zombies instantly)
- **Range**: 3x3 grid area
- **Visual**: Two cherries, explosion animation

### Zombie Types (MVP)

#### 1. Basic Zombie
- **Health**: 200
- **Speed**: 4.7s per tile
- **Damage**: 100 per bite (1 bite/sec)
- **Visual**: Grey suit, red tie, green skin

#### 2. Conehead Zombie
- **Health**: 640 (200 + 440 cone)
- **Same as basic with**: Orange cone hat
- **Visual**: Cone cracks then falls off

#### 3. Buckethead Zombie
- **Health**: 1370 (200 + 1170 bucket)
- **Same as basic with**: Metal bucket
- **Visual**: Bucket dents, then falls

#### 4. Flag Zombie
- **Health**: 200
- **Speed**: 3.8s per tile (20% faster)
- **Visual**: Red flag, leads waves

### Quality of Life Features

#### Input Handling
1. **Click seed packet** → Cursor shows plant ghost
2. **Hover over lawn** → Highlight valid placement
3. **Click tile** → Place plant (if valid)
4. **Click sun** → Collect
5. **Click shovel** → Remove plant mode
6. **Keyboard shortcuts**: Number keys 1-5 for seed packets

#### Visual Feedback
- **Invalid placement**: Red highlight, shake animation
- **Valid placement**: Green highlight, plant ghost
- **Plant ready**: Seed packet glows
- **Low health**: Flashing red overlay
- **Damage numbers**: Floating text on hit
- **Wave warning**: "HUGE WAVE" banner with shaking

#### Accessibility
- **Pause button**: Spacebar or click
- **Speed toggle**: Fast-forward for slow moments
- **Mute toggle**: (Even without sound, show icon)

---

## 5. Performance Optimization Strategy

### Object Pooling
```javascript
class ObjectPool {
  constructor(createFn, resetFn, size = 50) {
    this.available = Array(size).fill().map(createFn);
    this.active = new Set();
  }
  
  acquire() {
    const obj = this.available.pop() || this.createFn();
    this.active.add(obj);
    return obj;
  }
  
  release(obj) {
    this.resetFn(obj);
    this.active.delete(obj);
    this.available.push(obj);
  }
}

// Pools for:
// - Projectiles (peas)
// - Sun particles
// - Damage numbers
// - Explosion particles
```

### Spatial Grid for Collisions
```javascript
class SpatialGrid {
  constructor(cellSize = 100) {
    this.cellSize = cellSize;
    this.cells = new Map(); // key: "x,y", value: Set of entities
  }
  
  insert(entity) {
    const key = this.getKey(entity.x, entity.y);
    if (!this.cells.has(key)) this.cells.set(key, new Set());
    this.cells.get(key).add(entity);
  }
  
  query(x, y, radius) {
    // Return entities in neighboring cells within radius
  }
  
  clear() {
    this.cells.clear();
  }
}
```

### Entity Update Optimization
- **Sleeping entities**: Don't update off-screen or idle entities
- **Update throttling**: Update animation every 2nd frame if FPS drops
- **Early exit**: Skip collision checks for entities far apart

### Memory Management
- Pre-allocate arrays to expected capacity
- Avoid creating objects in hot paths (use temp variables)
- Use `requestAnimationFrame` with delta time, not `setInterval`

---

## 6. Visual Polish Details

### Animations

#### Plant Animations
- **Idle**: Gentle bobbing (sine wave)
- **Attack**: Forward lunge, mouth opens, projectile spawns
- **Hit**: Flash white, brief scale shake
- **Death**: Fade out, scale down

#### Zombie Animations
- **Walk**: Arms forward, lurching gait, head bob
- **Eat**: Biting motion, arms raised
- **Hit**: Recoil backward, flash red
- **Death**: Fall backward, fade out after delay

#### Projectile Animations
- **Pea**: Spinning motion trail
- **Snow pea**: Ice trail particles
- **Hit**: Small splash effect

### Particle Effects
- **Sun spawn**: Glow expand then contract
- **Sun collect**: Fly to counter with trail
- **Explosion**: Radial burst of colored particles
- **Plant placed**: Dirt puff from ground
- **Zombie death**: Smoke puff

### UI Polish
- **Seed packets**: 3D bevel effect, cooldown overlay
- **Sun counter**: Glow pulse when increasing
- **Progress bar**: Segmented, flag markers
- **Shovel**: Hover lift effect

### Screen Shake & Effects
- **Explosion**: Brief screen shake
- **Huge wave**: Warning banner with pulse
- **Game over**: Slow zoom to zombie eating brain

---

## 7. State Management

### Game States
```javascript
const GameState = {
  MENU: 'menu',           // Title screen
  SEED_SELECT: 'seed',    // Choose loadout (optional for MVP)
  PLAYING: 'playing',     // Active gameplay
  PAUSED: 'paused',       // Pause menu
  WAVE_COMPLETE: 'wave',  // Between waves (optional)
  GAME_OVER: 'gameover',  // Win/lose screen
  VICTORY: 'victory'      // Level complete
}
```

### State Transitions
- MENU → PLAYING: Click "Adventure"
- PLAYING → PAUSED: Press space/click pause
- PLAYING → GAME_OVER: Zombie reaches house
- PLAYING → VICTORY: All waves cleared

### Save/Resume (Optional)
- Store game state in localStorage
- Resume on page reload

---

## 8. Complete Feature List

### Core Features (Must Have)
- [x] 5-lane lawn grid system
- [x] 5 plant types (Sunflower, Peashooter, Snow Pea, Wall-nut, Cherry Bomb)
- [x] 4 zombie types (Basic, Conehead, Buckethead, Flag)
- [x] Sun economy system
- [x] Plant placement with seed packets
- [x] Zombie wave spawning
- [x] Combat system (projectiles, eating, damage)
- [x] Lawn mower last defense
- [x] Win/lose conditions
- [x] Pause functionality

### Polish Features (High Priority)
- [x] Smooth animations for all entities
- [x] Particle effects (explosions, sun collection)
- [x] Visual feedback for all actions
- [x] Damage number popups
- [x] Plant placement ghost/validation
- [x] Progress bar showing wave progress
- [x] "Huge Wave" warning banner
- [x] Screen shake on explosions
- [x] Keyboard shortcuts (1-5 for plants)
- [x] Hover tooltips showing stats

### Advanced Features (If Time Permits)
- [ ] Shovel tool to remove plants
- [ ] Multiple levels with increasing difficulty
- [ ] Almanac showing plant/zombie info
- [ ] Sound effects (muted by design constraint)
- [ ] Fast-forward button
- [ ] Seed selection before level
- [ ] Achievement system

---

## 9. Technical Specifications

### Target Performance
- **FPS**: Solid 60fps on modern browsers
- **Entity count**: Support 50+ zombies + 20+ plants simultaneously
- **Memory**: < 100MB heap usage
- **Startup**: < 2 seconds to playable

### Browser Support
- Chrome/Edge (latest 2 versions)
- Firefox (latest 2 versions)
- Safari (latest 2 versions)
- Mobile: Touch-friendly (optional stretch)

### Dependencies
- **None** - Pure vanilla JavaScript
- Single HTML file with inline CSS/JS for easy deployment

### File Structure
```
index.html          # Main entry, canvas setup
style.css           # Minimal styles
js/
├── main.js         # Entry point, initialization
├── config.js       # Constants, balance values
├── game.js         # Main game controller
├── ecs/
│   ├── entity.js
│   ├── component.js
│   └── system.js
├── systems/
│   ├── render.js
│   ├── combat.js
│   ├── movement.js
│   └── economy.js
├── entities/
│   ├── plant.js
│   ├── zombie.js
│   └── projectile.js
├── ui/
│   ├── seedpacket.js
│   └── hud.js
└── utils/
    ├── pool.js
    └── spatialgrid.js
```

---

## 10. Implementation Roadmap

### Phase 1: Foundation
1. Set up HTML5 Canvas and basic loop
2. Implement ECS framework
3. Create grid system and tile rendering
4. Basic entity spawning

### Phase 2: Core Mechanics
1. Plant placement system
2. Zombie spawning and movement
3. Projectile system
4. Basic combat (damage, death)
5. Sun economy

### Phase 3: Content
1. Implement all 5 plant types
2. Implement all 4 zombie types
3. Wave system with configuration
4. Lawn mowers

### Phase 4: Polish
1. All animations
2. Particle effects
3. UI/UX improvements
4. Visual feedback systems
5. Balance tuning

### Phase 5: Optimization
1. Object pooling
2. Spatial grid
3. Render optimizations
4. Performance testing

---

## Summary

This design prioritizes:
1. **Performance** through ECS, object pooling, and spatial hashing
2. **Visual appeal** through procedural graphics and extensive animations
3. **Polish** through QOL features and feedback systems
4. **Completeness** with all core PvZ mechanics represented

The architecture supports easy extension for additional plants, zombies, and features while maintaining the performance needed for dozens of simultaneous entities.
