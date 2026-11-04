# PdfCraft — instructions for agents

PdfCraft is a clean-room, open-source, Rust-native PDF application targeting Adobe Acrobat Pro parity. It runs natively on macOS, Windows, Linux and FreeBSD, and on the web via WASM. It is the sibling of `../photocraft` (a Photoshop-class editor) and follows the same conventions.

## This fork

This is `earnesthudson/pdfcraft`, the owner's own fork of `storytold/pdfcraft`. The owner's own rules for agents come first; where this file differs, follow the owner.

- Work only on what the owner asks. Don't choose work by yourself from any list or plan.
- Ask before building or running this code, and before anything the request doesn't cover.
- Changes reach `main` by a pull request against this fork. `main` takes no direct pushes.
- Don't open issues or pull requests on `storytold` repositories, and don't fetch code from them.
- Upstream code arrives only when the owner asks, through `fork-guard.sh sync` and its security check.
- The `storytold` repositories this project builds from have not all been verified. Don't add more of them.

## Non-negotiables
- **Assets: read `AGENTS.md` §1 before adding or showing any icon, image, font or document.** No assets from Adobe products, ever. Only openly licensed or contributor-original assets are allowed, each with an entry in `ATTRIBUTION.toml`. `cargo xtask assets` enforces this. `AGENTS.md` overrides this file.
- **Fonts live in [storytold/craft-fonts](https://github.com/storytold/craft-fonts).** Never commit font files here (`AGENTS.md` §1.4). It is the optional build input `CRAFT_FONTS_DIR`: `CRAFT_FONTS_DIR=<checkout>`, with a checkout the owner has approved, embeds the Japanese fonts (UI fallback, Japanese text in edited PDFs) and runs their tests, which otherwise skip. Code using `pdfcraft_fonts::CRAFT_FONTS` must work when it is empty.
- **Clean-room.**
  - Never read, disassemble or copy anything inside the Acrobat bundle (names and listings only). **Never open `Contents/Resources/JavaScripts/`.**
  - Behaviour comes from public docs, specs (ISO 32000-2, the Arlington model) and black-box observation (`plan/acrobat/`).
  - Never copy GPL/AGPL code. MuPDF, Ghostscript, Poppler, veraPDF and DSS run only as external oracle processes.
  - See `plan/README.md` §Clean-room and `plan/adr/0001`.
- **Privacy.** When observing Acrobat, use synthetic fixtures only. Never capture the Home view, recent files or account info. Capture by window id (`plan/acrobat/tools/`). Never commit Acrobat outputs, corpus files or personal data.
- **Layering.** Nothing below L7 depends on egui/winit/eframe/rfd (`plan/architecture.md` §3). `cos`/`filters`/`crypt`/`arlington` stay standalone.
- **Page input.** Page modules read `PagePointer` (`crates/ui-egui/src/page_pointer.rs`), never egui's raw pointer state (`hover_pos`, `interact_pos`, `latest_pos`, `press_origin`, button reads); `page_pointer::tests::page_modules_read_the_pointer_only_through_page_pointer` enforces it.
- **Page cursor.** Page modules claim the cursor for what is under the pointer (`view.cursor.claim(Layer::…, icon)`, `crates/ui-egui/src/canvas/page_cursor.rs`), never set it (`set_cursor_icon`, `cursor_icon =`, `on_hover_cursor`); `document_area` sets it once, from the claim on the highest layer the active tool reaches. `page_pointer::tests::page_modules_claim_the_cursor_and_never_set_it` enforces it.
- **Page presses.** A page module decides what of its own is under a point in one hit test, which returns a `Hit` (owner, layer, cursor, tip; `crates/ui-egui/src/canvas/page_router.rs`) and gives its cursor, its tip and its press. It acts on a press only when the press is its own or, for the active tool, nobody's: the page view tells it which (`PagePress`: its own, nobody's, or another page's), and it never tests the press point again or asks another module whether it wants it. Escape cancels any press the pages hold (`cancel_on_escape`), and that press is ignored until the button is up. `page_pointer::tests::page_modules_act_on_a_press_only_through_the_router` enforces it.
- **Fidelity.** The PDF object graph is the model. Preserve unknown data. Saves are incremental unless a full rewrite is required. Never silently drop or repair data without recording it.
- **Never panic.** Every PDF, script, MCP call, settings file and keystroke is untrusted input, and none of it may crash the app or lose the user's work. This outranks feature work. Return `Result` (the crate's error enum) and propagate with `?`, or fall back leniently and record it. In non-test code:
  - No `unwrap`/`expect`/`panic!`/`unreachable!`/`todo!` unless it is provably infallible, with a comment saying why.
  - No indexing or slicing with input-derived positions (use `get`, and slice strings only at char boundaries).
  - Use checked or saturating arithmetic on input-derived numbers, and guard against division by zero and NaN/inf casts.
  - Cap allocations sized by input, and bound recursion with depth limits or seen-sets.
  - Handle lock poisoning.
  - No `unsafe` (`unsafe_code = "forbid"`).
  - Every crash fix gets a synthetic regression test. See `AGENTS.md` §4.
- **Rust only** in the product and build (`xtask`). No handwritten JS/TS.
- **Time goes through `chrono`** (workspace dependency). Never hand-roll UTC/calendar conversion (days-to-date algorithms, leap-year math); a crate owns only its output format string. The one exception is range extension: past the years chrono can represent, shift the date by whole 400-year Gregorian cycles (146,097 days, after which the calendar repeats), let chrono compute the month and day, and add the cycles back to the year (splitting seconds into hours, minutes and seconds is plain arithmetic).
- **Review every update as you make it, at four zooms.** Not an end-of-task pass — a per-update habit, so a defect never gets built on. **Clause:** checked arithmetic, `get()` not `[i]`, char-boundary slicing, the right error through `?`, `<` against `<=`, the 1-based/0-based seam, capped allocations. **Function:** is it total — name an input that panics; walk empty, zero, max, malformed, cyclic; is the return type honest. **File/module:** two sources of truth, dead or duplicated code, is the public surface still smallest and the doc comment still true. **Project:** layering, wasm, undo/redo, save fidelity, encryption and permissions, a matching `automation` tool or registry entry, and whether a `parity/` status or note just became untrue. Scale the zoom to the change, and fix what a level turns up before widening.
  - Tests mirror the zooms: property → unit → crate integration → end-to-end.
  - **Prove a test fails before the fix by actually reverting the fix.** A test nobody has seen fail is not evidence. This has caught two worthless tests in this repo: one confinement test passed with path resolution switched off entirely, because the tools failed on a nonexistent document id before reaching the path.
  - **Rebasing is a review step, not a mechanical one.** Read every hunk you resolve, and the auto-merged hunks in the files you touched too — `main` moves fast enough that a file can change by hundreds of lines under a branch.
- **External PRs are held to safety, security and completeness before style.** Provenance first (no Adobe-derived asset, no committed font, no copied GPL/AGPL code, behaviour from the spec and not from reading another implementation), then no new panic path or `unsafe`, then `--root` confinement on every path including ones reached by nested dispatch, opt-in-only MCP and control channel, no new outbound network, then a headless `automation` tool, tests at the right zoom and an honest `parity/` entry. Provenance is the part a contributor cannot be expected to know, and the only part that cannot be undone after merge.
- **Quality gates** before every commit: `cargo fmt --check`, clippy with `-D warnings`, the tests, `cargo xtask layers`, `cargo xtask assets`, and the wasm check once `xtask ci` exists (M0).
  - New workspace crates must include `[lints] workspace = true` in `Cargo.toml` to inherit `[workspace.lints]`. Production crate roots must declare the required crate-level lint attributes before modules or other crate items; crate-level `//!` documentation may precede them.
  - **Scope builds and tests to the crates you touched** — `cargo clippy -p <crate> -j 2 --all-targets` and `cargo test -p <crate> -j 2`, under a `timeout`. A full `--workspace` build has locked up a development machine's CPU; it is CI's job and CI runs it on every PR. Run the workspace sweep locally only when you have a reason to and the machine is free.
- **Commits and PRs:** `main` is never pushed to directly. Branch from `main`, commit only green states, and open a pull request against `main` of this fork.

## Running and looking at the app
- `cargo run -p pdfcraft -- <file.pdf>` opens the desktop app.
- `cargo run -p pdfcraft-cli -- run --script steps.json --root DIR` drives the engine headlessly through the automation tools (`pdfcraft-cli tools` lists them). Use it, together with `page_render`, to check engine changes. `pdfcraft-cli mcp` is the opt-in MCP server (AGENTS.md §3).
- `cargo xtask fuzz --time 300` mutation-fuzzes open/render/edit/save in child processes. Findings land in `fuzz-out/findings/` (git-ignored; never commit corpus-derived files). Turn every real finding into a small synthetic regression test before fixing it.
- `cargo xtask parity [--partial]` reports Acrobat-parity progress from `parity/acrobat-features.toml`. Update the entry when a feature ships.
- `cargo xtask demo-pdf` builds `dist/demo/pdfcraft-showcase.pdf` (needs Chrome) for visual checks.
- For UI work, **look at the result**. Either launch `pdfcraft --control FILE doc.pdf` and use `pdfcraft-cli ui --control FILE screenshot --out x.png` (plus `inspect`, `click`, `key`, `type`, `command`, `set`), or take a headless shot with `cargo run -p pdfcraft-ui-egui --example shot`. Compare against `plan/acrobat/02-ui-ux.md`. Control-channel tests use kittest (`crates/ui-egui/tests/control.rs`).
- Parallel agents: use a separate `CARGO_TARGET_DIR` per agent and separate git worktrees.

## Current bootstrap debt (tracked in STATUS.md)
- `pdfcraft-render` renders through the `hayro` crate directly and inspects documents through `lopdf`. Both get replaced by `cos` / `model` / the DisplayList device (M1–M2, ADR-0004).

## CI workflows
This fork has no workflows and Actions is off. Don't add any unless the owner asks.
