# Confidence Monitor

An in-studio talent confidence monitor. Shows the OBS **program feed** with local-only overlays — producer notes, a timer, a clock, and a full built-in teleprompter. The overlays render **only on this monitor** and never touch the OBS output, so the broadcast/vdo.ninja feed is unaffected.

Single self-contained HTML file. No build, no server. One dependency: Firebase (Realtime Database), used only for the teleprompter's script library — everything else is `postMessage`/`localStorage` between the two windows on this one machine.

Absorbs the standalone [Prompter](https://github.com/chrisgrimm-jm/prompter) app — its script library, editor, transport, and appearance controls now live directly in this Control panel, and its scroll-and-display logic is native on this Display (no more separate prompter Control/Display windows to babysit alongside OBS and this app).

## Setup (same machine as OBS)
Two ways to get the OBS program feed into the Display. **Option A (OBS Virtual Camera) is the recommended one** — no picker, remembers itself.

**Option A — OBS Virtual Camera (recommended)**
1. In OBS, click **Start Virtual Camera**.
2. Open the [page](https://chrisgrimm-jm.github.io/confidence-monitor/) (**Control**) → **Open Talent Display**.
3. In Control's **Feed** card, choose **OBS Virtual Camera** from the dropdown. It's picked automatically when found; the first time (or if the list is empty) press **Refresh** and allow camera access once. The Display connects to it, and **Reconnect** re-tries the same camera. The card shows the Display's live status ("feed live (camera: …)"). The choice is remembered, so next time the Display opens it connects by itself, and if the camera drops (e.g. the virtual camera was off) it retries every 3 seconds. It only ever connects the camera you chose — it never falls back to another one (so a laptop webcam can't end up on the talent monitor).
4. Drag the Display to the talent's monitor and press **Fullscreen**. Drive everything from Control.

This is a camera, not a screen capture, so there's no mirror-loop risk and nothing to keep windowed or hidden. The overlays (teleprompter, notes, timer, clock) are drawn only in this page, never fed back into OBS, so anything else using the same virtual camera (Zoom, vdo.ninja) sees only the clean program video.

**Option B — window capture (fallback, no extra software)**
1. Open **Control** → **Open Talent Display**.
2. In OBS: right-click the program preview → **Windowed Projector (Program)**.
3. In the Display window's toolbar: click **Capture Feed** and pick that projector window (pick **Window**, not **Screen** — see note below). Browsers make you re-pick every time the Display reloads. Drag the Display to the talent's monitor and press **Fullscreen**.

For Option B, the OBS Windowed Projector can sit anywhere (even hidden); `getDisplayMedia` captures its pixels directly. You put the **Display window** on the talent monitor, not the projector — even if the Display window ends up completely covering the projector on the same monitor, window capture keeps working since it grabs that window's own render buffer, not whatever's on top of it. Capturing **Entire Screen** instead of a window will feed back into itself if the Display is fullscreen on that same monitor, so always pick the specific window.

If the capture picker only shows one window to choose from, that's almost always a macOS permission issue, not the app: grant the browser **Screen Recording** access in System Settings → Privacy & Security, then fully quit and relaunch the browser (not just reload the page).

Each overlay has a **SHOW/HIDE** button and, where relevant, a position/mode dropdown — all in the card header on the Control panel. Producer Note, Timer, and Clock also have a **Size** field for font size in px.

## Overlays
- **Producer note** — a text aside for the talent; 9-point placement (corners / edges / center); adjustable size.
- **Timer** — countdown or count-up; 9-point placement; turns red under 10s; adjustable size.
- **Clock** — wall-clock time of day, top-right; adjustable size.
- **Teleprompter** — built in. Script library (drag the ⠿ handle on any read to reorder it; the order is saved and shared — Companion's read buttons follow it too; paste ad copy straight from a Google Doc — colors/highlights carry over, or link a published Doc for auto-refresh), rich-text editor with trim points, transport (play/pause/scrub/nudge), and appearance (font size, line spacing, margins, theme, mirror/flip, reading line) — all in the Control panel's Teleprompter card. Synced to the Display via Firebase under a **Topic** (default `adread`; change it to run a different session — building/editing the library ahead of time from any device still works, same as the standalone app did). Modes: Full / Top band / Bottom band. Can be triggered externally — see **Companion / hardware triggers** below. Every appearance/transport setting — scroll speed, font size, line spacing, margins, theme, mirror/flip, reading line, and even scrub position — is saved **per read**, not globally: whatever the sliders show when you hit **Save to library** is what that read restores when it next goes live, so each sponsor read can have its own pacing/look.

## Control page: Talent Display, layout, and overlay positions
- **Talent Display card** — a green dot and line show whether the Display window is open (it sends a heartbeat every 2 seconds) with its size and whether it's fullscreen. **HIDE / SHOW** blanks the whole Display (feed, overlays, teleprompter) the same way the other cards' buttons work, and it's available from Companion too.
- **Test pattern** — color bars with the studio name, shown whenever the Display is hidden *or* has no feed, so a blank talent monitor is never ambiguous. Type the **Studio name** in the Talent Display card (saved per computer). In the desktop app it defaults to the Windows user name (each studio's PC is logged in as that studio), and the pattern also shows the computer name and LAN IP — a web page can't read those, so that line is desktop-app only.
- **Edit layout** (top right) — every panel gets a bar: drag it to reorder (the teleprompter's panels can move between their two columns, the cards within the right column), **Hide / Show** removes a panel from view without disabling anything. **Done** locks it in; **Reset layout** restores the default. Saved per computer.
- **Overlay Positions** — a mini talent screen: drag the producer note, timer or clock anywhere. The 9-spot dropdowns still work (picking one clears the custom spot; **RESET** clears all three). Dragging doesn't write to Firebase — only show/hide flags do.

## Companion / hardware triggers
A read can be put live from outside the browser — a Bitfocus Companion button, a Stream Deck, anything that can fire an HTTP request — by writing directly to the same Firebase Realtime Database the app already uses (open/unauthenticated, same as every other read/write this app does; no server of its own to run).

For Companion specifically, there's also a real module — [confidence-monitor-companion](https://github.com/chrisgrimm-jm/confidence-monitor-companion) — with a dropdown of read names (no typing exact names into every button), a feedback that colors a button while its read is live, and variables. The raw HTTP approach below still works fine and needs no module install.

In Companion, add a **Generic → HTTP Request** action per button:
- Method: `PATCH`
- URL: `https://pinpoint-abf21-default-rtdb.firebaseio.com/prompter/<topic>/trigger.json` (use the Topic shown in the Teleprompter card, `adread` by default)
- Header: `Content-Type: application/json`
- Body: `{"name":"<exact read name>","n":{".sv":"timestamp"}}`

The name is matched case-insensitively against the Script Library. `n` must be Firebase's server-timestamp placeholder (`{".sv":"timestamp"}`), not a fixed number — otherwise pressing the same button twice in a row won't fire the second time, since the app only reacts when the value increases. A name that doesn't match anything currently in the library shows an error in the Edit-read message area rather than silently doing nothing.

Show/hide toggles and timer transport go through a separate queue (the Companion module's presets already wire these up, so this is only needed for the raw-HTTP approach). Unlike the trigger endpoint above, this one is a real queue — each command is a **new child**, not an overwrite of one value — so that two commands for different elements landing close together (e.g. show the teleprompter, then hide the timer, a moment apart) both take effect instead of the later one silently winning:

- Method: `POST` (creates a new child — a plain `PATCH`/`PUT` to a fixed URL would go back to the one-value-wins-the-race problem this is built to avoid)
- URL: `https://pinpoint-abf21-default-rtdb.firebaseio.com/prompter/<topic>/uiq.json`
- Header: `Content-Type: application/json`
- Show/hide an overlay: `{"t":"show","key":"<key>","on":true}` — `key` is one of `promptShow` (teleprompter), `prodShow` (producer note), `timerShow`, `clockShow`.
- Toggle an overlay: `{"t":"toggle","key":"<key>"}` — flips whatever it's currently set to, same `key` values as above.
- Timer transport: `{"t":"timer","op":"start"}` — `op` is `start`, `pause`, or `reset`.

Confidence Monitor applies each queued command and deletes it immediately, so the queue stays effectively empty in normal operation — there's nothing to clean up.

Note this requires the Control page itself to be open in a browser tab — it's the one listening on Firebase and re-applying the change locally (the same way it already relays state to the Display), not something the Display or a server does on its own.

Control also mirrors the show/hide flags (teleprompter, producer note, timer, clock, and the Display itself — `displayShow`) **out** to `prompter/<topic>/uistate` (`{"promptShow":bool,"prodShow":bool,"timerShow":bool,"clockShow":bool}`) every time any of them changes, from any source (a button click in Control, or one of the commands above) — this is what lets Companion's "element is shown" feedback color a button correctly. Nothing else about the app's state is mirrored there.

## How it syncs
Control and Display run on the same machine. Control opens the Display and sends the whole state over `postMessage` on every change; the Display is a pure renderer. Control also persists to `localStorage`, so a refresh keeps your setup. The program feed is either the OBS Virtual Camera (`getUserMedia`) or a window capture (`getDisplayMedia`), set up in the Display.

The teleprompter is the exception: script library, active content, playback settings, and transport commands all flow over **Firebase** (shared `pinpoint-abf21` project, under `prompter/{topic}`), the same as the standalone app did — Control writes, Display reads, independent of `postMessage`. The Topic string and the SHOW/HIDE + Full/Top/Bottom mode still travel over the regular `postMessage`/`localStorage` state between Control and its Display, since those are this app's own layout concerns — but Control also listens on Firebase (`prompter/{topic}/ui`) for remote show/hide and timer commands (from Companion or a raw HTTP request), and re-applies them locally through that same `postMessage`/`localStorage` path, exactly as if you'd clicked the button yourself.

---
Jomboy Media · hosted on GitHub Pages
