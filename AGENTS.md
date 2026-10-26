# AGENTS.md — house rules for Chat Crowd

**BUILD-PLAN.md decides what to build and in what order.** Build only Core items, one stage at a time. Never build Stretch items unless asked by name, and never build Cut items.

Read SPEC.md (the overlay), SPEC-tuning.md (tuning rules), SPEC-host.md (the streamer's robot) and SPEC-players.md (the players page) before every task.

**Every tunable number comes from `tuning.json`** (see SPEC-tuning.md), never hard-coded in a feature. If you add a setting, add its key to `tuning.json` too. Where `tuning.json` and a spec disagree, `tuning.json` wins. They are the source of truth.

## Code

- Each page is one self-contained HTML file with inline JS and CSS: `overlay.html` and `players.html`. Shared data lives in `roster.json`. No build step, no bundler, no npm.
- Inside each HTML file, keep code in clearly fenced sections (`// ===== NAME =====`). Each section talks to the others only through the event bus and the shared `app` object.
- three.js is pinned to 0.186.1 through the import map in SPEC.md. Don't change the version unless asked.
- WebGPU first, with the WebGL 2 backend as automatic fallback. Every feature must work on both, even if post effects are reduced on WebGL 2.
- No secrets in code. The only token is the streamer's, stored in `localStorage` after login. Never put it in a URL, on screen or in the console.
- The pages are hosted on tcpoole.com and run in Chrome, captured by OBS with Window Capture. They don't need to work as an OBS browser source.
- Network calls only to Twitch (`irc-ws.chat.twitch.tv`, `eventsub.wss.twitch.tv`, `id.twitch.tv`, `api.twitch.tv`, `static-cdn.jtvnw.net`), the pinned CDN, Google Fonts, and the model bucket in `roster.json`.
- Wrap every `localStorage` / `sessionStorage` / IndexedDB call in try/catch. The page must work when storage is blocked.

## Assets

- Characters come only from `roster.json`. Don't hand-edit its URLs.
- The train (engine, cars, seats) is built in code. Don't load the backup Sketchfab train unless I ask by name. If I do, its CC-BY credit must go in the README and the setup panel.
- Anything else visual (city, track, cars, effects) is built in code.

## Working style

- Build one stage at a time, as listed in SPEC.md. Stop after each stage.
- After each task, list the files and sections you changed, how to test the change, and anything in the spec that was unclear.
- Every feature needs a Test panel button so it can be triggered without Twitch.
- Nothing appears on the overlay except the scene, name tags, bubbles, event effects and the Hype Train HUD. Debug info goes in the console (`?debug=1`) or the setup panel.
- If something in the spec seems wrong or impossible, say so and propose an alternative instead of guessing.

## Testing

- Check the console for errors on every page before calling a task done.
- Test with `?webgl=1` as well as WebGPU.
- Keep an eye on frame time. If a change costs more than 1 ms per frame, say so.
