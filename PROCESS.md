# PROCESS.md — Bulldozer Demolition

This document records the process of developing the project: the initial concept, the design
and code changes made along the way, the user's own test records and how each piece of feedback
was answered, the AI tools that were used and exactly what they were used for, and a final
section listing one suggestion that was **adopted**, one that was **modified**, and one that
was **rejected**.

---

## 1. Initial concept

### 1.1 Reading the brief

The assignment asked for an *interactive experience* that could not be carried by clicking
through text-based menus. The first design question was therefore not "what does the player
click" but "what is the verb". The answer became a single verb: **ram**.

The pitch that was settled on:

> You drive a bulldozer around a small cartoon city and flatten it. Every building cracks,
> tilts, topples, and shatters into debris that settles into rubble and fades away.

Two constraints shaped everything downstream:

- **Zero external art assets.** Everything — models, textures, audio — is generated in code
  (Three.js geometry + `CanvasTexture` + Web Audio API). The whole experience ships as a
  single `index.html`.
- **Physical honesty.** Destruction is not a scripted animation. Buildings are real rigid
  bodies; they break because the vehicle's momentum breaks them.

### 1.2 The first idea that was thrown away

The very first mechanical proposal was a **wrecking ball** — a swinging chain anchored above
the city that the player would drive a truck under, aiming the momentum at buildings. It was
rejected before any code was written.

**Why it was rejected:** the interaction was too narrow and too indirect. The player was not
acting, they were *positioning* — a single-axis timing toy. The brief asked for an
interactive experience, and a swinging pendulum is closer to "watch and trigger" than to
"play". A bulldozer gives the player **continuous, multi-axis freedom**: they choose the
approach angle, the approach speed, whether to reverse out or push straight through, and
which building to hit next. Freedom of movement *is* the game.

The result is that the shipped game has a bulldozer and **no** wrecking ball. The chain
anchor, the pendulum joint, and the swing-timing mechanic do not exist anywhere in the
codebase.

### 1.3 Feature set at the end of the concept phase

| Feature | Why it exists |
| --- | --- |
| 45° top-down follow camera | Reads as a tabletop diorama; the whole play area stays legible |
| WASD + Space driving | "Press a key and something explodes" — no tutorial needed |
| Staged destruction (crack → topple → shatter → fade) | A hit should read as a *sequence*, not a single pop |
| Squashing small props (trees, lamps, crates, walls) | Contrast — not everything should explode |
| First-hit-only camera shake | Reward for the meaningful hit; noise otherwise |
| Bullet time on the first building hit | Marks the moment the sandbox "opens up" |
| Procedural Web Audio | Engine pitch tied to speed, no audio files |
| Toon / cel-shaded look | Stylistic identity + reads well at low poly counts |

---

## 2. Design and code changes

The entire application lives in `index.html` (~1,080 lines). Three.js r160 and cannon-es
0.20 are loaded through an import map from unpkg. Vendored copies of both libraries are kept
in `lib/` so an offline fallback is a one-line import-map change away, although the shipped
import map currently points at the CDN.

### 2.1 Physics foundation

- `CANNON.World` with `gravity = (0, -22, 0)`, `friction = .45`, `restitution = .08` as
  defaults.
- **Four separate contact materials** — `matGround`, `matBox`, `matDozer`, `matWall` — with
  the dozer pairings deliberately low-friction (`matGround×matDozer` friction `.06`,
  restitution `0`) so the tracks grip instead of fighting the ground.
- `ARENA = 92` radius playfield bounded by four static walls.
- Rendering is decoupled from the simulation: physics runs at a fixed 1/60 step
  (`world.step(1/60, dt, 3)`), while meshes are drawn at an interpolated
  `renderPos()` / `renderQuat()` so the vehicle does not judder when the render frame and
  the physics step disagree.

### 2.2 Vehicle model

The bulldozer is a 110 kg rigid body with a compound shape (tracks + hull + blade). Driving
is a **velocity-blending** model rather than direct velocity assignment:

```javascript
const vx = fwd.x * dozer.speed, vz = fwd.z * dozer.speed;
b.velocity.x += (vx - b.velocity.x) * k;
b.velocity.z += (vz - b.velocity.z) * k;
```

`k` is `dt * 30` normally and drops to `dt * 12` for 0.12 s after a hit, so the contact
solver gets to settle the impact without the vehicle being shoved back into the wall.
Top speed is 46 units/s with an acceleration of 85–110, and `camera.fov` widens from 50° to
64° with speed to add a sense of velocity.

Steering is composed **directly onto the body's quaternion** around world Y rather than
being fed through cannon's angular-velocity integration:

```javascript
_yawQ.setFromAxisAngle(_YAXIS, yawRate * dt);
_yawQ.mult(b.quaternion, b.quaternion);
```

This means the steering input is honoured for exactly as long as the key is held, and cannot
be eaten by damping or by the contact solver.

### 2.3 Destruction system

Three distinct damage paths, driven by a `kind` tag on every destructible:

**Buildings** (`kind === 'building'`)
1. *Crack* — above 0 HP, the facade flashes emissive orange, dust puffs, chips spawn, and
   the whole group tilts slightly in proportion to accumulated damage.
2. *Topple* — at 0 HP the static body is removed and replaced with 2–4 horizontal rigid
   segments, each with mass proportional to its height, given a **gentle** initial velocity
   (max 4 units/s) toward the impact direction and a mild angular velocity. They fall under
   gravity and tumble.
3. *Shatter* — when a segment lands (low Y and low speed) or times out, it is replaced by
   10 smaller chunks with randomised sizes, velocities, spin, and damping.
4. *Fade* — debris chunks shrink over their last 1.2 s and are removed; the city visibly
   empties out.

**Small props** (`kind === 'tree' | 'lamp' | 'crate' | 'wall'`)
Squashed flat into a ground decal — a dark rim plate plus a thin body-coloured plate that
survives 6 seconds — with a dust puff and a few chips. They do **not** shatter.

**Cars** (`kind === 'car'`)
Take a direct hit above 13 units/s, lose HP, flash, then go through the older debris
shatter path.

### 2.4 Game-feel systems

- **Camera shake** is gated per-object with a `_shook` flag — every destructible shakes the
  camera exactly once, the first time it is struck. Wall scrapes are throttled to once per
  second. Debris contacts (`fragIds`) never shake at all.
- **Bullet time** (`triggerBullet()`) fires only on the first hit against a building.
  `timeScale` eases down to `0.25` for ~1.1 s and eases back out, driven by an exponential
  blend so the ramp is smooth in both directions. A 2.5 s cooldown prevents it from
  re-triggering on every consecutive building. A CSS overlay (`#bulletFx`) draws letterbox
  bars and speed lines.
- **Star burst** — a comic-style star sprite pops at every building hit, throttled to 350 ms
  per building. Sprites are pooled in `burstPool` and re-added to the scene on reuse.
- **Score & combo** — each demolition awards points; demolitions chained within 1.4 s stack
  a multiplier that is shown as a floating label.
- **Audio** — `engineUpdate()` pitch-shifts a saw oscillator with speed, `thump()` and
  `crashBig()` are filtered noise bursts, `clank()` is a short square-wave ping.

### 2.5 Art direction: the toon pipeline

The visual pass was a deliberate re-skin of the whole game:

- `MeshToonMaterial` with a 4-step `DataTexture` gradient map
  (`_toonSteps = [110, 170, 220, 255]`), `NoToneMapping`, and re-tuned hemisphere + sun
  lighting.
- **Inverted-hull outlines** — `addOutline(mesh)` clones the geometry with a
  `MeshBasicMaterial({ side: BackSide })` and scales it slightly along its normals. Applied
  to hero assets (bulldozer, trees, cars, lamps, crates, walls) to get ink-black edges.
- **Procedural canvas textures** — `brickTexture()`, `concreteTexture()`, `hazardTexture()`,
  `windowsTexture()`, `groundTexture()`, `dustTexture()`, `starTexture()`. Windows are drawn
  as a single texture-mapped plane per facade rather than one mesh per window, which cut
  per-building draw calls from over a hundred to single digits.
- The ground texture was edited to remove all zebra crossings (pedestrian crosswalks) at
  the user's request.
- **UI simplification** — the original HUD was reduced to score / buildings-left / speed,
  and the styling was rewritten to drop the generic "AI-looking" look in favour of a flat
  comic style with a restrained palette.
- All in-game text was translated from Chinese to English (`Flipped — press R to reset`,
  `CITY FLATTENED — press N to rebuild`, etc.).

### 2.6 Documentation

- `README.md` — concept, intended user, controls table, gameplay loop, three ways to run
  it, technical notes, and a known-limitations section.
- `docs/screenshot.png` — the project screenshot referenced by the README.
- `PROCESS.md` — this document.

---

## 3. User test records and how the feedback was answered

The user tested the build repeatedly and reported problems in roughly twenty rounds. Each
round is listed below in the order it happened.

| # | Reported by the user | Diagnosis | Fix |
| --- | --- | --- | --- |
| 1 | "Opened it in the browser and there is no scene and no bulldozer" | ES module + import map blocked over `file://` | Documented the local-server path and vendored copies of both libraries into `lib/` so an offline run is a one-line import-map change away |
| 2 | "It is stuck on the loading screen" | Slow CDN fetch blocking module evaluation | Preloaded libraries locally; added an explicit loading/error state |
| 3 | "Collision throws an error" (see screenshot) | Physics contact event handling dereferenced a missing body | Guarded the contact-event path and stopped surfacing physics errors as a UI banner (console only) |
| 4 | "The bulldozer is far too slow, and acceleration is too small — there is no sense of speed" | Top speed and accel were far too conservative; FOV never changed | Raised top speed to 46, accel to 85–110, added speed-driven FOV widening |
| 5 | "Camera shake on impact is too large and too long" | Shake amplitude was applied to camera position directly with no decay | Shorter, smaller decay (`camShake -= dt*8`), amplitude scaled per object kind |
| 6 | "Change the camera to a 45° top-down angle" | Camera was a trailing perspective cam | Fixed `CAM_OFF = (0, 34, 34)` — a true 45° isometric-style offset with damped target |
| 7 | "The same error UI is still there — just delete the error UI" | Errors were being rendered as an on-screen banner | Removed the banner entirely; errors now go to the console only |
| 8 | "Steering is very janky, please optimise it" | Steering was applied through angular velocity and fought the damping | Moved to direct quaternion yaw composition plus input smoothing |
| 9 | "Physics should feel realistic: a building should crack, then collapse, then break into pieces, then disappear" | Destruction was a single explosion of debris | Implemented the four-stage crack → topple → shatter → fade pipeline |
| 10 | "Trees, street lamps and crates should just be squashed flat" | Everything used the same shatter path | Added `squashSmall()` — flat decal, no rigid-body debris |
| 11 | "Too much UI, and the UI design looks very AI-generated" | Large, generic overlay | Cut the HUD down to three readouts and restyled it flat and comic-like |
| 12 | "Movement and steering still stutter for a moment" | Rendering was locked to physics steps; also speed was punished every frame on contact | Added render interpolation and a per-target 0.5 s hit-penalty cooldown instead of a per-frame multiplier |
| 13 | "Now it can't move at all" | The hit-penalty cooldown had been tuned too aggressively | Re-tuned the penalty to 8–18 % of speed, only once per 0.5 s per target |
| 14 | "The speed-blend coefficient is still too small, and acceleration is too small" | Blend factor and accel too low | Raised blend to `dt*30`, accel to 85–110 |
| 15 | "I don't think it's the engine — it feels like there is huge friction stopping the car" | The default `friction = .45` applied to the dozer's ground contact | Added dedicated contact materials with `matGround×matDozer` friction `.06` and restitution `0` |
| 16 | "Now it can't turn left or right" | A fix for the stutter had clamped steering to zero at low speed | Restored low-speed steering and added a grip curve (`grip = min(1, .45 + speed/18)`) |
| 17 | "When the vehicle hits a building it gets bounced into the air" | Top segment was launched with 4× acceleration and 20+ units/s of initial velocity, kicking the 110 kg vehicle like a ball | Reduced the fall-segment impulse to `min(4, 1.5 + impactV * .12)` |
| 18 | "There is a brief stall when hitting objects" | Contact was multiplying speed by `.55` **every frame** while pressed against a building | Replaced with a single 8–18 % penalty per 0.5 s per target |
| 19 | "I don't want the camera to shake on debris contact — only on the first hit" | Shake was triggered by any contact above a velocity threshold | Gated shake behind a per-object `_shook` flag; debris contacts explicitly excluded |
| 20 | "Same for other objects, not just buildings — but fragments of a destroyed object must not shake" | Same root cause | Applied the `_shook` gate to every destructible kind; fragments are tracked in `fragIds` and never shake |
| 21 | "Add bullet time, but only for the first hit on a **building**" | — | Implemented `triggerBullet()` on `d.kind === 'building'` only, with a 2.5 s cooldown and a CSS speed-line overlay |
| 22 | "The models are too simple — make them more refined and stylise the game" | Primitive-only geometry | Rebuilt the bulldozer, buildings, trees, cars, lamps and crates with multi-part procedural geometry and outlines |
| 23 | "Remove all the zebra crossings from the ground" (see screenshot) | Road texture included pedestrian crossings | Redrew the road markings in `groundTexture()` |
| 24 | "I want a star effect on every building, not just the first one" | Burst was tied to the first-hit flag | Moved the burst to a 350 ms-throttled check on every building hit |
| 25 | "The star effect disappeared after the first building" | Recycled sprites were removed from the scene but never re-added on reuse | Added `scene.add(b)` in `burst()` when a pooled sprite is retrieved; also moved the burst evaluation so it happens before the destructible is removed from the registry |

### 3.1 Overall verdict from the user

The final rounds of testing were positive: the steering issue was reported as fixed ("问题修复
了"), the stutter and bullet-time requests were confirmed working, and the user proceeded to
commission the README and this process document, which indicates the build was considered
feature-complete.

---

## 4. AI tools used and their specific purposes

This project was built with the assistance of an AI coding agent running inside **Cursor
IDE**. The following AI capabilities were used:

| AI capability | Concrete use in this project |
| --- | --- |
| **Code generation / implementation** | The whole of `index.html` — Three.js scene setup, the cannon-es world and contact materials, the vehicle dynamics, the three-path destruction system, the toon material factory, the procedural canvas texture library, the particle systems, the Web Audio synthesis, and the HUD. |
| **Debugging from error reports** | Reading the user's screenshots and console output to locate the physics-contact crash, the "stuck on loading" state, and the non-rendering scene. |
| **Iterative refactoring against live feedback** | Every round of the test log in §3 was implemented as a targeted edit and re-verified, rather than rewritten wholesale. |
| **Browser automation (`cursor-ide-browser` MCP)** | Serving the folder locally, opening it in a controlled browser tab, taking screenshots, and driving the game to reproduce the reported symptoms (blank scene, stutter, spin behaviour) before and after each fix. This is what made "the fix works" claims verifiable rather than assumed. |
| **Web / documentation lookup (`WebSearch`)** | Checking cannon-es contact-material and damping semantics, and confirming the `gradientMap` + inverted-hull technique for toon shading, rather than guessing at engine behaviour. |
| **Task tracking (`TodoWrite`)** | Breaking the multi-round art-direction and physics work into tracked items (toon pipeline, outline system, canvas textures, hero models, FX, performance regression check). |
| **Code navigation (`Grep` / `Read` / `git`)** | Locating the exact call sites of the control flow that had to change — e.g. every `shakeCam()` call, every `friction` assignment, the `processHits()` branch structure. |

No external 3D model, texture, or audio asset was generated by an image/audio AI, and no
image-generation model was used at any point. The art is entirely hand-written procedural
code, which was a deliberate constraint of the brief.

---

## 5. Suggestions: adopted, modified, rejected

### 5.1 Adopted — the "velocity blend" driving model

**Suggestion:** instead of writing the vehicle's velocity directly every frame, *blend* the
physics body's velocity toward a target velocity.

**Why it was adopted:** direct assignment made the vehicle feel like it was on rails — it
would ignore walls, keep full speed while scraping a building, and clip through obstacles.
Blending lets the contact solver subtract speed on impact, which produces the honest
deceleration the user asked for ("physics should feel real"), while the blend rate still
gives the driver enough authority to shove debris out of the way.

**How it landed in the code:** the blend coefficient `k` became the main tuning knob of the
project. It is deliberately lower for 0.12 s after a hit (`dt*12`) so the contact solver can
settle the collision, and higher otherwise (`dt*30`) so the vehicle does not get stuck behind
rubble.

### 5.2 Modified — the falling-building impulse

**Original suggestion:** when a building topples, drive its segments down hard — a 4×
acceleration multiplier on the impact velocity with initial speeds above 20 units/s — so the
collapse looks violent and the debris scatters widely.

**What it was modified to:** the impulse was reduced to
`push = min(4, 1.5 + impactV * .12)`, with a small angular velocity and gravity doing the
rest. The collapse still reads as a collapse, but the debris no longer behaves like an
explosion.

**Why it was modified:** the user reported that hitting a building launched the bulldozer into
the air. The diagnosis was that a 110 kg vehicle was being kicked by an arbitrarily fast
rigid body — the "violence" of the effect was coming at the cost of the vehicle feeling
real. Making the collapse *heavier and slower* actually reads better, because it looks like
mass rather than like an explosion.

### 5.3 Rejected — the wrecking ball

**Rejected suggestion:** a wrecking ball on a swinging chain as the primary demolition
mechanic.

**Why it was rejected:** it was decided against *before any code was written*, on the grounds
that it replaces agency with timing. The player would spend most of the session lining the
ball up rather than acting, and the only interesting decision — *when* to swing — has a
correct answer that is obvious once found. A bulldozer gives the player a 2D continuous
control space (approach angle × approach speed), which supports far more varied play and
matches the "press a key and something explodes" onboarding that the project needs.

**Consequence:** the shipped build has no chain, no pendulum joint, and no swinging mechanic.
The only remnants of the original idea are the *mass* and *weight* of the bulldozer itself,
which carry the same sensation of heavy slow-moving destruction.

---

## 6. Known limitations

- **First load needs a network** — the import map points at unpkg, so the CDN must be
  reachable. Vendored copies exist in `lib/` but are not wired into the import map yet.
  `file://` module restrictions also apply in some browsers, so a local server is
  recommended.
- **Destructible counts are bounded** — debris is capped (320 chunks) and falls asleep, so a
  very long session slowly runs out of things to destroy until `N` generates a new city.
- **Shadows** are single-cascade and follow the camera target; they are adequate for the
  toon look but not for large open areas.
- **Mobile** gets touch controls, but the vehicle feel is tuned for keyboard input.
- **No persistence** — the city is regenerated on every page load.
