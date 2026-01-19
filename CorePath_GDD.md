# CorePath
## Game Design Document v0.5

**Last Updated:** January 2026  
**Status:** Pre-Production / Design Phase

---

## Table of Contents
1. [Core Concept](#core-concept)
2. [Game Modes](#game-modes)
3. [Core Mechanics](#core-mechanics)
4. [Tower System](#tower-system)
5. [Upgrade System](#upgrade-system)
6. [Progression System](#progression-system)
7. [Map Generation](#map-generation)
8. [Enemy Types](#enemy-types)
9. [Economy System](#economy-system)
10. [Visual Theme](#visual-theme)
11. [Technical Considerations](#technical-considerations)
12. [Open Questions & Future Features](#open-questions--future-features)

---

## Core Concept

A single-screen, roguelike tower defense game where geometric enemies attempt to steal cores from your base. Players must build efficient mazes using randomly available tower types, balancing minimal spending with maximum defense to maximize end-of-wave bonus multipliers.

**Key Inspirations:**
- Defense Grid / Defense Grid 2 (core stealing, path manipulation)
- Geometry Wars (visual aesthetic)
- Roguelike elements (random tower selection, meta-progression)

**Target Platform:** Mobile (primary), with potential web/desktop versions

**Core Loop:**
1. Waves of enemies spawn and path to your base
2. Build towers to damage enemies AND create maze walls
3. Enemies steal cores and slowly carry them back
4. Destroy enemies to recover cores
5. Survive wave, earn bonus based on money × cores remaining
6. Spend money on new towers or upgrades
7. Repeat until victory or defeat

---

## Game Modes

### Short Game (10 Waves)
- **Duration:** ~10-15 minutes
- **Difficulty:** Linear scaling
- **Purpose:** Quick sessions, unlock progression rewards
- **Use Case:** Casual play, unlocking new towers/upgrades

### Medium Game (15 Waves)
- **Duration:** ~20-25 minutes
- **Difficulty:** Linear scaling
- **Purpose:** Standard experience, better rewards
- **Use Case:** Main gameplay loop

### Endless Mode
- **Duration:** Until defeat
- **Difficulty:** Exponential scaling
- **Purpose:** Challenge/Leaderboard mode
- **Special Rule:** Player selects their tower loadout (no RNG)
- **Use Case:** Skill testing with known tools

---

## Core Mechanics

### The Cores System

**Core Behavior:**
- **Starting Count:** 20 cores at base
- **Location:** Top of map in base zone (4×5 grid pattern)
- **Enemy Interaction:**
  - Enemies path to base to grab cores
  - Instant pickup (no channel time, just walk over)
  - Enemy immediately turns toward exit with stolen core
  - Enemies can only carry ONE core each
  
**Core Movement When Stolen:**
- **Speed:** Attached to enemy, moves at enemy's normal walking speed
- **No Catching:** Players cannot intercept cores in transit
- **Must Kill Enemy:** Only way to recover is destroying the carrier

**Core Recovery:**
- Destroyed enemy drops carried core at death location
- Dropped cores float back to base automatically
- **Float Speed:** 0.5 tiles/second
- Recovered cores immediately count for end-wave bonus

**Core Loss:**
- If enemy reaches spawn/exit with core, it's permanently lost
- Lost cores reduce multiplier for ALL remaining waves
- Cannot be recovered once exited

**Strategic Importance:**
- Each lost core permanently reduces bonus multiplier
- Creates tension: aggressive defense vs. accepting losses
- Rewards efficient tower placement (fewer towers = more money = bigger bonus)
- Risk/reward: cheap defense vs. protecting maximum multiplier

### End-of-Wave Bonus Formula
```
Bonus = Current Money × Cores at Base × 0.10
Example: $5,000 × 18 cores = $9,000 bonus
```

**Strategic Implications:**
- Incentivizes minimal tower spending early game
- Every tower reduces future bonus potential
- Lost cores hurt exponentially (less multiplier forever)
- Creates risk/reward: save money vs. guarantee defense

### Path Manipulation

**Core Mechanic:**
- Enemies use pathfinding (shortest route to base/exit)
- Towers are impassable obstacles
- Players create mazes to extend enemy travel time
- Longer paths = more time under fire = more efficient towers

**Pathfinding System (Technical):**

**Global Path Calculation:**
- Two paths maintained at world level:
  - **Path A:** Spawn → Base (for enemies seeking cores)
  - **Path B:** Base → Exit (for enemies carrying cores)
- Recalculated on EVERY tower placement
- Single-file optimal route (enemies follow center line)
- Shortest path algorithm (A* pathfinding)
- Deterministic (same layout = same path)

**Pre-Placement Validation:**
```javascript
onTowerPlaceAttempt(x, y):
  tempWorld = world.clone()
  tempWorld.addTower(x, y)
  
  // Check if valid path still exists
  if (!pathExists(spawn, base, tempWorld)):
    showError("Cannot block all paths!")
    return false // Prevent placement
  
  // Valid placement
  world.addTower(x, y)
  recalculateGlobalPaths()
  return true
```

**Enemy Pathfinding Behavior:**
- Enemies follow global path when on it
- If off-path (spawned, or path changed mid-movement):
  - Run quick local A* to nearest point on appropriate global path
  - Resume following global path from that point forward
- Enemies pass through each other (no collision detection)
- No slowdown from crowding at chokepoints
- Smooth pathing (no jittering or stuttering)

**Path Changes Mid-Wave:**
- When tower placed, global paths immediately recalculate
- All enemies update their target path instantly
- Enemies currently off-path: pathfind to nearest point on new path
- Enemies on old path: immediately adopt new path from current position
- Creates dynamic moments where enemies suddenly turn

**Performance Considerations:**
- Implement simple recalc-on-every-placement first
- Monitor performance with 100+ enemies on screen
- Backup plan: Add 0.5 second debounce if lag occurs
- Global pathfinding is cheap (runs once per tower)
- Local enemy pathfinding is cheap (only for off-path enemies)

**Critical Rule:**
- Player CANNOT fully block all paths
- At least one route must always exist to base
- Validation prevents impossible situations
- Encourages creative maze design within constraints

**Strategic Depth:**
- Maze design multiplies tower effectiveness
- Cheap towers as walls vs. expensive damage dealers
- Bridge chokepoints become critical control points
- Requires planning ahead for future tower placement
- Reactive building can redirect enemy flow mid-wave

### Wave Structure

**Between Waves:**
- **Prep Time:** 5-10 seconds between waves
- Purpose: Brief pause for towers to refocus and retarget
- Players can still build/upgrade during prep
- No "ready" button - automatic start after timer

**Enemy Spawning:**
- Enemies spawn in gradual trickle
- **Spawn Rate:** 1 enemy per 0.5 seconds (balanced flow)
- Creates natural spacing for towers to engage
- Prevents overwhelming simultaneous arrivals

**Dynamic Difficulty Scaling:**
- System checks player's current money at start of each wave
- Compares to expected money for that wave/playstyle
- Adjusts wave difficulty based on performance

**Scaling Formula:**
```javascript
expectedMoney = startingMoney + (wave × 500) + (wave × 1000) // kills + bonuses
performanceRatio = currentMoney / expectedMoney

if (performanceRatio > 1.3):
  // Player hoarding money, being too efficient
  enemyCount *= 1.15 // +15% more enemies
  addHarderEnemyTypes() // More Tanks/Heavies
  
if (performanceRatio < 0.7):
  // Player struggling, spent everything
  enemyCount *= 0.9 // -10% fewer enemies
  reduceHarderEnemyTypes() // Fewer Tanks/Heavies
```

**Purpose:**
- Encourages spending money (hoarding = harder waves)
- Helps struggling players (spent everything = slight relief)
- Prevents dominant "save everything for bonus" strategy
- Creates organic difficulty curve that adapts to skill
- Subtle nudges, not dramatic swings

**During Waves:**
- Can build and upgrade towers mid-wave (any time)
- Must react to cores being stolen
- Strategic tower placement crucial
- Wave ends when: all enemies destroyed OR all enemies exited

**Wave Progression:**
- Enemy count increases each wave
- Enemy health increases each wave
- Enemy speed increases each wave
- New enemy types introduced every 3-5 waves
- Combined enemy types in later waves (mixed challenges)

**Victory & Defeat Conditions:**

**Victory (Short/Medium):**
- Survive all waves (10 or 15)
- Must have at least 1 core remaining when last wave ends
- Last wave ends when: last enemy dies OR last enemy exits
- Score calculated from final money total

**Defeat:**
- Lose ALL 20 cores (all stolen and exited)
- Immediately ends run
- No final wave completion required
- Can happen mid-wave

**Score Calculation:**
- Final Score = Total Money Earned
- Includes: starting money + enemy kills + all wave bonuses
- Converted to BP at 1% rate at results screen

### Roguelike Elements

**Random Map Generation:**
- Core placement patterns vary
- Block layouts randomized (1-3 blocks)
- Bridge positions change
- No two runs identical

**Random Tower Selection:**
- Each run: 3-5 tower types from unlocked pool
- Must adapt strategy to available tools
- Luck plays a role, skill in adaptation crucial

**Meta-Progression:**
- Permanent unlocks between runs
- Build custom tower variants over time
- Unlock new tower types
- Gradually become more powerful

---

## Tower System

### Tower Stats & Balance (Prototype Values)

#### Gun Tower ($500)
- **Damage:** 20 per shot
- **Fire Rate:** 2 shots/second (40 DPS)
- **Range:** 3 tiles
- **Projectile Speed:** Fast
- **Target Priority:** First in range
- **Best Against:** Consistent damage, reliable all-arounder

#### AOE/Mortar Tower ($700)
- **Damage:** 15 per hit
- **Fire Rate:** 0.5 shots/second
- **Splash Radius:** 1.5 tiles
- **Range:** 5 tiles
- **Effective DPS:** 30-50 (depending on grouping)
- **Target Priority:** Highest enemy density
- **Best Against:** Grouped enemies, Swarms

#### Contact/Grinder Tower ($400)
- **Damage:** 50 per second
- **Fire Rate:** Continuous (to enemies in contact)
- **Range:** 0 tiles (contact only)
- **Area:** 1 tile
- **Highest DPS:** Requires maze design
- **Best Against:** Forced pathing situations, chokepoints

#### Long Range/Sniper Tower ($600)
- **Damage:** 100 per shot
- **Fire Rate:** 0.5 shots/second (50 DPS)
- **Range:** 7 tiles
- **Target Priority:** Strongest enemy
- **Best Against:** Tanks, Heavies, high-value targets

#### Support/Slower Tower ($400)
- **Damage:** 0
- **Slow Effect:** 40% speed reduction
- **Range:** 3 tiles
- **No DPS:** Pure support
- **Best With:** Combo with high-DPS towers
- **Best Against:** Fast enemies, buying time

### Tower Placement Rules

**Selling/Refunding Towers:**
- Can sell towers at any time
- Refund: 75% of original purchase price
- Upgrades are lost (not refunded separately)
- Encourages experimentation but with cost

**Movement:**
- Cannot move towers directly
- Must sell and rebuild in new location
- Strategic commitment matters

**Starting Placement:**
- Players choose all tower placements
- No pre-placed or fixed towers
- Full freedom from wave 1

---

## Core Mechanics

### Base Tower Types (5 Classes)

#### 1. Gun Tower
- **Type:** Single-target, balanced DPS
- **Range:** Medium
- **Fire Rate:** Steady
- **Damage:** Moderate
- **Cost:** $500 base

#### 2. AOE/Mortar Tower
- **Type:** Area damage, splash radius
- **Range:** Medium-Long
- **Fire Rate:** Slow
- **Damage:** High to group
- **Cost:** $700 base

#### 3. Contact/Grinder Tower
- **Type:** Zero range, continuous physical damage
- **Range:** Contact only (enemies must pass through)
- **Fire Rate:** Constant
- **Damage:** Very high (to enemies in contact)
- **Cost:** $400 base
- **Note:** Requires maze design to force enemies past it multiple times

#### 4. Long Range/Sniper Tower
- **Type:** Long range, slow fire, high damage
- **Range:** Very Long
- **Fire Rate:** Very slow
- **Damage:** Very high single-target
- **Cost:** $600 base

#### 5. Support/Slower Tower
- **Type:** Slows enemies in radius
- **Range:** Medium
- **Effect:** Reduces enemy speed
- **Damage:** None (pure support)
- **Cost:** $400 base

### Tower Selection Per Run
- Roguelike: 3-5 random towers from unlocked pool
- Endless Mode: Player chooses loadout
- Must work with what you're given
- Encourages unlocking variety

---

## Upgrade System

### Universal Upgrades (All Towers)

#### Range
- **Effect:** Increases attack/effect radius
- **Base Cost:** $200
- **Levels:**
  - Level I: +20% range
  - Level II: +40% range
  - Level III: +60% range

#### Damage
- **Effect:** Increases damage output
- **Base Cost:** $300
- **Levels:**
  - Level I: +25% damage
  - Level II: +50% damage
  - Level III: +100% damage

### Tower-Specific Upgrades

#### Gun Tower
**Fire Rate**
- Base Cost: $250
- Level I: +30% fire rate
- Level II: +60% fire rate
- Level III: +100% fire rate

**Penetration**
- Base Cost: $400
- Level I: Shots pierce 2 enemies
- Level II: Shots pierce 3 enemies

**Armor Pierce**
- Base Cost: $350
- Level I: +25% vs. tanks
- Level II: +50% vs. tanks
- Level III: +100% vs. tanks

#### AOE/Mortar Tower
**Splash Radius**
- Base Cost: $300
- Level I: +30% explosion area
- Level II: +60% explosion area
- Level III: +100% explosion area

**Burn**
- Base Cost: $400
- Level I: Light DoT (5 dmg/sec for 3 sec)
- Level II: Medium DoT (10 dmg/sec for 4 sec)
- Level III: Heavy DoT (15 dmg/sec for 5 sec)

**Cluster Bomb**
- Base Cost: $500
- Level I: Splits into 3 sub-projectiles
- Level II: Splits into 5 sub-projectiles

#### Contact/Grinder Tower
**Spin Speed**
- Base Cost: $250
- Level I: +40% damage rate
- Level II: +80% damage rate
- Level III: +120% damage rate

**Knockback**
- Base Cost: $350
- Level I: Small push (enemies take +1 hit)
- Level II: Medium push (enemies take +2 hits)

**Shred Armor**
- Base Cost: $400
- Level I: -5% defense per hit (stacks)
- Level II: -10% defense per hit (stacks)
- Level III: -15% defense per hit (stacks)

#### Long Range/Sniper Tower
**Crit Chance**
- Base Cost: $350
- Level I: 15% crit (3x damage)
- Level II: 30% crit (3x damage)
- Level III: 50% crit (3x damage)

**Execute**
- Base Cost: $400
- Level I: +50% dmg vs. <50% health enemies
- Level II: +100% dmg vs. <40% health enemies
- Level III: +150% dmg vs. <30% health enemies

**Headshot**
- Base Cost: $500
- Level I: 10% instant kill (basic enemies)
- Level II: 25% instant kill (basic enemies)

#### Support/Slower Tower
**Slow Amount**
- Base Cost: $200
- Level I: 30% slow
- Level II: 50% slow
- Level III: 70% slow

**Sticky Slow**
- Base Cost: $300
- Level I: Slow persists 1s after leaving range
- Level II: Slow persists 2s after leaving range
- Level III: Slow persists 3s after leaving range

**Weakening**
- Base Cost: $350
- Level I: Slowed enemies take +15% damage
- Level II: Slowed enemies take +30% damage
- Level III: Slowed enemies take +50% damage

### Upgrade Cost Scaling (During Run)

**Exponential Formula:**
```
Cost = Base Cost × 2^(level - 1)

Example (Damage upgrade, $300 base):
Level I:   $300  (×1)
Level II:  $600  (×2)
Level III: $1200 (×4)
Level IV:  $2400 (×8)
Level V:   $4800 (×16)
...continues infinitely
```

**Diminishing Returns (Per Tier):**
```
Tier I:    +25% (100% effectiveness)
Tier II:   +22% (88% effectiveness)
Tier III:  +19% (76%)
Tier IV:   +16% (64%)
Tier V:    +14% (56%)
Tier VI:   +12% (48%)
Tier VII:  +10% (40%)
Tier VIII: +9%  (36%)
Tier IX:   +8%  (32%)
Tier X:    +7%  (28%)
...continues diminishing toward ~+5% minimum per tier
```

**No Hard Cap:**
- Players can theoretically reach Damage XX or higher
- Exponential costs create natural limits
- Always something to chase in progression

**Purpose:**
- Prevents infinite scaling exploits via diminishing returns
- Encourages diversified upgrades (better ROI)
- Maintains challenge in Endless mode
- Provides infinite progression goals

---

## Progression System

### Blueprint Point (BP) Economy

**End-of-Run Conversion:**
```
Final Money → Blueprint Points at 1% rate

Examples:
$20,000 → 200 BP (decent Short run)
$50,000 → 500 BP (good Medium run)
$100,000 → 1,000 BP (excellent Medium run)
$150,000 → 1,500 BP (perfect optimized run)
```

**Display:**
- Shown only at end-of-run results screen
- NOT displayed during gameplay
- Keeps focus on strategy, not currency grinding

**Starting State:**
- New players start with 0 BP
- Gun Tower unlocked
- AOE Tower unlocked
- All upgrades at base level (none retained)
- Must complete first run to begin progression

### Permanent Retention System

**How It Works:**
- After each run, spend BP to permanently "retain" upgrades
- Retained upgrades are permanently unlocked
- No refunds - choices are permanent
- Build your custom tower arsenal over time

**What Retaining Does:**
- Unlocks that upgrade tier for future runs
- During runs, that upgrade becomes available to purchase
- Does NOT give passive bonuses (balanced approach)
- Must still purchase upgrades in-run with money

**Strategic Value:**
- Higher tier unlocks allow stronger builds
- More options = more versatility = better adaptation
- Unlock synergies between upgrades
- Specialize towers to your playstyle

### Retention Cost Formula (Unlimited Tiers)

**Base Formula:**
```
Retention Cost = (150 BP × 1.5^(tier - 1)) × (1 + Total Retained × 0.15)

Where:
- 150 BP = base cost for all upgrades
- 1.5^(tier - 1) = tier exponential multiplier
- Total Retained = count of ALL retained upgrades across everything
```

**Tier Costs (Before Retention Scaling):**
```
Tier I:    150 BP
Tier II:   225 BP
Tier III:  338 BP
Tier IV:   506 BP
Tier V:    759 BP
Tier VI:   1,139 BP
Tier VII:  1,708 BP
Tier VIII: 2,563 BP
Tier IX:   3,844 BP
Tier X:    5,766 BP
Tier XI:   8,649 BP
Tier XII:  12,974 BP
...continues infinitely
```

**With Retention Scaling Examples:**

**Early Game (5 total retained):**
```
Multiplier = 1 + (5 × 0.15) = 1.75

Tier I:   150 × 1.75 = 263 BP (~1 run)
Tier III: 338 × 1.75 = 591 BP (~1-2 runs)
Tier V:   759 × 1.75 = 1,328 BP (~2-3 runs)
```

**Mid Game (20 total retained):**
```
Multiplier = 1 + (20 × 0.15) = 4.0

Tier I:   150 × 4.0 = 600 BP (~1-2 runs)
Tier V:   759 × 4.0 = 3,036 BP (~3-6 runs)
Tier VII: 1,708 × 4.0 = 6,832 BP (~7-14 runs)
```

**Late Game (50 total retained):**
```
Multiplier = 1 + (50 × 0.15) = 8.5

Tier III: 338 × 8.5 = 2,873 BP (~3-6 runs)
Tier VII: 1,708 × 8.5 = 14,518 BP (~15-30 runs)
Tier X:   5,766 × 8.5 = 49,011 BP (~50-100 runs!)
```

**End Game (100+ total retained):**
```
Multiplier = 1 + (100 × 0.15) = 16.0

Tier X:   5,766 × 16.0 = 92,256 BP (enormous achievement)
Tier XII: 12,974 × 16.0 = 207,584 BP (ultimate endgame goal)
```

### Tower Unlock Costs

**Additional Tower Types:**
```
Sniper Tower: 100 BP (cheap, encourages variety)
Contact Tower: 100 BP
Support Tower: 100 BP
```

**Philosophy:**
- Tower variety is encouraged (low cost)
- Tower depth is earned (high tier costs)
- Horizontal progression cheaper than vertical

### Endless Mode BP Rewards

**Milestone Rewards:**
```
Wave 10:  200 BP
Wave 20:  400 BP
Wave 30:  600 BP
Wave 40:  800 BP
Wave 50:  1,000 BP
...scales linearly: Wave (X×10) = X × 200 BP
```

**Plus:** Money-to-BP conversion (1%) when run ends

**Purpose:**
- Endless becomes viable BP farming method
- Rewards skilled late-game play
- Provides alternative to Short/Medium grinding

### Natural Power Creep Control

**How The System Prevents Runaway Power:**

1. **Exponential Tier Costs**
   - Each tier costs 1.5× more than previous
   - Tier X costs 38× more than Tier I
   - Natural ceiling emerges from economics

2. **Retention Scaling**
   - Every retained upgrade makes future ones more expensive
   - 100 retentions = 16× cost multiplier
   - Impossible to "spam" cheap upgrades forever

3. **Diminishing Returns (In-Run)**
   - Each tier gives less benefit than previous
   - Encourages diversification over stacking
   - Tier X gives ~28% of Tier I's relative value

4. **Endless Mode Awareness**
   - Enemies scale with player power in Endless
   - Challenge remains even with progression
   - Short/Medium get easier (rewarding feel)

5. **No Passive Bonuses**
   - Must still purchase upgrades in-run
   - Retention just unlocks access
   - Economy tension remains throughout

### Progression Curve

**New Player (First 10 runs):**
- Earning 200-500 BP per run
- Unlocking Tier I-II across towers
- Learning which upgrades synergize
- Costs: 150-400 BP per retention

**Experienced Player (50+ runs, 20-30 retained):**
- Earning 500-1,000 BP per run
- Unlocking Tier III-V on favorites
- Specializing tower builds
- Costs: 1,000-5,000 BP per retention

**Veteran Player (200+ runs, 50-70 retained):**
- Earning 800-1,500+ BP per run
- Pushing Tier VII-X achievements
- Min-maxing builds
- Costs: 5,000-50,000 BP per retention

**Endgame Player (500+ runs, 100+ retained):**
- Endless mode farming
- Tier X+ on multiple towers
- Ultimate build optimization
- Costs: 50,000+ BP per retention

### Strategic Implications

**Early Focus:**
- Unlock variety (new towers)
- Tier I-II across multiple upgrade types
- Establish foundation for experimentation

**Mid Game Focus:**
- Specialize based on playstyle
- Tier III-V on most-used towers
- Unlock synergistic upgrades

**Late Game Focus:**
- Deep specialization
- Tier VII-X on core strategies
- Perfect score optimization

**Score Optimization Strategies:**
- Minimal tower spending → bigger end-wave bonuses
- Protect all cores → maximize multiplier
- Efficient tower placement → maximize per-dollar value
- Perfect maze design → reduce total tower count needed

---

## Map Generation

### Overall Layout Philosophy
- **Platform:** Mobile portrait orientation
- **Screen Size:** ~14 tiles wide × 22 tiles tall
- **Structure:** Multi-island with bridges
- **Variation:** 1-3 blocks randomized each run

### Vertical Layout
```
┌──────────────┐
│  BASE ZONE   │  ← Rows 1-3: 20 cores, no-build
├──────────────┤
│              │
│   Block 1    │  ← Random size/shape
│              │
├──────┬───────┤
│      │       │  ← 1-tile bridge (optional)
├──────┴───────┤
│              │
│   Block 2    │  ← Random (33% chance doesn't exist)
│              │
├──────┬───────┤
│      │       │  ← 1-tile bridge (optional)
├──────┴───────┤
│              │
│   Block 3    │  ← Random (50% chance doesn't exist)
│              │
├──────────────┤
│ ENEMY SPAWN  │  ← Rows 20-22: spawn zone
└──────────────┘
```

### Generation Algorithm

**Step 1: Determine Block Count**
```javascript
numBlocks = random(1, 3)
// Equal probability:
// 1 block = 33% (monolithic open area)
// 2 blocks = 33%
// 3 blocks = 33%
```

**Step 2: Divide Vertical Space**
```javascript
availableHeight = 22 - 5 = 17 tiles (exclude base + spawn)

if (numBlocks === 1) {
  blockHeights = [17] // Massive single platform
}
if (numBlocks === 2) {
  blockHeights = [10, 6] // 60/40 split + 1 bridge tile
}
if (numBlocks === 3) {
  blockHeights = [6, 5, 4] // ~thirds + 2 bridge tiles
}
```

**Step 3: Generate Block Shapes**
```javascript
for each block:
  width = random(8, 14) // Fits screen, allows variety
  height = assigned from blockHeights
  
  // 30% chance for L-shape
  if (random() < 0.3) {
    shape = generateLShape(width, height)
  } else {
    shape = rectangle(width, height)
  }
  
  // Center horizontally
  xOffset = (14 - width) / 2
```

**Step 4: L-Shape Generation**
```javascript
function generateLShape(width, height) {
  // Start with rectangle
  block = createRectangle(width, height)
  
  // Remove corner section
  corner = random(['topLeft', 'topRight', 'bottomLeft', 'bottomRight'])
  cutSize = random(2, 4) tiles
  
  block.removeCorner(corner, cutSize)
  
  return block
}

// Example L-shape:
//  ████████
//  ████████
//  ████████
//  ████
//  ████
```

**Step 5: Bridge Placement**
```javascript
for each bridge between blocks:
  bridgeWidth = 1 tile // ALWAYS 1-tile chokepoint
  bridgeX = random(3, 11) // Not at extreme edges
  
  // Connect blocks vertically
  upperBlock.bottomExit = bridgeX
  lowerBlock.topEntrance = bridgeX
}
```

**Step 6: Pathfinding Validation**
```javascript
// CRITICAL: Always validate path exists
afterGeneration() {
  pathExists = runPathfinding(spawn, base)
  
  if (!pathExists) {
    regenerateMap() // Regenerate if impossible
  }
}

// During gameplay: Prevent blocking all paths
canPlaceTower(x, y) {
  tempMap = currentMap.clone()
  tempMap.addWall(x, y)
  
  return pathfindingExists(spawn, base, tempMap)
}
```

### Base Zone Details
- **Location:** Top 3 rows of map
- **Cores:** 20 cores in cluster (4×5 grid pattern)
- **Building:** No towers allowed in base zone
- **Adjacency:** Towers can build near base (to defend cores)
- **Visual:** Clearly marked as "home" / safe zone

### Bridge Characteristics
- **Width:** Always 1 tile (extreme chokepoint)
- **Strategic Value:**
  - Natural choke point
  - High-value tower placement nearby
  - Forces enemy grouping
  - Maze design focal point
- **Cannot Block:** Must maintain path, but can create detours

### Block Characteristics
- **Buildable:** Pure open space (for now)
- **Future:** Could add obstacles/barriers within blocks
- **Variation:** Size and shape randomization
- **Strategy:** Larger blocks = more freedom, smaller = constraint challenge

---

## Enemy Types

### Initial Enemy Roster

#### 1. Basic
- **Health:** 100 (base)
- **Speed:** 1.0 (base speed)
- **Value:** $50
- **Behavior:** Standard pathfinding to cores
- **Visual:** Simple geometric shape (triangle/square)

#### 2. Fast
- **Health:** 60
- **Speed:** 1.8
- **Value:** $40
- **Behavior:** Rushes to cores quickly
- **Challenge:** Harder to hit, less time under fire
- **Visual:** Streamlined shape, motion trails

#### 3. Tank
- **Health:** 300
- **Speed:** 0.6
- **Value:** $80
- **Behavior:** Slow but durable, takes longer to steal core
- **Challenge:** Requires sustained damage or armor pierce
- **Visual:** Bulky geometric form

#### 4. Splitter
- **Health:** 80
- **Speed:** 1.0
- **Value:** $30 (×3 = $90 total if all killed)
- **Behavior:** Splits into 2-3 smaller enemies on death
- **Challenge:** Multiplies enemy count, overwhelms single-target
- **Visual:** Compound/clustered shape

#### 5. Heavy (Later Waves)
- **Health:** 200
- **Speed:** 1.4
- **Value:** $100
- **Behavior:** Fast AND tanky hybrid
- **Challenge:** Requires both coverage and sustained DPS
- **Visual:** Larger, angular, aggressive design

#### 6. Swarm (Later Waves)
- **Health:** 30
- **Speed:** 1.2
- **Value:** $20 each
- **Behavior:** Many weak enemies at once (10-20)
- **Challenge:** Overwhelms defenses, needs AOE
- **Visual:** Small shapes, appears in groups

### Enemy Introduction Schedule

**Wave-by-Wave Enemy Types:**

**Waves 1-3:** Basic only (learning phase)  
**Waves 4-6:** Basic + Fast (introduces speed variance)  
**Waves 7-9:** Basic + Fast + Tank (introduces durability challenge)  
**Wave 10+:** Basic + Fast + Tank + Splitter (introduces multiplication)  
**Wave 12+:** All types including Heavy (hybrid threat)  
**Wave 15+:** All types including Swarm (overwhelming numbers)

### Wave Composition Formulas

**Base Formula (Before Dynamic Scaling):**
```javascript
// Total enemy count
baseCount = 10 + (wave × 3)

// Enemy type distribution
if (wave <= 3):
  100% Basic
  
if (wave 4-6):
  70% Basic, 30% Fast
  
if (wave 7-9):
  50% Basic, 30% Fast, 20% Tank
  
if (wave 10-11):
  40% Basic, 25% Fast, 20% Tank, 15% Splitter
  
if (wave 12-14):
  30% Basic, 20% Fast, 25% Tank, 15% Splitter, 10% Heavy
  
if (wave 15+):
  20% Basic, 15% Fast, 20% Tank, 20% Splitter, 15% Heavy, 10% Swarm
```

**Examples:**

**Wave 1:**
- Count: 10 + (1 × 3) = 13 enemies
- Types: 13 Basic

**Wave 5:**
- Count: 10 + (5 × 3) = 25 enemies
- Types: 18 Basic, 7 Fast

**Wave 10:**
- Count: 10 + (10 × 3) = 40 enemies
- Types: 16 Basic, 10 Fast, 8 Tank, 6 Splitter

**Wave 15:**
- Count: 10 + (15 × 3) = 55 enemies
- Types: 11 Basic, 8 Fast, 11 Tank, 11 Splitter, 8 Heavy, 6 Swarm (×10 each = 60 total)

**Dynamic Difficulty Adjustment:**
- Applied AFTER base formula
- Modifies enemy count by ±10-15%
- Adjusts enemy type ratios (more/fewer hard enemies)
- See "Dynamic Difficulty Scaling" in Wave Structure section

### Difficulty Scaling

**Short/Medium Games (Linear):**
```javascript
enemyHealth = baseHealth × (1 + wave × 0.15)
enemyCount = baseCount (from formula above)
enemySpeed = baseSpeed × (1 + wave × 0.05)

Example Wave 10:
- Basic Health: 100 × 2.5 = 250 HP
- Tank Health: 300 × 2.5 = 750 HP
- Basic Speed: 1.0 × 1.5 = 1.5 tiles/sec
- Fast Speed: 1.8 × 1.5 = 2.7 tiles/sec
```

**Endless Mode (Exponential):**
```javascript
enemyHealth = baseHealth × (1.2 ^ wave) × (1 + playerProgression × 0.02)
enemyCount = baseCount × (1.15 ^ wave)
enemySpeed = baseSpeed × (1.05 ^ wave)

Example Wave 20 (with 20 permanent upgrades):
- Basic Health: 100 × 38.3 × 1.4 = 5,362 HP
- Enemy Count: ~246 enemies (from base formula × 16.4)
- Basic Speed: 1.0 × 2.65 = 2.65 tiles/sec
```

### Future Enemy Types (Ideas)
- **Flying:** Ignores some maze pathing
- **Shielded:** Must break shield before damage
- **Healing:** Regenerates health over time
- **Teleporter:** Jumps forward periodically
- **Spawner:** Creates new enemies while alive

---

## Economy System

### Starting Resources
- **Base:** $1,000 (Short/Medium modes)
- **Modified by:** Permanent unlocks (e.g., +$200, +$400)
- **Endless:** Scales with selected difficulty

### Income Sources

**Enemy Kills:**
- Basic: $50
- Fast: $40
- Tank: $80
- Splitter: $30 each (×3 = $90)
- Heavy: $100
- Swarm: $20 each

**End-of-Wave Bonus:**
```
Bonus = Current Money × Cores at Base × 0.10

Examples:
- $5,000 × 20 cores = $10,000 bonus
- $3,000 × 18 cores = $5,400 bonus
- $8,000 × 15 cores = $12,000 bonus
```

### Spending Categories

**Tower Placement:**
- Gun: $500
- AOE: $700
- Contact: $400
- Sniper: $600
- Support: $400

**Upgrades (see Upgrade System for details):**
- Universal (Range/Damage): $200-$300 base
- Specific: $250-$500 base
- Scaling: × 2^(level-1)

### Economic Strategy

**Early Game:**
- Minimal tower spending
- Maximize end-wave bonus
- Invest in high-efficiency placements

**Mid Game:**
- Balance defense vs. bonus growth
- Strategic upgrades on key towers
- Plan for harder waves

**Late Game:**
- Heavy investment required
- Bonus becomes secondary
- Survival prioritized over efficiency

**Endless:**
- Efficient scaling crucial
- Upgrade synergies critical
- Perfect maze design necessary

---

## Visual Theme

### Aesthetic: Geometric Abstraction

**Inspiration:** Geometry Wars, vector arcade games

**Color Palette:**
- Neon-on-black primary scheme
- High contrast for visibility
- Distinct colors per element type
  - Towers: Blues/Cyans
  - Enemies: Reds/Oranges/Yellows
  - Cores: Green/Teal glow
  - Player structures: Purple outlines

**Visual Style:**
- Clean vector graphics
- Minimal texture, maximum clarity
- Particle effects for impacts
- Glowing edges and outlines
- Smooth animations

### Entity Designs

**Enemies:**
- Basic: Triangle
- Fast: Elongated diamond
- Tank: Hexagon/Octagon
- Splitter: Compound triangles
- Heavy: Large angular polygon
- Swarm: Small squares/dots

**Towers:**
- Gun: Rotating turret form
- AOE: Mortar/launcher shape
- Contact: Spinning blade/grinder
- Sniper: Long barrel form
- Support: Pulsing field emitter

**Cores:**
- Glowing polyhedrons (icosahedron?)
- Pulsing light effect
- Trail when being carried
- Different glow when stolen vs. safe

**Effects:**
- Projectile trails
- Explosion particles
- Hit sparks
- Slow visual (frost/electric effect)
- Death bursts

### UI Design
- Minimal HUD
- Corner-positioned info
  - Top Left: Money, Wave, Cores
  - Top Right: Timer, Score
  - Bottom: Tower/Upgrade selection
- Semi-transparent panels
- Clear iconography
- Touch-friendly buttons (mobile)

---

## UI/UX Specifications

### Screen Layout (Mobile Portrait)

**Overall Structure:**
```
┌────────────────────┐
│   TOP BAR (5%)     │  Always visible
├────────────────────┤
│                    │
│                    │
│    GAME MAP        │  Main play area (75-85%)
│    (Touch input)   │
│                    │
│                    │
├────────────────────┤
│  BOTTOM BAR (10%)  │  Collapsed by default
└────────────────────┘
```

### Top Bar (Always Visible)

**Left Side:**
- **Money:** `$X,XXX` (large, clear font)
- **Wave:** `Wave X/10` (medium font)
- **Cores:** `❖ XX` (icon + number)

**Right Side:**
- **Next Wave Timer:** `5s` (countdown between waves)
- **Pause Button:** `||` (optional, for testing)

**Design:**
- Semi-transparent dark background
- High contrast white/cyan text
- ~5% of vertical screen space
- Fixed position, never scrolls

### Bottom Bar States

**STATE 1: Collapsed (Default)**
- Shows 5 tower type buttons in a row
- Each button shows:
  - Tower icon (simple geometric shape)
  - Cost below icon (`$500`)
- Greyed out if can't afford
- ~10% of vertical screen space

**STATE 2: Tower Selected (Placement Mode)**
- Same 5 tower buttons
- Selected tower highlighted
- Map shows valid placement grid overlay
- Tap map to place, tap button again to cancel
- Bottom bar same size as collapsed

**STATE 3: Upgrade Menu (Expanded)**
- Opens when existing tower is tapped
- Takes ~40% of vertical screen space
- Dims/darkens map behind it
- Shows:
  - Tower name/type at top
  - Current stats (damage, range, etc.)
  - Available upgrade list:
    - Upgrade name
    - Cost
    - Effect description
    - Button to purchase
  - **Sell Button** at bottom (red, 75% refund)
- **Close Button** (X) in top-right corner
- Scrollable if many upgrades

### Touch Interactions

**Map Interaction:**
- **Tap empty tile:** Select tile for building (if tower button pressed)
- **Tap existing tower:** Open upgrade menu
- **Tap empty space (no tower selected):** Deselect/close menu
- **Pinch zoom:** NOT implemented (fixed zoom for consistency)
- **Drag:** NOT implemented (prevent accidental moves)

**Tower Placement Flow:**
1. Tap tower type button → Button highlights, cursor ready
2. Tap valid map location → Confirmation dialog: "Build [Tower] for $X?" [Yes] [Cancel]
3. Confirm → Tower placed, money deducted, button deselects
4. Cancel → Return to step 1

**Tower Sell Flow:**
1. Tap existing tower → Upgrade menu opens
2. Tap "Sell Tower" button → Confirmation: "Sell for $X (75%)?" [Yes] [Cancel]
3. Confirm → Tower removed, money refunded

**Upgrade Purchase Flow:**
1. Tower selected, upgrade menu open
2. Tap upgrade button → Instant purchase (no confirmation for speed)
3. Money deducted, upgrade applied, button updates to next tier

### Visual Feedback

**Must Have (Prototype):**
- Tower range indicator (circle) when placing or selected
- Valid placement tiles (green tint)
- Invalid placement tiles (red tint/blocked by path validation)
- Selected tower highlight (glow/outline)
- Money flash when changed (green = gained, red = spent)

**Nice to Have (Post-Prototype):**
- Projectile trails
- Damage numbers floating up from enemies
- Hit sparks/impacts
- Tower firing animations
- Enemy death bursts
- Core glow/trail when carried

### Grid Display
- **Always On:** Grid lines visible on map at all times
- Helps with precise tower placement
- Subtle lines (don't overpower visuals)
- 1 tile = 1 grid square

---

## Tutorial & First Run Experience

### Tutorial Mode (Simplified First Run)

**Labeling:**
- Separate menu option: "Tutorial" alongside "Short/Medium/Endless"
- Clearly labeled as tutorial in UI
- Disappears from menu after completion

**Structure:**
- 5 waves (shorter than Short mode's 10)
- Only Basic enemies (no Fast/Tank/Splitter)
- Same starting money ($1,000)
- Same cores (20)
- Only Gun + AOE towers available
- No upgrades unlocked yet

**Rewards:**
- **0 Blueprint Points** (explicitly no progression)
- Completion unlocks Short/Medium/Endless modes
- Purpose: Learn mechanics without pressure

**Guidance Level:**
- Minimal text prompts:
  - Pre-Wave 1: "Enemies will steal your cores. Stop them!"
  - Pre-Wave 1: "Build towers to defend." (after 5 seconds of inaction)
  - After first core stolen: "Destroy enemies to recover cores!"
  - After Wave 3: "Use upgrades to make towers stronger."
- No forced actions, just hints
- Learn by doing

**One-Time Only:**
- After completion, Tutorial option removed from menu
- Cannot replay
- All future runs are normal Short/Medium/Endless

---

## Save System & Data Persistence

### What Gets Saved

**Player Profile:**
- Total Blueprint Points (current balance)
- Retained upgrades (list of all permanent unlocks)
- Towers unlocked (Gun, AOE, Sniper, Contact, Support)
- Tutorial completion status
- Total runs completed (stats)
- High scores per mode

**Run State:**
- NOT SAVED mid-run
- If player closes app, run is lost
- Encourages full session completion

### When Saves Occur

**After Each Run:**
- Run completes (win or lose)
- BP earned calculated (money × 1%)
- BP added to total
- Player can spend BP on retentions
- Save occurs after retention screen

**Auto-Save Frequency:**
- Immediately after BP spending
- No manual save button
- No save slots (single profile per device)

### Storage Method

**Prototype:**
- Local storage only (browser localStorage or mobile equivalent)
- Single device, no cloud sync
- JSON format for easy debugging

**Future:**
- Cloud save integration (post-launch)
- Multiple device sync
- Account system

**Data Structure Example:**
```json
{
  "blueprintPoints": 1250,
  "tutorialComplete": true,
  "retainedUpgrades": [
    {"tower": "gun", "upgrade": "damage", "tier": 1},
    {"tower": "gun", "upgrade": "range", "tier": 1},
    {"tower": "aoe", "upgrade": "damage", "tier": 2}
  ],
  "towersUnlocked": ["gun", "aoe", "sniper"],
  "stats": {
    "runsCompleted": 15,
    "shortHighScore": 45000,
    "mediumHighScore": 82000,
    "endlessWave": 23
  }
}
```

---

## Settings & Options

### Available Settings (Prototype)

**Sound:**
- NOT IMPLEMENTED (no audio in prototype)
- Placeholder for future

**Grid Display:**
- ALWAYS ON (no toggle)
- Essential for precise placement

**Confirm Actions:**
- Sell Tower: YES (confirmation dialog)
- Build Tower: YES (confirmation dialog)
- Buy Upgrade: NO (instant purchase for speed)

**Game Speed:**
- NOT IMPLEMENTED (1x only)
- Future: Premium unlock feature (1.5x, 2x speed)
- Endless mode grinding benefit

### Future Settings (Post-Prototype)

- Sound effects volume
- Music volume
- Colorblind mode
- Haptic feedback (mobile)
- Particle density
- Camera zoom level
- Show damage numbers
- Fast-forward (premium unlock)

---

## Technical Considerations

### Platform
- **Primary:** Mobile (iOS/Android)
- **Secondary:** Web browser
- **Tertiary:** Desktop (potential)

### Technology Stack (Proposed)
- **Framework:** React (web) / React Native (mobile)
- **Graphics:** Canvas/WebGL for performance
- **Pathfinding:** A* algorithm
- **State Management:** React context or Redux
- **Storage:** LocalStorage (web) / AsyncStorage (mobile)

### Performance Targets
- **Frame Rate:** 60 FPS minimum
- **Enemy Count:** 100+ on screen simultaneously
- **Load Time:** <3 seconds
- **Battery:** Minimal drain on mobile

### Mobile Optimization
- Touch controls (tap to place/select)
- Pinch to zoom (optional)
- Clear visual hierarchy
- Accessible buttons (44×44pt minimum)
- Portrait orientation primary
- Responsive to different screen sizes

### Accessibility
- Colorblind modes (optional)
- Adjustable text size
- Clear audio cues
- Haptic feedback (mobile)
- Tutorial system

---

## Open Questions & Future Features

### Questions to Resolve
1. **Exact tower stat values** (DPS, range in tiles, etc.)
2. **Prep time duration** between waves
3. **Particle effect intensity** (performance vs. visual appeal)
4. **Sound design** direction
5. **Tutorial structure** for new players

### Future Features (Post-Launch)
- **Daily Challenges** with fixed seeds
- **Leaderboards** for Endless mode
- **More Tower Types:**
  - Laser (energy damage, shields)
  - Lightning (chain attacks)
  - Poison (DoT specialist)
  - Builder (constructs walls)
- **More Enemy Types:**
  - Boss enemies
  - Flying units
  - Teleporters
- **More Game Modes:**
  - Boss Rush
  - Time Attack
  - Hardcore (permadeath progression)
- **Cosmetics:**
  - Tower skins
  - Enemy skins
  - Particle themes
  - UI themes
- **Social Features:**
  - Share builds
  - Challenge friends
  - Replay system

### Balancing Priorities
1. Ensure Short/Medium modes completable with skill
2. Endless mode challenging but fair
3. Meta-progression feels rewarding without being mandatory
4. RNG variance acceptable but not frustrating
5. All tower types viable in different situations

---

## Development Roadmap

### Phase 1: Prototype (Current)
- [ ] Basic map generation
- [ ] Simple tower placement
- [ ] Basic enemy pathfinding
- [ ] Core stealing mechanics
- [ ] Wave system
- [ ] Minimal UI

### Phase 2: Core Gameplay
- [ ] All 5 tower types functional
- [ ] All initial enemy types
- [ ] Upgrade system
- [ ] End-wave bonus
- [ ] Short/Medium/Endless modes
- [ ] Improved visuals

### Phase 3: Progression
- [ ] Blueprint Point system
- [ ] Permanent unlocks
- [ ] Meta-progression tree
- [ ] Save/load system
- [ ] Statistics tracking

### Phase 4: Polish
- [ ] Full visual effects
- [ ] Sound design
- [ ] Music
- [ ] Tutorial
- [ ] Balancing pass
- [ ] Performance optimization

### Phase 5: Launch
- [ ] Mobile builds
- [ ] App store submission
- [ ] Marketing materials
- [ ] Community setup
- [ ] Analytics integration

---

## Prototype Scope & Priorities

### MVP (Minimum Viable Prototype)

The goal is a PLAYABLE game loop that proves the core mechanics are fun. Focus on functionality over polish.

**MUST HAVE:**

**Core Systems:**
- Map generation (1-3 random blocks with bridges)
- Tower placement with path validation
- Global pathfinding (Spawn→Base, Base→Exit)
- Enemy movement along paths
- Core stealing and recovery
- Wave spawning and progression
- Money economy and wave bonuses
- Tower selling (75% refund)

**Towers (Minimum 2):**
- Gun Tower (single-target DPS)
- AOE Tower (splash damage)
- Basic stats: damage, fire rate, range, cost
- No upgrades yet (just base towers)

**Enemies (Minimum 2):**
- Basic (standard movement and health)
- Fast (higher speed, lower health)
- Health, speed, and money value
- Core stealing behavior

**UI:**
- Top bar: Money, Wave, Cores
- Bottom bar: Tower selection buttons
- Tower placement confirmation
- Sell tower confirmation
- Basic geometric shapes (no art needed)
- Grid overlay (always visible)

**Visuals:**
- Simple geometric shapes:
  - Towers: Circles/squares with distinct colors
  - Enemies: Triangles/diamonds color-coded by type
  - Cores: Small glowing hexagons
  - Map: Grid with block outlines
- No particle effects
- No animations (static sprites rotate/move)
- No projectile trails (instant hit or simple line)

**Waves:**
- Short mode only (10 waves)
- Wave 1-5: Only Basic enemies
- Wave 6-10: Basic + Fast
- Formula-based scaling (count, health, speed)

**No Audio:**
- Completely silent prototype
- Sound effects deferred to Phase 4

---

**CAN WAIT (Post-MVP):**

- Upgrades system (full implementation)
- Remaining 3 tower types (Sniper, Contact, Support)
- Remaining 4 enemy types (Tank, Splitter, Heavy, Swarm)
- Medium mode (15 waves)
- Endless mode
- Blueprint Points and progression
- Save system
- Tutorial mode
- Dynamic difficulty scaling
- Particle effects and polish
- Sound and music
- Settings menu
- Statistics tracking

---

### Implementation Order (Suggested)

**Week 1: Foundation**
1. Grid system and coordinate mapping
2. Basic map generation (single block first, then multi-block)
3. Camera/view setup for mobile portrait
4. UI framework (top bar, bottom bar)

**Week 2: Core Mechanics**
1. Tower placement system with validation
2. Global pathfinding (A* implementation)
3. Enemy spawning and movement
4. Money economy (earning, spending, display)

**Week 3: Gameplay Loop**
1. Core stealing behavior
2. Tower attacking (Gun and AOE)
3. Enemy health and death
4. Wave progression system
5. End-wave bonuses

**Week 4: Polish & Testing**
1. Tower selling with refund
2. UI improvements (confirmations, clarity)
3. Wave composition tuning
4. Balance testing
5. Bug fixes

---

### Testing Priorities

**What to Test First:**
1. **Is maze-building fun?** Does path manipulation feel strategic?
2. **Is tower placement satisfying?** Do confirmations feel good or annoying?
3. **Is the economy balanced?** Can players afford towers without grinding?
4. **Are wave bonuses compelling?** Does the multiplier create interesting decisions?
5. **Is pathfinding robust?** Do enemies ever get stuck or behave weirdly?

**Metrics to Track:**
- Average money at end of Wave 5
- Average money at end of Wave 10
- Number of towers typically placed
- Number of cores typically lost
- Player failure rate (which waves most common?)

**Known Risks:**
- Pathfinding performance with many enemies
- Path validation feeling restrictive or annoying
- Tower refund economy (too generous? too punishing?)
- Wave difficulty curve (too easy? too hard?)

---

### Post-Prototype Expansion Plan

Once the MVP is proven fun:

**Phase 2A: Content**
- Add remaining 3 towers (Sniper, Contact, Support)
- Add remaining 4 enemy types
- Add upgrade system (all tiers)
- Add Medium mode (15 waves)

**Phase 2B: Progression**
- Blueprint Points conversion
- Retention system
- Tower unlocks
- Save/load implementation

**Phase 2C: Endless Mode**
- Exponential scaling
- Milestone rewards
- Leaderboard foundation
- Player loadout selection

**Phase 3: Polish**
- Visual effects (particles, trails, impacts)
- Sound design (SFX for all actions)
- Music (menu and gameplay tracks)
- Tutorial mode
- Settings menu
- Improved graphics

**Phase 4: Launch Prep**
- Mobile optimization
- Performance testing
- Balance pass
- Bug fixing
- App store assets
- Marketing materials

---

## Design Principles

1. **Player Time Respect:** No manipulative engagement mechanics
2. **Skill Rewarded:** Better play = better outcomes
3. **Meaningful Choices:** Every decision matters
4. **Variety:** Replayability through randomization
5. **Fair Challenge:** Difficult but never unfair
6. **Transparent Systems:** Clear feedback on everything
7. **Progression Satisfaction:** Feel stronger without breaking game
8. **Mobile-First:** Touch-friendly, session-friendly

---

**End of Document**  
Version 0.5 | January 2026
