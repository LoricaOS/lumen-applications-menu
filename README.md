# lumen-applications-menu

The full-screen application launcher for **AspisOS**, a capability-based,
no-ambient-authority operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

lumen-applications-menu is the Launchpad-style application grid. It is a
standalone `/bin` binary (installed as `/bin/applications`) that is a client of
the [lumen](https://github.com/AspisOS/lumen) compositor: it opens a chromeless
fullscreen window, lays the installed apps out as a tile grid, and launches the
one the user picks. It is a component of the Lumen desktop, distributed as a
[herald](https://github.com/AspisOS/AspisOS) package.

## Role in the system

- Launched on demand (e.g. from the dock's applications icon), not by `vigil` —
  it ships no service definition. It opens a chromeless fullscreen window
  (`LUMEN_WIN_FLAG_FULLSCREEN`) filled with `C_TERM_BG`, which the compositor
  color-keys to frosted glass so the dimmed desktop shows through.
- Tiles come from `glyph_apps_scan()` over the `/apps` bundles. Clicking a tile
  or pressing Enter asks Lumen to spawn it via `LUMEN_OP_INVOKE`, then the
  launcher exits.
- Clicking outside any tile or pressing Esc dismisses it (macOS Launchpad
  behaviour). Arrow keys move the keyboard selection.

## Capabilities

lumen-applications-menu's cap policy (`pkg/etc/aegis/caps.d/applications`) holds
only the baseline:

```
service
```

It needs no extra capabilities: it draws through the compositor and launches
apps by asking Lumen to invoke them, so it holds no ambient authority of its own.

The herald package id (`lumen-applications-menu`) intentionally differs from the
binary it installs (`/bin/applications`): the package is a distribution name, the
binary is the runtime exec path. That id/exec-name mismatch — together with the
fact that it installs into `/bin` plus a cap policy — is exactly why it is a
`class=system` package: first-party and signature-trusted, installed verbatim by
herald.

## Building

lumen-applications-menu fetches a pinned [glyph](https://github.com/AspisOS/glyph)
toolkit artifact (the GUI libraries it links: glyph + libaudio + libauth +
libcitadel) and builds against it, then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-applications-menu.hpkg` (a `class=system` herald package) +
`lumen-applications-menu.hpkg.sig`.

## Package payload

```
/bin/applications                       the launcher binary
/etc/aegis/caps.d/applications          its capability policy (baseline service)
```

## Repository layout

```
src/        applications launcher source
pkg/        install-tree skeleton shipped verbatim (caps.d)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — the launcher is a Lumen client and launches apps through the
compositor, so installing it pulls [lumen](https://github.com/AspisOS/lumen)
(which in turn provides the desktop fonts).
