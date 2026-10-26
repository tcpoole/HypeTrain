# BUILD-PLAN.md — Chat Crowd core build order

## Rules for Codex

- **Core** = build it now, in stage order. **Stretch** = don't build it unless I ask for it by name. **Cut** = don't build it at all.
- Where this file and a spec disagree about scope, this file wins. Where a spec and `tuning.json` disagree about a number, `tuning.json` wins.
- Each stage ends with: the files and sections changed, how to test it, and what's left unclear. Stop and wait after each stage.
- Every Core feature gets a Test panel button so it works without Twitch.

## Core stages

### C1 — Foundation and the void (Mon 26 – Tue 27)

- Single-file `overlay.html`, pinned three.js import map, `WebGPURenderer` with WebGL 2 fallback, AgX tone mapping.
- **Post:** bloom only (emissive-driven).
- `tuning.json` loading and merge order, saved tweaks, Download tuning.json (SPEC-tuning.md section 1).
- `roster.json` loading, GLB cache, SkeletonUtils clones, model normalising.
- **One retarget function** for every Mixamo-named skeleton (SPEC-tuning.md section 3), with the `?debug=1` skeleton and clip player.
- Shared animation library loading and clip roles.
- **Void:** black floor fading to black, fog, faint dust. Shader spotlight per character (top-down falloff, rim, beam, pool, lamp, contact shadow, chat-colour tint, chat flare).
- **Beam-in / beam-out** using `Float_Levitate`. Crowd layout (scatter, rows, arc), max characters, longest-idle leaves first.
- **Cameras:** show cam, fly cam (`F`), `Esc` back to the show cam, double-click glide, saved cameras (1–5 / Ctrl+1–5).
- **Background and capture:** black default, `?bg=green|blue|magenta`, `?capture=1` (fixed 1920×1080, cursor hides), and keep rendering when the window is covered.
- **Minimal setup panel** (`` ` `` or `H`): channel name, background and capture toggles, the Test section, Download/Reset tuning. More gets added in later stages.
- **Lighting presets:** Noir (void default) only. The rest of `lightingPresets` in `tuning.json` are Stretch.
- **Done when:** "Add 100 fake chatters" fills the void with spotlit, idling characters at 60 fps, and `?webgl=1` works.

### C2 — Twitch chat and commands (Tue 27)

- Anonymous IRC, the event bus, ignore list, reconnect.
- **Name tags:** shared atlas, nearest 12 plus recent chatters, anti-overlap.
- **Chat bubbles** with Twitch emotes.
- **Commands:**
  - `!become` with numbers, codes, partial names, families and `random`; cooldowns; mods skip; saved picks; feedback bubbles.
  - `!dance` with library dances on the BPM (tap tempo in the panel).
  - `!sit` (`Sit_Floor_Idle`), `!stand`, `!wave`, `!cheer`, `!laugh`, `!sleep` (`Lay_On_Back`), `!players`.
- **Done when:** in a real chat, people appear on their first message, swap characters and dance.

### C3 — The host robot (Wed 28)

- Suit, helmet on the `Head` bone, `headGroup` frames exactly as in SPEC-tuning.md section 2. Fit editor and `?calibrate=1` view.
- **Webcam screen:** camera picker, `VideoTexture` on new frames only, centre-crop, mirror, zoom/pan, rounded mask and corner offsets from tuning, unlit/emissive, visor glass.
- **Camera off:** if there's no webcam, show a simple static glowing robot face (two eyes and a smile, from `host.robotFace` colours) so the screen is never black.
- **Mic, simple:** talk glow on the screen edge while talking.
- `tc-host-anims.glb` retargeted. Idle (`Weight_Shift` / `Arm_Stretching` plus breathing).
- **Gesture hotkeys** from `host.anim.keys` (Shift+0 stops), and Alt+D dance from the library.
- **Host mode (`G`), basic motion only:**
  - W walk, Shift run (and both mouse buttons), S backpedal (`Walking_Backwards` / `Running_Backward`), A/D turn (`Turn_L/R` when standing).
  - `Run_To_Stop` on release.
  - WoW camera: left-drag orbit, right-drag orbit-and-turn, wheel zoom, camera swings behind while moving. Speeds from `host.suit.walk`.
- **Host in the void:** front and centre in its warmer spotlight.
- **Done when:** I walk my robot around the crowd with my face on its screen and wave with Shift+1.

### C4 — Twitch login, EventSub and the Hype Train (Thu 29) ← gate

- **Login:** implicit grant, token in `localStorage`, validate hourly, Disconnect.
- **EventSub WebSocket:** welcome, subscribe within 10 s, reconnect handling. Subscribe to everything in SPEC.md section 4.
- **City-lite:**
  - A ring of instanced towers around the track with the hashed window shader, a few neon signs, and 2 billboards (channel name, Hype level).
  - Night gradient sky with a hazy moon, fog, wet-look ground.
  - **No** traffic, rain or GTAO.
- **Track and train:**
  - A closed `CatmullRomCurve3` loop, glowing rails, pylons, a simple platform with a HYPE CENTRAL neon sign.
  - A code-built maglev locomotive, matching code-built glass-domed observation cars, and one code-built seat instanced throughout (SPEC.md section 7), with cars following the curve on bogies. No downloaded train models.
- **Riders:** sit with library `Chair_Sit_*` clips, name tags through the glass.
- **Sequence:**
  1. Light-wipe from the void to the city.
  2. The train arrives.
  3. Riders beam in on the platform and appear in seats (top contributors first, then subs, cheers and gifts, then chatters).
  4. The train departs and loops. Speed rises per level, a car is added per level (max 8), and a "LEVEL N!" banner shows.
  5. HUD: level, progress bar, countdown, top 3 names.
  6. End: the train climbs into the sky, fireworks show the channel name, end card, back to the void with Hype Rider rings.
- **Host** on the conductor's platform at the front, waving at new riders.
- **City director:** 4 shots (chase cam, side tracking, rooftop wide, rider close-up), plus "now boarding" cuts.
- **All aboard** reward version (2 cars, 45 s).
- **Done when:** the Test panel's Full Hype Train plays start to finish with no errors, and a real login connects EventSub.

### C5 — Twitch moments, director and polish (Fri 30)

- **Raid:** the big raider at the front, raiders march in, a banner, a director cut.
- **Follow:** drops in from above with a dust burst.
- **Sub / resub:** a crown on the name tag for the session, gold light for 10 s.
- **Gift subs:** gift boxes rain on recipients.
- **Cheer:** fireworks scaled to bits.
- **Bonk:** `!bonk @user` (mods, broadcaster, or a reward) and `!getup`, with circling stars and `Sit_Floor_Dazed` when getting up. Light-hearted, cooldowns from `tuning.json` → `commands.bonk`.
- **Reward mapping UI:** spotlight, confetti, bonk, All aboard.
- **Void director (`V`):** 4 shots (wide, crane down, chatter close-up, host face-screen close-up), cuts on the beat, `!cam` requests, event interrupts.
- **Setup panel finished:** Twitch connection status, crowd sliders, host fit, commands on/off, reward mapping, credits.
- **README:** setup (Chrome + Window Capture, the two Chrome flags, camera note), controls, commands, credits.
- **Performance check:** 150 fake chatters plus the host at 60 fps on an RTX 3060-class GPU, and no memory growth over 30 minutes of fake chat.
- **Done when:** every Test panel button works, and a 20-minute live test on stream has no errors.

### P — Players page (any time, parallel task)

Build all of SPEC-players.md, with one simplification: cached thumbnails in the grid, and a live 3D viewer only in the detail view (no hover turntable on cards).

## Stretch (in order of value; only when asked)

1. Talking gestures from the mic (the clip list in SPEC-host.md section 7)
2. Face tracking: head pose and lean, in a worker
3. Animated robot-face mode driven by blendshapes, the `B` toggle, the `N` privacy key
4. Strafe (`Strafe_Walk/Run_L/R`) and jump (`Jumping_Up` → `Falling_Idle` → `Falling_To_Landing`)
5. More director shots (void and city)
6. Lurkers (`JOIN` / `PART`)
7. The other lighting presets, and the city's Storm and Dawn looks
8. City extras: flying traffic, rain, station details, the arrivals board
9. Voice triggers (speech recognition)
10. Webcam light spill, VHS look
11. TRAA, GTAO, dynamic resolution, per-pass timing
12. `!salute` and `!lay` as separate commands

## Cut (not for the hackathon)

- Crouch, crouched walking, front flip, backflip, running slide, falling roll
- Hover locomotion
