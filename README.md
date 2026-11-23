# easyvm-illumos

Walkthroughs and CI for **OpenIndiana** guests, driven by
[`easydev-org/easyvm`](https://github.com/easydev-org/easyvm).

## What lives here

`openindiana/steps/` - the `.walk` files for `install` and `smoke`, plus the reference images they
compare against. The other stages (`ssh`, `vm-configure`) are added once an install produces a disk
that boots.

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
