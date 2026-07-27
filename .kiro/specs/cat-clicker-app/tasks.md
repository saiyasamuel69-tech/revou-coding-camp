# Implementation Plan: Cat Clicker App

## Overview

Build the Cat Clicker App as a single `index.html` file using inline HTML, CSS (with Tailwind CDN), and Vanilla JavaScript. The implementation follows a layered approach: scaffold structure first, then add state and logic, then wire up animations and reactions, then add the decay loop and polish.

All property-based tests use [fast-check](https://github.com/dubzzz/fast-check) and run as a separate `tests.html` file that imports the extracted pure functions.

---

## Tasks

- [~] 1. Scaffold the single-file HTML structure
  - Create `index.html` with a valid HTML5 boilerplate
  - Add Tailwind CDN `<script>` tag in `<head>`
  - Add all required DOM containers: `#cat` (SVG wrapper), `#pet-counter`, `#happiness-bar`, `#happiness-fill`, `#speech-bubble`
  - Ensure the file can be opened via `file://` protocol with no errors
  - _Requirements: 7.1, 7.2, 7.3, 7.4_

- [ ] 2. Implement the Cat SVG character and idle animation
  - [-] 2.1 Embed inline SVG cat composed of ellipses (body/head), triangles (ears), and circles (eyes)
    - Apply base CSS classes for idle breathing/tail-sway looping keyframe animation (`cat--idle`)
    - Ensure the cat is visually distinct from the background with contrasting colours or outlines
    - Centre the cat in the viewport using Tailwind flex utilities; verify no horizontal overflow at 320 px and 1920 px
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 8.1, 8.2_

  - [ ]* 2.2 Write unit tests for initial DOM state
    - Assert cat SVG is present in DOM on load
    - Assert idle animation class is present before any interaction
    - Assert `#pet-counter` text is "0" on load
    - Assert `#happiness-fill` width is "0%" on load
    - Assert `#happiness-bar` (Happiness_Meter) is visible in DOM
    - _Requirements: 1.3, 4.3, 5.1_

- [ ] 3. Define app state and reaction pool
  - [~] 3.1 Declare the `state` object with all six fields: `petCount`, `happiness`, `lastPetTime`, `lastReactionIndex`, `isAnimating`, `pendingPets`
    - Initialise all fields to their documented defaults
    - Define the `REACTIONS` array with all 7 reaction objects (`purr`, `boop`, `happy`, `slow-blink`, `knead`, `roll`, `zoomies`), each with `id`, `emoji`, `animClass`, `animDuration`, `bubbleDuration`
    - _Requirements: 3.1, 4.3, 5.1_

  - [ ]* 3.2 Write property test for reaction pool constraints (Property 7)
    - **Property 7: Reaction timing constraints**
    - For every reaction in `REACTIONS`, assert `animDuration ∈ [400, 1200]` and `bubbleDuration ∈ [1500, 2500]`
    - **Validates: Requirements 3.3, 3.4**
    - Tag: `// Feature: cat-clicker-app, Property 7: reaction_timing_constraints`

- [ ] 4. Implement core pure functions
  - [~] 4.1 Implement `updateCounter()` — increment `state.petCount` by 1, update `#pet-counter` text
    - _Requirements: 4.2, 4.4_

  - [ ]* 4.2 Write property test for pet counter increment (Property 1)
    - **Property 1: Pet counter monotonic increment**
    - For any initial `petCount = n ≥ 0`, after calling `updateCounter()` once, assert `petCount === n + 1`
    - **Validates: Requirements 4.2, 4.4**
    - Tag: `// Feature: cat-clicker-app, Property 1: pet_counter_monotonic_increment`

  - [~] 4.3 Implement `increaseHappiness()` — add 10, cap at 100 with `Math.min`; set `state.lastPetTime = Date.now()`
    - _Requirements: 5.2_

  - [ ]* 4.4 Write property test for happiness increment cap (Property 3)
    - **Property 3: Happiness increment caps at 100**
    - For any `h ∈ [0, 100]`, set `state.happiness = h`, call `increaseHappiness()`, assert `state.happiness === Math.min(h + 10, 100)`
    - **Validates: Requirements 5.2**
    - Tag: `// Feature: cat-clicker-app, Property 3: happiness_increment_caps_at_100`

  - [~] 4.5 Implement `decayTick()` — decrease `state.happiness` by 1 only when `Date.now() - state.lastPetTime > 3000` and `state.happiness > 0`; floor at 0 with `Math.max`
    - _Requirements: 5.3_

  - [ ]* 4.6 Write property test for happiness decay floor (Property 4)
    - **Property 4: Happiness decay stops at 0**
    - For any `h ∈ [0, 100]`, set `state.happiness = h` and `state.lastPetTime = 0` (idle), call `decayTick()`, assert `state.happiness === Math.max(h - 1, 0)`
    - **Validates: Requirements 5.3**
    - Tag: `// Feature: cat-clicker-app, Property 4: happiness_decay_stops_at_0`

  - [~] 4.7 Implement `selectReaction()` — select a random reaction from `REACTIONS`; guarantee the selected index differs from `state.lastReactionIndex`; update `state.lastReactionIndex`
    - _Requirements: 3.2, 3.5_

  - [ ]* 4.8 Write property test for no-consecutive-repeat reactions (Property 5)
    - **Property 5: No consecutive identical reactions**
    - For any sequence of N ≥ 2 pet events, assert no two consecutive calls to `selectReaction()` return the same `id`
    - **Validates: Requirements 3.5**
    - Tag: `// Feature: cat-clicker-app, Property 5: no_consecutive_identical_reactions`

  - [ ]* 4.9 Write property test for reaction in pool bounds (Property 6)
    - **Property 6: Reaction selection is within pool bounds**
    - For any pet event, assert the returned reaction is an element present in `REACTIONS`
    - **Validates: Requirements 3.2**
    - Tag: `// Feature: cat-clicker-app, Property 6: reaction_selection_within_pool_bounds`

- [~] 5. Checkpoint — Ensure all pure function tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement `applyState()` and DOM rendering
  - [~] 6.1 Implement `applyState()` — reflect current state into DOM:
    - Set `#pet-counter` text to `state.petCount`
    - Set `#happiness-fill` inline `width` to `state.happiness + '%'`
    - Toggle `cat--sad` class on cat when `state.happiness === 0`
    - Toggle `cat--max` class on cat and `meter--max` class on meter when `state.happiness === 100`
    - _Requirements: 5.5, 5.6, 5.7_

  - [ ]* 6.2 Write property test for happiness meter fill proportion (Property 8)
    - **Property 8: Happiness meter fill is proportional**
    - For any `h ∈ [0, 100]`, call `applyState()` with `state.happiness = h`, assert `#happiness-fill` style width equals `h + '%'`
    - **Validates: Requirements 5.6**
    - Tag: `// Feature: cat-clicker-app, Property 8: happiness_meter_fill_proportional`

- [ ] 7. Implement reaction playback (`playReaction()`) and speech bubble
  - [~] 7.1 Implement `playReaction(reaction)` — set `state.isAnimating = true`; add `reaction.animClass` to cat; show `#speech-bubble` with `reaction.emoji` text and class `speech-bubble--visible`; after `reaction.animDuration` ms remove `animClass`; after `reaction.bubbleDuration` ms hide speech bubble; set `state.isAnimating = false`; process `state.pendingPets` if any
    - Add CSS for speech bubble fade-in (≤ 200 ms) and fade-out (≤ 300 ms) using Tailwind or inline `<style>`
    - Add CSS keyframe animations for all `cat--{reaction}` classes and `cat--sad` / `cat--max` states
    - _Requirements: 3.3, 3.4, 3.6, 5.4, 5.5, 6.1, 6.2, 6.4_

  - [ ]* 7.2 Write unit tests for speech bubble lifecycle
    - Assert speech bubble is not visible on load
    - Assert speech bubble becomes visible after a pet
    - Assert `cat--sad` is applied when happiness is 0 and removed on next pet
    - Assert `cat--max` is applied when happiness is 100
    - _Requirements: 3.4, 3.6, 5.4, 5.5_

- [ ] 8. Implement `handlePet()` and event wiring
  - [~] 8.1 Implement `handlePet(event)` — call `event.preventDefault()`; if `state.isAnimating` increment `state.pendingPets` and return; otherwise call `updateCounter()`, `increaseHappiness()`, `selectReaction()`, `playReaction()`, `applyState()`
    - Wire `click` and `touchend` event listeners on the cat element inside `DOMContentLoaded`
    - Wire `touchstart` with `preventDefault` to suppress scroll/zoom
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 8.2 Write unit tests for event handling
    - Assert mouse click on cat registers a pet (counter increments)
    - Assert touch tap on cat registers a pet (counter increments)
    - Assert `preventDefault` is called on touch events
    - Assert pets during animation are queued (not dropped)
    - _Requirements: 2.1, 2.3, 2.4, 2.5_

- [ ] 9. Implement happiness decay loop
  - [~] 9.1 Wire `decayTick()` to `setInterval` with 500 ms interval inside `DOMContentLoaded`
    - Ensure `applyState()` is called after each tick to sync DOM
    - Ensure guard `if (state.happiness > 0 && now - state.lastPetTime > 3000)` is respected
    - _Requirements: 5.3_

  - [ ]* 9.2 Write property test for happiness always bounded (Property 2)
    - **Property 2: Happiness is always bounded**
    - For any sequence of arbitrary pet events and decay ticks, assert `state.happiness` remains in `[0, 100]`
    - **Validates: Requirements 5.2, 5.3**
    - Tag: `// Feature: cat-clicker-app, Property 2: happiness_always_bounded`

- [ ] 10. Implement responsive layout and visual polish
  - [~] 10.1 Apply Tailwind responsive classes to centre the cat and stack the HUD (counter + meter) for all viewport widths 320 px–1920 px
    - Ensure no horizontal overflow at any supported width
    - Ensure the cat remains the visual centrepiece at all sizes
    - Add CSS `transition` on `#happiness-fill` of ≤ 300 ms
    - Ensure smooth CSS transition when cat changes between idle, reacting, and sad states
    - _Requirements: 8.1, 8.2, 8.3, 5.7, 6.4_

  - [ ]* 10.2 Write property test for no horizontal overflow (Property 9)
    - **Property 9: Viewport layout — no horizontal overflow**
    - For any viewport width `w ∈ [320, 1920]` px, assert `document.body.scrollWidth <= w` after resizing
    - **Validates: Requirements 8.1**
    - Tag: `// Feature: cat-clicker-app, Property 9: no_horizontal_overflow`

- [~] 11. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.
  - Verify the file opens from `file://` protocol with no console errors
  - Verify Tailwind CDN link is present and all core functionality works without additional packages

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP.
- Property-based tests should be written in a separate `tests.html` (or `tests.js`) file using [fast-check](https://github.com/dubzzz/fast-check) loaded from CDN, with pure functions extracted from `index.html` or imported via a shared module pattern.
- Each property test should run a minimum of 100 iterations.
- Checkpoints ensure incremental validation before proceeding to the next phase.
- All requirements references use the format `Requirement X.Y` matching `requirements.md`.

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1"] },
    { "id": 1, "tasks": ["2.2", "3.1"] },
    { "id": 2, "tasks": ["3.2", "4.1", "4.3", "4.5", "4.7"] },
    { "id": 3, "tasks": ["4.2", "4.4", "4.6", "4.8", "4.9", "6.1"] },
    { "id": 4, "tasks": ["6.2", "7.1"] },
    { "id": 5, "tasks": ["7.2", "8.1"] },
    { "id": 6, "tasks": ["8.2", "9.1"] },
    { "id": 7, "tasks": ["9.2", "10.1"] },
    { "id": 8, "tasks": ["10.2"] }
  ]
}
```
