---
description: "Scaffold a new Ayeeeen feature with audio contract, component, state machine, and ML integration. Use when starting any new feature."
---

# New Ayeeeen Feature

Use the `ayeeeen-dev` skill to build this feature end-to-end.

## Feature: {{ feature_name }}
## Trigger Gesture: {{ gesture_trigger }}

## Instructions

1. Follow the full 8-step procedure from the `ayeeeen-dev` skill.
2. Start by defining the audio contract for **{{ feature_name }}** using **{{ gesture_trigger }}** as the trigger gesture.
3. Create the component scaffold under `features/{{ feature_name }}/`.
4. Implement the state machine (idle → listening → processing → success/error).
5. Wire the gesture handler for **{{ gesture_trigger }}**.
6. Integrate the appropriate on-device ML model from the ML reference.
7. Add Arabic TTS output following the voice rules.
8. Add all accessibility props and run the shipping checklist.
