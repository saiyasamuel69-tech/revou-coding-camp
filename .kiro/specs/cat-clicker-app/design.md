# Design Document: Cat Clicker App

## Overview

The Cat Clicker App is a self-contained single HTML file that delivers a high-engagement, lighthearted interaction loop. A minimalist CSS/SVG cat sits centre-stage; users click or tap it to "pet" it, triggering random animated reactions and speech bubbles. A pet counter tallies interactions, and a happiness meter provides a live feedback loop that slowly decays when the cat is ignored, encouraging continued play.

The entire app ships as one `.html` file using Tailwind CSS loaded from CDN and Vanilla JavaScript — no bundler, no server, no dependencies to install.

---

## Architecture

The app follows a simple **single-file MVC-lite pattern** where all state lives in a plain JavaScript object, all rendering is driven by DOM mutations and CSS class toggling, and user events are wired up on DOMContentLoaded.

```
┌──────────────────────────────────────────────────────┐
│                    index.html                        │
│                                                      │
│  ┌─────────┐   events   ┌─────────────────────────┐ │
│  │  DOM /  │ ─────────► │   App Controller        │ │
│  │  SVG    │            │  handlePet()            │ │
│  │  Cat    │ ◄───────── │  selectReaction()       │ │
│  └─────────┘  mutations │  updateHappiness()      │ │
│                         │  updateCounter()        │ │
│  ┌─────────┐            │  decayLoop()            │ │
│  │ UI HUD  │ ◄───────── │  applyState()           │ │
│  │ counter │  mutations └─────────────────────────┘ │
│  │ meter   │                      │                  │
│  └─────────┘              ┌───────┴──────┐           │
│                           │  App State   │           │
│                           │  petCount    │           │
│                           │  happiness   │           │
│                           │  lastPetTime │           │
│                           │  lastReaction│           │
│                           │  isAnimating │           │
│                           └──────────────┘           │
└──────────────────────────────────────────────────────┘
```

**Data flow:**
1. User click/tap → `handlePet()` → mutate state → `applyState()` → DOM update
2. `decayLoop()` ticks every 500 ms → mutate happiness → `applyState()` → DOM update
3. All visual feedback is CSS-class-driven; JavaScript only assigns/removes classes and sets inline style for the meter fill width.

---

## Components and Interfaces

### 1. App State Object

```js
const state = {
  petCount: 0,           // integer >= 0
  happiness: 0,          // integer in [0, 100]
  lastPetTime: 0,        // timestamp (ms) of most recent pet
  lastReactionIndex: -1, // index of last reaction to prevent repeat
  isAnimating: false,    // whether a reaction animation is in flight
  pendingPets: 0,        // queued pets during animation
};
```

### 2. Reaction Pool

Each reaction is a plain object:

```js
{
  id: String,            // unique identifier
  emoji: String,         // speech bubble text / emoji
  animClass: String,     // CSS class name applied to cat during reaction
  animDuration: Number,  // ms, animation play time (400–1200)
  bubbleDuration: Number // ms, speech bubble visible time (1500–2500)
}
```

**Reaction Pool (minimum 6):**

| id | emoji / text | animClass | animDuration | bubbleDuration |
|---|---|---|---|---|
| `purr` | "Purrr… 😻" | `cat--purr` | 600 | 2000 |
| `boop` | "Boop! 👆" | `cat--boop` | 400 | 1500 |
| `happy` | "So happy! 😸" | `cat--happy` | 800 | 2000 |
| `slow-blink` | "I trust you 💕" | `cat--slow-blink` | 1200 | 2500 |
| `knead` | "Making biscuits 🍞" | `cat--knead` | 1000 | 2500 |
| `roll` | "Belly rubs! 🙈" | `cat--roll` | 900 | 2000 |
| `zoomies` | "ZOOMIES!! 🌀" | `cat--zoomies` | 500 | 1500 |

### 3. Cat SVG Character

Inline SVG embedded directly in the HTML. The cat is composed of simple geometric shapes (ellipses for the body/head, triangles for ears, circles for eyes). CSS classes control its visual state:

| CSS Class | State |
|---|---|
| *(none / base)* | idle — gentle breathing animation |
| `cat--sad` | happiness = 0 — drooping ears, closed eyes |
| `cat--max` | happiness = 100 — sparkles, extra-wide smile |
| `cat--{reaction}` | active reaction animation |

### 4. HUD (Heads-Up Display)

- **Pet Counter** — `<div id="pet-counter">` displays `petCount`
- **Happiness Meter** — `<div id="happiness-bar">` with an inner fill `<div id="happiness-fill">` whose inline `width` style is set to `happiness + '%'`
- **Speech Bubble** — `<div id="speech-bubble">` absolutely positioned above the cat; shown/hidden via CSS class `speech-bubble--visible`

### 5. Controller Functions

```js
// Entry point wired to click + touchend on cat element
function handlePet(event)

// Randomly select next reaction (no-repeat)
function selectReaction() → Reaction

// Apply chosen reaction to DOM (animation class + speech bubble)
function playReaction(reaction)

// Increment petCount, update DOM counter
function updateCounter()

// Add 10 to happiness (cap 100), handle threshold states
function increasHappiness()

// Called every 500ms by setInterval; decrease happiness by 1 when idle
function decayTick()

// Reflect current state into DOM (meter fill, cat class, counter text)
function applyState()
```

---

## Data Models

### State Invariants

- `state.petCount ≥ 0` always
- `state.happiness ∈ [0, 100]` always
- `state.lastReactionIndex ∈ [-1, REACTIONS.length - 1]` always
- `state.isAnimating ∈ {true, false}` always
- `state.pendingPets ≥ 0` always

### Reaction Constraints

- `REACTIONS.length ≥ 6`
- All reaction `id` values are unique strings
- Each `animDuration ∈ [400, 1200]` ms
- Each `bubbleDuration ∈ [1500, 2500]` ms

### Happiness Transitions

```
increasHappiness:   happiness' = min(happiness + 10, 100)
decayTick:          happiness' = max(happiness - 1, 0)  [only when now - lastPetTime > 3000]
```

### Special Thresholds

- `happiness === 100` → add class `cat--max` to cat, add class `meter--max` to meter bar for ≥ 1000 ms
- `happiness === 0` → add class `cat--sad` to cat; remove on next pet

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Pet counter monotonic increment

*For any* initial pet counter value `n ≥ 0`, after simulating exactly one pet event, the pet counter value should equal `n + 1`.

**Validates: Requirements 4.2, 4.4**

---

### Property 2: Happiness is always bounded

*For any* sequence of pet events and decay ticks of arbitrary length and order, the happiness value should always remain in the range [0, 100].

**Validates: Requirements 5.2, 5.3**

---

### Property 3: Happiness increment caps at 100

*For any* happiness value `h ∈ [0, 100]`, after one pet event the resulting happiness should equal `min(h + 10, 100)`.

**Validates: Requirements 5.2**

---

### Property 4: Happiness decay stops at 0

*For any* happiness value `h ∈ [0, 100]`, after one decay tick (with idle time > 3 s) the resulting happiness should equal `max(h - 1, 0)`.

**Validates: Requirements 5.3**

---

### Property 5: No consecutive identical reactions

*For any* sequence of `N ≥ 2` pet events, no two consecutive reactions should share the same reaction `id`.

**Validates: Requirements 3.5**

---

### Property 6: Reaction selection is within pool bounds

*For any* pet event, the selected reaction should be an element present in REACTIONS — i.e., the selection index is always in `[0, REACTIONS.length - 1]`.

**Validates: Requirements 3.2**

---

### Property 7: Reaction timing constraints

*For any* reaction in REACTIONS, its `animDuration` should be in [400, 1200] ms and its `bubbleDuration` should be in [1500, 2500] ms.

**Validates: Requirements 3.3, 3.4**

---

### Property 8: Happiness meter fill is proportional

*For any* happiness value `h ∈ [0, 100]`, the happiness fill element's width style should equal `h + '%'`.

**Validates: Requirements 5.6**

---

### Property 9: Viewport layout — no horizontal overflow

*For any* viewport width `w ∈ [320, 1920]` px, the document body's scroll width should not exceed `w`.

**Validates: Requirements 8.1**

---

## Error Handling

| Scenario | Handling Strategy |
|---|---|
| User pets rapidly during animation | `pendingPets` counter queues extra pets; processed sequentially after animation completes |
| Decay interval fires when happiness is already 0 | Guard: `if (state.happiness > 0)` before decrement |
| Happiness would exceed 100 from pet | Clamped with `Math.min(state.happiness + 10, 100)` |
| Touch event causes unintended scroll/zoom | `event.preventDefault()` called on `touchstart` and `touchend` on the cat element |
| Browser does not support CSS animations | Cat remains functional; interactions still register; degraded but not broken |

---

## Testing Strategy

PBT is applicable here because the core logic (happiness clamping, reaction selection, counter increments) consists of pure functions operating over a well-defined input space.

### Dual Testing Approach

**Unit Tests** — specific examples and edge cases:
- Cat SVG is present in DOM on load
- Pet counter initialises to 0
- Happiness initialises to 0
- `cat--sad` class applied when happiness = 0
- `cat--max` class applied when happiness = 100
- Mouse click and touch tap both register pets
- `preventDefault` called on touch events
- Idle animation class present before any interaction
- Happiness_Meter element is visible in DOM
- CSS transition on meter bar is ≤ 300 ms
- Reaction pool contains ≥ 6 unique reactions

**Property-Based Tests** — using [fast-check](https://github.com/dubzzz/fast-check) (JavaScript PBT library):

Each test should run a minimum of **100 iterations**.

| # | Property | Source |
|---|---|---|
| 1 | Pet counter increments by exactly 1 for any initial value | Property 1 |
| 2 | Happiness always in [0, 100] for any event sequence | Property 2 |
| 3 | `increaseHappiness` obeys `min(h+10, 100)` for any h | Property 3 |
| 4 | `decayTick` obeys `max(h-1, 0)` for any h | Property 4 |
| 5 | No two consecutive reactions are identical for any N-pet sequence | Property 5 |
| 6 | Selected reaction is always in REACTIONS pool for any pet | Property 6 |
| 7 | All reactions have timing in spec bounds | Property 7 |
| 8 | Meter fill = `h + '%'` for any h in [0, 100] | Property 8 |
| 9 | No horizontal overflow for any viewport width in [320, 1920] | Property 9 |

**Tag format:** `// Feature: cat-clicker-app, Property {N}: {property_text}`

**Integration / Smoke Tests:**
- File loads correctly from `file://` protocol
- No JavaScript console errors on load or interaction
- Tailwind CDN link resolves (network required)
- Manual: visual check of contrast, smoothness, and idle animation
