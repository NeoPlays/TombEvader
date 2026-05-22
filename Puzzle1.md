# Tomb Escape — Project Design Reference

**Course:** Multimedia 193.148 — SS25
**Group:** 3
**Engine:** Unreal Engine 5.6.1
**Deadline:** 26 May 2026, 23:59

---

## Concept

A first-person escape room game set in an Ancient Egyptian tomb. The player
is an archaeologist who steps into the tomb and accidentally triggers the
sealing mechanism. The whole game is about escaping.

Target gameplay length: **5–8 minutes** across **3 stages**.

---

## Game Arc (3 stages)

### Stage 1: The Sealed Tomb
- Player walks into tomb through one of three entrances
- All three doors slam shut simultaneously the moment any entrance is crossed
- 4 rooms total: a **main chamber** + 3 side chambers, all internally connected
- **Puzzle:** Picture/hieroglyph matching with light feedback (see below)
- **On solve:** Niagara portal appears in main chamber → step in → travel to Stage 2

**Atmosphere:** dark, claustrophobic, torchlight, dust

### Stage 2: The Void
- Pitch-dark room
- **Puzzle:** Microphone input puzzle
  - Uses audio amplitude (not speech recognition — more reliable, more atmospheric)
  - E.g. blow dust off a tablet, shout to break something, sustained tone to reveal hidden text
- **Reward on solve:** Player gains the **teleport ability**
- **On solve:** path opens to Stage 3

**Atmosphere:** sensory deprivation, audio-focused, no visual cues

### Stage 3: The Ascent
- High in the sky — vertigo, open air
- **Puzzle:** Jump-and-run platforming using the teleport ability earned in Stage 2
- This is the climax — player escapes by reaching the top

**Atmosphere:** open, vertiginous, contrast with the dark earlier stages

---

## Stage 1 Puzzle: Hieroglyph Picture Match (FINAL DESIGN)

### Layout
- **Main chamber:** entrance area, holds the 6 lights + 6 hieroglyphs on a wall (the puzzle's "answer key"). Portal spawns here on solve.
- **3 side chambers:** each contains **2 picture frames** the player can interact with. Total 6 frames.

### Core Mechanic
- Each picture frame cycles through several hieroglyph pictures when the player interacts (presses E)
- Each frame has one correct picture
- Main chamber has 6 lights, each with a hieroglyph symbol next to it on the wall — the hieroglyph is the answer key
- When a frame is set to its correct picture, the corresponding light in the main chamber turns on
- All 6 lights on → puzzle solved → portal appears

### Why this design works
- **The hieroglyphs on the wall are always visible** — they're the goal/answer key, not a reward. Players need to be able to see them or the puzzle becomes guessing.
- **The lights are the feedback** — instant confirmation when a frame matches
- **Players have a reason to traverse all rooms** — read the answers in the main chamber, then go set the frames in side chambers, then come back to verify
- **No "submit" button** — puzzle self-solves the moment the last correct picture is set
- **No frustration loop** — players can experiment freely, frames can be cycled and un-cycled
- **Atmospheric reward** — room transforms from mostly dark to fully lit as progress accumulates

### Blueprints needed

**`BP_PictureFrame`** (6 instances in level)
- Inherits from `BP_Interactable`
- Static Mesh: a flat plane / frame mesh
- Material with swappable texture parameter (`PictureTexture`)
- Variables:
  - `FrameID` (int, 1–6, Instance Editable)
  - `PictureOptions` (Array of Texture2D, Instance Editable)
  - `CorrectPictureIndex` (int, Instance Editable)
  - `CurrentIndex` (int, runtime)
  - `DynamicMaterial` (MID, runtime, created on BeginPlay)
  - Cached `Manager` reference (set on BeginPlay via `Get All Actors Of Class`)
- Override `OnInteract`:
  - Increment `CurrentIndex` (modulo array length)
  - Update material's `PictureTexture` parameter
  - Notify manager → `OnFrameChanged(FrameID, IsCorrect)`

**`BP_PictureManager`** (one instance in main chamber)
- Variables:
  - `Lights` (Array of Point Light references, Instance Editable, size 6)
  - `FrameStates` (Array of bool, size 6) — tracks which frames are currently correct
  - `IsSolved` (bool)
  - `PortalNiagara` (Niagara System ref) + spawn location
- Custom Event `OnFrameChanged(FrameID, IsCorrect)`:
  - `FrameStates[FrameID - 1] = IsCorrect`
  - `Lights[FrameID - 1].SetVisibility(IsCorrect)`
  - If all `FrameStates` are true → call `SolvePuzzle`
- Custom Event `SolvePuzzle`:
  - `IsSolved = true`
  - Spawn portal Niagara
  - Optional: triumphant sound, fade-up lighting, text reaction

### Hieroglyph wall (main chamber)
- 6 hieroglyph decals/planes placed next to each light on the wall
- **Always visible** (uses normal lit material with slight emissive boost so they read in low ambient light) — without this, the puzzle becomes guessing
- Not connected to anything in code — pure static decoration that serves as the puzzle's answer key
- The hieroglyph above each light shows the same image as the frame's correct picture

### Material setup
- `M_PictureFrame` material with a Texture parameter named `PictureTexture`
- Optionally Unlit + Emissive for a "magical glowing hieroglyph" look in the dark side chambers
- Each frame creates a Dynamic Material Instance on BeginPlay and sets the texture per `CurrentIndex`

---

## Player / Interaction System (current state)

### Pickup mechanic
- First-person line trace from camera (in `BP_FirstPersonCharacter`)
- Trace function `TraceForInteractable` is reusable — called from Tick (for crosshair state) and from `IA_Interact` (for interaction)
- UMG crosshair widget shows hover state when looking at interactable
- `IA_Interact` (Enhanced Input, bound to E) triggers `OnInteract(Caller)` on the hit `BP_Interactable`
- Carrying objects uses **UE5 Physics Handle component** (not attach-to-component) — natural physics, no manual transform math

### Inheritance pattern
- `BP_Interactable` is the parent class for all interactables (pictures, doors, future altar, etc.)
- Has no components — children provide their own meshes
- Defines virtual function `OnInteract(Caller)` — children override to define their behavior
- Player only knows `BP_Interactable` — doesn't care what specific class the child is. Polymorphism via override.

---

## Mandatory Requirements Coverage

| Requirement | How we cover it |
|-------------|-----------------|
| Image/UI feature | Picture frames change image based on user interaction (the cycle puzzle) |
| Text/UI feature | Text Render Actor reacts to puzzle progress / tomb sealing |
| Audio feature | Spatial audio (torches, footsteps) + microphone input in Stage 2 |
| Video feature | Intro cut-scene (player sealed in the tomb) |
| Niagara | Sand effects, portal appearance |
| World builder | Skybox + baked lights for tomb; Landscape tool outside entrance |
| Effects & Programming | Improved character motion (teleport, earned in Stage 2) |
| MetaHuman | TBD — possibly an NPC priest/ghost in the tomb, or in cut-scene |
| Basic UI | Reset, Exit, volume sliders, animated menu text |
| Avatar | First-person player + MetaHuman NPC |
| Environment | Tomb interior, landscape exterior, void room, sky platforming |

---

## Build Order

1. ✅ Door slam on entry
2. ✅ Crosshair UMG widget
3. ✅ Interact line trace + IA_Interact + BP_Interactable
4. ✅ Physics handle pickup
5. **Picture frame BP** — material with texture parameter, cycle on interact
6. **Picture manager BP** — track 6 frames, control 6 lights
7. **Level placement** — 6 frames in side chambers, 6 lights + hieroglyph decals in main chamber, hieroglyph textures imported
8. **Portal Niagara** — spawns on solve
9. **Level transition** — portal overlap → Stage 2 level
10. Stage 2 build (microphone puzzle)
11. Stage 3 build (sky platforming with teleport)
12. MetaHuman, cut-scene, UI polish
13. Build + test + video + docs

---

## Architecture Decisions

- **Game Mode per level** — holds per-stage puzzle state
- **Game Instance** — holds persistent state across levels (volume settings, abilities earned like teleport)
- **Single source of truth** — puzzle state lives in the manager, all actors call into it
- **Custom Events on individual actors** — e.g. doors close themselves when told to, rather than something else animating them. Keeps Timelines self-contained.
- **Separation of concerns** — pictures and lights are their own actors placed in the level. The manager only holds **references** to them and contains the logic. Manager does NOT contain pictures or lights as components.
- **BP_Interactable inheritance** — parent has no components, just defines the overridable `OnInteract` contract. Children provide their own meshes/components.

---

## Known Issues / Lessons So Far

- UE5 landscape brush "stuck in corner" bug: fixed by creating a temp new level, then reopening original (state reset)
- Timelines must live in the Blueprint of the thing being animated — calling one Timeline on multiple actors at once causes only the last one to animate. Wrap animation logic in a Custom Event on the target actor instead.
- Vertex color repair warning on map load: resave the map to bake the fix in permanently.
- `Set Collision Enabled` requires a Primitive Component (Static Mesh, Skeletal Mesh, etc.), not a Scene Component or plain Actor reference.
- When an Actor reference can't be assigned to a more specific class variable, either cast first or change the variable type. Generic `Actor` is fine for storage; cast when you need class-specific data.
- Physics Handle approach for grabbing/carrying is much cleaner than attach-to-component. No manual transform math, natural physics, easy release.