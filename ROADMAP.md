# Dripfolk -- Development Roadmap

## Vision

A multiplayer creature simulation where diverse, personality-driven creatures
interact, learn, evolve, and form communities. Creatures are generated from user
input, develop their own locomotion through reinforcement learning, and learn
from each other through interaction.

---

## Phase 1: Parametric Creature Generation Library

**Goal:** Replace hardcoded pixel-traced shapes with a parametric system that
generates visually diverse creatures from a small set of input parameters.

**What exists today:**
- One shape (`shadow-blob`) with 148 hand-traced coordinate pairs
- Fixed leg positions (4 pairs), fixed eyes (2), fixed body envelope
- All creatures look identical

**What we're building:**
- A `CreatureGenerator` that takes a config/seed and outputs `{ body, eyes, legs }`
- Body envelope defined by control parameters (width, height, taper, roundness, segments)
- Variable leg count, placement, and proportions derived from body plan
- Variable eye count, size, and placement
- Output format compatible with existing `CreatureRenderer` pipeline
- User-facing API: provide traits/preferences, get a unique creature

**Key constraint:** Output must match the existing shape format consumed by
`ShapeDeformer` and `CreatureRenderer` -- no renderer changes needed initially.

---

## Phase 2: Self-Reinforcement Learning for Locomotion

**Goal:** Creatures develop their own walking, movement, and interaction styles
based on their unique morphology.

**Depends on:** Phase 1 (diverse body plans)

**Approach:**
- Locomotion defined by evolvable gait parameters (stride length, phase offsets,
  undulation amplitude, duty cycle, leg coordination)
- Simple physics simulation for fitness evaluation
- Fitness function: distance covered, energy efficiency, stability
- Different morphologies naturally produce different locomotion strategies
- A 6-legged creature shouldn't walk like a 4-legged one

**Open questions:**
- Where does RL run? Server-side during simulation? Offline pre-computation?
- How complex should the physics model be?
- What constitutes "energy" for a creature?

---

## Phase 3: Inter-Creature Learning

**Goal:** Creatures learn from each other's evolved behaviors through interaction.

**Depends on:** Phase 2 (learnable parameters)

**Approach:**
- Creatures share/blend locomotion parameters through proximity + interaction
- Imitation learning between creatures with different body plans
- Knowledge transfer as a social mechanic (teaching, mimicry)
- Basis for community formation and breeding

**Open questions:**
- What does "learning from each other" look like visually?
- How does this connect to LLM-driven personality/conversation?
- What's the breeding mechanic -- parameter crossover? Morphology blending?

---

## Current Architecture

```
dripfolk/
  client/       -- Canvas2D browser client (Vite + colyseus.js)
  server/       -- Colyseus game server (20Hz tick, PostgreSQL, Redis)
  shared/       -- Shared types, shapes, constants
  docs/         -- Architecture spec, roadmap, reference images
  research/     -- Prototypes, experiments, reference materials
```

See `ARCHITECTURE.md` for the full technical specification.
