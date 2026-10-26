# SPEC-tuning.md — Tuning file and shared animation library

All hand-tuned numbers for Chat Crowd live in **`tuning.json`**. Read it and follow these rules in every stage. Where `tuning.json` and a number in SPEC.md, SPEC-host.md or SPEC-players.md disagree, **`tuning.json` wins**. The specs' numbers are only fallbacks.

## 1. How tuning works

1. At startup, `fetch('tuning.json')` and deep-merge it over the code's built-in defaults. If the fetch fails, run on the built-in defaults and log a warning.
2. Then apply the streamer's own saved tweaks from `localStorage` (key `chatcrowd.tuning.v1`), then URL parameters. So the order is: code defaults → `tuning.json` → saved tweaks → URL.
3. **Every slider, fit editor and preset in the setup panel reads from and writes to this one merged object.** No setting lives anywhere else.
4. The setup panel's Settings section gets:
   - **Download tuning.json:** the full merged object, pretty-printed, so I can commit it to the repo. The next deploy then has the tuning baked in for everyone.
   - **Copy / paste tuning JSON** for single sections or the whole thing.
   - **Reset section** and **Reset all**, back to `tuning.json` values (not code defaults).
5. Keep the structure and key names exactly as they appear in `tuning.json` so files stay compatible. New settings get new keys; never rename existing ones.

## 2. Reference frames (so the Skullhead numbers land in the right place)

The values came from Skullhead, so they only mean the same thing if they're applied in the same space:

- **Helmet:** load `funkbot2-head.glb` with **no auto-normalising**. Keep its raw GLB units and orientation.
  - Make a `headGroup` and parent it to the suit's `Head` bone.
  - Put the helmet inside it and apply `host.head.transform` (position, rotation in radians, scale) to the helmet.
  - The webcam screen (`host.screen`), the visor (`host.visor`) and the robot face (drawn on the screen) are **children of `headGroup`**, using their position/rotation/scale exactly as stored.
  - Then scale the whole `headGroup` by `host.head.sizeOnSuit` relative to the suit.
- **Screen shape:** `host.screen.scale.x/y` are the screen's width and height. `round` is the corner rounding (0–1). The `tlX…brY` keys are corner offsets in screen-local units. `light0`–`light5` are the spill light offsets in the screen's local axes (X right, Y up, Z forward), and `display` holds brightness, saturation, spill strength, reach, depth, spread and smoothing.
- **Visor:** `host.visor.surface` is a physical glass material (transmission, ior, thickness, attenuation, clearcoat, roughness, opacity, tint), plus curve and corner shaping, sized by `host.visor.scale`.
- **Robot face:** `host.robotFace` positions are fractions of the screen: `eyeY`, `mouthY`, `eyeSpacing` and the like run 0–1 from the screen's top-left. `depth` is the offset in front of the screen.
- **Tracking:** `host.tracking.lookX`, `lookY` and `roll` are the maximum head angles in degrees (these replace the ±45/30/25 in SPEC-host.md). `smoothing` is 0–1, and `followY`/`followZ` are how much the head's movement shifts the body.
- **Suit:** `host.suit.size` multiplies the suit after it's fitted to the host height. `pose` holds the idle arm and leg angles in degrees. `walk.speed` and `walk.sprint` are world units/s and the run multiplier. `walk.animSpeed` is the clip timescale.

Anything measured in world units in Skullhead's scene (crowd distances, spot heights, train seat spacing) was tuned at Skullhead's scale. Apply it as-is first. If the new scene's scale differs, add **one** global `worldScale` factor in `tuning.json` rather than changing each value.

**Calibration view:** `?calibrate=1` shows the host alone on a grey floor, from the front, side and three-quarter at once, with the screen showing a test pattern (centre cross and corner markers). That makes it easy to check the screen sits in the visor opening and to nudge it once with the fit editor if it doesn't.

## 3. One retargeting system for every character

The host suit, all 31 roster characters, the host pack (`tc-host-anims.glb`) and the shared library (`chatcrowd-anim-library.glb`) all use **Mixamo bone names**. The library's are bare (`Hips`, `Spine2`, `LeftForeArm`, …), and the others are prefixed `mixamorig:` or `mixamorig`.

- Write **one** retarget function: match bones by name after stripping the prefix, carry each keyframe's world-space rotation change from the source rest pose onto the target rest pose, keep only the vertical hips bob (scaled by hip-height ratio), and cache the result per clip and model.
- Bones the library doesn't have (fingers, toes, head-top) stay at the target's rest pose.
- Use it for the host and for every chatter.

## 4. The shared animation library

`roster.json → host.library` points to `chatcrowd-anim-library.glb`: 43 clips on a 22-bone Mixamo skeleton.

| Group | Clips | Used for |
| --- | --- | --- |
| Chair sits | `Chair_Sit_Relaxed`, `Chair_Sit_Groove`, `Chair_Sit_Bored`, `Chair_Sit_Chatty`, `Chair_Sit_Nodding`, `Chair_Sit_Asleep` | **Hype Train riders**, picked per rider and changed every so often. `Chair_Sit_Asleep` for idle riders, `Chair_Sit_Chatty` while they chat. The host's seat poses use the same clips. |
| Dances | `Dance_Hip_Hop_A/B/C`, `Dance_House_A/B`, `Dance_Locking`, `Dance_Robot_Hip_Hop`, `Dance_Tut`, `Dance_Wave_Hip_Hop`, `Dance_Slide`, `Dance_Classic`, `Dance_Belly_Dance`, `Dance_Maraschino_Step`, `Dance_Macarena`, `Dance_Northern_Soul` (+ `_Floor`, `_Spin`), `Dance_Brooklyn_Uprock`, `Dance_Uprock_1–4`, `Dance_Breakdance_*` (footwork, swipes, freezes, 1990 A–C), `Dance_Flair_A–C` | `!dance` for every chatter (replacing the procedural dances, which stay only as a fallback), the host's Alt+D dance and the `Dance_Hip_Hop_A` hotkey. Time-scale to the BPM. Breakdance and flair clips go in a "big moves" set used for hype moments only. |
| Floor | `Sit_Floor_Idle`, `Sit_Floor_Dazed` | `!sit`. `Sit_Floor_Dazed` for getting up after a bonk. |
| Other | `Lay_On_Back` (`!lay`, and sleeping on the floor), `Float_Levitate` (the beam-in arrival pose, rising out of the light) | |

The hotkeys in `host.anim.keys` use Skullhead's clip names. Map `Wave` to the pack's `Waving`. Everything else matches a clip in `tc-host-anims.glb` or the library.
