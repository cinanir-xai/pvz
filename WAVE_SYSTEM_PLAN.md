# Wave System Plan - Plants vs Zombies v3.0

## Overview
This document outlines the architecture for a progressive wave-based level system inspired by the original Plants vs Zombies. The system features ramping difficulty, three major boss waves, and dynamic spawning based on player performance.

---

## 1. Core Wave Mechanics

### 1.1 Wave Trigger System

```javascript
const WAVE_SYSTEM = {
  // Wave state tracking
  currentWave: 0,
  waveActive: false,
  zombiesSpawnedThisWave: 0,
  zombiesKilledThisWave: 0,
  
  // Spawn timing
  spawnInterval: 0,        // Dynamic based on wave
  lastSpawnTime: 0,
  
  // Wave progression threshold
  NEXT_WAVE_THRESHOLD: 0.5,  // 50% of wave must die before next wave
  
  // Boss wave flags
  isBossWave: false,
  bossWaveSpawned: false,
  
  // Wave configurations
  waveConfig: [...]  // See section 2
};
```

### 1.2 Wave State Machine

```
IDLE → SPAWNING → ACTIVE → THRESHOLD_REACHED → SPAWNING_NEXT
  ↑                                              ↓
  └──────────────── LEVEL_COMPLETE ←─────────────┘
```

**States:**
- **IDLE**: Before level starts, waiting for initial trigger
- **SPAWNING**: Currently spawning zombies for this wave
- **ACTIVE**: Wave fully spawned, waiting for threshold
- **THRESHOLD_REACHED**: 50% of wave killed, preparing next wave
- **LEVEL_COMPLETE**: All waves completed (victory)

### 1.3 Progression Logic

```javascript
function updateWaveSystem(currentTime) {
  // Check if we should spawn next zombie in current wave
  if (shouldSpawnZombie(currentTime)) {
    spawnNextZombieInWave();
  }
  
  // Check wave progression threshold
  if (canProgressToNextWave()) {
    startNextWave();
  }
}

function canProgressToNextWave() {
  if (WaveSystem.isBossWave && !WaveSystem.bossWaveSpawned) {
    return false; // Boss waves spawn all at once
  }
  
  const killRatio = WaveSystem.zombiesKilledThisWave / WaveSystem.zombiesSpawnedThisWave;
  return killRatio >= WAVE_SYSTEM.NEXT_WAVE_THRESHOLD;
}
```

---

## 2. Wave Composition & Difficulty Curve

### 2.1 Wave Configuration Structure

```javascript
const WAVE_CONFIG = [
  // Wave 1: Tutorial - Single basic zombie
  {
    waveNumber: 1,
    type: 'normal',
    totalZombies: 1,
    composition: { basic: 1, conehead: 0, buckethead: 0 },
    spawnDelay: 2000,      // ms between spawns
    preWaveDelay: 1000,    // ms before wave starts
    lanePattern: 'center'  // Spawn in center lane for visibility
  },
  
  // Wave 2: Introduction to multiple zombies
  {
    waveNumber: 2,
    type: 'normal',
    totalZombies: 3,
    composition: { basic: 3, conehead: 0, buckethead: 0 },
    spawnDelay: 2500,
    preWaveDelay: 3000,
    lanePattern: 'random'
  },
  
  // Wave 3: Slightly more pressure
  {
    waveNumber: 3,
    type: 'normal',
    totalZombies: 5,
    composition: { basic: 5, conehead: 0, buckethead: 0 },
    spawnDelay: 2200,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 4: Building up to first boss
  {
    waveNumber: 4,
    type: 'normal',
    totalZombies: 7,
    composition: { basic: 7, conehead: 0, buckethead: 0 },
    spawnDelay: 2000,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 5: BOSS WAVE - Horde of basics
  {
    waveNumber: 5,
    type: 'boss',
    totalZombies: 15,
    composition: { basic: 15, conehead: 0, buckethead: 0 },
    spawnDelay: 800,       // Fast spawning for horde feel
    preWaveDelay: 5000,    // Dramatic pause before boss wave
    lanePattern: 'all',    // Spawn in all lanes simultaneously
    special: 'A HUGE WAVE OF ZOMBIES IS APPROACHING!'
  },
  
  // Wave 6: Introduction of coneheads
  {
    waveNumber: 6,
    type: 'normal',
    totalZombies: 6,
    composition: { basic: 4, conehead: 2, buckethead: 0 },
    spawnDelay: 2000,
    preWaveDelay: 3000,
    lanePattern: 'random'
  },
  
  // Wave 7: Mixed pressure
  {
    waveNumber: 7,
    type: 'normal',
    totalZombies: 8,
    composition: { basic: 5, conehead: 3, buckethead: 0 },
    spawnDelay: 1800,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 8: More coneheads
  {
    waveNumber: 8,
    type: 'normal',
    totalZombies: 10,
    composition: { basic: 5, conehead: 5, buckethead: 0 },
    spawnDelay: 1700,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 9: Heavy pressure building
  {
    waveNumber: 9,
    type: 'normal',
    totalZombies: 12,
    composition: { basic: 6, conehead: 6, buckethead: 0 },
    spawnDelay: 1500,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 10: BOSS WAVE - Conehead invasion
  {
    waveNumber: 10,
    type: 'boss',
    totalZombies: 18,
    composition: { basic: 8, conehead: 10, buckethead: 0 },
    spawnDelay: 700,
    preWaveDelay: 5000,
    lanePattern: 'all',
    special: 'A HUGE WAVE OF ZOMBIES IS APPROACHING!'
  },
  
  // Wave 11: Introduction of bucketheads
  {
    waveNumber: 11,
    type: 'normal',
    totalZombies: 10,
    composition: { basic: 5, conehead: 3, buckethead: 2 },
    spawnDelay: 1800,
    preWaveDelay: 3000,
    lanePattern: 'random'
  },
  
  // Wave 12: Triple threat begins
  {
    waveNumber: 12,
    type: 'normal',
    totalZombies: 12,
    composition: { basic: 5, conehead: 4, buckethead: 3 },
    spawnDelay: 1600,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 13: Heavy mixed wave
  {
    waveNumber: 13,
    type: 'normal',
    totalZombies: 14,
    composition: { basic: 5, conehead: 5, buckethead: 4 },
    spawnDelay: 1400,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 14: Maximum pressure before final
  {
    waveNumber: 14,
    type: 'normal',
    totalZombies: 16,
    composition: { basic: 5, conehead: 5, buckethead: 6 },
    spawnDelay: 1200,
    preWaveDelay: 2000,
    lanePattern: 'random'
  },
  
  // Wave 15: FINAL BOSS WAVE - Everything
  {
    waveNumber: 15,
    type: 'boss',
    totalZombies: 25,
    composition: { basic: 10, conehead: 8, buckethead: 7 },
    spawnDelay: 500,       // Very fast spawning
    preWaveDelay: 8000,    // Long dramatic pause
    lanePattern: 'all',
    special: 'FINAL WAVE!'
  }
];
```

### 2.2 Difficulty Progression Analysis

| Wave Range | Total Zombies | New Element | Difficulty Rating |
|------------|---------------|-------------|-------------------|
| 1-4 | 1-7 | Learning basics | ★☆☆☆☆ |
| 5 (Boss) | 15 | Horde mechanic | ★★★☆☆ |
| 6-9 | 6-12 | Coneheads introduced | ★★☆☆☆ |
| 10 (Boss) | 18 | Heavy coneheads | ★★★★☆ |
| 11-14 | 10-16 | Bucketheads introduced | ★★★☆☆ |
| 15 (Final) | 25 | All types, fast spawn | ★★★★★ |

---

## 3. Spawning Logic & Algorithms

### 3.1 Zombie Type Selection

```javascript
function selectZombieTypeForWave(waveConfig) {
  const { composition, zombiesSpawned } = waveConfig;
  const remaining = {
    basic: composition.basic - zombiesSpawned.basic,
    conehead: composition.conehead - zombiesSpawned.conehead,
    buckethead: composition.buckethead - zombiesSpawned.buckethead
  };
  
  // Weighted random selection based on remaining zombies
  const totalRemaining = remaining.basic + remaining.conehead + remaining.buckethead;
  const rand = Math.random() * totalRemaining;
  
  if (rand < remaining.buckethead) return 'BUCKETHEAD';
  if (rand < remaining.buckethead + remaining.conehead) return 'CONEHEAD';
  return 'BASIC';
}
```

### 3.2 Lane Selection Patterns

```javascript
const LANE_PATTERNS = {
  // Random lane each time
  random: () => Math.floor(Math.random() * LAYOUT.LANES),
  
  // Center lane only (for tutorial)
  center: () => 2,  // Lane 3 (0-indexed)
  
  // Spawn in all lanes simultaneously for boss waves
  all: (spawnIndex, totalSpawns) => {
    // Distribute evenly across lanes
    return spawnIndex % LAYOUT.LANES;
  },
  
  // Concentrated (zombies cluster in adjacent lanes)
  concentrated: () => {
    const centerLane = Math.floor(LAYOUT.LANES / 2);
    const offset = Math.floor(Math.random() * 3) - 1; // -1, 0, or 1
    return Math.max(0, Math.min(LAYOUT.LANES - 1, centerLane + offset));
  }
};
```

### 3.3 Wave State Tracking

```javascript
const WaveTracker = {
  // Current wave info
  waveNumber: 0,
  zombiesSpawned: 0,
  zombiesKilled: 0,
  zombiesOnField: 0,
  
  // Spawn tracking for composition
  spawnCounts: {
    basic: 0,
    conehead: 0,
    buckethead: 0
  },
  
  // Wave timing
  waveStartTime: 0,
  lastSpawnTime: 0,
  
  // Boss wave specific
  bossWaveActive: false,
  bossWaveComplete: false,
  
  reset() {
    this.waveNumber = 0;
    this.zombiesSpawned = 0;
    this.zombiesKilled = 0;
    this.zombiesOnField = 0;
    this.spawnCounts = { basic: 0, conehead: 0, buckethead: 0 };
    this.bossWaveActive = false;
    this.bossWaveComplete = false;
  },
  
  onZombieSpawned(type) {
    this.zombiesSpawned++;
    this.zombiesOnField++;
    this.spawnCounts[type.toLowerCase()]++;
  },
  
  onZombieKilled() {
    this.zombiesKilled++;
    this.zombiesOnField--;
  }
};
```

---

## 4. UI & Visual Feedback

### 4.1 Wave Progress Bar

```javascript
function drawWaveProgress(ctx) {
  const barWidth = 200;
  const barHeight = 20;
  const x = DESIGN_WIDTH / 2 - barWidth / 2;
  const y = 10;
  
  // Background
  ctx.fillStyle = '#333';
  ctx.fillRect(x, y, barWidth, barHeight);
  
  // Progress fill (based on current wave progress)
  const currentWaveConfig = WAVE_CONFIG[WaveTracker.waveNumber - 1];
  const progress = WaveTracker.zombiesSpawned / currentWaveConfig.totalZombies;
  
  ctx.fillStyle = WaveTracker.bossWaveActive ? '#ff5722' : '#4caf50';
  ctx.fillRect(x, y, barWidth * progress, barHeight);
  
  // Border
  ctx.strokeStyle = '#fff';
  ctx.lineWidth = 2;
  ctx.strokeRect(x, y, barWidth, barHeight);
  
  // Wave number text
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 14px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText(`Wave ${WaveTracker.waveNumber}/15`, x + barWidth / 2, y + 15);
}
```

### 4.2 Boss Wave Warning

```javascript
function drawBossWaveWarning(ctx, currentTime) {
  const warningDuration = 5000; // 5 seconds
  const elapsed = currentTime - WaveTracker.waveStartTime;
  
  if (elapsed > warningDuration) return;
  
  // Flashing effect
  const flash = Math.sin(elapsed * 0.01) > 0;
  const alpha = 1 - (elapsed / warningDuration);
  
  ctx.save();
  ctx.globalAlpha = alpha;
  
  // Red warning background
  ctx.fillStyle = flash ? '#d32f2f' : '#b71c1c';
  ctx.fillRect(0, DESIGN_HEIGHT / 2 - 40, DESIGN_WIDTH, 80);
  
  // Warning text
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 36px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  
  const waveConfig = WAVE_CONFIG[WaveTracker.waveNumber - 1];
  ctx.fillText(waveConfig.special || 'HUGE WAVE!', DESIGN_WIDTH / 2, DESIGN_HEIGHT / 2);
  
  ctx.restore();
}
```

### 4.3 Wave Completion Notification

```javascript
function showWaveCompleteNotification(waveNumber) {
  // Brief "Wave X Complete!" text that fades out
  // Triggers when 50% threshold is reached
}
```

---

## 5. Balance Considerations

### 5.1 Plant Cost-Benefit Analysis

| Plant | Cost | Effectiveness | ROI Analysis |
|-------|------|---------------|--------------|
| Sunflower | 50 | Economy | Essential early game, pays for itself in 1 sun drop |
| Peashooter | 100 | Single target | Good DPS vs basic, slow vs armored |
| Wall-nut | 50 | Tank | Incredible value, buys 13+ seconds vs basic zombie |

### 5.2 Zombie Threat Assessment

| Zombie | HP | Time to Kill (1 Pea) | Threat Level |
|--------|-----|---------------------|--------------|
| Basic | 200 | 10 peas = 14s | Low |
| Conehead | 640 | 32 peas = 45s | Medium |
| Buckethead | 1370 | 69 peas = 96s | High |

### 5.3 Wave Timing Balance

- **Early waves (1-4)**: Slow spawns (2-2.5s), allow player to build economy
- **Mid waves (6-9)**: Medium spawns (1.5-2s), test basic defense
- **Boss waves (5, 10, 15)**: Fast spawns (0.5-0.8s), test panic response
- **Late waves (11-14)**: Fast spawns (1.2-1.8s), sustained pressure

### 5.4 Resource Economy Flow

```
Wave 1: Start with 150 sun → Can buy 3 sunflowers or mix
Wave 2-4: Natural sun + sunflower production ≈ 100-150 sun per wave
Wave 5+: Must have 3+ sunflowers to afford defense
Wave 10+: Need 5+ sunflowers for buckethead defense
Wave 15: Maximum economy required, all lanes must have defense
```

---

## 6. Implementation Roadmap

### Phase 1: Core Wave System
1. Create WaveTracker state object
2. Implement wave configuration data structure
3. Build wave progression logic (50% threshold)
4. Add wave state machine

### Phase 2: Spawning Logic
1. Implement zombie type selection algorithm
2. Create lane selection patterns
3. Build spawn timing system
4. Integrate with existing Spawner

### Phase 3: Boss Waves
1. Special boss wave handling (all-at-once spawn)
2. Warning banner system
3. Dramatic pre-wave delays
4. Lane-filling spawn patterns

### Phase 4: UI Integration
1. Wave progress bar
2. Boss wave warning display
3. Wave completion notifications
4. Final wave victory condition

### Phase 5: Balancing & Testing
1. Playtest wave 1-5 for early game balance
2. Playtest wave 6-10 for mid game difficulty spike
3. Playtest wave 11-15 for end game challenge
4. Adjust spawn timings based on feedback

---

## 7. Key Design Decisions

### Why 50% Threshold?
- Prevents player from being overwhelmed
- Rewards efficient play (kill fast = next wave faster)
- Maintains pressure without being unfair
- Allows recovery time if struggling

### Why 15 Waves?
- 5-8 minutes of gameplay (appropriate for web game)
- Three distinct phases (basic → conehead → buckethead)
- Three boss waves provide pacing beats
- Not too long to lose player interest

### Why This Spawn Pattern?
- Wave 1: Single zombie teaches mechanics
- Wave 5: First "oh no" moment with horde
- Wave 10: Second difficulty spike with coneheads
- Wave 15: Final exam requiring all skills

### Boss Wave Philosophy
- Spawn all zombies quickly for visual impact
- Create panic moments that test player composure
- Provide memorable set-piece encounters
- Break up normal wave rhythm

---

## Summary

This wave system creates a curated difficulty curve that:
1. **Teaches** mechanics in early waves (1-4)
2. **Tests** economy management in mid-game (6-9)
3. **Challenges** defensive positioning in late-game (11-14)
4. **Rewards** mastery in boss waves (5, 10, 15)

The 50% threshold system ensures the game adapts to player skill while the wave composition gradually introduces new threats. Players must learn to manage sun economy, lane coverage, and plant selection to survive all 15 waves.
