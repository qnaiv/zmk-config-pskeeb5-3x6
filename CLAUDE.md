# pskeeb5 3x6 zmk-config — working notes for Claude

This is a personal ZMK keymap config for a **6-column columnar split keyboard**
(pskeeb5 3x6, 44 keys, `nice_nano_v2` x2) with a **PS/2 trackpoint** on the
right/central half and **2 rotary encoders**. Forked from
[klesh/zmk-config-pskeeb5-3x6](https://github.com/klesh/zmk-config-pskeeb5-3x6),
owned here as `qnaiv/zmk-config-pskeeb5-3x6`.

Read [README.md](README.md) first — it documents the current keymap, Auto
Mouse Layer, and PS/2 driver fork from the user's point of view. This file is
about **how to work in this repo correctly**, not what the keymap currently
does (README/the keymap file itself are the source of truth for that — this
file can go stale, they can't).

## Repo layout

- `config/pskeeb5.keymap` — the entire keymap (behaviors, combos, layers).
  This is the file that gets edited for almost every request.
- `config/west.yml` — manifest. See "west.yml gotchas" below.
- `build.yaml` — GitHub Actions build matrix (`pskeeb5_left`, `pskeeb5_right`
  with `studio-rpc-usb-uart`, and `settings_reset`).
- `.github/workflows/build.yml` — calls the standard zmk-config build action.
- Firmware is **not** built locally — there's no local west/Zephyr toolchain
  set up in this working directory. Always push and let GitHub Actions build.

## Physical key position numbering

Positions are 0-indexed, 44 keys total, laid out row-major:
- Row0: 0–11 (left 0–5, right 6–11)
- Row1: 12–23
- Row2: 24–35
- Row3 (thumbs): 36–43, split 36–39 (left thumb cluster) / 40–43 (right thumb
  cluster) — so pos39 = left spacebar, pos40 = right spacebar, adjacent in
  the middle.

This comes from `default_transform` in `pskeeb5_layouts.dtsi` in the
`klesh/zmk` fork (`app/boards/shields/pskeeb5/pskeeb5_layouts.dtsi`, branch
`pskeeb5_3x6`) — fetch it from GitHub if you need to re-derive positions,
don't guess. **An off-by-one here has bitten this project before** (mouse
click positions were wrong by one column for a while) — double check against
that file or against the Artifact visualization (see below) rather than
counting columns in the keymap's ASCII art, which is easy to miscount.

## west.yml gotchas

```yaml
projects:
  - name: zmk
    remote: klesh
    revision: pskeeb5_3x6
    import:
      file: app/west.yml
      name-blocklist:
        - zmk-ydxc02-driver   # SSH-URL dependency, breaks CI auth — keep blocked
  - name: kb_zmk_ps2_mouse_trackpoint_driver
    remote: qnaiv
    revision: main   # our fork, overrides the one klesh/zmk's import pulls in
```

- `zmk-ydxc02-driver` must stay blocklisted — it's referenced via an SSH URL
  upstream and CI has no credentials for that, so leaving it in breaks every
  build with an auth error.
- The `kb_zmk_ps2_mouse_trackpoint_driver` entry here is a **deliberate
  override** of the same-named project pulled in by `klesh/zmk`'s own
  `app/west.yml` import. West resolves same-name project conflicts in favor
  of the importing (top-level) manifest, so this makes CI use
  `qnaiv/kb_zmk_ps2_mouse_trackpoint_driver` (our patched fork) instead of
  klesh's upstream version. `revision: main` means it always builds the
  fork's latest `main` — there's no pinned SHA, so pushing to that fork
  immediately changes what this repo builds next.

## The PS/2 driver fork (`qnaiv/kb_zmk_ps2_mouse_trackpoint_driver`)

Local clone: `C:\Users\nochi\vscode\kb_zmk_ps2_mouse_trackpoint_driver`
(separate repo/working directory from this one — remember to `cd` there and
push separately if you touch it).

Adds an `excluded-positions` devicetree property to
`zmk,input-listener-ps2` nodes: while the Auto Mouse Layer (or whatever
layer `layer-toggle` points at) is active, pressing any key position **not**
in `excluded-positions` deactivates that layer immediately, without waiting
for `layer-toggle-timeout-ms`.

**Critical implementation detail, already debugged once — don't redo this
mistake:** `zmk_position_state_changed` fires *before* ZMK resolves and
invokes the behavior bound to that position. Deactivating the layer
synchronously from that event handler makes ZMK resolve the triggering
keypress itself against the now-changed layer state, breaking it (e.g. a
click on the mouse layer stops registering as a click). The fix is to only
`k_work_schedule(..., K_NO_WAIT)` the deactivation from the listener, letting
it run after the current keypress has already been dispatched. See the
`IMPORTANT` comment block in `src/mouse/input_listener_ps2.c` in that repo
for the full explanation before touching this file again.

## Auto Mouse Layer (`mouse_layer`, layer 15) — `&none` vs `&trans`

Every position in `mouse_layer` that isn't a deliberate mouse binding must be
`&trans`, **not** `&none`. This was a real bug: `&none` does not "pass
through" to the layer below despite looking harmless — it's a real behavior
that silently absorbs the keypress and sends nothing. `&trans` is what falls
through to the base layer. Getting this wrong means ordinary typing (and
modifiers like Alt) silently do nothing while the trackpoint is in use, e.g.
holding Alt + tapping Tab repeatedly for app-switching would just send bare
Tab once the mouse layer exits, because the Alt keydown itself never
happened. If you add a new `&none` anywhere in `mouse_layer`, ask whether you
actually meant `&trans`.

`excluded-positions = <4 16 40>` in the `&mouse_ps2_input_listener` node at
the bottom of the keymap controls which keys *don't* end the layer early —
currently the two scroll keys (R=pos4, F=pos16) and left-click (pos40).
Right-click (pos41) and middle-click (pos42) are deliberately *not* in that
list, so pressing either exits the layer immediately after.

## Home row mods (`hml` / `hmr`) — positional hold-tap

Left-hand mods use behavior `hml`, right-hand mods use `hmr`
(`config/pskeeb5.keymap`, `behaviors` block). Both use
`hold-trigger-key-positions` + `hold-trigger-on-release` so that same-hand
typing rolls (e.g. "bad ") stay taps, while cross-hand chords (actual
shortcuts) reliably resolve to hold. The trigger list is normally just "the
other hand's key positions" — but a few keys are **thumb/OS-shortcut-neutral
and get added to both `hml` and `hmr`'s trigger lists regardless of hand**:

- Both spacebars (pos39, pos40) — Ctrl+Space (IME toggle) must work no
  matter which physical space bar is pressed alongside a held mod.
- Tab (pos0) — Alt+Tab/Ctrl+Tab/Shift+Tab are OS shortcuts, not same-hand
  letters, even though Tab and e.g. `S` (Alt) are both left-hand keys.

**If a "hold X, tap key Y, get plain Y instead of the modified version"
report comes in again**, the fix is almost always the same shape: check
whether Y's position is missing from the relevant `hml`/`hmr`
`hold-trigger-key-positions` list, and add it if Y is a shortcut-target key
rather than an ordinary same-hand letter. Don't reflexively add every key —
only genuinely hand-neutral/shortcut-target keys belong there, or same-hand
typing safety (the entire point of this positional design) erodes.

`flavor = "balanced"`, `tapping-term-ms = 220`, `require-prior-idle-ms = 150`
on both. Default hold-tap flavor in ZMK (used by the unflavored `lt` and
`mo_mkp` behaviors here) is `hold-preferred`.

## Auto Mouse Layer right-click (`mo_mkp`)

`mouse_layer`'s pos41 binding is `&mo_mkp 3 RCLK`: tap → right click, hold →
momentary Nav layer (same as the base layer's Fn key), so IJKL arrow
navigation is reachable while the trackpoint is in use. The layer still
exits immediately on *press* of pos41 either way (see excluded-positions
above) — that's driven by the raw key position in the PS/2 driver, not by
how the hold-tap resolves.

## Workflow for keymap/driver changes

1. Edit `config/pskeeb5.keymap` (or the driver repo, separately).
2. Commit and push directly to `main` (no PR workflow on this repo) with
   `Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>` on commits you
   author.
3. Check the build at
   `https://github.com/qnaiv/zmk-config-pskeeb5-3x6/actions` — poll the run
   for the pushed commit until it's `Success` (builds take ~2 minutes; the
   left/right/settings_reset matrix jobs report individually, then a final
   merge-artifacts job). If it fails, read the job logs before guessing.
4. Remind the user firmware needs to be **manually reflashed** on real
   hardware — a successful CI build proves it compiles, not that it works on
   the keyboard. Several "bug" reports in this project's history turned out
   to just be stale firmware (see README's PS/2 driver section for one
   example). Both halves need reflashing for most changes; the right/central
   half specifically for anything touching the PS/2 driver or trackpoint.
5. Update the keymap Artifact (see below) and `README.md` if the change is
   user-visible.

## Keymap visualization Artifact

There's a published Claude Artifact (HTML) visualizing the full keymap by
physical layout — ask the user for the current URL if you don't have it in
context (or check `README.md`, which links it), and re-publish to the same
URL after any keymap change so it stays in sync. Local source file lives in
the session's scratchpad directory as `pskeeb5_keymap.html`, but that path is
session-specific — you'll typically need to fetch/recreate it fresh, or ask
the user, rather than assuming a fixed path across sessions.

## Verifying ZMK semantics before assuming

This project has been bitten twice by assuming ZMK behavior semantics
instead of checking (`&none` vs `&trans`, and the position_state_changed
event ordering). Before relying on a specific ZMK core behavior, prefer
fetching the actual source from `klesh/zmk` (branch `pskeeb5_3x6`) on GitHub,
e.g. `https://raw.githubusercontent.com/klesh/zmk/pskeeb5_3x6/app/src/keymap.c`
or the relevant `dts/bindings/behaviors/*.yaml`, over recalling from memory.
