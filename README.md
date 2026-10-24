# lvx-esp32

A scriptable LVGL firmware for ESP32 touch panels: flash it once, then build the UI in JavaScript files you edit on a PC and hot-reload on the device without recompiling. The firmware embeds a QuickJS-ng engine and a curated LVGL binding surface; the apps are data it loads at boot.

**Hardware.** Everything here is developed and verified on a **Waveshare ESP32-S3-Touch-LCD-1.47** (ESP32-S3R8, 1.47" IPS 172×320, JD9853 controller, AXS5106L touch), which is the only board it has actually run on. That is a limit of what's been tested, not of the code: the runtime needs an ESP32 with PSRAM and any display LVGL can drive, at any resolution, and reaches hardware through three host hooks. What it does and doesn't require is in [`docs/portability.md`](docs/portability.md); this board's pinout and driver notes are in [`docs/hardware/README.md`](docs/hardware/README.md).

## What's here

### [`firmware/`](firmware/README.md) — the platform

Two Arduino libraries plus one sketch per board. [`quickjs-ng/`](firmware/quickjs-ng/README.md) is the vendored JS engine and [`lvgl-js-bindings/`](firmware/lvgl-js-bindings/README.md) is the LVGL binding layer; neither knows anything about a particular panel. [`boards/waveshare-s3-touch-147/`](firmware/boards/waveshare-s3-touch-147/README.md) is the board-specific half: pinout, display bring-up, and the three host hooks. Build and flash with `.\flash.ps1` from the repo root.

### [`app/`](app/) — the launcher and demo apps

`app.js` is the launcher, which lists `apps/*.js` and runs whichever you tap. Of those, `wifi.js` is the network setup screen the firmware opens on its own; `tasks`, `vitals` and `weather` are demos, there to exercise the platform and show what it can do. To build your own, see [Using it for your own project](#using-it-for-your-own-project). Deploy by copying to an SD card and long-pressing BOOT (≥700 ms), or push over serial with `.\push.ps1`. `selftest.js` is the on-hardware test of the binding surface. Script API: [`docs/binding-api.md`](docs/binding-api.md).

Apps can be written two ways. Directly against the bindings, which is what [`docs/binding-api.md`](docs/binding-api.md) documents and what [`app/selftest.js`](app/selftest.js) still does. Or as JSX components with hooks, which is what every shipped app now does: `src/*.jsx` builds to `apps/*.js` with `node tools/build-app.mjs`, using the reconciler in [`app/lib/ui.js`](app/lib/ui.js). That layer is pure JavaScript over the same bindings and adds nothing to the firmware — see [`docs/ui-runtime.md`](docs/ui-runtime.md).

Both remain supported. The pre-port version of each app is frozen in [`tools/fixtures/`](tools/fixtures/) as the baseline a parity test compares against, so the imperative style stays readable side by side with what replaced it.

## Docs

[`docs/`](docs/README.md) is indexed by topic. Starting points: [`docs/architecture.md`](docs/architecture.md) for the shape of things, [`docs/binding-api.md`](docs/binding-api.md) if you are writing an app, [`docs/runtime-architecture.md`](docs/runtime-architecture.md) if you are changing the runtime, and [`docs/hardware/README.md`](docs/hardware/README.md) for the board.

## Quick start

Build/deploy instructions (toolchain setup, the FQBN, the flash scripts, getting `app.js` onto the board) live in [`BUILDING.md`](BUILDING.md). The short version: run `.\flash.ps1` at the repo root. The board shows up as a `USB Serial Device` on the native USB-C port — use a **data** cable.

## Using it for your own project

What you take from this repo is the firmware. The demo apps are examples, not a base you have to build on.

1. **Flash the firmware once**, as in the quick start. On a different board, start from [`docs/portability.md`](docs/portability.md): you write a board sketch, and both libraries come over unchanged.
2. **Write an app** as `app/src/<name>.jsx`. The counter in [`docs/ui-runtime.md`](docs/ui-runtime.md#the-five-minute-version) is a working starting point and [`app/src/tasks.jsx`](app/src/tasks.jsx) is the fuller example. If you'd rather skip JSX, write plain JavaScript straight into `app/apps/<name>.js` against [`docs/binding-api.md`](docs/binding-api.md).
3. **Build and push it:** `node tools/build-app.mjs <name>`, then `.\push.ps1 app\apps\<name>.js`. The board writes it and runs it straight away, so the loop is edit, build, push. Copying `app.js` and `apps/` to an SD card and long-pressing BOOT works too.
4. **Pin it** so the board boots straight into your app with nothing drawn over it: long-press its row in the launcher, or send `pin /apps/<name>.js` over serial. A long-press of BOOT still reaches the launcher. See [Pinning one app](docs/binding-api.md#pinning-one-app).

What you can delete:

| Path | What it is | Keep it? |
|---|---|---|
| `app/app.js` (built from `src/app.jsx`) | The launcher | Yes, unless you write your own. BOOT long-press and the firmware's home button open it. |
| `app/apps/wifi.js` (built from `src/wifi.jsx`) | Wi-Fi setup | If any app uses `fetch()` or `wifi`. The firmware opens it when the network needs a person to fix it. |
| `app/lib/ui.js` | The JSX runtime, bundled into each built app | If you write JSX. |
| `app/apps/{tasks,vitals,weather}.js` and their `src/*.jsx` | Demos | No. |
| `app/selftest.js`, `app/ui-selftest.js`, `app/engine-probe.js` | On-board tests of the bindings, the JSX runtime and the engine | Only if you change the firmware or the runtime. |

Two things to know before deleting a demo. Removing it from the repo does not remove it from a board that already has it; `rm /apps/weather.js` over serial does. And [`tools/test-parity.mjs`](tools/test-parity.mjs) checks each demo against its original in `tools/fixtures/`, so remove a deleted demo's entries there too, or CI fails.
