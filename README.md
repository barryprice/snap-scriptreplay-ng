# snap-scriptreplay-ng

**[scriptreplay_ng](https://github.com/scoopex/scriptreplay_ng) as a [Snap](https://snapcraft.io) package.**

scriptreplay_ng replays a typescript recorded by `script(1)`, using the timing data to
reproduce the original typing and output delays — useful for reviewing recorded sessions,
demos and shell audit trails. This repo packages it as a strictly-confined Snap.

## Install

```bash
sudo snap install scriptreplay-ng-bp
sudo snap alias scriptreplay-ng-bp.scriptreplay-ng scriptreplay-ng
sudo snap alias scriptreplay-ng-bp.record-script-session record-script-session
```

## Usage

Replay a session (relies on the above aliases):

```bash
scriptreplay-ng -t timing typescript
scriptreplay-ng -a 2 -t ~/.script/2014-07-21/…/timing.gz ~/.script/…/typescript.gz
```

Compressed typescripts (`.bz2`, `.gz`, `.lz`, `.lzma`) are read directly; if `-t` is
omitted, a timing file next to the typescript is picked up automatically.

During playback: `-`/`d` slows down, `+`/`i` speeds up, `=`/`n` returns to normal speed,
`s`/`p` pauses, `c` continues, `q`/`f` quits.

### Recording

`record-script-session` wraps `script -t`, writing a gzipped typescript and timing file
under `$HOME/.script/<date>/<timestamp>-<name>/`. Inside a strict snap `$HOME` is the
snap's own user data directory, so the recording actually lands in
`~/snap/scriptreplay-ng-bp/current/.script/…`; the path is printed when the session ends:

```bash
record-script-session mysession
```

It records with `script(1)`, which starts `$SHELL` — set `SHELL=/bin/bash` if your login
shell is not part of the snap's runtime.

> **Confinement caveat.** The recorded shell runs *inside the snap's confined runtime*, so
> it only sees the snap's own environment — not your host's tools. This is fine for a quick
> demo, but to audit a real shell session, run `script -t` on the host and replay the result
> with this snap.

## Interfaces

| Interface | App | Notes |
| --------- | --- | ----- |
| `home` | both | auto-connected; read typescripts from `$HOME` and write recordings under `~/snap/scriptreplay-ng-bp/` |
| `removable-media` | `scriptreplay-ng` | **not** auto-connected — `sudo snap connect scriptreplay-ng-bp:removable-media` to replay from `/media` or `/mnt` |

## How it works

Upstream publishes no releases and no current tags, so the snap pins a specific commit on
`master`. Renovate tracks the branch head via the `git-refs` datasource and raises a pull
request when it moves, so updates are reviewed rather than picked up silently. The snap
version is derived from the pinned commit as `<commit date>+<short sha>`.

`scriptreplay-ng` is a Perl script and core26 ships no perl at all, so the snap stages `perl`,
`libterm-readkey-perl` (the one non-core module it uses), `bsdutils` (for `script(1)`) and
`xz-utils` (whose `lzcat` is an update-alternatives symlink the build recreates by hand);
`gzip`, `zcat` and `bzcat` come from the base. `PERL5LIB` is set per app so the staged
interpreter finds those modules under `$SNAP`.

### Downstream patch

The snap carries one patch, in [`patches/`](patches), applied during the pull step:

- **`0001-scriptreplay-open-compressed-typescripts-without-a-shell.patch`** — upstream opens
  a compressed typescript with a two-argument `open(SCRIPT, "zcat $file|")`, which Perl runs
  through `/bin/sh`. Replaying a file whose *name* contains shell metacharacters therefore
  executes it: `scriptreplay-ng 'evil ;touch PWNED; x.gz'` runs `touch` before any replay
  happens — a realistic hazard for typescripts unpacked from an archive or read off
  removable media. The patch returns the mode and arguments separately and uses the list
  form of `open()`, so the file name reaches the decompressor through `exec()` with no shell
  in between. It is offered upstream and carried here until it lands; `git apply` fails the
  build if a pin bump ever lands somewhere it no longer applies, so the fix cannot be
  silently dropped.

### Build gates

Every build gates on the packed tree, not the build environment: the staged perl must
compile `scriptreplay-ng` and load `Term::ReadKey`, `record-script-session` must parse, `-h`
must print usage, and upstream's own example session is replayed end-to-end. The replayed
bytes are compared against the typescript body — the transcript with `script(1)`'s
`Script started`/`Script done` lines removed — so a replay that stops early or garbles its
output fails, which a byte count would miss because the timing summary pads it. Anything
on stderr fails the gate too. The same session is then replayed from `.gz` and `.lzma`
copies named `evil ;touch PWNED; session.*`, which proves the patch above (no `PWNED` file
may appear), exercises the hand-made `lzcat` symlink, and shows each decompressor returns
the identical body. The perl series named in `PERL5LIB` is verified against the staged
interpreter, so an archive perl bump fails the build instead of shipping broken module
paths.

Each replay runs the primed `perl` directly, with `stdin` on `/dev/null` and under
`timeout --kill-after`, so a wedged replay is always killed. An earlier version wrapped
each replay in `script(1)` to give it a pty; a managed (LXD) build stopped on that line and
outlived its own `timeout`, leaving craft-parts waiting on the scriptlet's pipes. That was
never reproduced outside LXD, so the wrapper was removed rather than diagnosed — it is the
only part of the gate that needed a pty, and `script(1)` runs its child in a separate
session, out of reach of a `timeout` that signals its own process group. Nothing is lost:
`scriptreplay-ng`'s cbreak setup is a no-op off a terminal.

A CI workflow builds and lints the snap on every push and pull request;
[snapcraft.io](https://snapcraft.io) handles publishing to the Store on its own schedule.

## Disclaimer

This is a **community-maintained Snap**, not an official scriptreplay_ng project. Report
issues with the packaging here; report issues with the tool itself
[upstream](https://github.com/scoopex/scriptreplay_ng/issues).

Upstream ships no licence file; its manual page states the program is in the public domain,
so no SPDX licence is declared in `snap/snapcraft.yaml`.

## Credits

- **[scoopex/scriptreplay_ng](https://github.com/scoopex/scriptreplay_ng)** — the tool this Snap packages, by Marc Schöchlin, building on work by Joey Hess and Hendrik Brueckner.
- **[barryprice](https://github.com/barryprice)** — Snap packaging and maintenance.
