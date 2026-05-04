# David vs Goliath — 2D Sling Game

## Context

Greenfield game build. The existing repo contents will be wiped (user confirmed "feel free to overwrite this repo") and replaced with a 2D browser game on a new branch `claude/game-planning-and-build-bq5Gy`.

The game: side-view duel between David (left) and Goliath (right) on a battlefield with two armies in the background. The player controls David, aims the sling Angry-Birds style (click-drag-release on David, with a trajectory preview), and tries to land a rock on Goliath during the brief moment Goliath is taunting/laughing — the only pose where he's vulnerable. In every other pose Goliath ducks behind his shield and the rock bounces off. Periodically Goliath winds up and throws his spear at David; the player must duck David in a tight reaction window to avoid being skewered. Single hit either way ends the round.

The build will be driven by a ralph loop: a `prompt.md` plus an evolving `TASKS.md` checklist, so each loop iteration reads state, picks the next unchecked task, implements it, and ticks it off.

## Tech stack

- **Phaser 3** (via CDN, no build step) — 2D engine with sprite/animation/input/arcade-physics built in. Runs from a static `index.html`, which is friction-free for the ralph loop.
- **Vanilla ES modules** for source organization, no bundler.
- **No tests, no audio** for v1 (user said silent).
- Serve locally with `python3 -m http.server` for manual playtesting.

## Game design (locked from user answers)

| Decision | Choice |
|---|---|
| View | Side view, both characters on the ground |
| Aim | Angry-Birds: mouse-down on David, drag back to set angle + power, release to throw |
| Goliath hittable window | Only in the laughing/taunting pose; otherwise he ducks behind shield as rock approaches |
| Goliath spear throw | Periodic; wind-up telegraph (overhead-spear pose) gives David ~600 ms to duck |
| David dodge | Hold `Space` (or `↓`) to crouch; release to stand. Hit while standing during spear arrival = lose |
| Win | One rock landed during laughing pose = victory |
| Lose | One spear landed on standing David = defeat |
| Scope | Single duel; "click to restart" on win/lose |
| Audio | None |
| Background | Battlefield with two armies facing each other (Israelites left, Philistines right), simple parallax-free painted bg generated as a separate PNG or drawn programmatically |

### Goliath state machine

```
IDLE → (random) → TAUNTING → IDLE
IDLE → (random) → SPEAR_WINDUP → SPEAR_THROW → IDLE
At any time, if rock incoming AND state ≠ TAUNTING → DUCK (transient, blocks rock)
TAUNTING + rock contact → HIT → DEFEATED (game over: win)
```

Tunables in `src/config.js`:
- `tauntDurationMs` (default 900)
- `tauntCooldownMs` (default 1500–3000 random)
- `spearWindupMs` (default 600)
- `spearTravelMs` (default 450)
- `spearCooldownMs` (default 3000–5000 random)
- `rockGravity`, `rockMaxPower`
- Hitboxes for Goliath head (rock target) and David standing/crouching

## File layout

```
/
├── index.html                  # loads Phaser CDN + main.js as ES module
├── src/
│   ├── main.js                 # Phaser config, scene registration
│   ├── config.js               # all tunables
│   ├── BootScene.js            # asset preload
│   ├── GameScene.js            # gameplay: David, Goliath, rock, spear, hit detection
│   └── UIScene.js              # title, win/lose overlays, "click to restart"
├── assets/
│   └── sprites/
│       ├── david.png           # 4×2 grid, dropped by user
│       └── goliath.png         # 4×2 grid, dropped by user
├── prompt.md                   # ralph-loop prompt
├── TASKS.md                    # incremental task checklist for ralph
└── README.md                   # how to run + how to ralph
```

Sprite sheets are interpreted as **4 columns × 2 rows = 8 frames each** (reading order, top-left = frame 0). Frame indices for poses are kept in `config.js` so they can be corrected in one place if the interpretation is off.

Tentative frame mapping (verify on first iteration by visually comparing in-browser):
- **David**: 0=idle, 1=sling-ready, 2=aim-low, 3=aim-forward, 4=aim-up, 5=throw-release, 6=walk, 7=victory
- **Goliath**: 0=idle, 1=stance, 2=ducking-shield, 3=spear-overhead, 4=laughing-taunt (HITTABLE), 5=shield-down, 6=hit-reaction, 7=defeated

## Ralph loop design

`prompt.md` contents (high level):
1. Read `TASKS.md`. Pick the first unchecked `[ ]` task.
2. Read any files relevant to that task.
3. Implement it. Keep changes scoped to that task.
4. Manually verify by reasoning about the change (no test suite).
5. Mark the task `[x]` and commit with a one-line message naming the task.
6. If all tasks are checked, do nothing and exit.

`TASKS.md` initial checklist (each item is one ralph iteration's worth of work):
1. Wipe repo, scaffold `index.html`, `src/main.js`, empty scenes, hello-world Phaser canvas.
2. `BootScene` loads `david.png` and `goliath.png` as 4×2 spritesheets; `GameScene` renders one frame of each at the correct positions.
3. Draw battlefield background (sky, ground, two distant armies as silhouette rectangles for now — refine later).
4. Implement David sling aim: click-drag-release, trajectory preview dots, rock projectile with gravity.
5. Implement Goliath state machine + animations (idle / taunt / windup / throw / duck-on-rock).
6. Rock vs Goliath hit detection: only register a hit when state == TAUNTING and rock overlaps head hitbox.
7. Implement Goliath spear throw: spawn spear projectile from spear-overhead pose, travel toward David's standing-height position.
8. Implement David crouch: hold Space/↓ to lower hitbox; spear collides with standing hitbox only.
9. Win flow: Goliath plays hit → defeated frames; UIScene shows "Victory — click to restart".
10. Lose flow: David plays hit/fallen state; UIScene shows "Defeated — click to restart".
11. Polish: replace silhouette armies with hand-drawn battlefield bg; tune timing constants; verify frame mapping is correct, fix in `config.js` if not.

Each iteration is small enough to fit easily in one Claude turn. Items 1–10 are functional milestones; item 11 is iterative polish that the loop can repeat-and-refine.

## Verification

- `python3 -m http.server 8000` from repo root, open `http://localhost:8000`.
- Manual playtest checklist per task (documented in `TASKS.md` as part of each task's acceptance):
  - Sprites render upright at correct positions, no z-fighting.
  - Sling drag shows trajectory preview; release throws rock with arc.
  - Rock hits Goliath only during laughing pose; otherwise visibly blocked/dodged.
  - Spear wind-up is clearly telegraphed; ducking before/during throw avoids hit.
  - Win and lose end-states are reachable and restartable.
- No build step, so the only failure modes are runtime JS errors (visible in browser console) and visual bugs.

## Open items deferred to execution

- Exact frame index for "laughing" pose — I'll confirm visually on iteration 2 once the spritesheet is loaded; adjust `config.js` if needed.
- Battlefield background art — start with simple shapes, optionally upgrade later (out-of-scope for the must-ship loop).
