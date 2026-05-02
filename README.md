# xe-virt-repo

Prebuilt CachyOS / Arch Linux pacman repositories (built `-march=x86-64-v3 -mtune=generic`) for the [Intel Xe native-context virtualization stack](https://github.com/cmspam/xe-native-context-enablement). Lets you run OpenGL, Vulkan, and VA-API hardware video decode/encode inside a QEMU/KVM guest by routing DRM commands from the guest to the host's Intel Xe kernel driver — no full GPU passthrough required.

Packages are rebuilt automatically twice daily, tracking Arch upstream tags. Patches come from [`cmspam/xe-native-context-enablement`](https://github.com/cmspam/xe-native-context-enablement) and roll in automatically when updated there.

---

## Quick setup

The repositories live as GitHub Releases on this repo. You add them to `pacman.conf` like any third-party Arch repo.

### On the host (Arch / CachyOS with Intel Xe GPU)

Add this **above** the `[extra]` section in `/etc/pacman.conf`:

```ini
[xe-virt-host-v3]
SigLevel = Optional
Server = https://github.com/cmspam/xe-virt-repo/releases/download/latest-host
```

Then:

```bash
sudo pacman -Syu virglrenderer
```

You now have a `virglrenderer` built with `-Ddrm-renderers=xe-experimental`, which is what handles guest GPU commands on the host.

### Inside the guest VM (Arch / CachyOS)

Add this **above** the `[extra]` section in `/etc/pacman.conf`:

```ini
[xe-virt-guest-v3]
SigLevel = Optional
Server = https://github.com/cmspam/xe-virt-repo/releases/download/latest-guest
```

Then:

```bash
sudo pacman -Syu mesa lib32-mesa intel-media-driver
```

You now have:
- `mesa` / `lib32-mesa` with the `intel-virtio-experimental` Mesa feature enabled (powers iris OpenGL + ANV Vulkan over virtio)
- `intel-media-driver` patched with the vdrm shim (powers VA-API hardware video over virtio)

The `intel-media-driver` package also ships:
- `/etc/profile.d/xe-virt-vaapi.sh` — sets `LIBVA_DRIVER_NAME=iHD` for shell sessions if you haven't set it yourself
- `/usr/lib/environment.d/90-xe-virt-vaapi.conf` — same default for systemd-managed user sessions (graphical apps)

Both yield to a user-set `LIBVA_DRIVER_NAME` (e.g. in your dotfiles or `~/.config/environment.d/`), and both are uninstalled cleanly with `pacman -R intel-media-driver`.

That's it. Reboot the guest after install so Mesa changes take effect for every running app.

**Caveat**: this repo targets modern Intel Xe hardware. The iHD VA-API driver doesn't support pre-Broadwell iGPUs — if you accidentally install `xe-virt-guest-v3` on a host with old Intel hardware that needs the i965 driver, override `LIBVA_DRIVER_NAME` in your shell config or remove the package.

---

## QEMU configuration (host)

The Arch/CachyOS stock `qemu` package is recent enough — no special build needed. The minimum required options:

```
-accel kvm,honor-guest-pat=on
-device virtio-vga-gl,blob=on,hostmem=4G,drm_native_context=on
```

`drm_native_context=on` is the key flag. Without it, you get software rendering or full virgl translation, not Xe passthrough.

---

## What's actually in the repos

| Repo | Package | Patch | Extra build flag |
|------|---------|-------|------------------|
| `xe-virt-host-v3` | `virglrenderer` (+ split subpackages, if any) | [virglrenderer-xe-native-context.patch](https://github.com/cmspam/xe-native-context-enablement/blob/master/virglrenderer-xe-native-context.patch) | `-Ddrm-renderers=xe-experimental` |
| `xe-virt-guest-v3` | `mesa` (+ split subpackages: `vulkan-intel`, `opencl-clover-mesa`, etc.) | [mesa-01-xe-native-context-plus-iris-upload-fix.patch](https://github.com/cmspam/xe-native-context-enablement/blob/master/mesa-01-xe-native-context-plus-iris-upload-fix.patch) | `-D intel-virtio-experimental=true` |
| `xe-virt-guest-v3` | `lib32-mesa` (+ split subpackages) | same as `mesa` | same as `mesa` |
| `xe-virt-guest-v3` | `intel-media-driver` | [intel-media-driver-xe-native-context.patch](https://github.com/cmspam/xe-native-context-enablement/blob/master/intel-media-driver-xe-native-context.patch) | (none — patch is self-contained) |

All packages are built inside the `cachyos/cachyos-v3` Docker image, which sets `-march=x86-64-v3 -mtune=generic` in `/etc/makepkg.conf`. So everything you install is v3-optimized.

The `latest-host` and `latest-guest` GitHub Release URLs are stable — they get force-replaced on each new build. Older versions are not retained as releases (use `last_built_versions.txt` git history to know what was published when).

---

## Build pipeline (how the automation works)

[`.github/workflows/pkgbuild.yml`](./.github/workflows/pkgbuild.yml) runs twice a day on cron and on `workflow_dispatch`. Each run:

1. Polls Arch GitLab for the latest packaging tags of `virglrenderer`, `mesa`, `lib32-mesa`, `intel-media-driver`.
2. Diffs against [`last_built_versions.txt`](./last_built_versions.txt) — only rebuilds packages whose Arch tag changed.
3. For each package needing a rebuild:
   - Pulls the Arch `PKGBUILD` (and any companion files) at the latest tag.
   - Pulls the corresponding patch from `xe-native-context-enablement`.
   - Injects the patch into `source=()` + `prepare()`, marks `'SKIP'` in checksum arrays.
   - Injects the extra meson flag into `meson_options=()` (mesa, lib32-mesa) or directly into the `meson` invocation (virglrenderer).
   - Builds inside `cachyos/cachyos-v3` via `makepkg`.
4. Splits outputs by bucket (host / guest), merges them with the previous release's packages (so a mesa-only rebuild doesn't drop virglrenderer from the host repo... wait, those are different repos — but the principle holds within each repo: a mesa-only rebuild keeps the existing intel-media-driver pkgs in `latest-guest`).
5. Runs `repo-add` against the merged set to produce a fresh pacman db.
6. Force-replaces `latest-host` and/or `latest-guest` GitHub Releases with the new files.
7. Commits the updated `last_built_versions.txt` back to `main`.

To trigger a manual build: **Actions → Build and Publish xe-virt packages → Run workflow**.

### Failure modes that need a human

- **Arch restructures a PKGBUILD.** The patch-injection script is structural; if Arch renames `meson_options` or moves the configure call, injection fails loudly. Fix: update the matrix `inject_method` or the inject script.
- **A patch fails to apply on a new upstream version.** Build fails on the `patch -Np1` step. Fix: rebase the patch in `xe-native-context-enablement` and push there. Next cron run picks it up automatically.

GitHub Actions will email you on workflow failure. There is no silent staleness — either it ran and published, or it failed and you know.

---

## Companion repositories

- [**cmspam/xe-native-context-enablement**](https://github.com/cmspam/xe-native-context-enablement) — the upstream patches (this repo pulls them at build time).
- [**cmspam/qemu-patched**](https://github.com/cmspam/qemu-patched) — patched QEMU build with extra refresh-rate / VAAPI / latency tweaks (optional companion).

---

## Why this exists

Building Mesa from source on a typical user machine takes 30–60 minutes. Multiply that by the patch updates upstream gets, and most users never bother — they fall back to bare-metal i915, GPU passthrough, or stock virgl. This repo centralizes the build cost: one CI run produces v3-optimized binaries everyone can `pacman -Syu`. And because the cron tracks Arch tags directly, you never end up running a months-old build of mesa just because your patch repo went stale.

---

## License

The workflow scripts in this repository are licensed GPL-2.0 (see [LICENSE](./LICENSE)). The patches retain their upstream project licenses — see [xe-native-context-enablement](https://github.com/cmspam/xe-native-context-enablement) for details.
