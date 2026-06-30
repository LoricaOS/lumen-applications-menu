# lumen-applications-menu

The full-screen application launcher for [LoricaOS](https://github.com/LoricaOS/LoricaOS)
— a capability-based, no-ambient-authority x86-64 operating system built on the
from-scratch [Aegis](https://github.com/LoricaOS/Aegis) kernel.

lumen-applications-menu is a Launchpad-style application grid. It is a standalone
binary (installed at `/bin/applications`) and a client of the
[lumen](https://github.com/LoricaOS/lumen) compositor: it opens a chromeless
full-screen window, lays the installed apps out as a tile grid over the dimmed
desktop, launches the one the user picks, and dismisses itself. It is
distributed as a [herald](https://github.com/LoricaOS/LoricaOS) system package.

## Where it fits in LoricaOS

LoricaOS is decomposed into independent repositories:

| Repo | Role |
|------|------|
| `LoricaOS/Aegis` | The kernel: framebuffer, `AF_UNIX` sockets, the capability model, the syscalls everything graphical runs on. |
| `LoricaOS/lumen` | The compositor/display server. Owns the screen; every GUI process connects to `/run/lumen.sock` for a window. |
| `LoricaOS/glyph` | The GUI toolkit. Provides drawing primitives, fonts, procedural icons, the `/apps` bundle scanner (`glyph_apps_scan`), and the client side of lumen's window protocol (`lumen_client.h`). |
| `LoricaOS/lumen-applications-menu` | **This repo.** A lumen client that presents the installed-apps grid and brokers launches through the compositor. |

The launcher holds no display authority of its own — it does not touch the
framebuffer or input devices. It talks to lumen, which composites its surface and
forwards it input. It is typically opened from the dock's applications icon.
Everything graphical declares `depends=lumen`.

## What it does

`src/main.c` is a single-file lumen client:

- **Connect and open full-screen.** It calls `lumen_connect_retry()`, then
  `lumen_window_create_ex(..., LUMEN_WIN_FLAG_FULLSCREEN)` for a chromeless
  full-screen surface sized to the framebuffer.
- **Frosted backdrop.** It fills the whole surface with the compositor frost key
  (`C_TERM_BG`), which lumen color-keys to frosted glass so the desktop shows
  through dimmed. Tile highlights are drawn as off-key translucent rounded rects
  so they composite as a faint pane over that backdrop.
- **The grid.** Tiles come from `glyph_apps_scan()` over the `/apps` bundles.
  `layout_grid()` centres a grid whose column count is derived from the
  framebuffer width; each tile shows the app's procedural icon
  (`glyph_icon_draw()`) and its name, with long names clipped rather than allowed
  to bleed. If no apps are installed it draws a centred "No applications
  installed" message.
- **Input.** The mouse hover highlights the tile under the pointer. A left-click
  on a tile launches it; a click outside any tile dismisses the launcher (macOS
  Launchpad behaviour). The keyboard selection moves with the arrow keys (lumen
  folds CSI arrows to synthetic key codes for proxy windows) and Enter launches
  the selection; Esc dismisses.
- **Launch and exit.** Launching sends `lumen_invoke()` (`LUMEN_OP_INVOKE`) with
  the app's bundle id; lumen resolves it against the `/apps` registry and spawns
  it. The launcher then exits — it is transient, not a resident process. It also
  exits cleanly on `SIGTERM` (handled) and on `LUMEN_EV_CLOSE_REQUEST`.

## Capabilities

lumen-applications-menu ships a cap policy at
`pkg/etc/aegis/caps.d/applications` that grants only the baseline service
profile — no extra capabilities:

```
service
```

It needs none: it draws through the compositor rather than the raw framebuffer,
and it launches apps by asking lumen to invoke them rather than spawning them
itself. It holds no ambient authority.

The herald package id (`lumen-applications-menu`) intentionally differs from both
the binary it installs (`/bin/applications`) and the cap-policy basename
(`applications`): the id is a distribution name, the binary is the runtime exec
path. That id/exec-name divergence — together with installing into `/bin` and
`/etc/aegis` — is exactly why it is a `class=system` package: first-party and
signature-trusted, installed verbatim by herald.

## Building

lumen-applications-menu builds with a musl cross-compiler against a pinned
[glyph](https://github.com/LoricaOS/glyph) toolkit artifact, then packs a signed
herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

The `Makefile` fetches the toolkit, compiles `src/*.c` against it, and packs the
`.hpkg`:

- `GLYPH_VERSION` pins the glyph release fetched by `tools/fetch-glyph.sh` (local
  `vendor/` cache first, otherwise the GitHub release). Links
  `-lcitadel -laudio -lauth -lglyph` — a static archive only contributes the
  objects actually referenced.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption; defaults
  to `musl-gcc` on `PATH`). Point it at an Aegis-native `cc` to build on-device
  in the future.
- `HERALD_KEY` signs the package (ECDSA P-256).

Output: `lumen-applications-menu.hpkg` (a `class=system` herald package) plus its
detached `lumen-applications-menu.hpkg.sig`.

## Package payload

`lumen-applications-menu.hpkg` is a manifest-first, uncompressed POSIX `ustar`
archive with a detached ECDSA-P256/SHA-256 signature. herald installs the payload
tree verbatim at these paths:

```
/bin/applications                      the launcher binary (stripped)
/etc/aegis/caps.d/applications         capability policy (baseline service)
```

There is no vigil service: the launcher is started on demand (e.g. from the
dock), not at boot.

## Repository layout

```
src/main.c        the launcher client (connect, grid layout, render, invoke)
pkg/              install-tree skeleton shipped verbatim (caps.d)
tools/
  fetch-glyph.sh  fetch + unpack the pinned glyph toolkit artifact
  pack.sh         build + sign lumen-applications-menu.hpkg
Makefile          fetch toolkit → build → pack
VERSION           this component's version
GLYPH_VERSION     the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — the launcher is a lumen client and launches apps through the
compositor, so installing it pulls [lumen](https://github.com/LoricaOS/lumen)
(which in turn ships the desktop fonts every component inherits).
