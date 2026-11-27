# easyvm-solaris

Walkthroughs and CI for **illumos** guests - OpenIndiana, OmniOS, Tribblix and SmartOS - driven by
[`easydev-org/easyvm`](https://github.com/easydev-org/easyvm).

## What lives here

`openindiana/steps/`, `omnios/steps/`, `tribblix/steps/` and `smartos/steps/` - the `.walk` files for
`install` and `smoke`, one directory per distribution. OpenIndiana additionally has `ssh` and
`vm-configure`; the others get those once their install produces a disk that boots.

`smartos/media/` holds the two files `artifacts-publish.yml` adds to the SmartOS ISO: the answer file
and one line of loader configuration. Those are what make its install unattended.

## Two of the four have a desktop, and that is deliberate

| | desktop | why |
|---|---|---|
| OpenIndiana 2026.10 | **MATE**, lightdm | the GUI ISO carries it: 1368 packages, so the install copies a desktop off the disc and needs no network |
| Tribblix 0m41.1 | **Xfce**, SLiM | the `desktop` overlay, installed from the full ISO |
| OmniOS r151060 | none | MEASURED: its core and extra repositories hold 1109 packages between them and NONE of the desktop names - no mate, xfce, gnome, kde, lxde, no lightdm or slim. Only X11 libraries and headers (`ooce/x11/library/libX11` and friends). |
| SmartOS 20261121 | none | MEASURED: pkgsrc has 24,992 packages including `xfce4-session-4.18.4`, `lxde-1.0` and `modular-xorg-server-21.1.14`, but NO display manager (lightdm and slim are absent), and a desktop belongs nowhere near a hypervisor's global zone. |

OMNIOS AND SMARTOS ARE SERVER SYSTEMS, AND THEIR SMOKE STAGES EXPECT A TEXT LOGIN BECAUSE THAT IS
WHAT THEY PRODUCE. Forcing a desktop on either one would mean waiting for a graphical login they do
not have - and would test an arrangement the distribution does not ship.

So the four smoke gates differ on purpose, each matching its own system:

	openindiana   [[console, login], [Log, In]]      the GUI install reaches lightdm
	tribblix      [[tribblix, console, login]]       SLiM falls back to a text console here
	omnios        [[console, login], [omnios, console, login]]
	smartos       [[uscons, login]]                  its console login is on /dev/term/b

## The four installers are not the same shape

This is the one thing worth reading before touching any of these files, because the four guests
look alike on paper and behave nothing alike in a VM.

**OpenIndiana** is a TUI installer on a minimal image: F2 continues, F3 goes back, Space selects a
radio row, and `ret` does *not* move between fields on the Users page - `down` does. The whole
45-step sequence and each measurement behind it are in that file's header.

**OmniOS** is also a TUI, and is **pure keyboard** - a click on its `>` button leaves the screen
byte-identical, measured. Its pages are `ret` through: keyboard layout, main menu, `spc` to tick the
virtio disk, ZFS pool, hostname, time zone, and a `Yes` on the summary that is easy to miss. The disk
is then written as a 24 GiB ZFS image and a boot archive is rebuilt; **killing the machine between
those two publishes a disk whose loader answers `No rootfs module provided, aborting`** - measured
twice before the walk learned to wait for `Installation Complete`.

**Tribblix is not an installer at all.** It boots a live illumos system to a login prompt, and the
disk is written by `/root/live_install.sh -G c1t0d0 desktop` run by hand as root (`tribblix`). The
keyboard list that appears before the login is the live system's own. The install ends with
`property may be set but unable to remount filesystem`, which the project's page documents as the
sign of *success*, followed by `Welcome to Tribblix` - that second line is the completion witness.

**SmartOS has no installer binary at all.** It boots a live hypervisor image and runs `SmartOS Setup`,
an interactive TUI that this console cannot drive - at `Enter the default gateway IP [none]:` a `ret`,
`kp_enter`, `tab`, `spc` and releasing every modifier all do nothing, while letters type fine. So its
install is **unattended through a boot module**, which is what the project documents: the loader is
given `answers_json_load=YES` and a path, and the boot log then reads `Loading /local/answers.json...`
before the kernel starts.

Two keys in that answer file decide whether the result can boot at all, and the project's own example
file gets them wrong: `"bootpool": "none"` creates the pool and publishes a disk with **no ESP and no
MBR signature** (`0x1fe = 00 00`), while `"bootpool": "zones"` with `"pisource": "media"` runs
`piadm bootable -e -i media zones`, which writes an EFI partition and `0x1fe = 55 aa`. Measured both
ways; only the second disk starts.

## What makes these guests different

ALL FOUR ARE ILLUMOS DISTRIBUTIONS - illumos is the kernel, the way Linux is for Ubuntu. Package
management is `IPS` (`pkg install`) on OpenIndiana and OmniOS, `zap` overlays on Tribblix, pkgsrc on
SmartOS; service management is `SMF` (`svcadm`) on all four. None of it resembles systemd.

**OmniOS and SmartOS are server systems and have no desktop at all** - see the table above for the
repository evidence. Their smoke stages verify the boot and the login prompt and nothing else, because
there is no session to look at. **Tribblix is installed with the `desktop` overlay** so its smoke has
a session, which is why this repository mirrors the 1332 MiB full ISO rather than the 392 MiB minimal
one: the overlay installs from the medium and needs no network.

## The installed disk is started by its own loader, not by the firmware

Every guest here boots a disk written with a whole-disk EFI layout, and the console output differs
between them in a way that matters for what a smoke stage can wait on:

- **OpenIndiana** hands its console to the framebuffer once the kernel starts, so `console.log` stops
  at `Booting...` and the login prompt is on the *screen*. `smoke.walk` reads the frame.
- **OmniOS and Tribblix** keep writing to the serial line as well as the 1024x768 VGA, and both print
  `omnios console login:` / `tribblix console login:`. Their smoke stages read the frame too, which
  works either way.
- **SmartOS** prints its own `1. Boot SmartOS [Enter]` loader menu and then `smartos uscons login:` -
  `uscons` because its console login runs on `/dev/term/b`, not `/dev/console`. Its smoke stage waits
  for that prompt rather than for the menu, because the menu autoboots in about ten seconds and a
  ten-second poll can step over it while the machine is perfectly healthy.

## The media is mirrored, not fetched from upstream

A stage pulls its installer image from this repository's ghcr package, never from the publisher:
`artifacts-publish.yml` downloads each pinned URL once, checks its digest, packs it into a tar and
pushes one tag. Every stage afterwards asks the package.

```sh
gh workflow run artifacts-publish.yml -f which=iso-openindiana-2026.10-gui
gh workflow run artifacts-publish.yml -f which=iso-omnios-r151060
gh workflow run artifacts-publish.yml -f which=iso-tribblix-0m41.1
gh workflow run artifacts-publish.yml -f which=iso-smartos-20261121T172607Z

gh workflow run openindiana-2026.10-install.yml
gh workflow run omnios-r151060-install.yml
gh workflow run tribblix-0m41.1-install.yml
gh workflow run smartos-20261121T172607Z-install.yml

gh workflow run openindiana-2026.10-smoke.yml
gh workflow run omnios-r151060-smoke.yml
gh workflow run tribblix-0m41.1-smoke.yml
gh workflow run smartos-20261121T172607Z-smoke.yml
```

## What lives elsewhere, on purpose

The **engine** and the **CI machinery** are both in `easydev-org/easyvm`; these workflows call
`easydev-org/easyvm/.github/workflows/stage.yml@v0.0.4` for the second.

Disks pull from and publish to this repository's own ghcr package, `easydev-org/easyvm-solaris`.

