# SPEC-players.md — Players page (players.html)

Build `players.html` for Chat Crowd: a public gallery where Twitch viewers browse every character in `roster.json`, see them in 3D, and learn exactly what to type in chat to become one. Streamers link it in their panels and chat, and the overlay's `!players` and `!become` bubbles point to it. Read `AGENTS.md`, `SPEC.md` and `roster.json` first.

It must feel like the overlay's void: a black stage, each character standing in its own warm spotlight. It's a showcase, not a database page.

## 1. File and stack

- One self-contained `players.html` (inline JS and CSS) that loads `roster.json` with `fetch`.
- The same pinned import map as the overlay (three@0.186.1, `three/webgpu`, `three/tsl`, addons).
- `WebGPURenderer` with automatic WebGL 2 fallback, `?webgl=1` to force it, and AgX tone mapping.
- `GLTFLoader` + `DRACOLoader`, models cached by URL.
- Reuse the overlay's model normalising and clip-role logic in spirit (same height, feet on the floor, face +Z, same clip-name matching). The code lives in this file.

## 2. URL parameters

- `channel`: shows "Type these in **twitch.tv/{channel}** chat" in the header and links the channel. Without it, the page just says "in chat".
- `pick`: opens that character's detail view on load, by number or code (`?pick=6` or `?pick=grolthak`). That gives shareable links.
- `cmd`: the command shown everywhere, default `!become`. A streamer who renamed it can set `?cmd=!robot`.

## 3. Layout

### Header (sticky, compact)

- Title "Choose your character", one line of how-to: "Type the command in chat and your character changes on stream."
- The rules in small text: one swap every 30 minutes, mods skip the cooldown.
- Search box (matches name, code, epithet, and number).
- Family filter chips: All · Knights of the Tronixia Archive · Robots of the Bureau · Lily Paladins of Lu La Lain · The Dead of Brangus & Grimhollow. Use the labels from `roster.json`.
- A "Surprise me" button that opens a random character.

### Grid

- Responsive: 5 columns on wide screens, 3 on tablets, 2 on phones, with a 16 px gutter on mobile and no sideways scrolling.
- Each card shows:
  - The character's **3D thumbnail** (below) on a black spotlit stage.
  - A big number badge (`#6`).
  - The full name in a fantasy-serif display face, with the epithet underneath in italics.
  - The family as a small coloured tag.
  - Two command chips: `!become 6` and `!become grolthak`. Clicking a chip copies it to the clipboard and shows a "Copied — paste it in chat" toast.
- Hovering a card (or the card being centred on a phone) turns the character slowly and plays its idle.

### Thumbnails without 30 WebGL contexts

Browsers limit live 3D canvases, so **never create a renderer per card.**

- One hidden renderer renders each character once, in a 3/4 view under a spotlight, into a 512×640 image. Cache it in IndexedDB keyed by model URL and roster version.
- Cards show those images, so the grid stays instant on later visits.
- One shared live canvas follows whichever card is hovered and plays the turntable there. Render it with the same renderer, positioned over the card.
- Load models lazily as cards scroll into view (`IntersectionObserver`), at most 3 downloads at a time. Show a soft pulsing silhouette while each loads.

### Detail view (click a card, or `?pick=`)

- A full-screen overlay with a large 3D viewer on the left (on top on phones) and info on the right (below on phones).
- **Viewer:**
  - Black void, warm spotlight from above with a visible beam, rim light, light pool on the floor and a soft contact shadow, all matching the overlay.
  - Bloom on emissive parts.
  - Orbit controls: drag to rotate (horizontal only, plus slight vertical tilt), wheel or pinch to zoom within limits, double-click to reset.
  - Slow auto-rotate that pauses while dragging.
- **Pose buttons** under the viewer, showing only what this model can do: Idle, Walk, Run, Sit, Doze, each Dance clip it has (labelled from its clip names, made readable: `Hip_Hop_Dance` becomes "Hip hop dance"), plus the procedural Wave, Cheer and Salute. Clips crossfade in 0.4 s.
- **Info panel:**
  - Number, full name, epithet, family.
  - Large command block: `!become 6`, with "or" `!become grolthak`, each with a copy button.
  - Family shortcut: "Want any frog? `!become frog`".
  - A small "How it works" box: type it in chat, your character changes on stream, 30-minute cooldown.
- Previous / next arrows (and ← → keys) step through the roster without closing. `Esc` closes. The URL updates to `?pick=` so it can be shared.

### Footer

- Credits line and links read from an optional `site` block in `roster.json` (Twitch, Discord, Patreon). If the block is missing, show nothing.
- "Made with Chat Crowd."

## 4. Look and feel

- Pure black background, warm off-white text, one accent colour (amber, `#ffb36b`) for numbers and command chips.
- A display serif for names and a clean sans-serif for everything else, from Google Fonts.
- Cards have a faint top-down spotlight gradient on their stage area and lift slightly on hover.
- Respect `prefers-reduced-motion`: no auto-rotate, no hover turntable.

## 5. Accessibility and robustness

- Every card is a focusable button, the detail view traps focus while open, and copy buttons have labels.
- If WebGPU and WebGL are both unavailable, show the cached thumbnails (or a name card) and the commands, so the page is still useful.
- If a model fails to load, the card shows the name and commands with a "preview unavailable" note.
- Lighthouse performance score of at least 80 on desktop. First paint under 1.5 s, before any model loads.

## 6. Acceptance checks

- All 31 characters appear, with correct numbers and commands matching `roster.json`.
- Search and family filters work together.
- `?pick=6` opens Grol'thak, and `?channel=tc_poole` shows the channel line.
- Copy buttons copy exactly `!become 6` or `!become grolthak`.
- The second visit loads thumbnails from cache without downloading any model.
- It works on a phone at 390 px wide with no horizontal scroll.
- `?webgl=1` works.
