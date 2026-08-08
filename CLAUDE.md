# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page web tool (`index.html`) that generates boarding-pass barcodes (Aztec or PDF417) encoded per the IATA BCBP standard, entirely client-side. There is no server component and no package.json — this is a static site plus a C barcode library compiled to WebAssembly.

Live version: https://shooshx.github.io/BoardingBarcode

## Architecture

Two layers, glued together at the JS/WASM boundary:

1. **`index.html`** — all UI and BCBP formatting logic lives inline in `<script>` tags.
   - Form fields (name, booking ref, from/to airport, flight operator/number, date, class, seat, sequence number) are each formatted/padded (`fldDef` in the inline script defines width per field) and assembled into a single BCBP `M1...` string in `format()`.
   - `format()` calls `update()`, which invokes the compiled `run(type, data, ecc, symsize)` function (bound via `Module.cwrap`) to render the barcode as SVG, then draws it onto `<canvas id="theimg3">` via an `<img>`/`FileReader` data-URL round trip.
   - URL query params can pre-fill fields on load (see `queryFieldMap`/`applyQueryParams()` in `index.html`), e.g. `index.html?from=AMS&to=SFO`.
   - `codeTypeSel` switches between Aztec (type 0) and PDF417 (type 1); symbol size/ECC inputs map to `zint`'s `option_1`/`option_2`.

2. **`src/*.c`, `src/*.h`** — a subset of the [zint](https://github.com/zint/zint) barcode library (Aztec, PDF417, Reed-Solomon, SVG rendering) plus `src/main.cpp`, which is the app-specific entry point:
   - `run()` (exported to JS as `_run` via Emscripten) builds a `zint_symbol`, encodes the data, and prints SVG output which is captured into a global `std::string g_buf` via a custom `buf_printf` stdout shim, then handed to JS as `textOut` via `EM_ASM_`.
   - `escape_char_process()` handles `\n`, `\t`, etc. escape sequences in the raw input before encoding.

3. **`js_qr_brd.js`** — the Emscripten-generated glue/runtime, checked into the repo (large, single-line, minified, with the `.wasm` binary embedded as base64 — there is no separate `.wasm` file). This is a **build artifact**, not hand-written; regenerate it with the build scripts below rather than editing it directly.

## Building

Requires Emscripten (`emcc`) with the `%EMSCRIPTEN%` environment variable set to its install path. Run from the repo root (Windows `.bat` scripts):

- `em_debug_build.bat` — debug build (`-O0`, assertions, safe heap on)
- `em_rel_build.bat` — release build (`-O3`)

Both compile `src/aztec.c src/common.c src/library.c src/reedsol.c src/render.c src/svg.c src/main.cpp src/pdf417.c` into `js_qr_brd.html`/`js_qr_brd.js`, exporting only `_run`. In practice only the resulting `js_qr_brd.js` is kept/committed and referenced from `index.html` via `<script src="js_qr_brd.js">`.

There is also a Visual Studio solution (`qt_brd.sln` / `qt_brd.vcxproj`) that builds the same C/C++ sources as a native Windows executable (`main()` under `#ifdef WIN32` in `src/main.cpp`) — useful for debugging the barcode-generation logic natively without going through Emscripten.

No JS build step, package manager, linter, or test suite is present — `index.html` is opened directly (or served statically) to run the app.
