# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## What this is

A 2015 Pac-Man clone in vanilla HTML5 Canvas + jQuery 1.8.3, vendored and unmodified. Per `README.md`, this repo is deliberately an untouched legacy codebase being used as a testbed: review it, plan cleanups, then gradually migrate it to React.

There is no build step, no package manager, no test runner, no linter, and no CI. Nothing is transpiled or bundled. `index.html` loads plain `<script>` tags in a fixed order.

## Running it

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

Serve over HTTP rather than opening `index.html` from the filesystem. `js/sound.js` constructs `buzz.sound` objects against relative `./sound/*.mp3` paths at parse time, and `file://` blocks that in most browsers.

There is nothing to lint or test yet. Jay's global rules require CI on any repo that pushes to GitHub, so adding a tooling baseline (formatter, linter, a test harness, a GitHub Actions workflow) is outstanding work, not an existing capability to invoke.

## Architecture

### Everything is a global

There are no modules, classes, closures, or namespaces. Every variable and function in `js/*.js` is a global on `window`, written in `SCREAMING_SNAKE_CASE` for state and `camelCase` for functions. Cross-file calls are direct global calls: `js/pacman.js` calls `score()` from `game.js`, `affraidGhosts()` from `ghosts.js`, and `win()` from `game.js`.

The script order in `index.html` is load-bearing. `sound.js` and `tools.js` run side effects at parse time, and `paths.js` must populate `PATHS` before any movement check runs.

Consequence: renaming or scoping any global is a repo-wide change, and there is no way to grep imports to find callers. Grep the identifier itself across `js/` and `index.html`.

### Per-entity state lives in variable names, reached by `eval`

There is no ghost object. Each ghost has a parallel set of globals named by interpolation: `GHOST_BLINKY_POSITION_X`, `GHOST_PINKY_POSITION_X`, and so on. Every ghost function then reconstructs the name at runtime:

```js
eval('var positionX = GHOST_' + ghost.toUpperCase() + '_POSITION_X');
eval('GHOST_' + ghost.toUpperCase() + '_DIRECTION = tryDirection');
```

`js/ghosts.js` is ~700 lines almost entirely because of this. `js/game.js` uses the same trick in `score()` to place the combo label, and `js/pacman.js` uses it in `testGhostPacman()`. `js/home.js` duplicates the whole pattern again with a `_TRAILER_` infix for the attract-mode animation.

Collapsing this into `GHOSTS = { blinky: { x, y, ... } }` is the single highest-leverage refactor here and is a prerequisite for any React migration. It is also the reason the globals cannot be scoped: `eval` resolves against the global scope, so moving a variable into a module or closure silently breaks every accessor.

### The maze is a line-segment graph, not a tile grid

`js/paths.js` defines the entire walkable maze as ~50 strings of the form `"startX,startY-endX,endY"`, for example `"128,416-422,416"`. Positions are raw canvas pixels, not tile indices.

`canMovePacman()` and `canMoveGhost()` both work the same way: advance a copy of the position by `POSITION_STEP` (2px), then linear-scan every segment asking whether the candidate point falls inside one. If yes the move is legal, otherwise it is blocked. The two functions are near-identical copies of that loop.

This is why every coordinate in the codebase is a hardcoded pixel magic number, and why they must agree across four independent sources of truth:

- `js/paths.js` walkable rails
- `js/board.js` the drawn maze walls, ~300 lines of literal `lineTo`/`arcTo` calls
- `js/bubbles.js` `canAddBubble()`, a long chain of line/column range exclusions carving pellet gaps out of a 29x26 lattice, plus `correctionX()` and `getYFromLine()` for per-column and per-row pixel nudges
- spawn points and the special cases below

Change any one and the others drift. Moving a wall means editing the drawing, the path segment, and the pellet exclusion ranges together.

Special-cased coordinates to know about:

- `(276, 204)` is the ghost house door. Both `canMovePacman()` and `canMoveGhost()` hardcode a downward-move block there, so only ghosts in the eaten state (`STATE === -1`) can re-enter.
- `x === 2` and `x === 548` at `y === 258` are the tunnel warp, handled by literal equality checks in the move functions. Since steps are 2px and the check is `===`, changing `POSITION_STEP` to an odd number breaks the warp.

### Canvas layers

`#board` in `index.html` stacks nine absolutely positioned 550x550 canvases: board, paths, bubbles, fruits, pacman, and one per ghost. Each entity owns its layer, so redrawing is erase-then-draw of just its own rect (`erasePacman()` clears a box around the last position), never a full-scene clear.

`#canvas-paths` renders the red debug rails. The `D` key toggle for it in `index.html` is commented out.

Do not consolidate layers without rewriting the erase logic. Correctness depends on each entity having exclusive ownership of its canvas.

### Timing: `setInterval` plus a pausable shim

Two mechanisms coexist:

- Raw `setInterval` / `setTimeout` for animation loops, with the handle kept in a `*_TIMER` global sentineled to `-1` when stopped. There is no `requestAnimationFrame` and no delta time; speed is the interval in ms (`PACMAN_MOVING_SPEED = 15`).
- `Timer` in `js/tools.js`, a `setTimeout` wrapper adding `pause()` / `resume()` / `remain()`, used for anything that must survive a pause: fruit despawn, ghost frightened duration, ghost eaten duration, Pac-Man's buffered turn. These are sentineled to `null`, not `-1`.

Most callbacks are passed as **strings** (`setInterval('movePacman()', ...)`, `new Timer("cancelAffraidGhost('blinky')", ...)`). These are evaluated in global scope, which is another reason the globals must stay global.

Lifecycle verbs are consistently paired and must stay paired: `stop*` tears down and resets state, `pause*` preserves it, `resume*` restores it. `game.js` fans these out (`pauseGame()` calls `pauseTimes()`, `pausePacman()`, `pauseGhosts()`, `stopBlinkSuperBubbles()`). Adding a timer means wiring it into all three fan-outs plus `resetGhosts()` / `resetPacman()`, or it leaks across lives and levels.

Ghost speed changes are implemented by tearing the interval down and rebuilding it: `stopGhost()` then `moveGhost()`. See `testGhostTunnel()`.

### Control flow and state flags

Five boolean globals gate input and logic, checked together throughout `index.html`'s keydown handler and the move functions:

- `HOME` attract screen is showing
- `LOCK` cutscene or transition in progress, input ignored
- `PAUSE` player paused with `P`
- `PACMAN_DEAD` death animation playing, any key triggers respawn
- `GAMEOVER`

Level flow is `initGame()` to `ready()` to `go()`, then on clearing all pellets `win()` to `prepareNextLevel()` (the board flash) to `nextLevel()`. On death, `killPacman()` to `killingPacman()` to either `retry()` or `gameover()`.

Ghost `STATE` is a tri-state integer, not a boolean: `0` normal, `1` frightened, `-1` eaten and returning home. Ghost AI in `changeDirection()` is intentionally imperfect, mixing a greedy axis-choice toward Pac-Man with the randomizers in `tools.js` (`whatsYourProblem()`, `anyGoodIdea()`, `oneAxe()`) to vary each ghost's personality. Blinky chases most often, Pinky inverts its chosen direction, Inky chases occasionally, Clyde is purely random.

### Input

All input funnels through synthetic jQuery keyboard events. `index.html` defines `simulateKeydown()` / `simulateKeyup()`, and every touch and mouse control (`#control-up`, `#control-up-big`, etc.) dispatches a fake arrow keydown on `body` rather than calling the move functions. Keep new controls on that path so the flag gating stays in one place.

Turns are buffered: `movePacman(direction)` on an illegal turn stores it via `tryMovePacman()`, which sets a 1 second `Timer` to forget it. Each subsequent tick retries the buffered direction. A successful perpendicular turn grants a one-frame `speedUp` of 6px.

### Responsive layout drives audio

The CSS scales the whole game with `zoom` (60% to 135%) across four media query breakpoints in both `css/pacman.css` and `css/pacman-home.css`, and swaps the small `#control-*` pads for the large `#control-*-big` overlays on small screens.

Non-obvious coupling: `isAvailableSound()` in `js/sound.js` gates every audio call on `$("#sound").css("display") === "none"`, and `#sound` is an empty div hidden purely by those media queries. CSS breakpoints therefore decide whether sound plays at all. Editing the media query lists silently mutes or unmutes the game.

### Prototype patching

`js/tools.js` assigns `CanvasRenderingContext2D.prototype.roundRect` and `.oval` unconditionally. Modern browsers ship a native `roundRect(x, y, w, h, radii)`, and this overwrite replaces it with an incompatible `(startX, startY, endX, endY, radius)` signature. `js/board.js` depends on the patched version. Do not remove the patch without converting the `board.js` call sites, and do not assume native `roundRect` semantics anywhere in this codebase.

## Conventions in this codebase

Match the surrounding style when editing existing files: tabs for indentation, `var` only, a space inside the opening brace of function declarations (`function foo() { `), and a blank-line-free run of related globals at the top of each file. There are almost no comments, and the few that exist are commented-out code.

The French-influenced spellings and typos are consistent and load-bearing as identifiers: `getBoardCanevasContext` (not Canvas), `affraidGhosts` (not afraid), `PACMAN_MOUNTH_STATE` (not mouth), `LIFES` (not lives). Do not spot-fix these; they appear across files and a partial rename breaks the `eval` lookups.

Google Analytics (`UA-121647007-2`) is inlined in `index.html` and points at the original author's property.
