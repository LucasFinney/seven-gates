# Seven Gates

Notes for Claude (and any other collaborator) picking up this project. Read this before changing anything.

## What this is

An **interactive diagram dressed as a survival-horror game**. It teaches the Seven Bridges of Königsberg, and through it the basics of graph theory (Euler paths and vertex degree). It is not a full game. The horror is the delivery mechanism for the math.

The author is Lucas Finney: a mathematician (M.Sc. Applied Math), physics tutor and educational content creator. This is a portfolio piece for interactive STEM education. The tone he wants is "math as a puzzle you feel before you're told the rule."

### Why Königsberg

When Lucas tutored, Königsberg reliably hooked students. It's a grounded, physical problem that needed a whole new branch of math to resolve. The *Resident Evil 2* remake has a sewer puzzle of one-way gates that comes close to the same idea. Seven Gates leans into that.

### The arc

1. A series of escalating, **solvable** levels. Each is a graph: rooms are vertices and bridges are edges.
2. A final level with the **Königsberg layout** (4 rooms, 7 bridges, every room of odd degree). It is **deliberately unsolvable**.
3. At the final level the pursuer kills the player, and the "YOU DIED" screen offers exactly one option: **"Why?"** That button opens the explainer.
4. **The explainer.** The camera pulls up to a top-down view. Rooms morph into dots and bridges into lines, and the player's own failed route replays on the graph. Each room's degree is shown and the odd rooms light up. A short argument follows: a walk passing through a room uses its bridges in pairs, so an odd room leaves you stranded. Finally it flips back to the earlier levels to show why they worked.

Optional closing hook: Königsberg is now Kaliningrad, and the bridges changed after WWII. With the modern layout an open walk is reportedly possible but a closed loop isn't. **Verify this before using it.**

## Core design rules (don't break these)

- **The graph is the single source of truth.** Each level is plain data (`LEVEL`: rooms with positions and cameras, bridges as room pairs). Geometry, cameras, the HUD and the explainer are all generated from it, so the explainer always matches what the player walked.
- **One valve per bridge.** Each bridge has an iron gate at its midpoint. Holding E at the valve raises the gate. Passing through crumbles the half behind you, and reaching the far room crumbles the rest and counts the valve. So **every bridge is crossed exactly once**: the Euler path constraint, made physical.
- **Death is always the graph's fault, never the player's reflexes.** You die only when **stranded**: you're in a room with no standing bridges and valves are still left. The pursuer follows this rule too (built in step 2). Contact while you still have moves costs a few seconds, a stagger or knockback. Contact while stranded kills you. Failure is cheap (Retry resets in about two seconds). On the final level, "Why?" has to feel like an honest question, not "I was too slow."
- **The pursuer and the gun are decoration and pressure, not combat.** The pursuer wades through the water, so collapsed bridges don't stop it. It can't be killed. The shotgun only staggers and slows it.
- **Fixed cameras are pedagogical.** They deny the player an overview, so the graph is experienced locally, room by room. That's exactly how it feels to be stuck without Euler's insight. The explainer's pull-up to top-down **is** the abstraction step. Don't add a free camera or an in-game map. The `M` developer map exists for testing only.
- **Desktop and keyboard only for now.** Mobile is explicitly out of scope.

### Directed graphs (planned)

One-way gates, the RE2 nod, are a later mechanic. They teach the directed version of the theorem: in-degree must equal out-degree everywhere except possibly the start and end.

## Aesthetic

- **PS1 rendering.** The scene renders at 320×240 and is upscaled with nearest-neighbour filtering. Vertices snap to a grid in the vertex shader (the PS1 wobble). Lighting is per-vertex Lambert (Gouraud). Textures are tiny procedurally drawn canvases with no filtering, and a post-process pass adds Bayer dithering, colour quantization and a vignette. Textures are mapped affinely (no perspective correction), and water and pursuer animation step at 12 fps. Fog is heavy. Janky low-poly procedural models are intentional, so don't "improve" them toward realism.
- **Gothic, Bloodborne-ish, not police-station RE.** Stone rooms, narrow stone bridges over black water, iron portcullis gates, sconces and lanterns, a distant colonnade.
- **The player character** is a hunter in a long flowing coat with a tricorn hat and a scarf over the mouth. He carries a **double-barrel shotgun** (the anachronism is fine) in a **low two-handed carry**. The stock rides at the right side of the chest and the muzzle points down-left, past the front of the left knee. Arms are two-bone IK chains solved every frame so both hands stay on the gun. Walking adds torso counter-twist, gun sway and knee bend, and running raises the gun slightly. This pose took several iterations to get right, so be careful changing it.
- **Typography** uses IM Fell English SC (display) and IM Fell English (body). The palette is `--void`, `--bone`, `--ash`, `--blood`, `--ember` and `--iron`.
- **Audio** is all procedural Web Audio: brown-noise ambience, drips, valve creaks, gate clanks, crumbling stone.

## Controls

Tank controls. W/↑ walks forward, S/↓ steps back, A/D or ←/→ turn, Shift runs, Q quick-turns 180°, and E or Space interacts (hold E to turn a valve). F fires the shotgun, R reloads, and M toggles the developer map. Opening the page with `#debug` exposes `window.SG` (`S`, `level`, `collapseHalf`, `fire`, `openGate` and `step(n)`) for testing. `SG.step(n)` advances n fixed frames and draws once; use it for screenshots because the loop doesn't run while the browser pane is hidden.

## Current structure

Everything lives in **`index.html`**, a single self-contained page: CSS, HTML overlays, and one inline script. Three.js **r128** loads from cdnjs. r128 is pinned on purpose: the PS1 shader hook and per-vertex Lambert depend on it.

`index.html` deliberately has **no `<!doctype>`, `<html>`, `<head>` or `<body>` tags**. It is also published as a claude.ai artifact, and the publisher wraps the content in its own skeleton. Browsers open it fine locally as it is.

Script sections, in file order (search for the `/* ---------- ... ---------- */` banners):

| Section | What it does |
|---|---|
| Level data | `LEVEL`: rooms (position, name, camera) and bridges (`[roomA, roomB, pointTheSideCameraAvoids]`). Currently a "bowtie": two triangles sharing the centre room, 5 rooms and 6 bridges, all of even degree. |
| Seeded random | Deterministic RNG, so procedural textures and props are stable between loads. |
| PS1 materials | `psx()` injects vertex snapping into any material. Procedural canvas textures. Material palette `M`. |
| Renderer | 320×240 render target, then a post shader (dither, quantize, vignette, fade). Fog, lights, and 4:3 letterbox fitting. |
| Static world | Water plane (12 fps stepped swell) and the distant colonnade ring. |
| Player | `buildHunter()` builds the rig. `solveArm()` is two-bone IK. `poseHunter()` animates gait, gun carry and death collapse. |
| Level construction | `buildLevel()` turns `LEVEL` into rooms (floor, pillars, sconce, slime band, drain hatch, escape beam) and bridges (segmented halves, ruined piers, gate with portcullis, chains, valve wheel and lantern, side camera). `box()` shares geometry by size and `buildLevel()` disposes the rest, so retries don't leak. Light sources are recorded as data via `addSrc()` into `level.lights`. |
| Lights | `updateLights()` moves a fixed pool of 6 `PointLight`s (`LP`) onto the nearest sources each frame, with fades when a light changes source. A constant light count means no shader recompiles. |
| Geometry queries | `walkable()` handles collision against rooms, standing bridge halves and closed gates. `locate()` reports which room or bridge a point is on. |
| Pursuer | `buildPursuer()`, `resetPursuer()` and `updatePursuer()`: a tall drowned figure wading in the water, with states dormant, hunt and stagger, plus the contact rule (knockback, or death if stranded). |
| Audio | Procedural Web Audio helpers: ambience, drips, valve and gate sounds, cistern reverb, surface-aware footsteps, and the pursuer proximity drone and heartbeat (`audio.threat(v)`). |
| State | Global `S` (player position, heading, mode, falling debris, consumed count, stranded flag, clip and reserve ammo, route log), HUD helpers and `reset()`. |
| Particles | `FX`: pooled dust, chips, spray, smoke and water ripples. Hooked into falling debris, gates, the shotgun and pursuer footsteps. |
| Mechanics | `collapseHalf`, `consume` (also drops the gate and lantern into the water), `openGate`, `fire`, `startReload`, `setStranded`, `die` and `win`. |
| Input | Key handling. Mode flow is title, then play, then dying or won. |
| Update | The per-frame loop: scenery animation, valve hold, tank movement, crossing and collapse logic, the stranded check, the escape hatch, prompts, camera zone switching and pose. |
| Developer map | Top-down debug view with live remaining degree per room. |
| Loop | Renders to target, post-processes, and calls `requestAnimationFrame`. |

The win condition is all valves turned. The room you're standing in then opens a drain hatch with a light beam, and pressing E there wins.

## Build plan

1. ~~Core loop: one solvable level, tank controls, fixed cameras, valves, collapsing bridges, stranded detection.~~ **Done.**
2. ~~Pursuer and shotgun.~~ **Done** (PR #1 and PR #2). Decisions and why:
   - Stranding no longer kills on arrival. It sets `S.stranded`, and the pursuer reaching you kills. Death stays the graph's fault (you're stranded), the pursuer just delivers it. Stranding wakes the pursuer and speeds it up (3.0 vs 1.6) so waiting never becomes a long, boring timeout.
   - Contact with moves left: knockback in 0.1-unit steps that stop at the first non-`walkable()` spot, 1.5s stagger, 2s grace so it can't chain-stun. Never lethal, so it costs time and position, not a life.
   - **The pursuer can't hit you while you hold a valve.** Playtest found a single knockback jump teleporting the player through a closed gate; stepped knockback fixes the cause, and the valve exemption removes the frustration of being shoved mid-turn. Gameplay here is aesthetic, not a skill test.
   - Shotgun fires along your facing (no aim mode, to fit tank controls and fixed cameras). The clip holds 2 shells, reload takes 2.5s (auto when empty), and the level has a **limited total of shells** (`LEVEL.shells`, 8 for the bowtie). Why: a stranded player could otherwise stagger the pursuer forever and never reach death, so the stall is bounded instead of banning the shotgun. No ammo drops yet. A hit within 12 units and about ±18° staggers the pursuer 3s, then slows it for 6s.
   - `LEVEL.pursuer = {spawn, speed, strandedSpeed, contact, wakeDelay}` keeps the pursuer data-driven, with defaults in `PUR_DEFAULT` for omitted fields. It is read on every `reset()`, so each level can change it. A level with no `pursuer` has none, and stranding then kills directly. It wakes after `wakeDelay`, when you come within 14, or when you're stranded.
   - `S.route` logs rooms entered and bridges crossed (`{bridge, from, to}`) for the step 5 replay. It isn't displayed yet.
   - Knockback on a bridge pushes along the bridge axis (the rails block a sideways shove and the player would otherwise be pinned). Tunable numbers: pursuer speeds, contact radius (`contact`), stagger/grace times, shot cone and range, reload time.
   - Known placeholders (fixed in step 3): crude pursuer model and base clipping.
3. ~~Further PS1 polish.~~ **Done.** Lucas playtested it: sound and effects are good; the floor sigils were rejected and removed. Shipped in PR #3, with review fixes in PR #4. Decisions and why:
   - **Affine texture mapping** (`AFFINE` flag in `psx()`): uv*w and w varyings divided in the fragment shader, because GLSL ES 1.0 has no `noperspective`. Set the flag false if r128 chunks ever break it.
   - **The pursuer climbs onto rooms.** Over a room footprint its Y lerps to the floor; over open water it wades at `WATER_Y`. Contact stays planar distance, so the death rule is untouched. Rooms are solid, bridges are not.
   - Pursuer animation runs on a stepped 12 fps clock (PS1 keyframes); the hunter does not, because his feel matters.
   - **Floor sigils were tried and removed.** A dot-count inlay per room read as unclear, warped in and out of the floor during screenshake and was cut at some camera angles. Room identity is deferred to the decoration plan below.
   - **Lights are data, not objects.** `addSrc()` records sources in `level.lights`; `updateLights()` moves a fixed pool of 6 `PointLight`s (`LP`) onto the nearest ones with fades. Constant light count means no shader recompiles. Room sconces and bridge lanterns animate `src.intensity`, not a light.
   - Audio: generated-impulse reverb, stone vs bridge footsteps, a proximity drone and heartbeat from `audio.threat(v)`, chain clinks on gate open.
   - Tunables: pool size 6, reverb wet .33, threat range 26, water swell amplitude .1.
   - Review-fix decisions: pooled lights ramp in or out (`p.k`) only when they change source, and settled lights follow `src.intensity` directly so sconce flicker survives. Lantern light y follows its gate as it falls. `box()` shares geometry by dimensions (flagged `shared`) and `buildLevel()` disposes the rest, so retries don't leak. The second colonnade ring and moon disc were dropped: they sat beyond the fog's far plane (42) and above what the downward-pitched cameras can see. The particle pool is 192.
   - Known limitation: the pursuer ignores the bridge piers (and gate bases) and can wade through them. It is a placeholder (Lucas's call), so it isn't steered yet; revisit with the pursuer redesign.
   - Not verified: affine warping on other GPUs and performance on a weak machine (only checked in the in-app browser). The pursuer's head leaves the top of frame when it stands on a room next to the camera (accepted for now).
4. Level sequence: 2–3 undirected levels that teach the parity rule. One beat must be "only works if you start in the right room" (exactly two odd vertices). Then a one-way-gate level, then Königsberg (7 bridges, 4 odd rooms).
5. The death screen's "Why?" and the explainer.

### Backlog: low priority, decorative

None of these block Step 4 or 5. Pick them up only when Lucas asks, or once the level sequence and explainer are in.

- **Room decoration that fits each room's name, possibly with enclosed rooms.** Rooms are bare platforms, and names like THE OSSUARY and THE DROWNED CHAPEL (current names: VESTRY, OSSUARY, CISTERN HEART, WEIR, DROWNED CHAPEL, in `LEVEL.rooms[k].name`) promise more than a barren platform delivers. Lucas wants the dressing to match the name, and is open to rooms becoming enclosed spaces (walls, arches, a roof) rather than open slabs. Constraints to design around:
  - Fixed cameras: enclosing walls must not hide the player or the graph. Expect cut-away or low walls on the camera side, or per-zone visibility.
  - Keep `walkable()` and `locate()` working. They treat a room as a square footprint of half-width `H`, and bridges attach at its edges.
  - Author the dressing as data on the room (a per-name prop set), so geometry still comes from `LEVEL`.
  - Anything added must not z-fight, warp or clip under camera shake. The floor sigil inlay failed on all three, and also read as meaningless to the player.
  - Mind the light budget: sconces and lanterns are data sources feeding the pool of 6 `PointLight`s, so extra lit props should be emissive meshes, not new lights.
- **Injury effect: a regenerating-health illusion.** Pursuer contact (the knockback and stagger in `updatePursuer`) should look like taking damage: a red vignette that deepens as "health" drops, plus optional desaturation, heavier breathing or a limp, recovering over a few seconds. The real rule does not change: the player cannot die from damage and can only die by being stranded. So it must never read as a bar that can be emptied except by stranding. Repeated hits re-trigger or deepen the effect, capped below "dead". A stranded death can show it filling fully as the pursuer lands the killing blow. Likely hooks: the post shader (a `hurt` uniform next to `fade`, in the vignette term), `S.stagger` and `S.grace`, and `audio`.
- **Pursuer redesign.** The model and behaviour are still a placeholder (Lucas's own description). Known limitations to fix then: it wades through bridge piers and gate bases, and its head leaves the top of frame when it stands on a room.

As the file grows, splitting it into modules (level data, renderer, player, enemy, explainer) is reasonable. Keep it buildable as a single self-contained page, since it's shared as a published artifact.

## Development and testing

- **Serve it, don't just double-click it.** `.claude/launch.json` defines `seven-gates` (`python -m http.server 8123`). Open `http://localhost:8123/index.html#debug`. Add a `?v=N` query when reloading to dodge the cache.
- **Encoding.** `index.html` starts with `<meta charset="utf-8">`. Simple servers send no charset, and a literal non-ASCII character (the multiplication sign in the reserve counter) rendered as a garbled `A` with a tilde. Keep non-ASCII out of JS source: use \u00d7-style escapes, and HTML entities in markup. The Edit tool may silently normalise an escape back to the literal character, so check with `grep` or a small Python script.
- **Headless-style testing.** The browser pane does not run `requestAnimationFrame` while hidden. In `#debug`, drive the game with `SG.step(n)` (n fixed 1/60 s frames, one draw), and poke state through `SG.S`, `SG.level`, `SG.fire()`, `SG.openGate(b)` and `SG.collapseHalf(b,h,fromT)`. A useful recipe: click `#begin`, teleport the player by setting `SG.S.x` and `z`, set the pursuer to `state='hunt'` to wake it, `SG.step(60)`, then screenshot. Always check the console for shader compile errors after touching `psx()`.
- **Pitfall: branch from the latest `main`.** Step 3's branch was cut before PR #2 merged, so it silently lacked the limited-ammo feature until `main` was merged in. At the start of a session, `git fetch` and confirm the branch contains `origin/main`.
- **PRs.** Work lands through GitHub PRs (`LucasFinney/seven-gates`). Review fixes may be stacked on the step's PR; merge the base PR first.

## Session workflow

This file is the project's memory. Chat history is not, so anything decided in a session must land here before the session ends.

- **One session per build-plan step** (or one coherent chunk of it). Keep fine-tuning of animation and feel inside a single session. Start fresh when a step is finished or the topic changes. A new session reads this file and picks up.
- **Plan first for big steps** (pursuer, level sequence, explainer). Present the plan and check it against the core design rules before writing code.
- **Record decisions with their reason.** When a design decision is made in chat (a number, a rule, a behaviour), write it here with the why. Keep "Core design rules" and the build plan current. Tick off finished steps and note anything half-done or deferred.
- **Commit per working increment**, with a message saying what and why. Use a branch or worktree per step so failed experiments are cheap to discard.
- **Levels stay pure data.** Anything that bypasses `LEVEL` (hard-coded geometry, a separate route record for the explainer) is a red flag, because the explainer must match what the player walked.
- **End each session with a wrap-up:** what changed, what is not verified, what's next, and any decisions still to record here.
- **Review before merging** a larger step (`/code-review`), judged against the core design rules.
- Separable work, such as authoring level data for several levels, can run in its own session since it doesn't need the full history.

## Working with Lucas

- He tests on desktop and gives visual feedback with screenshots and reference images. Check visuals yourself where you can, such as screenshotting a local browser, before asking him to look.
- Animation and character feel matter a lot to him. Small, specific iterations work well.
- He prefers decorative extras to be parked in the backlog above, not built opportunistically. When he calls something a placeholder (the pursuer), don't invest in it.
- Graph theory terminology: he's a mathematician, so be precise (degree, Euler path vs circuit, directed vs undirected).
