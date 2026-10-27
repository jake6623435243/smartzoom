# SmartZoom

A camera app for iPhone that follows the subject you tap and zooms automatically. It zooms in as the subject moves away and out as it comes closer.

It's a web app (PWA). You put it online once, open it in Safari and add it to your Home Screen. After that it opens full screen like any other app. **You only need your phone**: no Mac, Xcode or App Store. The camera feed is processed on the phone and never uploaded.

---

## What's new in version 2

- **Library:** every photo and video is saved inside the app on your phone. Tap the thumbnail to open the library (All / Videos / Photos), rewatch videos full screen, swipe between them, and Save to Photos, Share or Delete. Captures are kept after you close the app.
- **Save to Photos straight away:** after each shot, the confirmation has a **Save to Photos** button.
- **Better recognition:** while following, the AI looks at a zoomed-in crop around the subject, so far-away subjects are found far more reliably. Tapping a subject that's too small to detect makes it look closer around your finger. A more accurate AI model is used automatically when your phone is fast enough (Settings › AI model).
- **Better at staying on target:** it remembers each subject's top and bottom colours separately, and keeps several looks of them (turned around, in shade). It won't jump to someone walking in front of the subject, and it checks that position, size and movement make sense. After a lighting change it re-learns the subject's look, but only once it's confident.
- **Smoother zoom:** zoom eases into and out of movement. It ignores size wobble that's just measurement noise, and ignores sizes from partly hidden or doubtful sightings.
- **Easier to use:** quick zoom buttons (1×, 2×, 3×, 5×…) that glide smoothly, a **Resume auto zoom** button after you zoom by hand, double-tap anywhere to release the subject, a red frame while recording, an optional framing grid, grouped settings and one-time tips.

## Put it on your iPhone (phone only, about 10 minutes, free)

You need to host the 6 app files at an `https://` address once. Safari only gives web pages camera access over https. These steps use GitHub Pages, which is free, and everything can be done in Safari on the phone.

1. **Unzip the app.** Save `SmartZoom-app.zip` to the Files app, then tap it. Files unzips it into a folder with `index.html`, `sw.js`, `manifest.webmanifest`, three `.png` icons and this README.
2. **Create a GitHub account** at [github.com](https://github.com) in Safari, if you don't have one.
3. **Create a repository.** Tap **+** (or your avatar menu), then **New repository**. Name it `smartzoom`, choose **Public**, and tap **Create repository**.
4. **Upload the files.** On the new repository's page, tap the **uploading an existing file** link. Tap **choose your files**, open the unzipped folder, select all the files, then tap **Commit changes**.
5. **Turn on GitHub Pages.** In the repository, open **Settings**, then **Pages**. Under *Build and deployment*, set **Source** to *Deploy from a branch* and **Branch** to `main` with folder `/ (root)`. Tap **Save**.
   *If a button isn't visible on the mobile layout, tap **aA** in Safari's address bar and choose **Request Desktop Website**.*
6. **Wait one or two minutes**, then open `https://YOUR-USERNAME.github.io/smartzoom/` in Safari.
7. **Install it.** Tap the **Share** button, then **Add to Home Screen**. Open SmartZoom from the Home Screen icon.
8. **First launch needs internet once.** It downloads the AI library and model, about 15 MB. After that it works offline.

Any other static web host with https works too. Upload the same files.

**Requirements:** iOS 17 or later for real camera zoom; older versions fall back to digital zoom, labelled as such. Newer iPhones run the AI faster, so tracking is smoother.

**Try it without walking around:** open `https://YOUR-USERNAME.github.io/smartzoom/?demo`. A rendered scene with people, a dog and a car replaces the camera, so you can see how tracking and auto zoom behave.

---

## Using it

| | |
|---|---|
| **Follow a subject** | Tap a person, animal or vehicle. Faint corner marks show what the AI can see. |
| **Tracking box** | Amber = following. Pulsing blue = looking for it again. Coral = lost; it keeps searching, or you can tap the subject again. |
| **Release** | Stops following. Tapping another subject switches to it. |
| **AI tracking** | Turns detection on or off. Off saves battery. |
| **Auto zoom** | Turns automatic zoom on or off. Tracking continues when auto zoom is off. |
| **Zoom scale** | Drag the scale or pinch the picture to zoom manually. Auto zoom pauses for 2.5 s, then resumes. The amber marker on the scale shows where auto zoom is heading. |
| **Photo / Video** | Tracking and auto zoom keep running while you shoot or record. |
| **Thumbnail** | Opens the library: everything you've shot, newest first. Tap a video to rewatch it; swipe for the next one. **Save to Photos**, **Share** and **Delete** are at the bottom. |
| **Quick zoom** | Tap 1×, 2×, 3×… to glide to that zoom. While following, auto zoom pauses for a few seconds after you zoom by hand; **Resume auto zoom** hands control back at once. |
| **Double-tap** | Anywhere on the picture releases the subject. |
| **Settings** | Subject size (small/medium/large), zoom speed, tracking sensitivity, auto zoom, maximum auto zoom, preferred camera, video quality and frame rate (only what your camera supports), battery saver. |

The status pill at the top always says what's happening. For example: "Following the person — 3.4×, zooming in, moving away" or "Looking for the dog — Widening the view to find it".

---

## Why a web app, and what that means

You asked for something that needs only a phone. On iPhone, installing a native app requires a Mac with Xcode, or the App Store. A web app installed from Safari is the only route that needs nothing else. Safari now exposes enough to build this properly:

- **Real zoom, not a crop.** Since Safari 17 (iOS 17), a web page can set the camera's `zoom`. On iPhones with several rear lenses, the default back camera is Apple's multi-lens camera, and iOS switches between the ultra-wide, wide and telephoto lenses by itself as zoom changes. Digital zoom is used only where the hardware has no optical lens left, or as a clearly labelled fallback where a browser doesn't expose zoom at all.
- **On-device AI.** [MediaPipe](https://ai.google.dev/edge/mediapipe/solutions/vision/object_detector/web_js) EfficientDet-Lite0 runs in the browser using the phone's graphics chip, and detects 80 everyday categories (people, dogs, cats, horses, birds, cars, motorbikes, bikes, buses, trucks, common objects).
- **Offline.** A service worker caches the app, AI library and model after the first visit.

Trade-offs compared with a native Swift app are listed under *Known limitations* below.

### Architecture

```
 camera.js ─ getUserMedia stream, hardware zoom (applyConstraints) or digital fallback,
     │        front/back, interruptions (calls), stream restarts
     ▼
 detector.js ─ MediaPipe object detection on the live <video>, plus a colour signature
     │          (hue/saturation histogram) for each detection
     ▼
 core.js ─ SubjectTracker: keeps hold of ONE subject across frames
     │      AutoZoomController: turns the subject's size/position into a smooth zoom path
     │      MotionEstimator: getting closer / moving away
     ▼
 app.js ─ render loop, overlay, tap/pinch, zoom scale, status, settings, lifecycle
 capture.js ─ photos (frame grab) and video (MediaRecorder), save via share sheet
 demo.js ─ rendered test scene + simulated detector (?demo), used by the browser tests
```

`core.js` has no browser dependencies, so the exact same code runs in the app and in the Node test suite.

### How the tracking works

- **Same group only.** A tracked person can never become a car. Label flips within a group (car ↔ truck) are tolerated.
- **Motion model.** Position and size are predicted between frames, so the box moves smoothly even when the detector skips a frame.
- **Zoom-independent coordinates.** Tracking happens in coordinates that undo the camera's zoom, so the app's own zooming never looks like the subject moving.
- **Look-alikes.** Each candidate is scored on overlap with the prediction, colour similarity and distance. The colour test adapts to the scene: a candidate whose colours are clearly worse than this subject's usual match is rejected unless it overlaps almost exactly. This stops it jumping to someone in a similar jacket who walks in front.
- **When crowded,** a neighbour's box can't drag the motion estimate along. If two candidates are equally plausible, the tracker waits a frame instead of guessing.
- **Losing and finding.** After 0.6 s unseen the state becomes *Looking for*, and it searches a widening area for a strong colour match confirmed on consecutive frames. After 4 s it becomes *Lost*, and it searches the whole frame with a stricter colour test.

### How the auto zoom works

1. **Apparent size.** It measures the subject's size as a fraction of the frame, divided by the zoom it was captured at. That gives a value that doesn't change when the camera zooms, so the lag between asking for zoom and getting it can't cause oscillation.
2. **Outlier rejection.** A 5-sample median plus light smoothing ignores one-off detector glitches. Boxes touching the frame edge are cut off, so they're only allowed to make it zoom *out*.
3. **Target zoom.** Target zoom = chosen subject size ÷ apparent size.
4. **Framing limit.** Zoom can't pan, so it never zooms so far that the subject would leave the frame. It accounts for where the subject has been moving over the last 1.5 s, and zooms out urgently if the subject reaches an edge.
5. **Deadband.** Small size wobble (breathing, arm swing) is ignored. A steadily walking subject is followed continuously.
6. **Smooth motion.** A critically damped spring in log-zoom space eases in and out with no overshoot, so 1→2× feels the same as 4→8×. It has a speed limit (Gentle/Normal/Quick), and zooming out may be 1.5× faster than zooming in. Feed-forward from the subject's size trend removes the lag when someone walks steadily towards or away from you.
7. **Subject lost.** It holds zoom for 1.2 s, then gently zooms out to give the tracker a wider view to find the subject again.
8. **Limits.** It always stays within the device's real zoom range and your *Maximum auto zoom* setting.

### Distance estimation

Browsers expose no LiDAR, depth or calibrated focus data. Distance changes are estimated from apparent size: for a rigid subject, distance ∝ 1 / apparent size. For people, height is used, because swinging arms and legs make width noisy. A slope over the last ~1.2 s gives the *getting closer* / *moving away* readout and steers the zoom feed-forward. The app does not show distances in metres, because it can't measure them reliably.

---

## Testing

There was no iPhone available while building this, so **nothing below was run on a phone**. All three suites are simulated.

| Suite | What it runs | Result |
|---|---|---|
| `tests/unit.mjs` | Unit tests of the core: geometry, colour signatures, tap selection, zoom compensation, limits, manual pause | 10/10 pass |
| `tests/scenarios.mjs` | **Simulated world.** The real `core.js` with a camera whose zoom lags 80 ms, a detector with 60 ms latency, jitter, misses (more when small, dark or hidden behind someone) and false positives. Covers your 10 cases. Repeated over 11 random seeds. | 32/32 checks pass on every seed |
| `tests/e2e.mjs` | **Headless Chromium at iPhone size** running the built app: demo scene end to end, plus the real camera code path using Chromium's fake camera | 25/25 pass |

Scenario results (seed 0):

| # | Case | Result |
|---|---|---|
| 1 | Person walks towards camera (15 m → 1.5 m) | Tracked 100%; median framing error 11%; zooms out to 1×; max zoom change 2.3% per frame |
| 2 | Person walks away (3 m → 19 m) | Tracked 100%; framing error 4%; zooms in to 6.6× without hunting |
| 3 | Person moves side to side | Tracked 100%; subject clipped 0% of frames (zoom leaves room for the movement) |
| 4 | Car approaching fast, detector flickering car/truck | Tracked 100%; clipped 0% |
| 5 | Four people, including a similar-coloured person crossing in front and an identically dressed person nearby | Tracked 100%; locked onto someone else 0% |
| 6 | Subject hidden 1.6 s behind an obstacle | Shows *Looking for*, reacquired 0.2 s after reappearing, no jump to the other person |
| 7 | Low light (40% missed detections, noisy colours) | Tracked 100%; zoom stayed smooth, no hunting |
| 8 | Phone with only 1–2× zoom | Never exceeds 2×; sits at the limit without hunting |
| 9 | Switching rear → front camera | Zoom stays within the new range; tracking resumes after re-selecting |
| 10 | Recording while auto zoom is active | Max zoom change 2.2% per frame (no visible jumps); framing error 11% while the subject keeps moving |

The browser test (demo scene) followed the subject 100% of the time over 24 s as it walked between 2 m and 14 m. Auto zoom went from 1× when the walker was near to about 4.5× when far, and it never switched to the crossing walker. It also checked photo and video capture while tracking, settings, manual-zoom pause, Release, camera switching, releasing the camera in the background and restarting it, the first-launch explanation, the denied-permission screen, and behaviour when the AI model can't download.

Bugs found and fixed through these tests:
- Subject pushed out of frame when moving sideways. Fixed with motion-aware framing.
- Tracker switching to a similar-coloured person who walked in front. Fixed with the adaptive colour test and crowd damping.
- Steady ~20% zoom lag behind an approaching subject. Fixed with feed-forward.
- The search zoom-out repeatedly halving the zoom.
- The status showing "zooming out" during a manual-zoom pause.

Run them yourself (on a computer, optional): `npm install`, `npm test`, `npm run test:e2e`.

---

## Version 2 benchmark (simulated)

`tests/benchmark.mjs` is a harder simulation than the scenarios above. Its detector misses subjects that are only a few pixels tall, and colour signatures come from rendered pixels with lighting and noise. Averages over 20 runs, version 1 → version 2:

| Scenario | On target | On the wrong person | Time to lock | Peak zoom acceleration |
|---|---|---|---|---|
| Far subject (60 m → 20 m) | 94.5% → 99.9% | 0% → 0% | 2.34 s → 0.02 s | 10.9 → 4.7 |
| Busy street, 6 people incl. look-alikes | 64.0% → 99.0% | 35.0% → 0.6% | — | 8.6 → 4.8 |
| Hidden 3 s, look-alike waiting nearby | 96.9% → 98.4% | 0.1% → 0% | — | 6.3 → 4.7 |
| Sun to shade, then dusk, with a passer-by | 90.4% → 82.6% | 8.0% → 0% | — | 7.6 → 4.8 |
| Same shirt, different trousers, crossing | 100% → 100% | 0% → 0% | — | 8.0 → 4.7 |
| Car approaching fast | 100% → 100% | 0% → 0% | 0.53 s → 0 s | 7.1 → 4.9 |
| Standing still | 100% → 100% | 0% → 0% | — | 10.4 → 4.7 |

One trade-off: in dusk with a similar-looking passer-by, version 2 sometimes says "Looking for…" and waits rather than risk following the wrong person. Lower peak acceleration means gentler starts and stops. These are simulations, not phone tests.

## Known limitations

- **Not yet run on a real iPhone.** The Safari-specific parts (zoom constraint, MediaPipe on the iPhone's graphics chip, MediaRecorder) follow Apple's and Google's documented APIs but haven't been exercised on a device. If something misbehaves on first launch, tell me what you see.
- **Which lens is active isn't visible.** Browsers don't report which physical lens is in use, so the app says "camera zoom", never "optical". When iOS switches lenses you may see a small jump, as in the Camera app. Safari's zoom numbers may not match the Camera app's (on some models Safari's 1× might be the ultra-wide); the app shows exactly what Safari reports.
- **No LiDAR or depth from the web.** Distance is the apparent-size trend described above.
- **Photos are frames from the video stream,** up to the selected video quality (1080p or 4K). They aren't full 12/48 MP stills, and there's no Night mode or HDR processing.
- **Saving to Photos uses the share sheet** ("Save Image" / "Save Video"). Web apps can't write to Photos directly. The in-app library lives in SmartZoom's own storage on the phone. Deleting the app from the Home Screen, or clearing Safari's website data, deletes the library too, so save keepers to Photos. Long 4K videos use a lot of space (roughly 200 MB per minute).
- **Tap only on detected subjects.** The detector knows 80 categories. Very small or distant subjects (under ~5% of the frame) are often not detected until you zoom in manually. It can't follow arbitrary unrecognised objects.
- **Look-alikes.** Re-identification uses colour and position. Two people dressed identically who swap places out of view can be confused.
- **Front camera** usually has no hardware zoom in Safari, so zoom there is digital (labelled).
- **Older iPhones / iOS < 17:** digital zoom only, and detection may run at a few frames per second. Battery saver helps.
- **The camera stops when you leave the app.** This is by design and an iOS rule: it never runs in the background.
- **First launch needs a connection** to download ~15 MB of AI library and model, from jsDelivr and Google's model storage. After that it's fully offline.

---

## Project structure

```
SmartZoomCam/
├── src/                     app source (what runs in the browser)
│   ├── index.html           page, styles, markup
│   ├── app.js               UI, render loop, gestures, settings, lifecycle
│   ├── core.js              tracker, auto-zoom controller, distance trend (no browser APIs)
│   ├── camera.js            camera stream, zoom, interruptions
│   ├── detector.js          MediaPipe detection + colour signatures
│   ├── capture.js           photo, video, saving
│   ├── library.js           in-app library (IndexedDB): save, list, play, delete
│   ├── demo.js              demo scene / test harness (?demo)
│   ├── sw.js                offline cache
│   ├── manifest.webmanifest Home Screen app metadata
│   └── *.png                icons
├── dist/                    built app: the 6 files you upload (all JS inlined into index.html)
├── tools/build.mjs          builds dist/
├── tests/                   unit.mjs, scenarios.mjs, benchmark.mjs (hard mode), e2e.mjs, screenshots/
└── package.json
```

Rebuilding is only needed if you change `src/`: run `node tools/build.mjs` on any computer with Node.js. You don't need to rebuild to install the app.

## Privacy

- **On-device processing.** Camera frames are processed on the phone and never sent anywhere.
- **No servers of its own.** The app has no analytics, accounts or server.
- **One-time downloads.** The only network requests are the one-time downloads of the AI library, model and font.
- **Permissions when needed.** Camera access is requested when you turn the camera on. The microphone is requested only the first time you record a video; if you decline, videos are recorded without sound.
