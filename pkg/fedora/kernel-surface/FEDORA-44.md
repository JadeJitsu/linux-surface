# Building kernel-surface on Fedora 44

The pre-built [linux-surface Fedora repository](https://pkg.surfacelinux.com/fedora/)
currently ships builds up to **Fedora 43** (kernel 6.18.x). There is no `f44/`
repository yet, so on **Fedora 44** you build the kernel from source using
[Kernel Ark](https://gitlab.com/cki-project/kernel-ark).

This directory contains everything the build needs:

- `build-linux-surface.py` — top-level driver. Pins the kernel-ark tag and
  feeds it the surface patches + config.
- `build-ark.py` — clones/updates kernel-ark, applies patches and config
  overrides, and runs `make dist-rpms`.
- `patches/` — Fedora-specific spec-template patches.
- `configs/fedora.config` — Fedora config fragment.
- `secureboot/` — optional Secure Boot signing keys (see below).

The surface patch set itself lives in the top-level
[`patches/6.19/`](../../../patches/6.19/) and
[`configs/surface-6.19.config`](../../../configs/surface-6.19.config),
which `build-linux-surface.py` locates automatically from `PACKAGE_TAG`.

## 1. Install build dependencies

Base tooling:

```bash
sudo dnf install -y git make gcc flex bison bzip2 rpm-build python3-pyyaml
```

Kernel Ark has a long `BuildRequires` list. The easiest way to install exactly
what the pinned kernel needs is to let it tell you. Clone the pinned kernel-ark
tag first (the build script will reuse this clone):

```bash
cd pkg/fedora/kernel-surface
git clone https://gitlab.com/cki-project/kernel-ark
cd kernel-ark
git fetch --tags && git checkout kernel-6.19.8-0   # must match PACKAGE_TAG
make dist-get-buildreqs
# install whatever it reports as missing:
sudo dnf -y install $(make dist-get-buildreqs | grep "Missing dependencies:" | cut -d":" -f2-)
```

> `system-sb-certs` (if listed) comes from the RHEL/CentOS-Stream CRB repo.

## 2. Build the RPMs

```bash
cd pkg/fedora/kernel-surface
python3 build-linux-surface.py --mode rpms
```

This:

1. Clones `kernel-ark` (or reuses `./kernel-ark`) and checks out
   `PACKAGE_TAG` (currently `kernel-6.19.8-0`).
2. Applies the `patches/6.19/*.patch` surface set.
3. Applies the Fedora-local `patches/*.patch` and `configs/fedora.config`.
4. Runs `make dist-rpms` with `SPECPACKAGE_NAME=kernel-surface`,
   `DISTLOCALVERSION=.surface`, and the reduced build options
   (`+up +baseonly -debuginfo -doc -headers -efiuki`).

Built RPMs are copied to `./out/`.

> **Note on the pinned tag.** `build-linux-surface.py` pins
> `PACKAGE_TAG = "kernel-6.19.8-0"`. The 6.19 surface patch set is generated
> against that tree and applies cleanly (all 24 base + 6 Fedora-local patches
> verified). Do not bump the tag without re-validating the patch set — newer
> kernel-ark 6.19.x tags have since merged an *upstream* version of the
> dirty-frag rxrpc fixes, which conflicts with the surface copies of those
> patches (0020/0021).

## 3. Secure Boot (optional)

If `secureboot/MOK.crt` and `secureboot/MOK.key` are absent, the build falls
back to Red Hat's test keys and the resulting kernel **will not boot with
Secure Boot enabled**. To sign your own keys, follow the upstream
[Secure Boot wiki page](https://github.com/linux-surface/linux-surface/wiki/Secure-Boot)
and drop the cert/key into `secureboot/` before building.

## 4. Install

```bash
sudo dnf install -y ./out/kernel-surface-*.rpm
# for building out-of-tree modules (e.g. DKMS):
sudo dnf install -y ./out/kernel-surface-devel-*.rpm
```

Also install the companion utilities from the pre-built repo (these are not
kernel-version-specific and work on F44):

```bash
sudo dnf config-manager addrepo --from-repofile=https://pkg.surfacelinux.com/fedora/linux-surface.repo
sudo dnf install -y iptsd libwacom-surface
```

> The `linux-surface.repo` uses `f$releasever/`, which does not exist for f44
> yet. If `dnf` errors on the repo, install `iptsd` / `libwacom-surface` from
> the `f43/` directory directly, or build them from source.

## 5. Make it the default kernel

Fedora boots the newest kernel by default, which will be the stock 7.x kernel.
Use the helper in
[`contrib/fedora-default-kernel/`](../../../contrib/fedora-default-kernel/)
to pin the surface kernel as the default, then reboot.

## Troubleshooting

- **Stuck at boot loading the ramdisk (Intel):** install the Processor
  Microcode firmware (`sudo dnf install -y microcode_ctl` or via
  Additional Drivers).
- **Patch conflict on a newer tag:** see the pinned-tag note in step 2.
- **`make dist-rpms` fails on a missing build dep:** re-run
  `make dist-get-buildreqs` and install what it reports.
