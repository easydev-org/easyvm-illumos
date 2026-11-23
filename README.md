# easyvm-illumos

Walkthroughs and CI for **OpenIndiana** guests, driven by
[`easydev-org/easyvm`](https://github.com/easydev-org/easyvm).

## What lives here

`openindiana/steps/` - the `.walk` files for `install` and `smoke`. The other stages (`ssh`,
`vm-configure`) are added once an install produces a disk that boots.

## The install is a text installer, and it is driven like one

`openindiana/steps/install.walk` is 45 steps and was driven end to end by hand on the pod before it
was committed, then run on CI: `openindiana-2026.10-install.yml` reached `completed/success` and
published a 1049 MiB disk. The parts that are not guessable, each measured:

- **`ret` does not move between fields on the Users page.** `admin` typed, Return pressed, cursor
  still in Root password, second `admin` landed in the same box. `down` is the field mover, so each
  of the six fields is typed and then stepped away from.
- **The Network page's red bottom line is a hint, not an error.** It stays after "Automatically" is
  selected, and F2 is what moves on. The cursor starts in Computer Name, one `down` from the first
  radio row, and `spc` selects it.
- **The GPT warning's cursor starts on Cancel**, so it takes `right` then Return. Without the
  `right` the dialog closes and nothing is installed.
- **The Fdisk page proposes and preselects "Use the whole disk (EFI)"**, which matches the GPT label
  the warning just accepted, so F2 is enough.
- **The time zone default is UTC/GMT**, and the date page is preloaded from the firmware clock.

The installed disk is then started by **illumos's own loader in the disk's boot slice**, not by a
firmware ESP entry. `smoke.walk` therefore reads the **screen**, not the serial console: the illumos
kernel hands its console to the framebuffer once it starts, so `console.log` stops at `Booting...`
and never receives the `openindiana console login:` prompt that a complete boot prints. Measured on
run 45226041744, where the recording showed that prompt while the console read timed out.

## What makes this guest different

OPENINDIANA IS AN ILLUMOS DISTRIBUTION - illumos is the kernel, and this is one of the
systems built on it, the way Ubuntu is built on Linux. Its package manager is `IPS` (`pkg install`)
and its service manager is `SMF` (`svcadm`), neither of which resembles systemd. THE MINIMAL IMAGE IS
THE ONE MIRRORED: 543 MiB against the live DVD's 2.47 GiB, and an install needs no live desktop.

## The media is mirrored, not fetched from upstream

A stage pulls its installer image from this repository's ghcr package, never from the publisher:
`artifacts-publish.yml` downloads `iso-openindiana-2026.10-minimal`, checks its digest, packs it into a tar and pushes one
tag. Every stage afterwards asks the package.

```sh
gh workflow run artifacts-publish.yml -f which=iso-openindiana-2026.10-minimal
gh workflow run openindiana-2026.10-install.yml
gh workflow run openindiana-2026.10-smoke.yml
```

## What lives elsewhere, on purpose

The **engine** and the **CI machinery** are both in `easydev-org/easyvm`; these workflows call
`easydev-org/easyvm/.github/workflows/stage.yml@v0.0.3-dev` for the second.

Disks pull from and publish to this repository's own ghcr package, `easydev-org/easyvm-illumos`.
