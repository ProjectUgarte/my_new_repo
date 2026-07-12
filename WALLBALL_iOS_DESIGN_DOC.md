# WallBall iOS Design Document

## Overview
WallBall is a retro-inspired puzzle game based on the 1992 classic JezzBall. The player builds walls to partition a playing field, trapping bouncing balls. Fill 75% of the field to advance. The game is currently a single-file HTML5 canvas game (~1375 lines) and needs to be ported to native iOS.

**Live version:** https://projectugarte.github.io/wallball/
**Source:** https://github.com/ProjectUgarte/wallball

---

## Game Flow

### Screens
1. **Start Screen** — Title ("WALLBALL" in pixel font), instructions, START button, "by project(u)" brand text, large decorative ball bouncing behind semi-transparent overlay
2. **Gameplay** — Active play with HUD, balls bouncing, player building walls
3. **Level Complete** — Overlay showing score, fill %, time bonus, CONTINUE button, decorative ball bouncing behind
4. **Game Over** — "GAME OVER" in red pixel font, final score, level reached, leaderboard with name entry (if high score), RETRY button, decorative ball bouncing behind

### Screen Transitions
- Start Screen -> Gameplay: immediate on START tap
- Gameplay -> Level Complete: 500ms delay with blue flash effect, then overlay
- Gameplay -> Game Over: 500ms delay, then overlay  
- Level Complete -> Gameplay: immediate on CONTINUE tap
- Game Over -> Gameplay: immediate on RETRY tap

---

## Core Mechanics

### Grid System
- The playing field is a pixel grid
- **CELL_SIZE = 6 points** (each cell is 6x6)
- Grid dimensions: `gridW = floor(screenWidth / 6)`, `gridH = floor(screenHeight / 6)`
- Grid is a flat array of `UInt8`: `0 = open, 1 = wall, 2 = filled/claimed`
- **GRID_GAP = 1** — 1-point gap between cells creates the visible grid lines

### Grid Initialization
- Border walls: all edge cells set to `1`
- Top UI area: all cells in the top 48 points (`UI_TOP_PAD = 48`, so `ceil(48/6) = 8` rows) set to `1`
- `totalCells` = count of all cells that start as `0` (used for fill % calculation)

### Walls
- Player taps + swipes to set direction (horizontal or vertical)
- Wall grows from the tap point in both directions simultaneously
- **WALL_SPEED = 2 cells per frame** (at 60fps baseline, delta-time scaled with fractional accumulator)
- Each wall head advances one cell per step, checking if the next cell is already wall/filled or out of bounds
- When both heads reach existing walls/boundaries, the wall is complete
- On completion, trigger flood fill to recalculate claimed area
- Score += number of cells in the completed wall

### Wall-Ball Collision (Life Loss)
- While a wall is actively building, check if any ball overlaps any cell of that wall
- Distance check: `sqrt((ball.x - cellCenterX)^2 + (ball.y - cellCenterY)^2) < ball.radius + CELL_SIZE`
- On hit: remove the wall (set all its cells back to `0`), lose 1 life, red flash effect
- If lives reach 0, trigger game over

### Flood Fill (Area Claiming)
1. Set all `0` cells to `2` (tentatively filled)
2. For each ball, flood fill from the ball's grid position, converting `2` back to `0`
3. Any cells still marked `2` are claimed — they're in areas with no balls
4. `fillPercent = round((filledCells / totalCells) * 100)`

### Balls

#### Spawning
- Level 1: 2 balls, Level 2: 3 balls, etc. (`ballCount = level + 1`)
- Random positions within safe margins (3x ball radius from edges)
- Random direction angles, speed based on level

#### Movement Constants
- **BALL_RADIUS = 21 points** (diameter ~7 grid cells — large enough to see two-tone detail)
- **BALL_SPEED_BASE = 2.8 points/frame** (at 60fps)
- **BALL_SPEED_INCREMENT = 0.2** per level
- Speed = `BALL_SPEED_BASE + (level - 1) * BALL_SPEED_INCREMENT`
- Each ball has independent spin: `spinSpeed = (0.02 + random * 0.02) * randomSign`

#### Delta-Time Movement
- All movement is normalized to 60fps: `scale = delta * 60`
- Ball position: `x += vx * scale`, `y += vy * scale`
- **Critical: Substep collision** — movement is broken into steps no larger than `CELL_SIZE * 0.5` (3 points) to prevent tunneling through 1-cell-wide walls
- `steps = max(1, ceil(totalDistance / maxStep))`
- Each substep: move, check horizontal collision, check vertical collision

#### Collision Detection
- Sample multiple points along the ball's leading edge in each axis
- Horizontal: check `edgeX = ball.x + sign(vx) * radius` at Y offsets from `-radius` to `+radius` in CELL_SIZE increments
- Vertical: same approach on the Y axis
- On collision: reverse velocity, nudge ball away by `2 * velocity`
- Boundary clamping: keep ball within `wallEdge + radius` from all sides

### Ball Rendering (Two-Tone Spinning Sphere)
- Pixelated circular sprite rendered cell-by-cell
- Each cell position is tested against the ball radius for circular clipping
- The dividing line between halves is determined by rotating each pixel's position by the ball's current angle: `rotY = px * sin(angle) + py * cos(angle)`
- `rotY < 0` = red half, `rotY >= 0` = white half
- Dark outline ring: cells where `dist > radius - CELL_SIZE * 1.2`
- Each cell has a random flicker: `flick = 0.93 + random * 0.07`
- **Red half color:** `rgb(230 * shade * flick, 35 * shade * flick, 35 * shade * flick)` where `shade = max(0.4, 1 - normDist * 0.6)`
- **White half color:** `rgb(220 * shade * flick, 220 * shade * flick, 230 * shade * flick)` where `shade = max(0.55, 1 - normDist * 0.4)`
- **Outline color:** `rgb(20 * flick, 20 * flick, 25 * flick)`
- Center each pixel cell by offsetting `-CELL_SIZE/2` to avoid the top-left drift artifact
- The `drawBall()` function is reused for both gameplay balls and the decorative menu ball

### Scoring
- Completing a wall: `+wallCellCount`
- Level complete bonus: `fillPercent * 10 + lives * 50 + round(timeRemaining) * 5`
- Leaderboard: top 5 scores stored locally, name entry (6 char max) on high score

---

## Visual Design

### Color Palette
| Element | Color |
|---------|-------|
| Background (gameplay) | `#040804` (near black) |
| Open play area cells | `#121e30` (dark blue tint) |
| Grid lines | `rgba(20, 30, 45, 0.3)` |
| Static walls (visible) | `rgb(30*f, 80*f, 140*f)` where f = 0.9-1.0 flicker |
| Interior walls (claimed zones) | `#040804` (hidden, matches background) |
| Claimed/filled areas | `#040804` (black) |
| UI text | `#cceeff` with `#55bbff` glow |
| Timer (urgent, <=10s) | `#ff3333` with pulse animation |
| Buttons | `#55bbff` fill, `#0a0a0a` text |
| Overlay background | `rgba(0, 0, 0, 0.8)` |
| Flash (life lost) | `#ff3333` |
| Flash (level complete) | `#55bbff` |

### Wall Building Animation
Active wall cells have elaborate animated rendering:
- **Wave pulse**: brightness fades over 50 frames from placement
- **Proximity glow**: brighter near the origin point, fades over 60 cells
- **Shimmer**: `sin(frameCount * 0.3 + dist * 0.5) * 0.15`
- **Size boost**: cells near origin pulse larger (up to +6px)
- **Outer halo**: soft glow at `rgba(85, 187, 255, intensity * 0.12)`
- **Leading edge spark**: white flash for first 6 frames of each new cell
- **Origin point**: expanding shockwave rings (5 concentric), bright center with throb animation

### Interior Wall Hiding
- Completed walls that don't border any open cell (val=0) are rendered black, blending into claimed areas
- Exception: walls in the top UI rows (y < uiRows) always render as visible blue walls

### Scanline / CRT Effects
- Horizontal scanlines: `rgba(0,0,0,0.06)` every 3rd pixel row
- Radial vignette: gradient from transparent at center to `rgba(0,0,0,0.4)` at edges

### Pixel Font System
- Custom 5x7 pixel bitmap font for "WALLBALL" title and "GAME OVER"
- Glyphs defined: A, B, E, G, L, M, O, R, V, W, space
- Rendered to offscreen canvas with configurable pixel size
- Two passes: dark shadow offset by 2px, then main color
- Edge shading on right and bottom edges of each pixel
- Title: 8px blocks, "GAME OVER": 6px blocks
- Title color: `#88ccff` highlight / `#55bbff` main, shadow `#1a3050`
- Game over color: `#cc3333` highlight / `#991111` main, shadow `#2a0808`

### 8-Bit Hearts (Lives Display)
- 9x8 pixel bitmap rendered to a small canvas
- 3px per pixel cell
- Filled heart: `#ff2222` with `#ff6666` highlight on top-left quadrant
- Empty/lost heart: `#441111`
- Max 3 lives displayed

### Menu Ball
- Decorative ball bouncing behind the overlay on start, game over, and level complete screens
- **MENU_BALL_RADIUS = BALL_RADIUS * 2** (42 points, twice the gameplay size)
- Velocity: `vx = 1.8, vy = 1.4` (delta-time scaled)
- Bounces off screen edges with 4px margin
- Rendered using the same `drawBall()` function
- Clears the canvas each frame with `#040804`
- Stops when gameplay starts, resumes when overlay appears (synced with setTimeout to avoid premature appearance)

---

## HUD (Heads-Up Display)
- Fixed bar at top of screen, hidden on start screen and game over
- Layout: `LVL 01` (left) | `0%` (center) | `0:59` (center-right) | hearts (right)
- Font: Courier New monospace, 28px, bold, 3px letter-spacing
- Text glow: `0 0 12px #55bbff88, 0 0 4px #55bbff, 0 0 20px #55bbff44`
- Gradient background fading to transparent
- Timer turns red + pulses when <= 10 seconds
- Fill percent shown without leading zeros (0%, 5%, 75% — not 075%)

---

## Input System

### Touch (Primary)
- **touchstart**: record start position and time
- **touchmove**: if drag > 15px, determine direction (horizontal vs vertical) and show directional hint line
- **touchend**: if drag < 10px, default to horizontal. Otherwise use dominant axis. Create wall at start position.
- Swipe hint: 60px blue line showing wall direction while dragging

### Mouse (Fallback for Desktop)
- Same logic as touch using mousedown/mousemove/mouseup

---

## Audio System

### Tracks
| Track | File | Loop | Volume | When |
|-------|------|------|--------|------|
| Menu | menu.mp3 | Yes | 0.4 | Start screen, game over screen |
| Gameplay | gameplay.mp3 | Yes | 0.4 | During active gameplay |
| Level Clear | levelclear2.mp3 | No | 0.5 | Level complete |

### Behavior
- Crossfade between tracks: 400ms fade out of previous, then start new track
- On iOS, use AVAudioSession with `.ambient` or `.playback` category
- Menu music should start as early as possible (the web version attempts autoplay and falls back to first interaction)

---

## Timing & Difficulty

| Parameter | Value |
|-----------|-------|
| Base time per level | 60 seconds |
| Time added per level | +10 seconds |
| Target fill to advance | 75% |
| Starting lives | 3 |
| Ball count | level + 1 |
| Ball speed | 2.8 + (level-1) * 0.2 |
| Wall build speed | 2 cells/frame (at 60fps) |

---

## Leaderboard
- Top 5 scores stored locally (UserDefaults on iOS)
- Each entry: `{ name: String (6 char max, uppercase), score: Int, level: Int, date: Date }`
- Sorted by score descending
- On game over, if score qualifies, show arcade-style name input (6 character max, uppercase)
- Current score highlighted in the list
- Consider Game Center integration for global leaderboards (future enhancement)

---

## iOS-Specific Considerations

### Framework Recommendation
**SpriteKit** is the best fit:
- Built-in game loop with delta time
- Efficient 2D sprite rendering
- Easy audio integration via `SKAudioNode` or `AVAudioPlayer`
- Physics engine available (though this game uses custom collision)
- Built-in support for touch handling

### Performance Notes
- The grid can be 60-100+ columns by 120-150+ rows on modern iPhones
- Rendering every cell every frame is fine for SpriteKit with `SKShapeNode` or preferably batch-rendered via `SKSpriteNode` with generated textures
- The wall animation with per-cell glow effects is the most expensive part — consider pre-rendering animated states or using shaders
- Flood fill runs on wall completion only, not every frame — it's fast enough with the stack-based approach

### Audio
- Use `AVAudioPlayer` for music tracks (supports looping, volume control, crossfading)
- Consider `AVAudioSession.sharedInstance().setCategory(.ambient)` to respect silent mode, or `.playback` to play regardless

### Haptics
- Vibrate on life lost (wall destroyed by ball)
- Light tap on wall completion
- Success haptic on level complete
- Use `UIImpactFeedbackGenerator` and `UINotificationFeedbackGenerator`

### Screen Adaptation
- The game already adapts to screen size (grid fills available space)
- Safe area insets: ensure the top UI bar accounts for notch/Dynamic Island
- Support both portrait orientations; consider landscape as optional

### Assets to Include
- `audio/menu.mp3` (7.4MB)
- `audio/gameplay.mp3` (5.5MB)
- `audio/levelclear2.mp3` (5.5MB)
- App icon: generated from `icon-generator.html` (512x512 and 1024x1024 PNGs available)

---

## Brand
- "by project(u)" shown on start screen only
- Subtle styling: 13px, 2px letter-spacing, `rgba(85, 187, 255, 0.4)`, monospace
- Positioned below the START button with generous spacing (72px margin-top)
