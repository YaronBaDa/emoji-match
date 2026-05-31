# FADE — Game Specification

> **One-file HTML memory-matching puzzle game.**  
> Live: `https://yaronbada.github.io/emoji-match/`  
> Repo: `YaronBaDa/emoji-match`  
> Deployed via GitHub Actions (`.github/workflows/deploy.yml`)

---

## 1. Game Concept

FADE is a **daily + free-play emoji memory-matching game** built for mobile portrait. The board starts as a grid of face-down cards. The player flips two cards at a time to find matching emoji pairs. Matched cards **pop, wobble, and vanish completely**, progressively revealing a large daily-changing mystery emoji hidden underneath. A contextual ability banner announces special events.

The game uses an **elapsed-time counter** (counting up from the first flip) rather than a countdown timer.

---

## 2. Tech Stack

| Layer | Choice |
|-------|--------|
| **File** | Single `index.html` (~75 KB) |
| **Styling** | Inline CSS + Tailwind CDN (`cdn.tailwindcss.com`) |
| **Logic** | Vanilla ES6+ JavaScript, no frameworks |
| **State** | Single global `S` object + `localStorage` for stats |
| **Deploy** | GitHub Pages via GitHub Actions |
| **Fonts** | `Unbounded` (headings), `Space Mono` (stats/timer) |

---

## 3. Screens & Flow

```
Landing Screen ──► Game Screen ──► Game Over Overlay
     ▲                │               │
     └────────────────┴───────────────┘   (Replay / Back to Menu)
```

### 3.1 Landing Screen (`#landingScreen`)
- **Hero**: Animated mini 3×2 grid with auto-flipping demo cards
- **Title**: "FADE" in large bold white
- **Tagline**: "Match them before they fade."
- **Daily Challenge Card**: Shows today's date, streak, lock status if already played
- **▶️ Play Classic button** (default): Cyan gradient, launches 16-card game immediately
- **Tier Buttons**:
  - Warm-up — 12 cards (3×4)
  - Classic — 16 cards (4×4)
  - Hard — 20 cards (4×5)
  - Brutal — 24 cards (4×6)

### 3.2 Game Screen (`#gameScreen`)
- **Timer bar** (3px cyan strip at top)
- **Compact HUD** (bento-box layout):
  - Left: "FADE" title card
  - Right: 🏆 Score / 🕹️ Moves / 🔥 Combo / ⏱️ Elapsed timer
- **Grid area** (`#gridWrapper`): Cards + underlay emoji behind them
- **Ability banner** (`#gameStatusAnnouncer`): Fixed top-center, slides in/out
- **Victory banner** (`#victoryBanner`): "Victory!" overlay during 3s sparkle

### 3.3 Game Over Overlay (`#gameOverOverlay`)
- Tempo map grid showing per-card flip counts
- Summary line: "Completed in X moves and Y minutes"
- Tomorrow countdown (for daily mode)
- Buttons: **Play Again** | **Back to Menu**

---

## 4. Game State (`S`)

```js
const S = {
  score: 0,
  combo: 0,
  maxCombo: 0,
  moves: 0,
  flipHistory: [],      // "🟩" | "🟥" | "🔥"
  pairTimes: [],        // ms timestamps per match
  totalFlips: 0,
  flipCounts: [],       // per-card flip counter (tempo map)
  pairFlipCounts: {},   // per-emoji flip counter
  emojis: [],           // active emoji pool
  totalPairs: 0,
  matchedPairs: 0,
  flippedIndices: [],   // currently flipped cards
  cards: [],            // {emoji, matched, flipped, element, index}
  isProcessing: false,  // lock during match check
  lastMatchTime: 0,     // for combo chaining
  gameActive: false,
  elapsedTime: 0,       // seconds, count-up
  elapsedTimerId: null,
  firstFlipTime: 0,     // performance.now() on first flip
  isBoardLocked: false, // during ability animations
  boardLockTimer: null,
  bombTriggered: false, // cluster bomb once-per-game
  clairvoyanceActive: false,
  mode: 'classic',      // 'classic' | 'daily'
};
```

---

## 5. Core Mechanics

### 5.1 Card Flip
- Click / tap a face-down card → flips face-up (CSS `rotateY(180deg)`)
- Two cards flipped → `isProcessing = true`, 400ms delay → `checkMatch()`

### 5.2 Match
- If emojis match:
  - Both cards get `.matched` class
  - `matchedPairs++`, score += `BASE_SCORE × (combo + 1)`
  - Floating combo text appears at match midpoint
  - Cards trigger `matchVanish` animation (pop → wobble → scale to 0)
  - **Special ability triggered** if emoji matches `ABILITY_TRIGGERS`
- If mismatch:
  - Cards get `.mismatch` → shake animation → flip back
  - `combo = 0`

### 5.3 Elapsed Timer
- Starts on **first card flip** (`firstFlipTime = performance.now()`)
- Increments `S.elapsedTime` every 50ms
- Displayed as `M:SS.ms` in HUD (e.g. `2:27.04`)
- Timer bar shows progress vs 5-minute reference (not a countdown)

### 5.4 Combo System
- Match within 2000ms of previous match → `combo++`
- Combo multiplies score: `BASE_SCORE × (combo + 1)`
- Mismatch resets combo to 0
- HUD shows "🔥 xN" when combo > 0

### 5.5 Underlay Reveal
- A large daily-changing emoji sits behind the card grid (`#underlayGrid`)
- Opacity starts at 0.30 during play
- First match adds `.underlay-breathe` (gentle pulse animation)
- On victory: `.underlay-victory` → full opacity + golden sparkle glow for 3 seconds
- Cards vanishing progressively reveals more of the underlay

---

## 6. Grid Layout Logic

`getGridLayout(count)` — **portrait-first** algorithm:

```js
function getGridLayout(count) {
  let cols = Math.floor(Math.sqrt(count));
  cols = Math.max(3, cols);
  let rows = Math.ceil(count / cols);
  // Ensure cols <= rows (taller than wide)
  while (rows < cols && cols > 3) {
    cols--;
    rows = Math.ceil(count / cols);
  }
  return { cols, rows };
}
```

| Cards | Layout | Aspect |
|-------|--------|--------|
| 12 | 3×4 | Portrait |
| 16 | 4×4 | Square |
| 20 | 4×5 | Portrait |
| 24 | 4×6 | Portrait |

Grid fills available viewport via `calculateCardSize()` — max size constrained by wrapper width/height minus gap.

---

## 7. Special Card Abilities

Four emoji pairs trigger abilities when matched:

| Emoji | Ability | Effect |
|-------|---------|--------|
| `🔮` | **Clairvoyance** | All unrevealed cards flip face-up for 1.0s via `rotateY` CSS, then flip back. `isBoardLocked = true` during reveal. Banner: "🔮 CLAIRVOYANCE: Photographic memory test!" |
| `⏰` | **Time Warp** | Deducts 10s from `elapsedTime` (floor 0). Floating green "-10s" text at match location. Banner: "⏰ TIME WARP: 10 seconds shaved off!" |
| `🔓` | **Security Lockdown** | `isBoardLocked = true` for 3.0s. Frost overlay (`❄️`) appears over grid. Time keeps running. Banner: "🔓 SECURITY LOCKDOWN: Inputs frozen for 3s!" |
| `💣` | **Cluster Bomb** | Once per game. Screen shake on grid. Injects 4 new cards (2 new emoji pairs from backup pool). Grid recalculates `cols/rows`. Banner: "💣 DETONATION: 4 new cards added to the grid!" |

### 7.1 Board Lock
- `isBoardLocked = true` during: ability animations, clairvoyance, lockdown
- `onCardClick()` returns early if locked
- `#gameScreen` gets `.board-locked` overlay during lockdown

---

## 8. CSS Animation Classes

| Class | Animation | Use |
|-------|-----------|-----|
| `.card.flipped .card-inner` | `rotateY(180deg)` | Card flip |
| `.card.mismatch .card-inner` | `shake` (translateX wobble) | Mismatch feedback |
| `.reveal-mode .card.matched` | `matchVanish` (pop→wobble→scale 0) | Match disappearance |
| `.explosion-shake` | `explosionShake` (10-keyframe screen shake) | Cluster bomb |
| `.freeze-overlay` | Frost blur + `❄️` icon pulse | Security lockdown |
| `.floating-time-bonus` | `floatUp` (green "-10s") | Time warp |
| `.clairvoyance-flip .card-inner` | `rotateY(180deg)` smooth | Clairvoyance reveal |
| `#gameStatusAnnouncer.show` | Slide + scale in | Ability banner |
| `#victoryBanner.show` | Scale bounce in | Victory text |
| `.underlay-breathe` | Gentle opacity pulse | First match+ |
| `.underlay-victory` | Full brightness + sparkle | All pairs found |
| `.combo-pulse` | Scale pulse | Combo UI |
| `.grid-fade-in` | Fade + scale in | Grid spawn |
| `.screen-shake` | Violent translate shake | Screen shake effect |

---

## 9. Daily Challenge

- **Seed**: Deterministic via `cyrb128(YYYY-MM-DD + '_fade_daily_v1')`
- **Emoji pool**: Same 24 emojis, shuffled deterministically per day
- **Lock**: Once played today, card shows countdown until next midnight UTC
- **Stats**: Streak tracking (resets if missed a day)

---

## 10. Stats Persistence

`localStorage` key: `fade_stats_v2`

```js
{
  tierPlays:   { 12: N, 16: N, 20: N, 24: N },
  tierWins:    { 12: N, 16: N, 20: N, 24: N },
  tierAvgMoves:{ 12: N, 16: N, 20: N, 24: N },
  streak: 0,
  maxStreak: 0,
  lastDate: '',
  dailyPlayed: '',
  dailyWon: false,
}
```

---

## 11. Emoji Pools

### 11.1 Standard Pool (`ALL_EMOJIS`) — 28 emojis
```
💀 🤡 🧠 🔥 🤫 👑 💅 🤠
🚀 👽 🥷 🥑 🦄 🌈 🎯 ⚡
🍕 🧩 🛸 🎸 🎪 🐲 🎨 🍭
🔮 ⏰ 🔓 💣
```

### 11.2 Daily Mystery Pool (`DAILY_EMOJIS`) — 80 emojis
Animals, fantasy, space, food, music, etc. Selected by daily hash.

### 11.3 Ability Triggers
```js
ABILITY_TRIGGERS: {
  CLAIRVOYANCE: '🔮',
  TIMEWARP:     '⏰',
  LOCKDOWN:     '🔓',
  DETONATION:   '💣',
}
```

---

## 12. Key Functions

| Function | Purpose |
|----------|---------|
| `startGameFromTier(count)` | Launch classic mode with N cards |
| `startDaily()` | Launch daily seeded 16-card mode |
| `startGame(count)` | Show game screen, set grid dims, call `initGame()` |
| `initGame()` | Reset state, call `buildGrid()`, start game |
| `buildGrid()` | Create card DOM, shuffle emojis, size cards, build underlay |
| `calculateCardSize()` | Compute max card size from viewport |
| `onCardClick(index)` | Handle flip, start timer on first flip, trigger match check |
| `checkMatch()` | Compare two flipped cards |
| `handleMatch(i1, i2)` | Mark matched, score, combo, trigger ability |
| `handleMismatch(i1, i2)` | Shake and flip back |
| `triggerClairvoyance()` | 1s reveal all unrevealed cards |
| `triggerTimeWarp(x, y)` | -10s from elapsed time, floating text |
| `triggerSecurityLockdown()` | 3s board lock with frost overlay |
| `triggerClusterBomb()` | +4 cards, recalc grid, screen shake |
| `showBanner(msg, class)` | Show contextual ability banner |
| `updateElapsedDisplay()` | Format and write timer to HUD |
| `startTimer()` / `stopTimer()` | 50ms interval for elapsed time |
| `victory()` | Stop game, show victory banner + sparkle, then overlay |
| `gameOver()` | Stop game, show overlay immediately |
| `showOverlay()` | Hide HUD, show game over with tempo map + stats |
| `renderTempoMap()` | Build per-card flip count grid |
| `getGridLayout(count)` | Portrait-friendly cols/rows calculation |
| `buildUnderlayGrid(size, count)` | Size and position the background emoji |
| `getDailyEmoji()` | Hash-based daily mystery emoji |

---

## 13. Feature Flags (`GAME_CONFIG`)

```js
{
  COMPACT_HUD: true,           // Use bento-box HUD
  VANISH_MATCHES: true,        // Matched cards disappear
  UNDERLAY_REVEAL: true,       // Background emoji reveal
  EMOJI_SCALE_PERCENT: 52,     // Front emoji size % of card
  ABILITY_TRIGGERS: { ... },   // Special card mapping
  GRID_COLS: 4,                // Set by getGridLayout()
  GRID_ROWS: 4,
}
```

---

## 14. Constants

```js
BASE_SCORE = 100;
GRID_GAP = 6;
CLAIRVOYANCE_DURATION = 1000;   // ms
LOCKDOWN_DURATION = 3000;       // ms
TIMEWARP_DEDUCTION = 10;        // seconds
```

---

## 15. Responsive Behavior

- **Mobile-first**: Designed for ~375px–430px portrait width
- `calculateCardSize()` auto-shrinks cards to fit viewport
- Grid gap = 6px, padding = 12px
- `@media (max-width: 380px)` → smaller border radii, dots, hero board
- `@media (min-width: 500px)` → larger center dots
- Resize listener rebuilds grid while preserving matched state

---

## 16. Known Architecture Notes

1. **Single file**: All HTML, CSS, JS in `index.html`. No build step.
2. **No external JS deps**: Vanilla JS only. Tailwind loaded via CDN.
3. **Deterministic daily**: `cyrb128` + `mulberry32` PRNG for reproducible daily boards.
4. **Touch + click**: Both `click` and `touchstart` (with `preventDefault`) bound to cards.
5. **Adrenaline mode** class exists but unused (legacy from countdown timer era).
6. **X-Ray system** fully removed (was manual button, now abilities trigger on match).
7. **Vanish mode** CSS exists but `UNDERLAY_REVEAL` takes precedence via `.reveal-mode`.

---

## 17. File Structure

```
emoji-match/
├── index.html          # Entire game (HTML + CSS + JS)
├── .github/
│   └── workflows/
│       └── deploy.yml  # GitHub Actions → GitHub Pages
└── FADE_GAME_SPEC.md   # This document
```

---

*Generated for cross-LLM context sharing. Last updated: 2025-05-30.*
