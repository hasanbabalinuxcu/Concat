# Mobil Çoklu Seçim Toggle'u Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A toggle button (desk tray + phone toolbar) that makes timeline taps additive, so phone users can select clips one by one and merge them.

**Architecture:** One `multi_select: bool` on `TimelinePane` (mirroring the existing `pan_mode` tray-toggle pattern), published to a new `multi-select` property on the `Editor` Slint global. The two Rust gesture handlers (`on_clip_pressed`, `on_band_selected`) OR the flag into their `additive` argument — no changes to `lanes.slint` input handling.

**Tech Stack:** Rust (slint bindings), Slint UI, `scripts/locales.py` for i18n, GitHub Actions `mobile.yml` for the APK.

**Spec:** `docs/superpowers/specs/2026-11-05-mobile-multi-select-toggle-design.md`

## Global Constraints
- Workspace root is `src/` (Cargo.toml lives there), not the repo root.
- Pinned toolchain is 1.93.1 (CI); local cargo is 1.99.1 without rustup — code must compile on 1.93.1 (plain stable Rust, nothing newer).
- Follow the `pan_mode` pattern exactly at every layer; do not invent new plumbing shapes.
- i18n: every new `I18n.t("...")` key must exist in `locales/en.json` or `python3 scripts/locales.py --check` fails CI. Other locales missing the key do NOT fail (only stale keys do).
- CI gates: `cargo fmt --check`, `python3 scripts/locales.py --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo nextest run --workspace`.
- Do not modify `ui/timeline/lanes.slint` (input handling is unchanged; the OR happens in Rust).

## Review Focus
1. **Silent Slint wiring break** — a property/callback declared but not bound through `Editor → seat → pane → tray` compiles fine but the button is dead. The full chain is spelled out in Task 2; verify each layer's two lines exist.
2. **Phone toggle stranded** — the phone toolbar swaps between the no-clip bar and the clip bar; the toggle must be in BOTH (Task 3) or the user cannot turn it off once a clip is selected. Verify both ToolItems exist with `current: Editor.multi-select`.
3. **`locales.py --check` exit 1** — en.json missing `tray.multiSelect` fails CI. Task 4 runs the check.
4. **Toggle ON + tap on empty lane no longer clears selection** — additive band with an empty caught set leaves the selection alone (same as shift-click today). Intended; note it for manual verification.
5. **Toggle ON + trim-handle press on a selected clip** — additive toggles it out of selection, then the trim proceeds (`gestures.rs:40-67`, pre-existing shift behavior). Intended parity with desktop shift; note for manual verification.

---

### Task 1: Rust state, message, publish, and the additive OR

**Files:**
- Modify: `src/crates/concat/src/panes/timeline.rs` (enum :51-93, struct :96-116, Default :118-131, `update` match :136-149)
- Modify: `src/crates/concat/src/lib.rs` (tray handlers :1103-1130, `on_clip_pressed` :1201-1206, `on_band_selected` :1247-1281)
- Modify: `src/crates/concat/src/studio/publish.rs` (:414)

**Interfaces:**
- Consumes: nothing new (existing `TimelineMsg`, `on_window!`, `state.lanes`).
- Produces: `TimelineMsg::MultiSelectChanged(bool)`, `TimelinePane.multi_select: bool`, `Editor::on_multi_select_changed(bool)`, generated `set_multi_select(bool)` — Tasks 2-4 rely on these names.

- [ ] **Step 1: Add the message variant, field, default, and match arm in `panes/timeline.rs`**

```rust
// enum TimelineMsg, right after PanChanged(bool):
/// The tray's or the phone's multi-select button: taps add to the
/// selection instead of replacing it — the touch world's Shift.
MultiSelectChanged(bool),

// struct TimelinePane, right after `pub pan_mode: bool,`:
/// Multi-select mode: a press adds a clip to the selection rather than
/// taking it, so a set can be gathered for Merge or Delete.
pub multi_select: bool,

// Default impl, after `pan_mode: false,`:
multi_select: false,

// update() match, right after the PanChanged arm:
TimelineMsg::MultiSelectChanged(on) => self.multi_select = on,
```

- [ ] **Step 2: Register the handler in `lib.rs`, right after `on_pan_changed` (:1107-1109)**

```rust
editor.on_multi_select_changed(on_window!(|state, on: bool| {
    state.handle(Msg::Timeline(TimelineMsg::MultiSelectChanged(on)));
}));
```

- [ ] **Step 3: OR the flag into `on_clip_pressed` (`lib.rs:1201-1206`)**

Replace the body with:

```rust
let additive = additive || state.lanes.multi_select;
state.clip_pressed(id.as_str(), additive, edge);
```

- [ ] **Step 4: OR the flag into `on_band_selected` (`lib.rs:1247`)**

After the existing `state.flush_commit();` line, insert:

```rust
let additive = additive || state.lanes.multi_select;
```

(The `if additive { ... } else { ... }` below already does the right thing: additive with an empty caught set leaves the selection untouched.)

- [ ] **Step 5: Publish the flag in `publish.rs`, right after `editor.set_pan_mode(...)` (:414)**

```rust
editor.set_multi_select(self.lanes.multi_select);
```

- [ ] **Step 6: Verify compilation**

Run: `cargo check -p concat` (cwd: `src/`)
Expected: `Finished` with no errors. First run downloads/compiles dependencies and may take a long time on this device; that is expected. If the toolchain cannot build at all locally, note it and defer to CI (Step 8).

- [ ] **Step 7: Commit**

```bash
git add src/crates/concat/src/panes/timeline.rs src/crates/concat/src/lib.rs src/crates/concat/src/studio/publish.rs
git commit -m "feat(timeline): multi-select mode state and additive tap glue"
```

- [ ] **Step 8: (conditional) If local check failed for environment reasons, record it** — CI will run fmt/clippy/nextest; do not attempt to install a toolchain.

---

### Task 2: Desk chain — Editor global → seat → pane → tray button

**Files:**
- Modify: `src/crates/concat/ui/editor.slint` (global properties :340-391, callbacks :393-430)
- Modify: `src/crates/concat/ui/workspace/seat.slint` (:408, :441)
- Modify: `src/crates/concat/ui/workspace/timeline-pane.slint` (properties :38-61, callbacks :63-71, tray instantiation :327-386)
- Modify: `src/crates/concat/ui/timeline/tray.slint` (properties :155-189, callbacks :191-217, buttons :300-312)

**Interfaces:**
- Consumes: `Editor::on_multi_select_changed` from Task 1.
- Produces: `Editor.multi-select: bool`, `Editor.multi-select-changed(on: bool)` — Task 3's phone ToolItems read/invoke exactly these.

- [ ] **Step 1: Declare on the `Editor` global (`editor.slint`)**

After the `pan-mode` property (:350):

```slint
/// Multi-select: taps add to the selection instead of replacing it —
/// the touch world's Shift, for gathering a set to merge or delete.
in-out property <bool> multi-select: false;
```

After `callback pan-changed(on: bool);` (:401):

```slint
callback multi-select-changed(on: bool);
```

- [ ] **Step 2: Thread through `seat.slint`**

After `pan-mode: Editor.pan-mode;` (:408):

```slint
multi-select: Editor.multi-select;
```

After `pan-changed(on) => { Editor.pan-changed(on); }` (:441):

```slint
multi-select-changed(on) => { Editor.multi-select-changed(on); }
```

- [ ] **Step 3: Thread through `timeline-pane.slint`**

Properties, after `in property <bool> pan-mode: false;` (:43):

```slint
in property <bool> multi-select: false;
```

Callbacks, after `callback pan-changed(on: bool);` (:67):

```slint
callback multi-select-changed(on: bool);
```

Tray bindings, after `pan-mode: root.pan-mode;` (:332) and `pan-changed(on) => ...` (:353):

```slint
multi-select: root.multi-select;
multi-select-changed(on) => { root.multi-select-changed(on); }
```

- [ ] **Step 4: Declare and add the button in `tray.slint`**

Property, after `in property <bool> pan-mode: false;` (:160):

```slint
/// Multi-select: on, a tap adds a clip to the selection.
in property <bool> multi-select: false;
```

Callback, after `callback pan-changed(on: bool);` (:195):

```slint
callback multi-select-changed(on: bool);
```

Button — inserted immediately BEFORE the merge `TrayButton` (:300), mirroring the razor toggle's `checked` pattern:

```slint
// Multi-select: on, a tap adds a clip to the selection instead of
// taking it — the touch world's Shift — so a set can be gathered
// for Merge or Delete.
TrayButton {
    glyph: Glyph.check;
    label: I18n.t("tray.multiSelect");
    icon-size: 16px;
    checked: root.multi-select;
    clicked => { root.multi-select-changed(!root.multi-select); }
}
```

- [ ] **Step 5: Trace the chain (manual, this is the Review Focus #1 failure)**

Confirm all eight links exist: `tray` button → `tray.multi-select-changed` → `timeline-pane` callback forwarding → `seat` callback forwarding → `Editor.multi-select-changed` → Rust `on_multi_select_changed`; and back: Rust `set_multi_select` → `Editor.multi-select` → `seat` binding → `pane` binding → `tray` `checked`.

- [ ] **Step 6: Verify compilation**

Run: `cargo check -p concat` (cwd: `src/`)
Expected: `Finished` with no errors (Slint files compile via `slint::include_modules!`).

- [ ] **Step 7: Commit**

```bash
git add src/crates/concat/ui/editor.slint src/crates/concat/ui/workspace/seat.slint src/crates/concat/ui/workspace/timeline-pane.slint src/crates/concat/ui/timeline/tray.slint
git commit -m "feat(tray): multi-select toggle button"
```

---

### Task 3: Phone toolbar — the toggle in both tool bars

**Files:**
- Modify: `src/crates/concat/ui/phone/editor.slint` (no-clip bar :931-1017, clip bar :1024-1197)

**Interfaces:**
- Consumes: `Editor.multi-select`, `Editor.multi-select-changed(bool)` from Task 2.
- Produces: nothing.

- [ ] **Step 1: Add the ToolItem to the no-clip bar**

Immediately after the `transitions` ToolItem (:976-981) and before the `captions` ToolItem (:982):

```slint
// Gather clips one tap at a time: the touch world's Shift, for a
// set to merge or delete.
ToolItem {
    glyph: Glyph.check;
    label: I18n.t("tray.multiSelect");
    current: Editor.multi-select;
    clicked => { Editor.multi-select-changed(!Editor.multi-select); }
}
```

- [ ] **Step 2: Add the same ToolItem to the clip bar**

Immediately after the `delete` ToolItem (:1043-1047), before the `is-shape` block (:1049):

Same block as Step 1 (identical).

- [ ] **Step 3: Verify compilation**

Run: `cargo check -p concat` (cwd: `src/`)
Expected: `Finished` with no errors.

- [ ] **Step 4: Commit**

```bash
git add src/crates/concat/ui/phone/editor.slint
git commit -m "feat(phone): multi-select toggle in both tool bars"
```

---

### Task 4: i18n key

**Files:**
- Modify: `src/crates/concat/locales/en.json` (~:943, alphabetical position among `tray.*`)
- Modify: `src/crates/concat/locales/tr.json` (same position)

**Interfaces:**
- Consumes: key name `tray.multiSelect` used by Tasks 2-3.
- Produces: passing `scripts/locales.py --check`.

- [ ] **Step 1: Add the English line to `en.json`**

```json
"tray.multiSelect": "Multi-select",
```

- [ ] **Step 2: Add the Turkish line to `tr.json`**

```json
"tray.multiSelect": "Çoklu seçim",
```

- [ ] **Step 3: Normalize and check**

Run: `python3 scripts/locales.py` then `python3 scripts/locales.py --check` (cwd: repo root)
Expected: check exits 0 (`en.json is out of date` must not appear; other locales' `missing` lines are informational and do not fail).

- [ ] **Step 4: Commit**

```bash
git add src/crates/concat/locales/en.json src/crates/concat/locales/tr.json
git commit -m "i18n: tray.multiSelect"
```

---

### Task 5: Final verification, push, CI

**Files:** none new.

- [ ] **Step 1: Formatting**

Run: `cargo fmt --check` (cwd: `src/`)
Expected: clean. If rustfmt is unavailable locally, note it — CI runs it.

- [ ] **Step 2: Lint (narrow, local)**

Run: `cargo clippy -p concat --all-targets -- -D warnings` (cwd: `src/`)
Expected: no warnings. (Full workspace clippy runs in CI regardless.)

- [ ] **Step 3: Full diff self-review**

Run: `git diff main --stat` and skim `git diff` — confirm no accidental edits, all names match the plan (`multi-select`, `multi-select-changed`, `MultiSelectChanged`, `multi_select`, `set_multi_select`, `on_multi_select_changed`, `tray.multiSelect`).

- [ ] **Step 4: Push (requires GitHub auth — ask the user here)**

Push to `main` triggers `.github/workflows/mobile.yml` → artifact `Concat-android-arm64`, plus `ci.yml` gates. This environment has no `gh` login and no stored credentials; ask the user to either run `gh auth login`, provide a token, or push from their own machine.

```bash
git push origin main
```

- [ ] **Step 5: Watch CI**

Confirm `ci.yml` (fmt, locales, clippy, nextest) and `mobile.yml` (Android build + emulator smoke) are green; tell the user where to download the APK artifact.
