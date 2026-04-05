# Version 1.0 - Core Systems Plan

## Overview
Version 1.0 establishes the foundational architecture: a grid-based lawn where players place plants that shoot projectiles at zombies walking from right to left. No economy, no complex visuals, just the core loop working correctly.

---

## 1. Core Architecture Principles

### Why These Principles Matter
The decisions made here cascade to all future versions. We need:
- **Single Source of Truth**: One place for each type of data prevents bugs
- **Separation of Concerns**: Game logic vs rendering vs input handling
- **Data-Driven Design**: Behavior defined in configuration, not hardcoded
- **Composition over Inheritance**: Mix behaviors rather than deep class hierarchies

### The Three Core Data Stores

```javascript
// 1. GRID - Spatial indexing and placement
// 2. ENTITIES - All living things (plants, zombies, projectiles)
// 3. SYSTEMS - Logic that operates on entities each frame
```

---

## 2. Grid System Design

### Why a Grid System?
Plants vs Zombies is fundamentally tile-based. Each plant occupies exactly one tile. Zombies move across tiles but interact with plants at tile boundaries. A grid gives us:
- O(1) plant lookup: `grid[lane][col]`
- O(1) placement validation
- Easy lane-based queries for zombie AI

### Grid Data Structure

```javascript
const GRID_CONFIG = {
  LANES: 5,
  COLUMNS: 9,
  CELL_WIDTH: 80,
  CELL_HEIGHT: 100,
  OFFSET_X: 250,  // Space before first column (for mowers)
  OFFSET_Y: 150   // Space for UI at top
};

// Grid stores references to entities
// grid[lane][col] = plantEntity or null
const grid = Array(5).fill(null).map(() => Array(9).fill(null));
```

### Coordinate Conversion Functions

```javascript
/**
 * Convert grid coordinates to pixel coordinates (center of cell)
 * Used when spawning plants at grid positions
 */
function gridToPixel(gridX, gridY) {
  return {
    x: GRID_CONFIG.OFFSET_X + gridX * GRID_CONFIG.CELL_WIDTH + GRID_CONFIG.CELL_WIDTH / 2,
    y: GRID_CONFIG.OFFSET_Y + gridY * GRID_CONFIG.CELL_HEIGHT + GRID_CONFIG.CELL_HEIGHT / 2
  };
}

/**
 * Convert pixel coordinates to grid coordinates
 * Used when clicking to place plants
 * Returns null if outside grid bounds
 */
function pixelToGrid(pixelX, pixelY) {
  const gridX = Math.floor((pixelX - GRID_CONFIG.OFFSET_X) / GRID_CONFIG.CELL_WIDTH);
  const gridY = Math.floor((pixelY - GRID_CONFIG.OFFSET_Y) / GRID_CONFIG.CELL_HEIGHT);
  
  if (gridX < 0 || gridX >= GRID_CONFIG.COLUMNS || 
      gridY < 0 || gridY >= GRID_CONFIG.LANES) {
    return null;
  }
  
  return { x: gridX, y: gridY };
}

/**
 * Get the lane (row) for a given Y pixel coordinate
 * Used for zombie lane assignment and projectile collision
 */
function getLaneFromY(pixelY) {
  const lane = Math.floor((pixelY - GRID_CONFIG.OFFSET_Y) / GRID_CONFIG.CELL_HEIGHT);
  return (lane >= 0 && lane < GRID_CONFIG.LANES) ? lane : null;
}
```

### Why These Functions?
- **Centralized coordinate math**: One place to change if we adjust grid sizing
- **Bounds checking**: Prevents array access errors
- **Clear naming**: `gridToPixel` vs `pixelToGrid` - no confusion about direction

---

## 3. Entity System Architecture

### Entity Definition
An entity is a plain JavaScript object with a unique ID and components:

```javascript
const entity = {
  id: 123,                    // Unique identifier
  type: 'plant',              // 'plant' | 'zombie' | 'projectile'
  active: true,               // Set to false for pooling/deletion
  
  // Components (added based on type)
  transform: { x, y, width, height },
  health: { current, max },
  plant: { type, cooldown, lastShot },
  zombie: { lane, speed, state },
  projectile: { damage, speed, lane },
  render: { color, shape, radius }
};
```

### Why Plain Objects?
- **Serialization**: Easy to save/load game state
- **Inspection**: Console.log shows everything
- **Flexibility**: Add/remove components at runtime
- **Performance**: No class overhead, direct property access

### Entity Manager

```javascript
const EntityManager = {
  entities: new Map(),        // id -> entity
  plants: new Set(),          // Fast iteration
  zombies: new Set(),         // Fast iteration
  projectiles: new Set(),     // Fast iteration
  nextId: 1,
  
  create(type) {
    const entity = { id: this.nextId++, type, active: true };
    this.entities.set(entity.id, entity);
    this[`${type}s`].add(entity);
    return entity;
  },
  
  destroy(entity) {
    entity.active = false;
    this.entities.delete(entity.id);
    this[`${entity.type}s`].delete(entity);
  },
  
  getByType(type) {
    return this[`${type}s`];
  }
};
```

### Why Separate Collections?
- **Cache locality**: Iterate only relevant entities
- **No type checks**: Systems know what they're getting
- **Fast cleanup**: Delete from Map is O(1)

---

## 4. Plant System Design

### Plant Type Definition (Data-Driven)

```javascript
const PLANT_TYPES = {
  PEASHOOTER: {
    id: 'peashooter',
    name: 'Peashooter',
    health: 300,
    cooldown: 1400,           // ms between shots
    projectile: 'pea',
    color: '#4caf50',
    radius: 25,
    // Future properties (ignored in v1.0):
    // cost: 100,
    // recharge: 7500,
    description: 'Shoots peas at zombies'
  }
};
```

### Why Data-Driven?
- **Balance changes**: Edit numbers without touching code
- **New plants**: Add entry to PLANT_TYPES, logic stays same
- **UI generation**: Build seed packets from this data

### Plant Factory Function

```javascript
function createPlant(typeId, gridX, gridY) {
  const type = PLANT_TYPES[typeId];
  const pos = gridToPixel(gridX, gridY);
  
  const plant = EntityManager.create('plant');
  
  plant.transform = {
    x: pos.x,
    y: pos.y,
    width: 50,
    height: 60
  };
  
  plant.health = {
    current: type.health,
    max: type.health
  };
  
  plant.plant = {
    type: typeId,
    gridX,
    gridY,
    cooldown: type.cooldown,
    lastShot: 0  // Timestamp of last shot
  };
  
  plant.render = {
    color: type.color,
    radius: type.radius,
    shape: 'circle'
  };
  
  // Register in grid for collision lookup
  grid[gridY][gridX] = plant;
  
  return plant;
}
```

### Plant Update Logic

```javascript
function updatePlants(deltaTime, currentTime) {
  for (const plant of EntityManager.getByType('plant')) {
    if (!plant.active) continue;
    
    // Check if can shoot
    if (currentTime - plant.plant.lastShot >= plant.plant.cooldown) {
      // Check if zombie in same lane
      if (hasZombieInLane(plant.plant.gridY, plant.transform.x)) {
        shootProjectile(plant);
        plant.plant.lastShot = currentTime;
      }
    }
  }
}

/**
 * Check if any zombie exists in this lane, to the right of x position
 * This is O(n) where n = zombies. For v1.0 this is fine (< 50 zombies).
 * Future: Could use lane-based spatial indexing.
 */
function hasZombieInLane(lane, fromX) {
  for (const zombie of EntityManager.getByType('zombie')) {
    if (!zombie.active) continue;
    if (zombie.zombie.lane === lane && zombie.transform.x > fromX) {
      return true;
    }
  }
  return false;
}
```

### Why This Approach?
- **Plants are passive**: They don't track targets, just check "is zombie in lane?"
- **Simple condition**: No complex AI, just lane + position check
- **Extensible**: Different plant types just have different cooldown/projectile

---

## 5. Zombie System Design

### Zombie State Machine

Zombies have distinct behavioral states. Using a state machine prevents bugs and makes behavior predictable:

```javascript
const ZOMBIE_STATES = {
  WALKING: 'walking',     // Moving left
  EATING: 'eating',       // Damaging plant
  DYING: 'dying'          // Death animation, then cleanup
};
```

### Why a State Machine?
- **Clear transitions**: Walking → Eating (when at plant), Eating → Walking (when plant dies)
- **No conflicting behaviors**: Can't walk and eat simultaneously
- **Extensible**: Easy to add FROZEN, STUNNED states later

### Zombie Type Definition

```javascript
const ZOMBIE_TYPES = {
  BASIC: {
    id: 'basic',
    name: 'Basic Zombie',
    health: 200,
    speed: 20,              // pixels per second
    damage: 100,            // damage per bite
    eatCooldown: 1000,      // ms between bites
    color: '#8bc34a',       // Green skin
    radius: 30
  }
};
```

### Zombie Factory Function

```javascript
function createZombie(typeId, lane) {
  const type = ZOMBIE_TYPES[typeId];
  const startX = GRID_CONFIG.OFFSET_X + GRID_CONFIG.COLUMNS * GRID_CONFIG.CELL_WIDTH + 50;
  const startY = GRID_CONFIG.OFFSET_Y + lane * GRID_CONFIG.CELL_HEIGHT + GRID_CONFIG.CELL_HEIGHT / 2;
  
  const zombie = EntityManager.create('zombie');
  
  zombie.transform = {
    x: startX,
    y: startY,
    width: 40,
    height: 70
  };
  
  zombie.health = {
    current: type.health,
    max: type.health
  };
  
  zombie.zombie = {
    type: typeId,
    lane,
    state: ZOMBIE_STATES.WALKING,
    speed: type.speed,
    damage: type.damage,
    eatCooldown: type.eatCooldown,
    lastBite: 0,
    targetPlant: null  // Reference to plant being eaten
  };
  
  zombie.render = {
    color: type.color,
    radius: type.radius,
    shape: 'rect'
  };
  
  return zombie;
}
```

### Zombie Update Logic

```javascript
function updateZombies(deltaTime, currentTime) {
  for (const zombie of EntityManager.getByType('zombie')) {
    if (!zombie.active) continue;
    
    switch (zombie.zombie.state) {
      case ZOMBIE_STATES.WALKING:
        updateZombieWalking(zombie, deltaTime, currentTime);
        break;
      case ZOMBIE_STATES.EATING:
        updateZombieEating(zombie, deltaTime, currentTime);
        break;
      case ZOMBIE_STATES.DYING:
        updateZombieDying(zombie, deltaTime);
        break;
    }
  }
}

function updateZombieWalking(zombie, deltaTime, currentTime) {
  // Check if there's a plant in front
  const plant = getPlantInFront(zombie);
  
  if (plant) {
    // Transition to eating
    zombie.zombie.state = ZOMBIE_STATES.EATING;
    zombie.zombie.targetPlant = plant;
    zombie.zombie.lastBite = currentTime;
  } else {
    // Move left
    zombie.transform.x -= zombie.zombie.speed * (deltaTime / 1000);
    
    // Check if reached house (game over condition for v1.0)
    if (zombie.transform.x < GRID_CONFIG.OFFSET_X - 50) {
      triggerGameOver();
    }
  }
}

function updateZombieEating(zombie, deltaTime, currentTime) {
  const plant = zombie.zombie.targetPlant;
  
  // If plant died, go back to walking
  if (!plant || !plant.active) {
    zombie.zombie.state = ZOMBIE_STATES.WALKING;
    zombie.zombie.targetPlant = null;
    return;
  }
  
  // Bite the plant
  if (currentTime - zombie.zombie.lastBite >= zombie.zombie.eatCooldown) {
    damageEntity(plant, zombie.zombie.damage);
    zombie.zombie.lastBite = currentTime;
    
    // Visual feedback (v1.0: just console or simple effect)
    showBiteEffect(plant.transform.x, plant.transform.y);
  }
}

/**
 * Get the plant directly in front of this zombie
 * Returns null if no plant or plant is behind zombie
 */
function getPlantInFront(zombie) {
  const lane = zombie.zombie.lane;
  const zombieX = zombie.transform.x;
  
  // Check which column the zombie is at
  const col = Math.floor((zombieX - GRID_CONFIG.OFFSET_X) / GRID_CONFIG.CELL_WIDTH);
  
  // If in valid column range, check for plant
  if (col >= 0 && col < GRID_CONFIG.COLUMNS) {
    const plant = grid[lane][col];
    if (plant && plant.active) {
      // Check if zombie is close enough (within cell)
      const plantPos = gridToPixel(col, lane);
      const distance = Math.abs(zombieX - plantPos.x);
      if (distance < 30) {  // Within 30 pixels
        return plant;
      }
    }
  }
  
  return null;
}
```

### Why This State Machine Design?
- **Explicit states**: No hidden behavior, state is always clear
- **Clean transitions**: Each state handles its own logic
- **Easy to extend**: Add new states without changing existing ones
- **Separation**: Walking logic doesn't know about eating logic

---

## 6. Projectile System Design

### Projectile Definition

```javascript
const PROJECTILE_TYPES = {
  PEA: {
    id: 'pea',
    speed: 300,             // pixels per second
    damage: 20,
    radius: 8,
    color: '#76ff03'
  }
};
```

### Projectile Factory

```javascript
function createProjectile(typeId, x, y, lane) {
  const type = PROJECTILE_TYPES[typeId];
  
  const projectile = EntityManager.create('projectile');
  
  projectile.transform = {
    x,
    y,
    width: type.radius * 2,
    height: type.radius * 2
  };
  
  projectile.projectile = {
    type: typeId,
    lane,
    speed: type.speed,
    damage: type.damage
  };
  
  projectile.render = {
    color: type.color,
    radius: type.radius,
    shape: 'circle'
  };
  
  return projectile;
}

function shootProjectile(plant) {
  const type = PLANT_TYPES[plant.plant.type];
  createProjectile(
    type.projectile,
    plant.transform.x + 20,  // Slightly in front of plant
    plant.transform.y,
    plant.plant.gridY
  );
}
```

### Projectile Update & Collision

```javascript
function updateProjectiles(deltaTime) {
  for (const projectile of EntityManager.getByType('projectile')) {
    if (!projectile.active) continue;
    
    // Move right
    projectile.transform.x += projectile.projectile.speed * (deltaTime / 1000);
    
    // Check collision with zombies
    const hitZombie = checkProjectileCollision(projectile);
    
    if (hitZombie) {
      damageEntity(hitZombie, projectile.projectile.damage);
      EntityManager.destroy(projectile);
      showHitEffect(projectile.transform.x, projectile.transform.y);
    }
    
    // Remove if off screen
    else if (projectile.transform.x > canvas.width) {
      EntityManager.destroy(projectile);
    }
  }
}

/**
 * Check if projectile hits any zombie in same lane
 * Returns first zombie hit (closest to plant)
 */
function checkProjectileCollision(projectile) {
  const lane = projectile.projectile.lane;
  const projX = projectile.transform.x;
  const projRadius = projectile.render.radius;
  
  for (const zombie of EntityManager.getByType('zombie')) {
    if (!zombie.active) continue;
    if (zombie.zombie.lane !== lane) continue;
    
    const zombieX = zombie.transform.x;
    const zombieRadius = zombie.render.radius;
    
    // Simple circle collision
    const distance = Math.abs(projX - zombieX);
    if (distance < projRadius + zombieRadius) {
      return zombie;
    }
  }
  
  return null;
}
```

### Why This Collision Approach?
- **Lane-based filtering**: Only check zombies in same lane (optimization)
- **Circle collision**: Simple and effective for v1.0
- **First hit**: Peas pass through if zombie already dead (handled by damage check)

---

## 7. Combat & Damage System

### Unified Damage Function

```javascript
function damageEntity(entity, amount) {
  if (!entity.active) return;
  
  entity.health.current -= amount;
  
  // Visual feedback (v1.0: flash color)
  triggerDamageFlash(entity);
  
  if (entity.health.current <= 0) {
    killEntity(entity);
  }
}

function killEntity(entity) {
  if (entity.type === 'zombie') {
    // Zombie death: change state to dying for animation
    entity.zombie.state = ZOMBIE_STATES.DYING;
    entity.zombie.deathTime = performance.now();
  } else if (entity.type === 'plant') {
    // Plant death: clear from grid and destroy
    const { gridX, gridY } = entity.plant;
    if (grid[gridY][gridX] === entity) {
      grid[gridY][gridX] = null;
    }
    EntityManager.destroy(entity);
  }
}

function updateZombieDying(zombie, deltaTime) {
  // Simple death animation: fade out over 500ms
  const elapsed = performance.now() - zombie.zombie.deathTime;
  const duration = 500;
  
  if (elapsed >= duration) {
    EntityManager.destroy(zombie);
  } else {
    // Fade out (v1.0: could just shrink)
    const progress = elapsed / duration;
    zombie.render.radius = 30 * (1 - progress);
  }
}
```

### Why a Unified Damage Function?
- **Consistent handling**: All damage goes through one path
- **Easy to extend**: Add armor, invulnerability, damage types here
- **Centralized feedback**: One place for damage visuals

---

## 8. Input Handling

### Mouse/Touch Input

```javascript
const InputHandler = {
  selectedPlant: null,  // 'peashooter' or null
  
  init(canvas) {
    canvas.addEventListener('click', (e) => this.handleClick(e));
    canvas.addEventListener('mousemove', (e) => this.handleMouseMove(e));
  },
  
  handleClick(e) {
    const rect = canvas.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    
    // Check if clicking seed packet (UI area)
    const packet = getSeedPacketAt(x, y);
    if (packet) {
      this.selectedPlant = packet.plantType;
      return;
    }
    
    // Try to place plant
    if (this.selectedPlant) {
      const gridPos = pixelToGrid(x, y);
      if (gridPos && canPlacePlant(gridPos.x, gridPos.y)) {
        createPlant(this.selectedPlant, gridPos.x, gridPos.y);
        // For v1.0, don't deselect (free placement)
      }
    }
  },
  
  handleMouseMove(e) {
    // For hover effects (v1.0: highlight grid cell)
    const rect = canvas.getBoundingClientRect();
    this.mouseX = e.clientX - rect.left;
    this.mouseY = e.clientY - rect.top;
  }
};

function canPlacePlant(gridX, gridY) {
  // Check bounds
  if (gridX < 0 || gridX >= GRID_CONFIG.COLUMNS ||
      gridY < 0 || gridY >= GRID_CONFIG.LANES) {
    return false;
  }
  
  // Check if occupied
  return grid[gridY][gridX] === null;
}
```

### Why This Input Structure?
- **State-based**: Selected plant persists between clicks
- **Validation**: `canPlacePlant` is reusable (for UI feedback too)
- **Extensible**: Easy to add right-click cancel, keyboard shortcuts

---

## 9. Game Loop & Update Order

### The Golden Rule of Update Order
**"Dependents must update before dependencies"**

If A affects B, update A first. This prevents frame-lag bugs.

### Update Sequence

```javascript
function gameLoop(currentTime) {
  const deltaTime = currentTime - lastTime;
  lastTime = currentTime;
  
  // 1. UPDATE PHASE (in dependency order)
  // First: Entities that make decisions
  updateZombies(deltaTime, currentTime);
  
  // Second: Entities that react to decisions
  updatePlants(deltaTime, currentTime);
  
  // Third: Entities created by plants
  updateProjectiles(deltaTime);
  
  // Fourth: Cleanup
  cleanupDestroyedEntities();
  
  // 2. SPAWN PHASE
  updateSpawning(currentTime);
  
  // 3. RENDER PHASE
  render();
  
  requestAnimationFrame(gameLoop);
}
```

### Why This Order?

1. **Zombies first**: They move, potentially reaching plants
2. **Plants second**: They check zombie positions, shoot if needed
3. **Projectiles third**: They move, check collisions with zombies (already moved)
4. **Spawn last**: New zombies appear after everything settles

This order ensures:
- Plants shoot at current zombie positions (not last frame's)
- Projectiles hit zombies that have already moved this frame
- No frame-delayed reactions

### Spawning System (v1.0 Simple)

```javascript
const Spawner = {
  spawnInterval: 5000,      // Spawn every 5 seconds for v1.0
  lastSpawn: 0,
  
  update(currentTime) {
    if (currentTime - this.lastSpawn >= this.spawnInterval) {
      this.spawnZombie();
      this.lastSpawn = currentTime;
    }
  },
  
  spawnZombie() {
    const lane = Math.floor(Math.random() * GRID_CONFIG.LANES);
    createZombie('BASIC', lane);
  }
};
```

---

## 10. Rendering System

### Render Order (Back to Front)

```javascript
function render() {
  // Clear canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  
  // 1. Background
  drawBackground();
  drawGrid();
  
  // 2. Plants (behind zombies)
  for (const plant of EntityManager.getByType('plant')) {
    if (plant.active) drawPlant(plant);
  }
  
  // 3. Zombies
  for (const zombie of EntityManager.getByType('zombie')) {
    if (zombie.active) drawZombie(zombie);
  }
  
  // 4. Projectiles
  for (const projectile of EntityManager.getByType('projectile')) {
    if (projectile.active) drawProjectile(projectile);
  }
  
  // 5. UI
  drawUI();
  drawPlacementGhost();
}
```

### Basic Drawing Functions

```javascript
function drawPlant(plant) {
  const { x, y } = plant.transform;
  const { color, radius } = plant.render;
  
  // Simple circle for v1.0
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(x, y, radius, 0, Math.PI * 2);
  ctx.fill();
  
  // Health indicator (simple bar)
  const healthPct = plant.health.current / plant.health.max;
  ctx.fillStyle = healthPct > 0.5 ? '#4caf50' : '#f44336';
  ctx.fillRect(x - 20, y - radius - 10, 40 * healthPct, 5);
}

function drawZombie(zombie) {
  const { x, y } = zombie.transform;
  const { color } = zombie.render;
  
  // Rectangle for v1.0
  ctx.fillStyle = color;
  ctx.fillRect(x - 20, y - 35, 40, 70);
  
  // Eyes looking left
  ctx.fillStyle = 'white';
  ctx.fillRect(x - 15, y - 20, 10, 10);
  ctx.fillRect(x - 15, y, 10, 10);
  ctx.fillStyle = 'black';
  ctx.fillRect(x - 12, y - 17, 4, 4);
  ctx.fillRect(x - 12, y + 3, 4, 4);
}

function drawProjectile(projectile) {
  const { x, y } = projectile.transform;
  const { color, radius } = projectile.render;
  
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(x, y, radius, 0, Math.PI * 2);
  ctx.fill();
}
```

### Why Simple Shapes?
- **v1.0 focus**: Get game loop working, visuals come later
- **Clear hitboxes**: Circle/rect collision is obvious
- **Performance**: No image loading, no sprite sheets

---

## 11. File Structure (v1.0)

```
index.html
├── style.css (minimal styles)
└── js/
    ├── config.js          # GRID_CONFIG, PLANT_TYPES, ZOMBIE_TYPES
    ├── utils.js           # gridToPixel, pixelToGrid, etc.
    ├── entityManager.js   # Entity creation/destruction
    ├── plant.js           # Plant factory, update, render
    ├── zombie.js          # Zombie factory, update (state machine), render
    ├── projectile.js      # Projectile factory, update, render
    ├── combat.js          # damageEntity, killEntity
    ├── input.js           # Mouse handling
    ├── spawner.js         # Zombie spawning logic
    └── game.js            # Main loop, init
```

### Why This Structure?
- **Single responsibility**: Each file does one thing
- **Clear imports**: game.js imports everything, orchestrates
- **Easy to test**: Each module can be tested independently
- **Future-proof**: Adding sun system = new file, doesn't touch existing

---

## 12. Key Design Decisions Summary

| Decision | Rationale |
|----------|-----------|
| Plain objects for entities | Serializable, flexible, fast property access |
| Separate Maps per entity type | O(1) lookup, cache-friendly iteration |
| Grid array for placement | O(1) plant lookup by position |
| State machine for zombies | Clear behavior, prevents bugs, extensible |
| Data-driven type definitions | Balance without code changes |
| Unified damageEntity() | Consistent handling, one place for effects |
| Update order: zombies → plants → projectiles | Dependencies update before dependents |
| Functional factories | Pure creation logic, easy to test |

---

## 13. v1.0 Success Criteria

The game is complete when:

- [ ] Grid displays as 5x9 cells
- [ ] Clicking places peashooters (free, unlimited)
- [ ] Peashooters shoot peas when zombie in lane
- [ ] Peas move right-to-left and damage zombies
- [ ] Zombies spawn periodically from right
- [ ] Zombies walk left, stop at plants, eat them
- [ ] Plants die when health reaches 0
- [ ] Zombies die when health reaches 0
- [ ] Game ends when zombie reaches left edge
- [ ] Solid 60fps with 20+ zombies and 10+ plants

Once v1.0 works, we add:
- Sun economy (v1.1)
- Multiple plant types (v1.2)
- Multiple zombie types (v1.3)
- Lawn mowers (v1.4)
- Visual polish (v2.0)
