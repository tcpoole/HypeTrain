# SPEC.md — Chat Crowd overlay (overlay.html)

You are building **Chat Crowd**, a Twitch overlay for the Codex × TwitchCon hackathon. It's a web page hosted on my site, which the streamer opens in Chrome and brings into OBS with Window Capture. Every Twitch chatter appears as a 3D character standing in their own spotlight in an endless black void. When a Twitch Hype Train starts, the overlay cuts to a neon futuristic city and an actual train pulls into a station. The chatters who fueled the hype board it and ride through the skyline, gaining a car with every Hype Train level.

Build it from scratch in this repo. Read `AGENTS.md` and `roster.json` before you start. This document is the spec: save it as `SPEC.md` if it isn't already, and build it in the stages listed at the end. Do **one stage per task**, and stop after each stage so I can test it.

## 1. Files

```
overlay.html     the whole overlay: one self-contained HTML file (inline JS and CSS)
roster.json      the character list (already provided — do not hand-edit the URLs)
players.html     the public character gallery (separate prompt, not this one)
README.md        setup, controls, chat commands, credits
```

`overlay.html` is a single file, which is how I like my apps. Inside it, organise the code into clearly fenced sections (`// ===== TWITCH CHAT =====`, `// ===== CROWD =====`, and so on), each a self-contained block that talks to the rest only through the event bus and a small shared `app` object. No build step, no npm.

## 2. Tech stack

### Import map (pin exactly this version)

```html
<script type="importmap">
{ "imports": {
  "three": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.webgpu.js",
  "three/webgpu": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.webgpu.js",
  "three/tsl": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.tsl.js",
  "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.186.1/examples/jsm/"
} }
</script>
```

### Renderer

- `WebGPURenderer` with `{ antialias: false, powerPreference: 'high-performance' }`. Await `renderer.init()`.
- **Automatic fallback:** if WebGPU isn't available, the same renderer runs on its WebGL 2 backend. Also support `?webgl=1` to force the WebGL 2 backend for testing (`forceWebGL: true`). Log which backend is active, and show it in the setup panel.
- **Background:** solid black by default, because the void is a full-screen scene. Window Capture can't carry transparency, so for streamers who want the crowd over gameplay, `?bg=green`, `?bg=blue` or `?bg=magenta` fills the background with a pure key colour for an OBS chroma-key filter. Keep the key colour out of bloom, fog and the vignette so the key stays clean.
- **Capture mode:** `?capture=1` (also a toggle in the setup panel) renders at exactly 1920×1080, letterboxed to fit the window, so the OBS capture never changes size. It also hides the mouse cursor after 2 s without movement and guarantees no UI shows unless the panel is toggled open.
- Tone mapping: `AgXToneMapping`, exposure adjustable per lighting preset. Output colour space sRGB.
- Pixel ratio capped at `min(devicePixelRatio, 1.5)`.
- **Dynamic resolution:** measure frame time over a rolling 2-second window. If it's above 18 ms, lower the render scale in 0.1 steps down to 0.6. If it's below 13 ms, raise it back toward 1.0. Show the current scale and fps in the setup panel.
- **Shader warm-up:** after models load, call `renderer.compileAsync(scene, camera)` before revealing anything, so a new character arriving never causes a hitch.
- When nothing is animating (empty void, no chat for 60 s), drop to 30 fps. Don't pause on `document.hidden`: in window-capture use, the window can be behind others and must keep rendering. The README tells streamers to launch Chrome with `--disable-backgrounding-occluded-windows --disable-renderer-backgrounding` and never minimise the window.

### Post-processing (TSL `PostProcessing` pipeline)

- Scene pass with MRT output (colour + emissive) so **bloom is driven by emissive only**, without a second scene pass. Bloom strength, radius and threshold come from the lighting preset.
- **TRAA** (temporal reprojection anti-aliasing) on WebGPU. On the WebGL 2 backend, fall back to FXAA if TRAA misbehaves.
- **GTAO** ambient occlusion in the city scene only (it costs too much in the void and isn't needed there).
- Final grade: contrast, saturation and a vignette applied as a TSL node at the end of the chain, all set by the lighting preset.
- Every pass can be toggled in the setup panel, and the panel shows the cost of each in ms if the backend supports timestamp queries.

### Loading

- `GLTFLoader` with `DRACOLoader` (decoder path `three/addons/libs/draco/gltf/`). The host's suit and helmet are Draco-compressed, and most models use `EXT_texture_webp`.
- Cache each GLB once by URL. Make characters with `SkeletonUtils.clone()` so every viewer of the same model shares geometry, textures and clips.
- Load lazily: a model only downloads when someone picks it or is randomly assigned it. Preload the 6 most-used models after startup.
- If a model fails to load, show that viewer as a glowing capsule with their name tag and retry once after 30 s. Never let a failed load break the overlay.

## 3. Characters (`roster.json`)

`roster.json` lists 31 characters. Each has `n` (1–31), `code`, `name`, `epithet`, `family` (`knight`, `robot`, `frog`, `undead`), `scale` and `url`. Families have chat aliases.

### Normalising models

- Every model is a skinned mesh, but they arrive at different sizes and orientations. On first load, measure the bounding box in its rest pose and scale it so the character stands at `app.settings.charHeight` (default 1.8 world units) × the entry's `scale` (frogs are 0.5). Put its feet on y = 0 and have it face +Z.
- Material look: keep the original textures. Set `envMapIntensity` to 0.6, clamp roughness to at least 0.35 so nothing looks like plastic, and keep emissive maps (bloom picks them up).

### Animation clips

The GLBs contain clips such as `Running`, `Walking`, `Sit_Cross_Legged`, `Sit_and_Doze_Off`, `Chair_Sit_Idle_F`, `Sit_and_Drink`, `FunnyDancing_01`, `Hip_Hop_Dance`, `You_Groove`, `All_Night_Dance`, `Bass_Beats`, `Cherish_Pop_Dance`, `Arm_Circle_Shuffle` and `Dozing_Elderly`. They do **not** all have the same clips, and none has an idle, wave or jump clip.

- At load, sort each model's clips into roles by name: `walk`, `run`, `sitFloor`, `sitChair`, `doze`, `dance[]`. Ignore clips named `restpose` or `Armature|clip0|baselayer`.
- **Anything missing is built procedurally from the bones**, layered on the rest pose:
  - **Idle:** breathing (chest and shoulders rise slowly), weight shift side to side, slow head glances that sometimes look at the camera. Arms hang slightly away from the body with a small elbow bend, never a T-pose.
  - **Wave:** right upper arm raised, forearm swinging.
  - **Salute:** right hand to brow, held 2 s.
  - **Cheer:** both arms up, two small hops.
  - **Laugh:** shoulders bounce, head tips back.
  - **Sleep:** if there's no `doze` clip, the head drops and breathing slows. Floating "Z z z" sprites rise above the head.
  - **Jump** and **crouch** for the host avatar (see section 6).
  - **Sit on the floor** if there's no `sitFloor` clip: hips lowered, thighs forward, knees up.
- Blend between roles with `AnimationMixer` crossfades of 0.4 s. Procedural layers blend in and out with the same timing.
- **Far-away characters animate at a lower rate** (every 2nd frame beyond 25 units, every 4th beyond 50).

### Who looks like what

- A chatter who hasn't picked gets a model chosen by hashing their Twitch user ID, so the same person always gets the same default.
- `!become` picks are saved per login in `localStorage` (key `chatcrowd.picks.v1`), so a viewer keeps their character across streams. Wrap all storage in try/catch.

## 4. Twitch

### Config

At the top of the script: `const TWITCH_CLIENT_ID = '...'` (I'll fill it in) and `const REDIRECT_URI = location.origin + location.pathname`.

### Chat, with no login

- Anonymous IRC over `wss://irc-ws.chat.twitch.tv:443`, nick `justinfan` + 5 random digits. Request `twitch.tv/tags twitch.tv/commands twitch.tv/membership`.
- Parse tags for `user-id`, `display-name`, `color`, `badges` (broadcaster, moderator, vip, subscriber, founder) and `emotes`.
- `JOIN` and `PART` from the membership capability feed the **lurker** feature.
- Reconnect with exponential backoff (1 s → 30 s), and answer `PING` with `PONG`.
- **Ignore list:** nightbot, streamelements, streamlabs, moobot, fossabot, wizebot, soundalerts, sery_bot, kofistreambot, commanderroot. The streamer can edit it.

### Events, with one login

- **Implicit grant:** the setup panel has a "Connect Twitch" button. It sends the streamer to `https://id.twitch.tv/oauth2/authorize` with `response_type=token`, the client ID, the redirect URI, a random `state` kept in `sessionStorage`, and the scopes `channel:read:hype_train channel:read:redemptions moderator:read:followers channel:read:subscriptions bits:read`. On return, read `access_token` from `location.hash`, check `state`, clear the hash from the address bar, and store the token in `localStorage` (key `chatcrowd.token`) so it survives reloads in the same Chrome. A "Disconnect" button deletes it.
- Call `https://id.twitch.tv/oauth2/validate` at startup and every hour. If the token is invalid, show "Reconnect Twitch" in the setup panel only. The overlay itself carries on with chat-only features and shows no errors on stream.
- Look up the broadcaster's user ID with `GET https://api.twitch.tv/helix/users`.
- **EventSub WebSocket** at `wss://eventsub.wss.twitch.tv/ws`. On `session_welcome`, create every subscription (via `POST /helix/eventsub/subscriptions`) within 10 s. On `session_reconnect`, connect to `reconnect_url` and close the old socket only once the new one sends its welcome. Treat a missing keepalive as a disconnect.
- Subscriptions:
  - `channel.hype_train.begin`, `channel.hype_train.progress` and `channel.hype_train.end`, all version 2
  - `channel.channel_points_custom_reward_redemption.add`
  - `channel.raid` with `to_broadcaster_user_id`
  - `channel.follow` v2 (with `moderator_user_id` set to the broadcaster)
  - `channel.subscribe`, `channel.subscription.gift` and `channel.cheer`
- Check the payload field names against https://dev.twitch.tv/docs/eventsub/eventsub-subscription-types/ when you implement each handler. Log every raw payload to the console behind `?debug=1`.

### Token safety

The token never goes in the URL, so it can't leak if the address bar is ever captured. It lives in `localStorage` only and is never printed on screen or in the console. The status line shows "Connected as {name}".

### Event bus

Every input becomes one event shape and goes through one bus. Features only listen to the bus.

```
{ type, user: { id, login, displayName, color, badges }, data, source: 'irc'|'eventsub'|'test', at }
```

Types: `join`, `part`, `chat`, `command`, `reward`, `raid`, `follow`, `sub`, `gift`, `cheer`, `hype_begin`, `hype_progress`, `hype_end`.

## 5. Scene A: The Void (default)

An endless black space. No visible floor edge, horizon or walls. Each viewer stands in their own pool of light. It should look like a theatre with the house lights off.

### Space

- Floor: a huge plane with a dark, slightly glossy material whose reflectivity fades to pure black by distance from the centre, so it never ends visibly. Exponential fog to black.
- Very faint drifting dust particles across the whole space (GPU instanced points, less than 2,000).

### Per-character spotlight (shader-level, no real lights)

Do **not** add a `SpotLight` per viewer. Light each character in its material with a TSL node that computes a spotlight from the lamp position above its head. That way 150 viewers cost nothing extra in lights.

- **Top-down falloff:** bright on head and shoulders, fading down the body (adjustable "drama").
- **Rim light** catching the edges.
- **Visible beam:** an additive cone mesh from the lamp to the floor, fading along its length, with slow-moving dust specks inside.
- **Light pool** on the floor: a soft disc with an adjustable edge.
- **Lamp fixture** at the top: a small emissive disc that blooms.
- **Soft contact shadow:** a dark blurred disc under the feet.
- **Tint:** the light takes on a little of the viewer's Twitch chat colour (adjustable, default 20%).
- **Chat flare:** when they chat, their light flares to 1.6× and settles back over 1 s.
- Settings for all of these: spot brightness, height, pool radius, edge softness, beam strength, dust, rim, drama, tint, ambient (the house lights).

### The host

The broadcaster's own character stands front and centre in a bigger, warmer spotlight. It uses their `!become` pick. This is the character the WASD controls drive (section 6).

### Crowd layout

- Viewers stand behind and around the host. Spots are pre-computed best-first (nearest the host and most visible to the camera first).
- Layouts: `scatter` (default, seeded random with no overlaps), `rows`, and `arc`. Settings: spacing, depth, width, widen toward the back, scatter jitter, a clear lane behind the host, offset left/right and forward/back, and a seed for a new random layout.
- Each character turns partly toward the host (default 30%) and partly toward the camera, plus a random ±14° so the crowd doesn't look cloned.
- Changing a layout setting eases everyone to their new spots over 2 s.

### Arriving and leaving

- **Beam-in:** a viewer's first message (or `JOIN` if lurkers are on) brings their light in first, then the character materialises from the top of the beam down over 1.6 s, using a dissolve with a glowing edge.
- **Beam-out:** reverse it when they `PART`, after `idleMinutes` of silence (default 0 = never), or when the crowd is full and the longest-idle viewer has to make room.
- Max characters: 150, adjustable from 1 to 300.
- **Lurkers:** if on, people seen via `JOIN` who haven't chatted appear in the back rows at 40% light. They brighten fully the first time they chat.

### Name tags and chat bubbles

- Canvas-texture billboards, drawn into one shared atlas, never a DOM element per viewer.
- **Name tags:** the display name in their chat colour, with a dark outline, above the head. Show tags for the nearest 12, plus anyone who chatted in the last 45 s. Hide all tags beyond 260 units.
- **Anti-overlap:** nearer tags stay put, and farther ones step up so tags never sit on top of each other.
- **Chat bubbles:** the message above the speaker for 7 s (adjustable, 0 turns them off), truncated at 90 characters. Twitch emotes are drawn inline as images from `https://static-cdn.jtvnw.net/emoticons/v2/{id}/default/dark/1.0`, using the `emotes` tag positions.
- Badges: a small crown for subscribers, a sword for mods, a star for VIPs and a "Hype Rider" golden ring for this session's train riders (section 7).

### Dancing

- One shared beat clock. BPM defaults to 118, and the setup panel has tap tempo.
- `!dance` plays one of the model's dance clips if it has any, time-scaled to the BPM. Models without one use procedural dances: bounce, two-step, arm pump, shoulder shimmy, robot, side shuffle, head bang, sway. Each dancer gets a random style so the floor looks mixed. `!dance` again switches style.
- Dancers' knees bend on the beat, and the body drops to match.

## 6. Camera and controls

The overlay runs in a Chrome window that OBS captures, and I drive it live by clicking into that window, so it needs full controls. Keys only act when the page has focus and the setup panel's text fields aren't being typed in.

### Camera modes

| Key | Mode | Behaviour |
| --- | --- | --- |
| (default) | Show cam | A slow, composed drift, framed on the host with the crowd behind. It gently pushes in and out over 40 s and never moves fast. |
| `V` | Director | Automatic cinematic cuts (below). Press again to return to the show cam. |
| `F` | Fly cam | Free flight. Mouse look with left-drag (pointer lock while held), **W/A/S/D** to move, **Q/E** down/up, **Shift** for 4× speed, **Alt** for 0.25× (precise), wheel changes base speed. |
| `G` | Host mode | WoW-style third-person on the host avatar (below). |
| `Esc` | — | Leaves any mode and glides back to the show cam. |

### Host mode (WoW-style)

- **W/S:** walk forward or backpedal (play `Walking` reversed for backpedalling, at 0.7×).
- **A/D:** turn. If the right mouse button is held, they strafe instead. **Q/E** always strafe. Strafing plays `Walking` with the body twisted 45° toward the direction of travel and the hips leaning into it.
- **Shift:** toggles walk/run (`Running` clip). Both mouse buttons held: run forward.
- **Space:** procedural jump (crouch-anticipation, arc, land with a knee bend).
- **C:** crouch toggle (procedural knee and hip bend, half walking speed).
- **Mouse:** left-drag orbits the camera around the character. Right-drag orbits **and** turns the character to face where the camera looks. Wheel zooms from 0.35 to 14 units. Pointer lock while a button is held.
- The camera swings back behind the character while it moves, and pulls in when the character is close to another character rather than clipping through them.
- Walking speed 1.4 units/s, running 4.2 units/s, turning 180°/s. Locomotion blends in 0.25 s.
- Movement is limited to a 60-unit radius around the centre of the void.

### Everywhere

- **Double-click any character:** the camera glides over (1.2 s, eased) and frames them. Their name card fades in at the bottom of the screen.
- **Alt+X or Home:** glide back to the show cam.
- **1–5:** recall a saved camera position. **Ctrl+1–5** saves the current one. They persist in `localStorage`. (Shift+number is reserved for the host's gestures in SPEC-host.md.)
- **Mouse wheel** in the show cam: dolly in and out. **Middle-drag:** pan.

### Director (`V`)

The director cuts between shots on the beat (every 4 or 8 bars):

- Wide establishing shot of the whole crowd.
- Slow crane down from above.
- Dolly along the front row.
- Close-up of whoever chatted most recently.
- Worm's-eye push-in on the host.
- Slow 360° orbit of a random viewer with their name card.
- Telephoto profile of a dancer.

Chat steers it: anyone typing `!cam` is queued for a close-up (one per viewer per 2 minutes). Big events (reward, raid, sub) interrupt with an immediate close-up of that viewer. Cuts are hard cuts, or a 0.3 s whip-pan between related shots.

## 7. Scene B: The Hype Train (the centerpiece)

### The city

A neon megacity at night, built procedurally in code. It's a different world from the void, and the cut to it should feel like an event.

- **Towers:** about 2,000 buildings as a handful of `InstancedMesh` batches, with varied heights, setbacks, rooftop antennae and blinking aircraft-warning lights.
- **Windows:** a TSL shader gives each window cell a hashed on/off state, warm or cool colour, and the occasional flicker. The floor count comes from the building height so the windows line up.
- **Neon and screens:** neon signs (canvas textures, flickering, bloom) and giant holographic billboards. Some billboards show the channel name and live Hype Train level, the rest show made-up brand names. No real-world brands.
- **Sky:** a deep blue-violet gradient with smog glow near the horizon, a few stars and a big hazy moon. Height fog thickens toward the street.
- **Street:** wet asphalt with neon reflections (low roughness plus the environment map), and street-level light strips.
- **Flying traffic:** instanced vehicles on 6 to 8 looping lanes at different heights, with emissive headlights, tail lights and short light trails. Some lanes cross the track.
- **Optional rain:** instanced GPU streaks, toggled in the panel, default off.
- **Budget:** under 300 draw calls for the whole city. Frustum-cull the instance batches by city block.

### The track

- An elevated maglev line on a closed `CatmullRomCurve3` loop through the city, about 2 km. It weaves between towers, runs through a lit tunnel section and climbs over a big bend with a skyline view.
- Glowing guide rails, pylons every 40 m, and a station called **HYPE CENTRAL**. The station has a platform, a neon sign, an arrivals board showing the riders' names, and glass doors.

### The train

The whole train is **built in code**: no downloaded models. Engine, cars and seats share one design language so it reads as one train, and this is a showcase of what Codex can build.

- **Locomotive (code-built):** a sleek maglev engine about 1.6× a passenger car's length.
  - **Body:** a long aerodynamic nose that tapers to a rounded point (lathe or extruded profile, smoothed), a wraparound tinted cockpit windscreen, and side panel lines.
  - **Underside:** a slim hover skirt over the guide rail with a soft glowing gap underneath, and no wheels.
  - **Lights:** emissive light strips along the sides and the nose edge (accent colour from `tuning.json`, default cyan with a warm amber second accent) that bloom at night. Twin headlights cast visible beams down the track, and running lights on the roof.
  - **Materials:** dark gunmetal paint with a clearcoat sheen (`MeshPhysicalMaterial`, metalness about 0.6, roughness about 0.3, clearcoat 1), picking up neon reflections from the city. Panels can be a few slightly different shades so it doesn't look like one flat plastic piece.
  - **Branding:** the channel name in glowing letters along the side (a canvas texture), and a small "HYPE EXPRESS" plate on the nose.
  - **Pulse:** on each Hype Train level-up, the light strips pulse brighter for 1 s and the hover glow flares.
  - **Performance:** under 20k triangles and a handful of materials.
- **Passenger cars:** built to match the engine. They're **glass-domed observation cars** so passengers are always visible from outside: the same gunmetal lower body, light strips and hover skirt, a lit floor, seat rows (default 8 rows × 4 seats with an aisle), and coupling joints with a small glowing connector between cars.
- **Seats (code-built):** a simple sci-fi seat, a moulded shell with a seat pan, backrest, headrest, side bolsters and a thin glowing edge strip. Use one geometry instanced across every car.
- **Backup only (don't use unless I ask):** `flying_futuristic_train_with_uncleaned_graffiti-optimized.glb` in my R2 bucket (CC-BY-4.0, `@blue.blender.print` on Sketchfab). If I ask for the swap, load it as the locomotive, scale and orient it to the track, and add its credit to the README and the setup panel.
- **Riders:** each rider sits in a seat using their model's `sitChair` clip if it has one. Otherwise build a chair-sit from the bones: thighs level on the seat, shins hanging down, hips far enough forward that knees clear the seat edge. Riders keep idling (glances, chatting bubbles, waving out of the dome).
- **Name tags** show through the glass.
- Cars follow the curve at fixed spacing (a car's front and back bogies each sample the curve and the body sits between them), so the train bends realistically through curves.

### The sequence

1. **`hype_begin`:**
   - The void plays a transition: every spotlight swings up, then a white light-wipe.
   - Cut to the city. Wide skyline shot, the train glides into HYPE CENTRAL, title card: "HYPE TRAIN — ALL ABOARD".
2. **Boarding:**
   - Riders materialise on the platform with the same beam-in effect as the void, walk to the doors (`Walking` clip), and appear in their seats.
   - Seat order: top contributors from the payload first (gold glow, front car), then anyone who subs, gifts or cheers during the train, then everyone who chats during the train (standing-room crowd in the back cars if seats run out).
   - The arrivals board lists names as they board.
3. **Departure and ride:**
   - The train pulls out and loops the city.
   - Speed rises with each level (level 1 = 20 m/s, +6 m/s per level, max 60).
4. **`hype_progress`:**
   - Updates the HUD.
   - On a level-up, a new observation car glides in on a side track and couples on (sparks plus a clunk sound). Banner: "LEVEL 3!"
   - New contributors appear in the new car.
   - Maximum 8 cars, after which levels add a gold trim to the cars instead.
5. **HUD:**
   - A slim bar at the top of the screen: Hype Train level, a progress bar from `progress` / `goal`, and a countdown to `expires_at`.
   - The top three contributors' names.
   - Clean, readable on a 1080p stream, with no clutter.
6. **`hype_end`:**
   - The train speeds into the tunnel, bursts out, and climbs into the sky over the skyline.
   - Fireworks spell the channel name.
   - End card: "HYPE TRAIN COMPLETE — LEVEL N" with the rider count.
   - Fade back to the void. Every rider gets the golden **Hype Rider** ring on their name tag for the rest of the session.
7. **Director in the city** cuts between:
   - Chase cam behind the last car
   - Side tracking shot alongside the train
   - Flyby from a passing vehicle
   - Interior aisle shot down the dome
   - Rooftop wide shot as the train passes
   - Hero close-up of the top contributor
   - "Now boarding" cuts to each new rider

   The fly cam (`F`) works in the city too.

### "All aboard" (for channels without real Hype Trains)

- A channel-point reward the streamer maps to "All aboard" in the setup panel.
- It runs a short version: 2 cars, the redeemer plus the next 10 chatters, a 45-second loop, then back to the void.
- Same visuals, no HUD bar, title card "ALL ABOARD — courtesy of {name}".
- Cooldown 5 minutes.

## 8. Chat commands

| Command | Aliases | Who | What it does |
| --- | --- | --- | --- |
| `!become <n\|name\|family\|random>` | `!robot`, `!droid`, `!knight` | everyone | Switches to roster character `n` (1–31), a code or part of a name (`grol` finds Grol'thak), a random member of a family (`!become frog`), or `random`. With no argument, a bubble shows "pick a character: !become 1–31 · see the roster at {playersUrl}". Cooldown 30 minutes per viewer, a 5-second gap between anyone's swaps, and mods and the broadcaster skip both. Feedback bubble: "⚡ is now Sir Reginald Fleur-de-Crumb". If they're on cooldown: "⏳ swap recharging · 12 min left", shown at most once per 45 s. |
| `!dance` | | everyone | Dance on the beat. Again = new style. |
| `!sit` | | everyone | Sit on the floor. |
| `!stand` | `!up` | everyone | Back to idle. |
| `!wave` | `!hi` | everyone | Wave. |
| `!salute` | | everyone | Salute. |
| `!cheer` | `!clap`, `!woo`, `!yay` | everyone | Cheer. |
| `!laugh` | `!lol`, `!haha` | everyone | Laugh. |
| `!sleep` | | everyone | Doze until they next chat. |
| `!cam` | | everyone | Ask the director for a close-up. |
| `!players` | | everyone | Bubble with the players page link. |
| `!bonk @user` | | mods, broadcaster, or a reward | A cartoon mallet bonks the target, who falls over with circling stars until they type `!getup` (or after 60 s). Light-hearted: no blood and no damage effects. |

- Commands are case-insensitive and ignore extra text after the command.
- Viewers who haven't chatted yet are spawned by their first command.
- Settings can turn any command off.

## 9. Other Twitch moments

- **Raid:** the raiding streamer beams in as a 2× size character at the front, then their viewers march in from the darkness behind as generic characters wearing the raider's colour (capped at 150, or the number of free spots). Banner: "RAID — {name} and {count} raiders". The director cuts to them.
- **Follow:** the new follower drops in from above into a free spot with a small dust burst.
- **Sub / resub:** a crown appears on their name tag for the session, and their light turns gold for 10 s.
- **Gift subs:** small gift boxes rain on the recipients, or on the gifter if recipients aren't known.
- **Cheer:** fireworks above the cheerer, scaled to the bit amount.
- **Channel-point rewards:** the streamer maps reward titles to actions in the setup panel: spotlight (follows the redeemer for 20 s), confetti, bonk, All aboard, or none.

## 10. Lighting presets

Each preset sets exposure, bloom (strength, radius, threshold), the grade (contrast, saturation, hue), vignette, environment-light colours, spotlight colour and a subtle flicker:

- **Noir** (default for the void): warm-white spots, deep blacks, strong vignette.
- **Golden hour:** amber spots, soft bloom.
- **Neon nebula:** violet and cyan spots, high bloom.
- **Red alert:** red spots, slow pulse.
- **Clear:** neutral, low vignette.

The city has its own preset, **Neon night**, and can switch to Storm (heavy fog, rain on) or Dawn (pink sky, windows fading out).

## 11. Setup panel

Hidden by default so it never shows in OBS. `` ` `` (backtick) or `H` toggles it. It's a dark glass side panel with collapsible sections:

1. **Channel:** channel name field, Connect / Reconnect Twitch, connection status lights (chat, EventSub, token expiry).
2. **Look:** lighting preset, background (black / green / blue / magenta key), capture mode, render scale and fps readout, backend (WebGPU or WebGL 2), post-pass toggles.
3. **Crowd:** every slider from section 5, layout picker, max characters, lurkers on/off, name tag and bubble settings.
4. **Music:** BPM, tap tempo, dance energy.
5. **Rewards:** after login, the channel's custom rewards (from `GET /helix/channel_points/custom_rewards`) each with an action dropdown.
6. **Commands:** on/off per command, cooldowns, subs-only option for `!become`, and the ignore list.
7. **Test:** buttons that fire fake events through the bus:
   - Add 1 / 10 / 100 fake chatters
   - Random chat
   - Every command
   - Raid of 40
   - Follow, sub, gift 5, cheer 500
   - Each mapped reward
   - **Full Hype Train** (begin → five level-ups over 2 minutes → end)
   - All aboard

   Everything must be testable without a Twitch account.
8. **Settings:** "Copy settings JSON" and "Paste settings JSON" (so I can tune and share a look without screenshots), "Copy launch URL" (all non-default settings in the query string, never the token), and "Reset to defaults".
9. **Credits:** "Characters and robot by TC Poole · animations from Adobe Mixamo · built with OpenAI Codex", and a link to the players page.

All settings persist in `localStorage` (try/catch), and URL parameters override them. URL parameters: `channel`, `preset`, `layout`, `max`, `bpm`, `lurkers`, `bg` (`black`/`green`/`blue`/`magenta`), `capture`, `webgl`, `debug`, `players` (players page URL), `scene` (`void`/`city`, to preview the city directly).

## 12. Performance targets

- 150 characters in the void at 60 fps on an RTX 3060-class GPU at 1080p. I stream from an RTX 3080, and judges may be on less.
- The city with a full 8-car train at 60 fps on the same GPU.
- No frame over 50 ms when a new character arrives (thanks to the shader warm-up and cached models).
- Memory stays flat over a 4-hour stream: dispose geometry, materials, textures and mixers when characters leave, and pool bubble and name-tag canvases.

## 13. Acceptance checks

- Opens with `?channel=tc_poole` and no login: chatters appear, commands work, and there are no console errors.
- Login works, the token survives a reload, and EventSub reconnects after a 30-second network drop.
- Every Test panel button produces the right result.
- A full simulated Hype Train plays start to finish, then returns to the void with Hype Rider rings.
- WebGL 2 fallback (`?webgl=1`) runs everything, perhaps with fewer post effects.
- Opened from tcpoole.com in Chrome with `?capture=1` and captured with OBS Window Capture: a steady 1920×1080 picture, no panel or cursor visible, and it keeps rendering while other windows are on top of it.
- `?bg=green` with an OBS chroma-key filter leaves a clean cut-out, with no green fringe from bloom or fog.

## 14. Build stages

Do one stage per task. After each one, list the files you changed, how to test it, and anything you weren't sure about.

1. **Renderer and the void:** renderer with fallback, post pipeline, black void space, host character in its spotlight, show cam, fly cam, setup panel shell, settings save/load.
2. **Characters:** roster loading, normalising, clip roles, procedural idle and gestures, shader spotlights with beams and pools, beam-in and beam-out, crowd layout. Test buttons for fake chatters.
3. **Twitch chat:** IRC, event bus, name tags, bubbles with emotes, all chat commands including `!become` with cooldowns and saved picks, lurkers.
4. **Host mode:** WoW-style controls with locomotion, jump and crouch. Double-click glide, saved cameras, director.
5. **Twitch login and EventSub:** implicit grant, validate, subscriptions, reconnect. Raid, follow, sub, gift, cheer and reward moments. Reward mapping UI.
6. **The city:** towers, windows, neon, billboards, sky, street, traffic, track, station.
7. **The Hype Train:** locomotive, observation cars, seats, riders, full sequence, HUD, city director, All aboard. Full-train test button.
8. **Polish:** performance pass against section 12, dynamic resolution, memory checks, lighting presets, README with setup steps, controls, commands and credits.
