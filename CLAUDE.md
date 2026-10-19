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

- **PS1 rendering.** The scene renders at 320×240 and is upscaled with nearest-neighbour filtering. Vertices snap to a grid in the vertex shader (the PS1 wobble). Lighting is per-vertex Lambert (Gouraud). Textures are tiny procedurally drawn canvases with no filtering, and a post-process pass adds Bayer dithering, colour quantization and a vignette. Fog is heavy. Janky low-poly procedural models are intentional, so don't "improve" them toward realism.
- **Gothic, Bloodborne-ish, not police-station RE.** Stone rooms, narrow stone bridges over black water, iron portcullis gates, sconces and lanterns, a distant colonnade.
- **The player character** is a hunter in a long flowing coat with a tricorn hat and a scarf over the mouth. He carries a **double-barrel shotgun** (the anachronism is fine) in a **low two-handed carry**. The stock rides at the right side of the chest and the muzzle points down-left, past the front of the left knee. Arms are two-bone IK chains solved every frame so both hands stay on the gun. Walking adds torso counter-twist, gun sway and knee bend, and running raises the gun slightly. This pose took several iterations to get right, so be careful changing it.
- **Typography** uses IM Fell English SC (display) and IM Fell English (body). The palette is `--void`, `--bone`, `--ash`, `--blood`, `--ember` and `--iron`.
- **Audio** is all procedural Web Audio: brown-noise ambience, drips, valve creaks, gate clanks, crumbling stone.

## Controls

Tank controls. W/↑ walks forward, S/↓ steps back, A/D or ←/→ turn, Shift runs, Q quick-turns 180°, and E or Space interacts (hold E to turn a valve). F fires the shotgun, R reloads, and M toggles the developer map. Opening the page with `#debug` exposes `window.SG` (`S` and `level`) for testing.

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
| Static world | Water plane and distant colonnade. |
| Player | `buildHunter()` builds the rig. `solveArm()` is two-bone IK. `poseHunter()` animates gait, gun carry and death collapse. |
| Level construction | `buildLevel()` turns `LEVEL` into rooms (floor, pillars, sconce, drain hatch, escape beam) and bridges (segmented halves, gate with portcullis, valve wheel and lantern, side camera). |
| Geometry queries | `walkable()` handles collision against rooms, standing bridge halves and closed gates. `locate()` reports which room or bridge a point is on. |
| Pursuer | `buildPursuer()`, `resetPursuer()` and `updatePursuer()`: a tall drowned figure wading in the water, with states dormant, hunt and stagger, plus the contact rule (knockback, or death if stranded). |
| Audio | Procedural Web Audio helpers. |
| State | Global `S` (player position, heading, mode, falling debris, consumed count, stranded flag, ammo, route log), HUD helpers and `reset()`. |
| Mechanics | `collapseHalf`, `consume` (also drops the gate and lantern into the water), `openGate`, `fire`, `startReload`, `setStranded`, `die` and `win`. |
| Input | Key handling. Mode flow is title, then play, then dying or won. |
| Update | The per-frame loop: scenery animation, valve hold, tank movement, crossing and collapse logic, the stranded check, the escape hatch, prompts, camera zone switching and pose. |
| Developer map | Top-down debug view with live remaining degree per room. |
| Loop | Renders to target, post-processes, and calls `requestAnimationFrame`. |

The win condition is all valves turned. The room you're standing in then opens a drain hatch with a light beam, and pressing E there wins.

## Build plan

1. ~~Core loop: one solvable level, tank controls, fixed cameras, valves, collapsing bridges, stranded detection.~~ **Done.**
2. ~~Pursuer and shotgun.~~ **Built, awaiting Lucas's playtest.** Decisions and why:
   - Stranding no longer kills on arrival. It sets `S.stranded`, and the pursuer reaching you kills. Death stays the graph's fault (you're stranded), the pursuer just delivers it. Stranding wakes the pursuer and speeds it up (3.0 vs 1.6) so waiting never becomes a long, boring timeout.
   - Contact with moves left: knockback in 0.1-unit steps that stop at the first non-`walkable()` spot, 1.5s stagger, 2s grace so it can't chain-stun. Never lethal, so it costs time and position, not a life.
   - **The pursuer can't hit you while you hold a valve.** Playtest found a single knockback jump teleporting the player through a closed gate; stepped knockback fixes the cause, and the valve exemption removes the frustration of being shoved mid-turn. Gameplay here is aesthetic, not a skill test.
   - Shotgun fires along your facing (no aim mode, to fit tank controls and fixed cameras). The clip holds 2 shells, reload takes 2.5s (auto when empty), and the level has a **limited total of shells** (`LEVEL.shells`, 8 for the bowtie). Why: a stranded player could otherwise stagger the pursuer forever and never reach death, so the stall is bounded instead of banning the shotgun. No ammo drops yet. A hit within 12 units and about ±18° staggers the pursuer 3s, then slows it for 6s.
   - `LEVEL.pursuer = {spawn, speed, strandedSpeed, contact, wakeDelay}` keeps the pursuer data-driven, with defaults in `PUR_DEFAULT` for omitted fields. It is read on every `reset()`, so each level can change it. A level with no `pursuer` has none, and stranding then kills directly. It wakes after `wakeDelay`, when you come within 14, or when you're stranded.
   - `S.route` logs rooms entered and bridges crossed (`{bridge, from, to}`) for the step 5 replay. It isn't displayed yet.
   - Knockback on a bridge pushes along the bridge axis (the rails block a sideways shove and the player would otherwise be pinned). Tunable numbers: pursuer speeds, contact radius (`contact`), stagger/grace times, shot cone and range, reload time.
   - Known placeholders: the pursuer model is crude and clips through room bases when it reaches you (it ignores geometry). Step 3 polish.
3. Further PS1 polish.
4. Level sequence: 2–3 undirected levels that teach the parity rule. One beat must be "only works if you start in the right room" (exactly two odd vertices). Then a one-way-gate level, then Königsberg (7 bridges, 4 odd rooms).
5. The death screen's "Why?" and the explainer.

As the file grows, splitting it into modules (level data, renderer, player, enemy, explainer) is reasonable. Keep it buildable as a single self-contained page, since it's shared as a published artifact.

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
- Graph theory terminology: he's a mathematician, so be precise (degree, Euler path vs circuit, directed vs undirected).
