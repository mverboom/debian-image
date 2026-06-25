# debian-image — internals

Implementation details for the `debian-image` toolkit: the internal `_cleanup`
companion script, what the per-user-facing scripts actually do internally,
caveats, architecture mapping, and the project layout. This is reference
material for maintainers and people debugging a build; you do **not** need it
to use the toolkit — see [README.md](README.md) for usage.

## `_cleanup` (internal)

This is the companion script that `create-debian-rootfs` bind-mounts into the
build container at `/root/cleanup.sh` and runs there. **Do not run it
directly.** All of its configuration is passed in via environment variables
(`-e` flags in `create-debian-rootfs`): `WITH_PVE`, `WITH_BACKPORTS`,
`WITH_ZFS`, `WITH_KERNEL`, `WITH_REFIND`, `CODENAME`, `ROOT_PASSWORD`,
`FULL_PKG_LIST`, `PVE_KEYRING_URL`, `PVE_SUITE`, `PVE_FORMAT`, `KERNEL_PKG`,
`HEADERS_PKG`, `BACKPORTS_KERNEL_PKGS`, `BACKPORTS_ZFS_PKGS`, `MAIN_PKG_LIST`,
`ESPROOT`.

It is responsible for:

- `apt-get update` / package installation (Debian, backports, ZFS repos, or
  the Proxmox repo depending on flags).
- Downloading and installing the Proxmox keyring; adding the
  `pve-no-subscription` repo (DEB822 for trixie+, legacy format otherwise);
  disabling the PVE enterprise repo; removing the stock Debian kernel and
  `os-prober` for PVE builds.
- Writing `RESUME=none` and `noresume` so the initramfs doesn't wait for a
  non-existent resume/swap device.
- Generating the initramfs for the installed kernel.
- Force-creating essential `/sbin/init`, `/sbin/halt`, `/sbin/reboot`, …
  symlinks that maintainer scripts sometimes skip in containers.
- Setting / unlocking the root password.
- Stripping Docker artifacts (`.dockerenv`, `Dockerfile`), apt caches/lists,
  `/tmp` and `/var/tmp`, hostname, `resolv.conf`, `machine-id`, logs, SSH host
  keys, udev persistent-net rules, user caches, bash history, `lost+found`.
- Re-creating essential mount-point and `if-up.d`/`if-down.d` directories.
- Staging rEFInd onto `${ESPROOT}/EFI/refind` when requested.

### What gets cleaned up

| Item | Action |
| --- | --- |
| `.dockerenv`, `Dockerfile` | Removed |
| `/var/cache/apt/` | Cleaned (no `.deb` files) |
| `/var/lib/apt/lists/` | Removed |
| `/tmp/`, `/var/tmp/` | Emptied |
| `/etc/hostname` | Cleared |
| `/etc/resolv.conf` | Cleared |
| `/etc/machine-id` | Truncated (regenerated on first boot) |
| `/var/log/*` | Cleaned/truncated |
| `/etc/ssh/ssh_host_*` | Removed (regenerated on first boot) |
| `/root/.bash_history` | Removed |
| `/lost+found` | Removed |

Repository configurations (`/etc/apt/sources.list*`,
`/etc/apt/sources.list.d/backports.list`,
`/etc/apt/sources.list.d/pve-install-repo.*`) are **preserved** so that
`apt-get` works on the deployed system. Backports, contrib/non-free
components, and the PVE repository are retained if `-B`, `-z`, or `-P` was used.

## `test-rootfs` — what it checks

- Critical top-level directories (`bin`, `etc`, `lib`, `usr`, `var`, `proc`,
  `sys`, `dev`, `run`, `boot`).
- Critical binaries/config (`/bin/sh`, `/bin/bash`, `/etc/passwd`,
  `/etc/shadow`, `/etc/fstab`, `/etc/os-release`, `sshd`, `ssh`, …).
- No leftover Docker artifacts (`.dockerenv`, `Dockerfile`).
- Clean apt cache (no `.deb` files).
- Ownership/permissions from tar headers: `/etc/shadow` `0/42` mode `640`,
  `/etc/passwd` `0/0` mode `644`, `/usr/bin/su` setuid, `/tmp` sticky bit,
  presence of setuid/setgid files, ownership diversity, and that the boot
  tarball is all root-owned.
- `/etc/machine-id` is empty/absent.
- `/etc/os-release` identifies as Debian.
- Optional: kernel+initramfs, rEFInd, ZFS, PVE (see the `-k`/`-r`/`-z`/`-P`
  flags).

Exits `0` if all checks pass, `1` if any fail.

## `setup-efi` — what it does

1. Detects or creates an ESP on the target disk (GPT label if none, FAT32,
   `boot`/`esp` flags), or uses an existing partition (`-p`), or an
   already-mounted directory (`-e`).
2. Detects the latest kernel and matching initramfs in the rootfs.
3. Copies rEFInd (EFI binary, sample config, icons, tools) from the rootfs to
   `${ESP_DIR}/EFI/refind`.
4. Generates a `refind.conf` pointing at the detected kernel/initramfs, with
   `noresume` and a placeholder `root=ROOT_PART` you must edit.
5. Registers rEFInd as the default EFI boot entry via `efibootmgr` (unless
   `-n`).
6. Unmounts the ESP if it mounted it itself (EXIT trap).

## Notes & caveats

- `create-debian-rootfs` reaps leftover `debian-rootfs-build-*` containers from
  interrupted previous runs at startup, and removes its own container via an
  EXIT/INT/TERM/HUP/QUIT trap.
- The split layout deliberately keeps `/boot` as an empty mount point in the
  rootfs tarball so you can mount a separate `/boot` partition onto it.
- `/etc/shadow` ownership (`root:shadow`, `0/42`) and setuid bits are
  preserved by manipulating the tar stream directly rather than extracting to
  disk, which a non-root build user could not preserve.
- For PVE builds, `ifupdown` is **not** installed (PVE manages networking);
  the PVE enterprise repository is disabled in favor of `pve-no-subscription`.
- `setup-efi` always tells you to edit `ROOT_PART` in `refind.conf`; it cannot
  guess your root partition. Use `blkid` to find the UUID.

## Architecture support

| Architecture | Kernel package | `--arch` |
| --- | --- | --- |
| x86-64 | `linux-image-amd64` | `amd64` |
| ARM 64-bit | `linux-image-arm64` | `arm64` |
| ARM 32-bit | `linux-image-armmp` | `armhf` |

## Project structure

```
.
├── create-debian-rootfs   # Build a rootfs + boot tarball (host, uses Docker)
├── _cleanup               # Configure & clean the rootfs (inside the container)
├── test-rootfs            # Verify a produced tarball (host)
├── setup-efi              # Install rEFInd on the ESP (host, needs root)
├── README.md              # Usage guide (user-facing)
└── INTERNALS.md           # This file (implementation details)
```

`_cleanup` is an internal companion script — you never invoke it directly;
`create-debian-rootfs` bind-mounts it into the build container.