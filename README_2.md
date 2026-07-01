# Contribution #2: `key` may be incorrect inside an event handler

**Contribution Number:** 2  
**Student:** Quoc-Viet Bui  
**Issue:** [processing/p5.js#7569 — key may be incorrect inside an event handler](https://github.com/processing/p5.js/issues/7569)  
**Status:** Phase I — In Progress

---

## Why I Chose This Issue

In p5.js 1.11.3, the global `key` variable holds the wrong character inside `keyReleased` (and, on Firefox/Linux, `keyPressed` fires continuously while a key is held). This is a regression from 1.11.2 that breaks a bread-and-butter p5.js interaction pattern — reading `key`/`keyCode` in keyboard event handlers — making any sketch that relies on it (drag-with-spacebar, arrow-key movement, etc.) behave unpredictably. The root cause is a one-line removal in PR #7435, so the fix is small and localized, yet it has a direct, visible impact on sketch authors.

I chose this issue because it is sharply scoped to `src/events/keyboard.js`, the root cause is already pinpointed in the maintainer thread (limzykenneth identified the removed line and davepagurek blessed reverting it), and fixing it touches real input-handling semantics — a good way to learn how p5.js wires native DOM events to its global `key`/`keyCode` state. The added Firefox/Linux `e.repeat` complication makes the testing strategy non-trivial without pulling in deep architecture changes.

---

## Understanding the Issue

### Problem Description

PR #7435 (fixing #7282) removed the assignment `key = e.key;` from both `_onKeyDown` and `_onKeyUp` in `src/events/keyboard.js`. As a result, the global `key` is no longer updated on each keydown/keyup event; it only reflects the most recently "typed" key, so when two keys are partially overlapping (e.g. `d` held, then `a` pressed and released, then `d` released), `key` inside `keyReleased` is `a` rather than the `d` that fired the event. The same PR introduced an `e.repeat`-based guard that misbehaves on Firefox/Linux, so `keyPressed` fires repeatedly instead of once per press.

### Expected Behavior

- Inside `keyReleased`, `key` should equal the key that was actually released (i.e. `ev.key`).
- Inside `keyPressed`, the handler should fire once on the initial keydown (not on browser-induced key-repeat) across all supported browsers — including Firefox on Linux.
- Behavior should match p5.js 1.11.2 for sketches that don't rely on the new `e.repeat` semantics.

### Current Behavior

```js
function keyPressed(ev) {
  console.log(`pressed ${ev.key} (p5.key = ${key})`);
}

function keyReleased(ev) {
  console.log(`released ${ev.key} (p5.key = ${key})`);
}
```

Steps: press `d`, press+release `a`, release `d`.

- 1.11.3, `keyReleased`: `key` is `a`, `ev.key` is `d`. **Mismatch.**
- 1.11.3, Firefox/Linux: `keyPressed` fires continuously while `d` is held (should fire once).
- 1.11.2 and earlier: both behaviors are correct.

### Affected Components

- `src/events/keyboard.js` — `_onKeyDown` and `_onKeyUp` handlers; the removed `key = e.key;` line (~L627 before removal) and the `e.repeat` repeat-detection guard added in PR #7435.
- PR #7435 (commit `c1c553d4`) — introduced the regression; reference for what the fix must preserve.
- [p5.js `key` reference](https://p5js.org/reference/p5/key/) — semantic contract for `key` in event handlers.
- Cross-browser follow-ups:
  - Firefox on Linux: `e.repeat` unreliable (see comment by limzykenneth).
  - Chromium: separate upstream bug [issues/40940886](https://issues.chromium.org/issues/40940886).

---

## Reproduction Process

### Environment Setup

*To be filled in during Phase II.*

### Steps to Reproduce

1. Open p5.js Web Editor (or local sketch) with p5.js `1.11.3`.
2. Paste the `keyPressed`/`keyReleased` snippet above.
3. Press and hold `d`; pressing then releasing `a`; release `d`.
4. Observe the console: `keyReleased` reports `p5.key = a` while `ev.key = d`.
5. On Firefox/Linux: hold `d` — `keyPressed` fires repeatedly (confirm before fix).

### Reproduction Evidence

- **Commit showing reproduction:** *To be filled in during Phase II*
- **Screenshots/logs:** *To be filled in during Phase II*
- **My findings:** *To be filled in during Phase II*

---

## Solution Approach

### Analysis

*To be filled in during Phase II.*

### Proposed Solution

*To be filled in during Phase II.*

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** *To be filled in during Phase II*

**Match:** *To be filled in during Phase II*

**Plan:** *To be filled in during Phase II*

**Implement:** *To be filled in during Phase II*

**Review:** *To be filled in during Phase II*

**Evaluate:** *To be filled in during Phase II*

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: *To be filled in during Phase III*
- [ ] Test case 2: *To be filled in during Phase III*
- [ ] Test case 3: *To be filled in during Phase III*

### Integration Tests

- [ ] *To be filled in during Phase III*

### Manual Testing

*To be filled in during Phase III.*

---

## Implementation Notes

### Week 1 Progress

Selected issue, read the maintainer thread to pin the root cause to the removed `key = e.key;` line in PR #7435 and the `e.repeat` repeat-detection gate, set up README.

### Code Changes

- **Files modified:** None yet
- **Key commits:** *To be filled in during Phase III*
- **Approach decisions:** *To be filled in during Phase III*

---

## Pull Request

**PR Link:** *To be filled in during Phase IV*

**PR Description:** *To be filled in during Phase IV*

**Maintainer Feedback:**
- *To be filled in during Phase IV*

**Status:** Not yet submitted

---

## Learnings & Reflections

### Technical Skills Gained

*To be filled in at completion.*

### Challenges Overcome

*To be filled in at completion.*

### What I'd Do Differently Next Time

*To be filled in at completion.*

---

## Resources Used

- [Issue #7569](https://github.com/processing/p5.js/issues/7569)
- [PR #7435](https://github.com/processing/p5.js/pull/7435) (regression source, commit `c1c553d4`)
- [Issue #7282](https://github.com/processing/p5.js/issues/7282) (the bug #7435 was fixing)
- [Chromium bug 40940886 — keyboard auto-repeat](https://issues.chromium.org/issues/40940886)
- [p5.js `key` reference](https://p5js.org/reference/p5/key/)
- `src/events/keyboard.js`