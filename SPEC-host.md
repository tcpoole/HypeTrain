# SPEC-host.md — The streamer's robot (webcam head + black suit)

Add the streamer's own avatar to Chat Crowd. **This replaces "The host" in SPEC.md section 5, "Host mode" in section 6, and Stage 4.** Everything else in SPEC.md stays.

The host is a **robot head on a black bodysuit**. The hollow robot head has a screen inside its visor showing **the streamer's live webcam**. Streamers want their face on stream, and this puts it inside the show instead of in a box in the corner. The head follows the streamer's real head movement through face tracking. When the camera is off, or the streamer wants privacy, the screen switches to an animated robot face driven by the same tracking. The body is a rigged suit with real Mixamo animations for walking, running, gestures and dancing. The host stands front and centre in the void, rides at the front of the Hype Train, and is driven with the WoW-style controls.

The page runs in Chrome on tcpoole.com (HTTPS), so camera, mic and speech recognition all work normally. It does not need to run as an OBS browser source.

## 1. Assets (all in `roster.json` → `host`)

| Part | File | Notes |
| --- | --- | --- |
| Body | `tccharacter/riggedtcblackundersuit.glb` | 0.7 MB. Black bodysuit, one skinned mesh, Mixamo skeleton (74 joints, `mixamorig:` names), Draco + WebP. Includes `Walking` and `Running` clips. |
| Head | `funkbot2-head.glb` | 4 MB. Hollow robot helmet, one rigid mesh, Draco + WebP. The webcam screen goes inside it. |
| Animations | `tccharacter/tc-host-anims.glb` | 2.8 MB. Animation-only (skeletons plus 45 clips, no meshes), Mixamo skeleton. |

The 45 clips in `tc-host-anims.glb`:

| Group | Clips |
| --- | --- |
| Gestures | `Acknowledging`, `Angry_Gesture`, `Annoyed_Head_Shake`, `Being_Cocky`, `Dismissing_Gesture`, `Happy_Hand_Gesture`, `Hard_Head_Nod`, `Head_Nod_Yes`, `Lengthy_Head_Nod`, `Look_Away_Gesture`, `Relieved_Sigh`, `Sarcastic_Head_Nod`, `Shaking_Head_No`, `Thoughtful_Head_Shake`, `Shrugging`, `Pointing`, `Laughing`, `Rude`, `Salute`, `Waving`, `Poolespellcastsimple` |
| Idles | `Weight_Shift`, `Arm_Stretching`, `Dwarf_Idle`, `Sitting_Idle` |
| Locomotion | `Walking_Backwards`, `Running_Backward`, `Running` (a newer run, preferred over the suit's own), `Run_To_Stop`, `Strafe_Walk_L/R`, `Strafe_Run_L/R`, `Turn_L/R`, `Standing_To_Crouched`, `Crouching_Idle`, `Crouched_Walking` |
| One-shot moves | `Jumping_Up`, `Falling_Idle`, `Falling_To_Landing`, `Falling_To_Roll`, `Front_Flip`, `Running_Slide`, `Backflip` |

## 2. Retargeting (important)

The clips use the Mixamo skeleton, but **their rest pose and units differ from the suit's**. For example, the clips are in centimetres with a 0.01 scale, and the head and arm rest rotations differ. Playing them directly will twist the limbs.

- **Match bones by name** after stripping the `mixamorig` / `mixamorig:` prefix. Some files use the colon and some don't.
- **Rotations:** retarget each clip once at load. For every bone and keyframe, take the clip's rotation relative to the clip's own rest pose in world space, then apply that same world-space change to the suit's rest pose and convert back to the suit's local space. Cache the retargeted clips.
- **Hips position:** keep only the vertical bob, scaled by the ratio of hip heights (suit hips rest at about 1.08). Strip horizontal travel so every clip plays in place. The code moves the root for locomotion and for travelling moves (`Front_Flip`, `Running_Slide`, `Falling_To_Roll`), at speeds measured from the original clip's hip travel.
- Add a debug view behind `?debug=1` that shows the skeleton and lets me play any clip, so retargeting problems are easy to spot.

## 3. Assembly

- **Suit:** load it and scale so the host stands 1.45× a chatter's height (adjustable). Keep its material black with a slight sheen: roughness about 0.45 and a faint rim from the environment so the silhouette reads against the black void. The setup panel has controls for surface detail (normal strength), roughness and brightness.
- **Head:** parent the helmet to the suit's `Head` bone. Default head size is 0.72 of the helmet's natural size relative to the suit's shoulders, with position, rotation and size offsets in the setup panel. The helmet must fully hide the suit's own head.
- **Head follows the body:** gestures like `Head_Nod_Yes` and `Shaking_Head_No` move the `Head` bone, so the helmet nods and shakes with them. Face tracking (section 5) layers on top.
- **Fit editor** in a new "Host" section of the setup panel: body size and offset, head size/offset/rotation, suit look, with Copy fit JSON, Paste fit JSON and Reset. The fit saves with the other settings.

## 4. The webcam screen

- **Camera:** `getUserMedia({ video: { width: 1280, height: 720, frameRate: 30 } })`. The setup panel has a device dropdown (`enumerateDevices`), Start/Stop camera, and a status line.
- **Video texture:** a `VideoTexture` that updates only when a new frame arrives (`requestVideoFrameCallback`).
- **Screen:** a quad inside the helmet, about 0.46 forward of the head centre, sized about 1.15 × 0.86 of the helmet width. It must have:
  - **Centre-crop, never squash.** The video keeps its aspect ratio and fills the screen.
  - Zoom (1–3×) and pan X/Y.
  - **Face-centred framing (default on):** use the face-tracking landmarks to keep the streamer's face centred and sized in the screen as they lean around. Smooth it (default 0.8) so it never jitters. Setting: how much space around the face (default 2.4× the face height).
  - Mirror toggle (default on).
  - Rounded-corner mask (0 = square, 1 = oval, default 0.85) plus four corner offsets so the screen can be shaped to sit exactly in the visor opening.
  - **Unlit and emissive,** so the face stays readable in any lighting, with brightness and saturation controls. A slight bloom on the screen edges.
- **Visor glass:** a thin curved glass layer just in front of the screen with fresnel edge reflections from the environment. A "clear visor" option makes it nearly invisible.
- **Light spill:** the screen lights the inside rim of the helmet with one small `PointLight` (short range) whose colour and intensity follow the webcam's average colour. Get that colour by drawing the video element into an 8×8 canvas 10 times a second. Never read back from a WebGPU canvas.
- **VHS look (optional, off by default):** grain, scanlines, colour bleed, tape jitter, tracking wobble, vignette and desaturation, each adjustable. Applies to the screen only.

## 5. Face tracking

- **MediaPipe Tasks Vision `FaceLandmarker`** running in a **Web Worker**, not on the main thread. Pin a specific version from jsDelivr, with `outputFaceBlendshapes: true` and `outputFacialTransformationMatrixes: true`. Send frames to the worker as `ImageBitmap`s at up to 30 fps.
- **Head pose:** yaw, pitch and roll from the transformation matrix are added on top of the animation on the `Head` bone (clamped to ±45° yaw, ±30° pitch, ±25° roll). The `Neck` takes 40% of it and `Spine2` 15%, so it looks natural. Smooth with a One Euro filter (adjustable).
- **Lean:** the face's position in the camera frame shifts the upper spine slightly in that direction.
- **Calibrate** button: stores the current pose as neutral.
- If tracking loses the face for over 1 s, the head eases back to the animation's pose and resumes when the face returns.

## 6. Robot face mode

- **B** toggles between webcam and robot face. The setup panel has the same toggle. The switch plays a 1.1 s scanline-wipe reveal.
- The robot face is drawn on the same screen with the same rounded mask: two glowing eyes, eyebrows and a mouth in cyan line art with bloom.
- It's driven by the blendshapes: `eyeBlinkLeft/Right` and `eyeSquintLeft/Right` for the eyes; `browInnerUp`, `browOuterUpLeft/Right` and `browDownLeft/Right` for the brows; `jawOpen`, `mouthSmileLeft/Right` and `mouthFrownLeft/Right` for the mouth; and `mouthPucker` and `mouthFunnel` for an O-shaped mouth.
- It has a gain per feature group, expression smoothing (default 0.08 s), and a "Calibrate neutral face" button.
- **It switches on automatically** if the camera is denied, fails or is stopped, so the host never shows a black screen. With no camera at all, it idles: slow blinks, glances, and the mouth moving with the mic.
- **N** is a privacy hotkey: instantly robot face, camera paused, until pressed again.

## 7. Mic, talking gestures and voice triggers

- Optional mic input (Start mic button, device dropdown), analysed only and never played back or recorded.
- An RMS envelope with a noise gate (adjustable, default level 0.16) drives:
  - **Talk glow:** the screen edge brightens while talking.
  - **Robot-face mouth** when there are no blendshapes.
  - **Talking gestures:** while the streamer talks, every few seconds (default gap 3 s) the body plays a gesture from this list, blended over the idle at 0.75 strength: `Happy_Hand_Gesture`, `Dismissing_Gesture`, `Being_Cocky`, `Acknowledging`, `Thoughtful_Head_Shake`, `Look_Away_Gesture`, `Lengthy_Head_Nod`, `Weight_Shift`. Gestures only affect the upper body by default, so they work while walking (a "gestures move the legs too" option turns that off).
- **Voice triggers (Chrome speech recognition, off by default):** the streamer says a phrase and the body plays an animation. Editable in the setup panel as `phrases => Clip`, one per line. Defaults:

  ```
  hello, hi there, hey everybody, welcome => Waving
  yes, yeah, exactly, correct => Head_Nod_Yes
  no way, nope, nah => Shaking_Head_No
  seriously, come on => Annoyed_Head_Shake
  whatever => Dismissing_Gesture
  phew, finally => Relieved_Sigh
  let's go, awesome, amazing => Happy_Hand_Gesture
  i don't know, dunno, no idea, beats me => Shrugging
  hmm => Thoughtful_Head_Shake
  salute => Salute
  ```

  Language setting defaults to en-US. Recognition restarts itself when Chrome stops it.

## 8. Moving the host (host mode, key `G`)

It uses the same WoW-style controls as SPEC.md section 6, now playing real clips.

| Input | Clip |
| --- | --- |
| Standing | `Weight_Shift` and `Arm_Stretching` mixed with a procedural breathing layer (`Dwarf_Idle` is excluded from the idle rotation) |
| W | `Walking` (suit) |
| W + Shift, or both mouse buttons | `Running` (pack) |
| S | `Walking_Backwards`, or `Running_Backward` with Shift |
| Q / E, or A / D with right mouse held | `Strafe_Walk_L/R`, or `Strafe_Run_L/R` with Shift |
| A / D standing still | `Turn_L/R` while the body rotates |
| Release after running | `Run_To_Stop` |
| Space | `Jumping_Up` → `Falling_Idle` while airborne → `Falling_To_Landing` |
| C | `Standing_To_Crouched` → `Crouching_Idle`; moving while crouched = `Crouched_Walking` |
| Alt+F | `Front_Flip` (travels forward) |
| Alt+S while running | `Running_Slide` (travels) |

- Locomotion blends in 0.25 s. Speeds and turning come from SPEC.md (walk 1.4, run 4.2, turn 180°/s), with walk and run timescales matched so the feet don't slide.
- The camera swings behind the host while it moves.
- **Gesture hotkeys (editable in the setup panel):**

  | Key | Clip |
  | --- | --- |
  | Shift+1 | `Waving` |
  | Shift+2 | `Head_Nod_Yes` |
  | Shift+3 | `Shaking_Head_No` |
  | Shift+4 | `Happy_Hand_Gesture` |
  | Shift+5 | `Dismissing_Gesture` |
  | Shift+6 | `Being_Cocky` |
  | Shift+7 | `Relieved_Sigh` |
  | Shift+8 | `Angry_Gesture` |
  | Shift+9 | `Salute` |
  | Shift+0 | Stop |

  The setup panel also has a button for every clip in the pack.
- **Dancing:** **Alt+D** cycles through the dance clips in the shared animation library (SPEC-tuning.md section 4), time-scaled to the BPM. Press it again for the next dance, and Shift+0 stops.
- **Hotkeys, speeds, gesture strength, voice triggers and all fit values come from `tuning.json`** (SPEC-tuning.md). The tables above are fallbacks.
- The host stays within the 60-unit radius and passes through the crowd without colliding (crowd members lean aside slightly as it passes).

## 9. The host in the show

- **Void:** front and centre in its larger, warmer spotlight, with the crowd behind. The show cam frames it.
- **Director shots added:**
  - Face-screen close-up with the crowd's lights out of focus behind.
  - Slow orbit around the host.
  - Over-the-shoulder shot from behind the host looking at the crowd.
  - "Host reacts" close-up when a big event happens (raid, Hype Train, reward). The host also plays `Waving` or `Happy_Hand_Gesture` automatically.
- **Hype Train:** the host rides a conductor's platform at the front of the first observation car, standing, facing forward, and swaying with the train on curves. During boarding, it waves (`Waving`) at each new rider. Director shots add a close-up of the face screen with city lights streaking past, and a front shot of the train with the host leading it.
- **Raids:** the host turns to face the incoming raiders and plays `Salute`.
- **Hype Rider:** the host's screen frame gets the same golden ring as riders after a train.
- **Other characters:** a setup option lets the streamer use a roster character instead of the robot (with no webcam), for streamers who'd rather not appear.

## 10. Chrome and OBS setup (for the README)

- Open the overlay on tcpoole.com in Chrome, allow the camera and mic, and bring it into OBS with **Window Capture** (Chrome window, "Windows 10 (1903 and up)" capture method).
- Launch Chrome with `--disable-backgrounding-occluded-windows --disable-renderer-backgrounding`, and never minimise the window, or it stops rendering.
- **A webcam usually can't be open in two apps at once on Windows.** If OBS already has a Video Capture source for the same webcam, remove it. The robot *is* the facecam now.
- Webcam frames, mic audio and speech never leave the page, except that Chrome's speech recognition may use Google's servers. Say that next to the voice-trigger toggle.

## 11. Performance

- The host adds at most 1.5 ms per frame on an RTX 3060-class GPU, not counting face tracking, which runs in the worker.
- The video texture uploads only on new camera frames.
- Robot-face drawing happens on a 512×512 canvas texture, redrawn only when the expression changes.
- Retargeted clips are computed once and cached.

## 12. Test panel additions

- **Fake webcam:** use a looping test video or animated test pattern as the camera, so the screen, framing and VHS can be tested without a camera.
- **Fake expressions:** sliders for each blendshape group to preview the robot face.
- **Fake talking:** a simulated mic envelope to test talking gestures.
- **Fake voice trigger:** a text box that fires a phrase as if it were spoken.
- A button per clip.

## 13. Build stages (replace Stage 4 in SPEC.md)

- **Stage 4a:** load the suit, head and pack; retargeting with the debug skeleton view; head on the `Head` bone; fit editor; webcam screen with crop/framing/mask/visor/spill; robot face mode; privacy key; fake webcam.
- **Stage 4b:** face tracking in a worker, head pose and lean, blendshape-driven robot face, mic, talking gestures, voice triggers.
- **Stage 4c:** host mode with clip-based locomotion, jump, crouch, flips, gesture hotkeys, procedural dance, double-click glide, saved cameras, director (including the new host shots).
- **Stage 7 addition:** the host on the conductor's platform during the Hype Train.
