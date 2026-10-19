# Seven Gates

A PS1-style survival-horror take on the **Seven Bridges of Königsberg**: an interactive graph theory explainer.

You're a hunter in a flooded Gothic cistern. Every bridge is barred by an iron gate. Turn its valve to pass, and the bridge crumbles behind you. Raise every gate to escape. The levels escalate until the last one, which can't be solved, and its death screen asks **"Why?"**

## Play

Open `index.html` in a desktop browser. A keyboard is required.

| Key | Action |
|---|---|
| W / ↑ | Walk forward |
| S / ↓ | Step back |
| A D / ← → | Turn |
| Shift | Run |
| Q | Quick turn |
| E / Space | Use (hold to turn a valve) |
| F | Fire the shotgun |
| R | Reload |
| M | Developer map |

## Status

Early prototype. The core loop works: tank controls, fixed cameras, valves, collapsing bridges and stranded detection. A pursuer wades through the water and a shotgun can stagger it. The pursuer kills only when you are stranded. The full level sequence and the explainer are still to come.

Built with [Three.js](https://threejs.org/) r128. No build step.
