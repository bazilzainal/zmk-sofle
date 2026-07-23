# Eyelash Sofle — ZMK Architecture & Notes

Personal reference for how this keyboard firmware repo is put together, the core
ZMK concepts behind it, and the state of this fork versus upstream. Written as
study notes, not vendor docs.

---

## 1. What this repo is

A **ZMK firmware config** for the *Eyelash Sofle* — a split 6×4 + thumb-cluster
keyboard with a rotary encoder, RGB underglow, backlight, and a nice_view display,
driven by an nRF52840 MCU.

It's a **fork of the vendor repo** `a741725193/zmk-sofle` (the seller — the
qq.com contact in the README). Our own customisations live on top of that fork.

There is **no local build toolchain**. You push commits; GitHub Actions compiles
the `.uf2` files you flash onto each half.

---

## 2. The build pipeline (how a push becomes firmware)

```
git push
   │
   ├── .github/workflows/build.yml
   │      └── calls zmkfirmware/zmk .github/workflows/build-user-config.yml
   │             ├── reads build.yaml        → the build MATRIX (what to build)
   │             ├── reads config/west.yml   → the dependency MANIFEST (what to fetch)
   │             └── outputs .uf2 artifacts  → download & flash
   │
   └── .github/workflows/draw.yml (only on config/** changes)
          └── calls keymap-drawer → regenerates keymap-drawer/eyelash_sofle.svg
                                     (the layout image embedded in the README)
```

Key files:

| File | Role |
|---|---|
| `build.yaml` | Build matrix — each entry is one firmware target (a board/shield + options). |
| `config/west.yml` | Dependency manifest — which repos/branches west pulls in. |
| `config/eyelash_sofle.keymap` | **The keymap that actually builds** (layers, behaviors). |
| `config/eyelash_sofle.conf` | Kconfig user overrides (sleep timeout, RGB, mouse, etc.). |
| `config/eyelash_sofle.json` | Physical layout description (used by ZMK Studio / editors). |
| `keymap_drawer.config.yaml` | Styling for the auto-generated keymap SVG. |
| `zephyr/module.yml` | Declares this repo itself as a Zephyr/ZMK module (`board_root: .`). |
| `boards/arm/eyelash_sofle/` | A **vendored copy** of the board definition (see §4). |

---

## 3. The core concept: **board model vs shield model**

This is the single most important idea for understanding — and eventually
updating — this repo.

ZMK separates two things:

- **The controller** — the MCU (here an nRF52840): flash, USB, Bluetooth, power.
- **The physical keyboard** — the switch matrix, encoder, and display wiring.

There are two ways to express that split:

### Board model (what THIS fork still uses)

The keyboard *is* a full custom **board**. `boards/arm/eyelash_sofle/` contains
the entire board port:

- `*.dts` — device tree: pin assignments, the key matrix, peripherals.
- `*_defconfig` — the SoC selection (nRF52840), flash layout, USB stack.
- `Kconfig.board` / `Kconfig.defconfig` — board Kconfig symbols and defaults.

The whole PCB + MCU is treated as one integrated unit that *we* own end to end.

### Shield model (what UPSTREAM has migrated to)

The keyboard is just a **shield** — a matrix overlay — that plugs onto a
*standard, maintained* controller board like `nice_nano_v2`. The shield dir holds
essentially only an `.overlay` (the matrix wiring) + `Kconfig.shield`; the generic
controller supplies all the SoC/flash/USB plumbing.

In the shield model `build.yaml` looks like:

```yaml
- board: nice_nano_v2
  shield: eyelash_sofle_left nice_view
```

versus our board model:

```yaml
- board: eyelash_sofle_left
  shield: nice_view
```

### Why shields win (the trade-off)

**Separation of concerns / composition over monolith.** When a new ZMK release or
a new controller lands, the maintained `nice_nano_v2` board absorbs the change —
the shield (just matrix wiring) rarely needs to move. The board model means *we*
re-own all the SoC/flash/USB details forever, and have to hand-maintain them
against every upstream ZMK change.

The board model's only real upside is being fully self-contained (no dependency on
an external controller definition) — which mostly matters for a genuinely bespoke
PCB, not for one that's nice_nano-compatible.

---

## 4. `west` and the dependency manifest

`west` is Zephyr's meta build tool: a **multi-repo manifest** that stitches
several git repos into one workspace. `config/west.yml` is roughly a lockfile —
it lists each project and the exact `revision` (branch/tag) to check out.

Our current `config/west.yml` pulls in:

- `zmkfirmware/zmk` @ **`main`** — the core ZMK firmware.
- `eyelash_sofle` from `a741725193/zmk-sofle` @ **`refactor`** — the board def.
- a peripheral RGB animation module.

### Two different `main` branches — don't confuse them

- **`a741725193/zmk-sofle@main`** — the *vendor / upstream* repo. This is the
  parent our fork was cut from; "upstream main" in the divergence notes = this.
- **`zmkfirmware/zmk@main`** — the *ZMK firmware project itself*, a separate repo.
  This is the ZMK source our `west.yml` pins.

### Pinning a moving branch is a footgun

We pin `zmk@main` — a branch that keeps advancing. Any upstream ZMK change can
break our build with **zero changes on our side**, and we'd only find out the next
time we try to flash. Upstream deliberately stopped doing this: they now pin a
specific fork/tag (`cormoran/zmk@v0.3-branch+dya`) plus feature modules, so their
builds are reproducible. Pin stable revisions, not moving branches.

---

## 5. Known redundancy / cleanup in this repo

Recorded so it's not re-discovered later. (As of 2026-07-23; none fixed yet.)

1. **Board defined twice — collision risk.** The board is vendored locally in
   `boards/arm/eyelash_sofle/` (activated via `zephyr/module.yml`'s `board_root: .`)
   **and** imported through `config/west.yml`. Two board-roots provide the same
   `eyelash_sofle_left`/`_right`; one silently shadows the other. It builds only
   because they're near-identical. **Fix: pick one source of truth** — either keep
   the vendored copy and drop the west import, or delete the local board + module
   and keep the import.
2. **Two keymaps, one is dead.** `config/eyelash_sofle.keymap` is what builds
   (ZMK's `ZMK_CONFIG` keymap always overrides a board's default). The board's
   `boards/arm/eyelash_sofle/eyelash_sofle.keymap` is never used — dead weight.
3. **Copy-paste noise** in `config/eyelash_sofle.keymap`:
   `#include <dt-bindings/zmk/pointing.h>` appears 3×; the mouse `#define`s
   (`ZMK_MOUSE_DEFAULT_MOVE_VAL` / `_SCRL_VAL`) are defined twice.
4. **Foreign drawer files.** `keymap-drawer/corne.*` and `keymap-drawer/sofle.*`
   are for other keyboards — template leftovers.
5. **Broken README links.** After the rename dance (`README_EN.md`→`README.md`,
   `README.md`→`README_CN.md`), `README.md` links to a `README_EN.md` that no
   longer exists and mislabels the language columns.

---

## 6. This fork vs upstream — and the update path

**State (2026-07-23):** our `main` is **74 commits behind** upstream
`a741725193/zmk-sofle@main` and **5 ahead** (the 5 = our keymap customisations +
README renames).

Upstream didn't just add commits — they **re-architected**:

| Area | This fork (`@refactor`, frozen Dec 2024) | Upstream `main` (active, 2026) |
|---|---|---|
| Keyboard model | Custom **board** (`boards/arm/`) | **Shield** on `nice_nano_v2` (`boards/shields/`) |
| ZMK source | mainline `zmkfirmware/zmk@main` (moving) | pinned `cormoran/zmk@v0.3-branch+dya` + modules |
| Pointing | `CONFIG_ZMK_MOUSE` (old name) | `CONFIG_ZMK_POINTING` (renamed subsystem) |
| Extras | — | soft-off, debounce tuning, dyastudio adaptation, battery history, BLE mgmt |

Also note the `refactor` branch our `west.yml` pins is itself ~88 commits behind
upstream's own `main` — we're pinned to a **dead branch**.

### Recommended path: migrate, don't merge

A plain `git merge upstream/main` would be a conflict nightmare — board→shield is a
structural change. Treat it as a **replay/port**:

1. Start fresh from upstream `main`'s thin shield structure (shields + their
   `west.yml`).
2. **Port our keymap** (`config/eyelash_sofle.keymap`) on top — the valuable part.
   Rename mouse behaviors/defines to their `pointing` equivalents as you go.
3. Update `build.yaml` entries to `board: nice_nano_v2` + `shield: eyelash_sofle_left nice_view`.
4. **Flash and test on the real keyboard** before trusting it. Always keep a
   known-good `.uf2` as a rollback — firmware isn't `git revert`-able off the board.

Payoff: soft-off (real battery savings), a stable *pinned* ZMK instead of a moving
target, and staying on the branch upstream actually maintains.

---

## 7. Questions worth being able to answer

- If we *don't* migrate: what's the plan for the day `zmkfirmware/zmk@main`
  changes an API our old board relies on — would we find out before the next flash?
- Does the board model give *us* anything specific, or is vendoring purely
  inherited from the template?
- Why is pinning a tag/fork reproducible where pinning `@main` isn't — and what's
  the equivalent lesson in any dependency manifest (npm, Cargo, go.mod)?
