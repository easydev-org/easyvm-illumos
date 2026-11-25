# easyvm-solaris

Walkthroughs and CI for **illumos** guests - OpenIndiana, OmniOS and Tribblix - driven by
[`easydev-org/easyvm`](https://github.com/easydev-org/easyvm).

## What lives here

`openindiana/steps/`, `omnios/steps/` and `tribblix/steps/` - the `.walk` files for `install` and
`smoke`. OpenIndiana additionally has `ssh` and `vm-configure`; the other two get those once their
install produces a disk that boots.

## The three installers are not the same shape

This is the one thing worth reading before touching any of these files, because the three guests
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

## What makes these guests different

ALL THREE ARE ILLUMOS DISTRIBUTIONS - illumos is the kernel, the way Linux is for Ubuntu. Package
management is `IPS` (`pkg install`) on OpenIndiana and OmniOS, `zap` overlays on Tribblix; service
management is `SMF` (`svcadm`) on all three. None of it resembles systemd.

**OmniOS is a server system and has no desktop at all.** Its smoke stage verifies the boot, the ZFS
pool and the login prompt, and nothing else - there is no session to look at. **Tribblix is
installed with the `desktop` overlay** so its smoke has a session, which is why this repository
mirrors the 1332 MiB full ISO rather than the 392 MiB minimal one: the overlay installs from the
medium and needs no network.

## The installed disk is started by its own loader, not by the firmware

Every guest here boots a disk written with a whole-disk EFI layout, and the console output differs
between them in a way that matters for what a smoke stage can wait on:

- **OpenIndiana** hands its console to the framebuffer once the kernel starts, so `console.log` stops
  at `Booting...` and the login prompt is on the *screen*. `smoke.walk` reads the frame.
- **OmniOS and Tribblix** keep writing to the serial line as well as the 1024x768 VGA, and both print
  `omnios console login:` / `tribblix console login:`. Their smoke stages read the frame too, which
  works either way.

## The media is mirrored, not fetched from upstream

A stage pulls its installer image from this repository's ghcr package, never from the publisher:
`artifacts-publish.yml` downloads each pinned URL once, checks its digest, packs it into a tar and
pushes one tag. Every stage afterwards asks the package.

```sh
gh workflow run artifacts-publish.yml -f which=iso-openindiana-2026.10-minimal
gh workflow run artifacts-publish.yml -f which=iso-omnios-r151060
gh workflow run artifacts-publish.yml -f which=iso-tribblix-0m41.1

gh workflow run openindiana-2026.10-install.yml
gh workflow run omnios-r151060-install.yml
gh workflow run tribblix-0m41.1-install.yml

gh workflow run openindiana-2026.10-smoke.yml
gh workflow run omnios-r151060-smoke.yml
gh workflow run tribblix-0m41.1-smoke.yml
```

## What lives elsewhere, on purpose

The **engine** and the **CI machinery** are both in `easydev-org/easyvm`; these workflows call
`easydev-org/easyvm/.github/workflows/stage.yml@v0.0.4` for the second.

Disks pull from and publish to this repository's own ghcr package, `easydev-org/easyvm-solaris`.

