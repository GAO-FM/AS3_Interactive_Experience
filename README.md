# Bulldozer Demolition

A stylized 3D demolition sandbox for the browser. You drive a bulldozer around a small
cartoon city and flatten it — every building cracks, tilts, topples, and shatters into
debris that settles into rubble and fades away.

## The idea

The brief for this assignment asked for an *interactive experience* that could not be
carried by clicking through text-based menus. So instead of dialogue, the interaction is
**destructive play**: the only verb in the game is *ram*, and the whole city is the
feedback surface. Every hit is answered with cracks, dust, debris, screen shake, and a
comic-style star burst.

The visual direction is a **toon / cel-shaded diorama** — flat color bands, hard shadow
edges, ink-black outlines around hero objects, and hand-drawn-feeling canvas textures for
brick, concrete, and windows. Everything is generated procedurally in code, so the whole
experience ships as a single HTML file with no external art assets.

The camera sits at a fixed **45-degree top-down** angle and follows the vehicle, so the
play area always reads as a small tabletop model you are pushing around.

## Intended user

Anyone who can open a browser tab — no install, no account, no build step. It suits:

- **Players / casual users** who want a satisfying "break things" toy with no rules to learn.
- **Judges and reviewers** of the assignment, who need to grasp the concept in seconds.
- **Design and 3D students** who want to read a compact reference implementation of
  rigid-body demolition, toon shading, and procedural asset generation.

Controls are deliberately minimal (WASD + Space) so the first-time experience is
"press a key and something explodes."

## Main interaction

| Input     | Action                                 |
| --------- | -------------------------------------- |
| `W` / `↑` | Throttle forward                       |
| `S` / `↓` | Reverse                                |
| `A` / `←` | Steer left                             |
| `D` / `→` | Steer right                            |
| `Space`   | Brake                                  |
| `R`       | Right the bulldozer (it can flip over) |
| `N`       | Generate a new city                    |
| `M`       | Mute / unmute sound                    |

Touch controls appear automatically on coarse-pointer devices (phones and tablets).

### What happens when you hit something

Destruction is staged, so a hit reads as a short sequence rather than one explosion:

1. **Crack** — the surface flashes, dust puffs, and small chips fall off. The structure
   visibly tilts as it accumulates damage.
2. **Topple** — the building splits into 2–4 horizontal rigid segments that tumble
   forward under gravity.
3. **Shatter** — on landing (or after a timeout) each segment bursts into smaller debris
   chunks with finite lifetimes.
4. **Fade** — debris shrinks and disappears, leaving the city progressively emptier.

Small props behave differently on purpose: **trees, street lamps, crates, and brick walls
are squashed flat** into ground decals instead of shattering.

### Game feel details

- **First-hit-only screen shake.** The camera shakes once when you first strike any
  intact object. Scraping debris, re-hitting the same wreck, or grinding through rubble
  never shakes the camera.
- **Bullet time on the first building hit.** The first time you hit a building, the whole
  simulation drops to 0.25x speed for about 1.1 seconds, with letterbox bars and comic
  speed lines. Repeated hits on the same building do not re-trigger it, and a 2.5-second
  cooldown prevents it from firing on every consecutive building.
- **Speed matters.** The faster you arrive, the more damage you do.
- **Score and combo.** Destructions award points, and chaining demolitions within a short
  window adds a multiplier.
- **Star bursts.** Every building hit pops a comic-style star, not just the first one.
- **Audio.** Procedurally generated with the Web Audio API — engine hum pitched to speed,
  impact thumps, metallic clanks, and crash noise. No audio files are used.

## How to run it

The project is a single self-contained `index.html`. Three.js and cannon-es load from a
CDN via an import map, so **you need an internet connection on first load**.

### Option A — just open it

Open `index.html` in a modern browser. Because the page uses ES modules and an import
map, some browsers block module loading over the `file://` protocol; if you see a blank
screen, use Option B.

### Option B — local server (recommended)

From the project folder:

```bash
python -m http.server 8000
```

Then visit <http://localhost:8000>.

Any static server works equally well (`npx serve`, `php -S localhost:8000`, VS Code's
Live Server extension, and so on).

### Option C — GitHub Pages

This is a static site with no build step, so it deploys cleanly. If a Pages site is
enabled for this repository, the live URL follows the pattern:

```
https://gao-fm.github.io/AS3_Interactive_Experience/
```

> **Note:** the live link above is only valid once GitHub Pages has been enabled for the
> repo (Settings → Pages → Deploy from a branch → `main` / root). At the time of
> writing this was **not yet enabled**, so the URL currently returns a 404. To publish it,
> enable Pages on the `main` branch, then the URL above will work as-is with no code
> changes.

## Technical notes

- **Renderer:** Three.js r160 (`MeshToonMaterial` with a 4-step `gradientMap`),
  cel-shaded lighting, inverted-hull outlines for hero assets.
- **Physics:** cannon-es 0.20 — rigid-body demolition with custom contact materials.
- **Structure:** one `index.html`, ~1000 lines. Procedural geometry and canvas textures
  only; no external models, images, or audio.
- **Performance:** the city renders in roughly 300 draw calls / ~7k triangles. Building
  windows are drawn as a single texture-mapped plane per facade rather than per-window
  meshes, which cut per-building draw calls from over a hundred to single digits.
