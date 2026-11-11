# AGENTS.md

Instructions for AI coding agents working on `6666666666` (`app.mrlunev.6666666666`), a JavaScript application for the BUSY Bar. See [README.md](./README.md) for the full build, packaging and manifest reference.

The app runs on the bar itself, in the firmware's embedded [JerryScript](https://compat-table.github.io/compat-table/es2016plus/#jerryscript3_0_0) runtime. It is not Node.js and not a browser.

<!-- busy-emulator:begin -->
## No BUSY Bar needed: this project runs on BUSY Bar Emulator

BUSY Bar Emulator created this project. It runs the bar's firmware on this computer and has its own Node, npm and busy-cli, so use its command line for everything: do not run `npm`, `npx`, `node` or `tsc` yourself, and do not look for a bar on USB. Where the rest of this file or `README.md` says `npm run …`, use the command below.

`busy-emulator` below is the command line's full path: `~/.busy-emulator/bin/busy-emulator` on macOS, `%LOCALAPPDATA%\busy-emulator\bin\busy-emulator.exe` on Windows. Run each command on its own, not joined with others. Every command takes `--json`; a failure has a `code`, a `message` and a `hint` that names the next command. The emulator is installed, so none of this needs a question to the person: build, run, look at the screen, fix.

| To | Run in this folder |
| --- | --- |
| Build, install and start the app after an edit | `busy-emulator dev --once` (exit code 0: the app runs) |
| Only build; only install and start | `busy-emulator build`; `busy-emulator run` |
| Check the types | `busy-emulator typecheck` |
| Read the app's console: `console.log`, errors, why it ended | `busy-emulator logs` (this app's lines; `busy-emulator logs --since-run`: its latest launch only; an `emulator:` line is the emulator's note on an unhandled rejection or an API call the bar refused) |
| See the screen | `busy-emulator screenshot --scale 8`, then read `screenshots/front-<n>@8x.png`; the back display: `busy-emulator screenshot --back --scale 4` |
| Make a short video of the app for a post | `busy-emulator record` (5 s of the emulator's window, `media/clip-<n>.mp4`; `--layout post` the bar alone, 1080×1920; `--gif` a GIF of the front display) |
| Press a button or turn the wheel | `busy-emulator press ok` (also `start`, `back`, `up`, `down`; `--times 3`; `--hold 800` for a long press) |
| Let time pass: a timer, a countdown, the hour | `busy-emulator time advance 25m` (the app's timers due in between fire now); `busy-emulator time set 14:59:50`; `busy-emulator time reset` |
| Read or change the app's settings | `busy-emulator settings get`; `busy-emulator settings set hour24=false --restart` |
| Stop the app | `busy-emulator stop` |
| Make the app's icons | `busy-emulator icon --text AB` (one or two letters) or `busy-emulator icon --from picture.png` |
| Check the app against the app catalog's rules | `busy-emulator check` (`--json` names only what needs something); `busy-emulator submit` then writes the files for the catalog's pull request. The license is the author's choice: ask them, do not pick one |

### The bar's API in short

- Draw: `await device.DisplayDraw({ application_name: manifest.id, priority: 50, elements })` (`device` from `@shared/device`). The front display is 72×16 in colour; `display: "back"` puts an element on the back one, 160×80 in grey, whose right 12 px are the firmware's status bar. Every element has an `id` (drawing an `id` again replaces it; its `type` cannot change), `x`, `y`, `align` (`top_left`, `center`, `bottom_right` and the like) and `z_index` (0 or more; higher is on top). Colours are `#RRGGBBAA`. `device.DisplayClear({ application_name: manifest.id })` takes them all away.
- Elements: `{ id, type: "text", text, font, color, x, y, align }` (ASCII; other letters may come out as a box); `{ id, type: "rectangle", x, y, width, height, fill: "solid", fill_colors: ["#FF0000FF"], border_width: 0 }`; `{ id, type: "countdown", timestamp: String(unixSeconds), direction: "time_left", show_hours: "when_non_zero" }`, which counts by itself.
- Fonts, by these exact names, with their line height in px: `tiny` 6, `small` 9, `normal` 11, `condensed` 11, `bold` 11, `large` 13, `extra_large` 14 (capitals and digits only), `global` 11, `superscript` 11.
- Layout (`render()`, `measure()`, `row`, `column`, `stack` from `@busy-app/busy-lib`) needs each font's metrics, and the build bundles them only for a font written as a string literal, `font: "bold"`, in a file under `src/`. A font from a variable, the short form `{ font }`, or a name only in `shared/` builds and then fails as the app runs: `font "large" has no metrics loaded`. For such a font add, once at the top of `src/main.ts`, only for the fonts it lays out (each takes about 9 KiB of the app's 128 KiB heap, `global` about 60 KiB; registering every font runs out of memory): `import { registerFontMaps } from "@busy-app/busy-lib"; import { large } from "@busy-app/busy-lib/fonts"; registerFontMaps(large);`. Elements placed by `x`, `y` and `align` without `render()` need no metrics. `busy-emulator build` warns about the cases it sees.
- Input: `listen("input", (event) => …)` returns the function that detaches it. `ok`, `start`, `back`: `press`, `short`, `release`; held: `press`, `long`, `repeat` again and again, `release`. The wheel: `{ key: "encoder", action: "clockwise", delta: 1 }`, or `"counterclockwise"` and `-1`. A short Back should end the app, as the template's does.
- Timers and time: `setTimeout`, `setInterval`, `Date.now()` (the bar's clock); `busy-emulator time advance` moves all three, not the network.
- `fetch`: `res.ok`, `res.status`, `res.json()`, `res.text()`; a 404 resolves. A failure rejects with a string, not an `Error`: `"socket error"`, or `"Inactivity timeout"` after 5 s without data, before the answer or in its middle. There is no `TextDecoder` or `TextEncoder`. A request to this computer (`127.0.0.1`) is refused unless allowed: `busy-emulator allow-local 8080`.
- Storage: `localStorage.getItem(key)` and `setItem(key, text)` keep strings across launches; a missing key gives `null`. The first run logs `[JsLocalStorage] Failed to open file`: harmless.
- Catch every promise and `console.error` what it rejects with: a rejection nobody catches shows only as the emulator's `emulator:` note in `busy-emulator logs`.

`not_running`: run `busy-emulator start` first. `untrusted_project` (a folder the emulator did not create): run `busy-emulator trust <folder>`, which asks the person in a dialog. Another project is made with `busy-emulator create <new-folder-name>`: a folder that does not exist yet, not `.` in a folder with files.

### Settings

`src/appmeta/settings.json` describes the app's settings; the bar draws their screen and their web page itself. `busy-emulator build` checks the file and names the field at fault. The smallest valid file:

```json
{"format_version":1,"version":1,"fields":{"hour24":{"type":"boolean","label":"24-hour clock","default":true}}}
```

`fields` is a map keyed by each field's id (lower-case letters, digits, `_`), not a list. Every field has `type`, `label` and `default`. The types: `boolean`; `integer` (optional `min`, `max`, `step`); `string` (optional `max_length`, 64 unless given); `enum` (`"options": [{ "value": "a", "label": "A" }]`, `default` is one of the values); `color` (`"#RRGGBB"`); `time` (`"HH:MM"`); `geolocation`; `group` (its own `fields`). Raise `version` to drop the values kept on the bar.

Read the values with `loadValues()` from `@shared/settings` (this project's copy asks the bar's API, `GET /api/apps/settings`):

```ts
import { loadValues } from "@shared/settings";

const values = (await loadValues()) ?? {};
const hour24 = values.hour24 !== false;
```

The bar answers with every field: the saved value, or the file's `default`. Read the settings when the app starts; after `busy-emulator settings set` start the app again (`--restart` does). An app cannot write its settings.
<!-- busy-emulator:end -->

## Firmware API boundary

The BUSY Bar firmware is an external dependency of this app.

- Never modify or patch BUSY Bar firmware source code as part of work on this project. Treat the firmware repository as read-only reference material.
- If the required capability is not available in the target firmware, use a supported fallback or degrade gracefully. If neither is possible, report the missing capability and required API version; do not implement a firmware change.
- Before treating a capability as missing, check the target device at `http://<device-host>/docs` and `GET /api/version`, or inspect the matching firmware OpenAPI definitions.
- If the target firmware lacks a required capability, use a supported fallback, degrade gracefully, or report the required API version. Do not invent an endpoint or silently depend on unreleased firmware behavior.

## Entry point and lifecycle

`src/main.ts` or `src/main.js` is the entry point. The build emits it as `scripts/main.js`, the file the firmware starts. It is a script, not a module: the firmware runs it from top to bottom once, it must not export anything, and it starts the app by calling `main()` at the end.

**`main()` is not run in an infinite loop.** It is a one-time launch. The firmware does not keep the app alive because `main()` was called, and it does not restart it when `main()` returns. The app stays alive only while something the script started is still pending:

- an input listener registered with `listen("input", ...)`;
- a pending `setTimeout` or a running `setInterval`;
- a `fetch` request that has not settled yet.

When `main()` returns, the app keeps running while at least one of those remains. Once the last listener is unbound, every interval is cleared, every timeout is cleared or has fired, and every outstanding `fetch` has settled, the JavaScript process exits and the app closes. The bar takes the screen back.

That makes cleanup part of the app design, not an afterthought:

- Keep the reason the app stays alive explicit: one listener or timer per thing the app is doing.
- To quit, release everything: clear every timer, unbind every listener, and let outstanding fetches settle. There is no exit function — releasing the last handle is the exit.
- Do the same cleanup on every path that ends a mode or the app, not only in the `back` handler.
- Do not unbind the last handler from inside that handler's own callback: the process would stop while the callback is still running. Schedule it instead, as the template does with `setTimeout(unbind, 0)`.
- `listen()` accepts one handler at a time and throws if a handler is already attached. Unbind the old one before attaching another.
- Keep `main()` synchronous and start asynchronous work from it explicitly. Give every promise a rejection handler (`void work().catch(report)` or `try`/`catch`); an unhandled rejection must not be the only error path.
- Do not keep the app open with a dummy long-running timer "just in case", and never busy-wait to stay alive.

The shape of a minimal app:

```ts
function main(): void {
  const timer = setInterval(() => {
    // redraw
  }, 1000);

  const unbind = listen("input", (event) => {
    if (event.key !== "back" || event.action !== "short") return;

    clearInterval(timer);
    // The unbind is the last handle: schedule it so this callback returns first.
    setTimeout(unbind, 0);
  });
}

main();
```

## Conserve device resources

The BUSY Bar is a small embedded device. The JavaScript runtime, the web server that serves the bar's HTTP API, and the rest of the firmware share the same CPU and memory. An app that is greedy slows down the whole bar, not just itself.

### Memory

- `heap_size_kib` in `src/appmeta/manifest.json` is the entire heap for the app: 1–512 KiB, 128 KiB by default. Keep the default unless measurements show the app needs more; a larger heap takes memory from other subsystems. Prefer reducing allocations over raising the heap.
- Avoid large buffers, long-lived caches, and unbounded arrays. Keep only the state the app needs and what is currently on screen.
- Send display elements from the data you already have instead of keeping parallel copies of every frame.
- Parse only the fields you need from responses; do not retain whole payloads.
- Logging costs memory and time. Do not log inside tight loops or per-frame redraws.

### Requests and timers

- Every API call is an HTTP request handled by the device's web server. It costs CPU and memory on the bar and can be slow. Poll on human time scales (seconds or minutes), never per frame and never in a loop.
- Keep at most one display request in flight. When one is still unanswered, skip or coalesce the next update rather than queueing it.
- Do not retry immediately on failure. Use a bounded backoff, and stop retrying while the device reports that it is offline or still connecting.
- Send only what changed. Elements are addressed by `id`, so update an element instead of redrawing the whole screen; clear removed ids once with `DisplayClear` instead of resending an empty frame.
- Run timers only while something needs updating, and clear them when the app is idle. Do not use `setInterval(..., 0)` or a `while` loop as a wait.
- Prefer coarse, aligned timers: redraw on the minute rather than every 100 ms, and refresh remote data every few minutes rather than every few seconds.
- Package images, animations and sounds with the app instead of downloading them at runtime, and keep them small. Do not decode large files into memory.

When in doubt, choose the cheaper option: a slower refresh, fewer requests, less data on screen.

## Display, input and settings

- The screen is 72×16 pixels. Build a tree with `row`, `column` and `stack` from `@busy-app/busy-lib`, render it with `render()`, and send the elements with `device.DisplayDraw({ application_name: APP, priority: 50, elements })`. Use `manifest.id` as `APP`.
- Give every element a stable `id`. The device replaces elements with the same id, so only changed elements need to be sent.
- Remove elements that disappear with `device.DisplayClear({ application_name: APP, element_ids: [...] })`.
- `priority` is 1–100. Keep it at 50 unless the app deliberately needs to take the screen from another one.
- Colors are `#RRGGBBAA`.
- `listen("input", handler)` delivers `ok`, `start` and `back` with a `press`, `release`, `short`, `long` or `repeat` action, and `encoder` with a direction and a `delta` of `1` or `-1`. It returns an unbind function.
- **Use `short` as the default button action, not `press`.** `short` fires once, after a normal press and release; `press` fires immediately on key down and is followed by `short` on release, so treating `press` as a tap reacts before the gesture is complete and can double-fire. Use `long` and `repeat` for hold gestures, and `press`/`release` only when the down and up edges matter.
- Settings are described in `src/appmeta/settings.json`. Read them with `loadValues()` from `@shared/settings` (the format and an example are in the emulator's section above); it returns `null` only when the app has no settings file or the bar did not answer. An app cannot save its settings. Do not put secrets in settings files.

## Code rules

- The runtime is not Node.js and not a browser: no `fs`, `path`, `process`, `document` or `window`. `console`, `fetch` with `Request`/`Response`/`Headers`/`URL`, timers, `localStorage`, `Date`, `JSON` and standard ECMAScript are available.
- `localStorage` persists small values per app, but every `setItem` rewrites the whole store to flash. Keep values small and writes rare; use `@shared/settings` for user-visible settings.
- `src/` is the app and the only folder the build scans. `shared/` holds app-level helpers imported as `@shared/*`. The entry point must stay named `main.ts` or `main.js`.
- TypeScript is strict when the app is written in `.ts`; run `busy-emulator typecheck`. The entry point may be plain JavaScript (`src/main.js`) instead.
- Third-party packages must run in this runtime: no Node built-ins and no DOM. Prefer `@busy-app/busy-lib` and the existing `@shared/*` helpers over adding dependencies.
- Never edit `dist/`; it is generated by `busy-emulator build`.
- Report failures with `console.error` and a stable app prefix, and keep one failed request from stopping timers or input handling.
- Put resources under `src/images`, `src/animations` and `src/sounds` in the formats the bar reads (`.png`/`.image`, `.anim`, `.snd`). Other formats are packaged into `resources/` and cannot be displayed by the bar.
- Keep this file and `README.md` accurate when conventions change.

## Before finishing

Run `busy-emulator typecheck` and `busy-emulator build`, then check:

- [ ] The app quits cleanly when `back` (or the app's quit action) is used: no timer or listener outlives it.
- [ ] No path leaks an interval, timeout or listener; errors do not leave timers running.
- [ ] Memory use is bounded: no unbounded caches, buffers or per-frame copies.
- [ ] Button activation uses `short`, not `press`; holds use `long` and `repeat`.
- [ ] Requests are infrequent and do not pile up; failures back off instead of retrying in a tight loop.
- [ ] Frames send only changed elements and are redrawn only when something visible changes.
- [ ] `src/appmeta/manifest.json` has a valid `id`, `name` and semver `version`, and `api_version` is no higher than needed.
