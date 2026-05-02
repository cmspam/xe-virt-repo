# xe-virt-repo

Pacman repositories of patched Mesa, virglrenderer, and intel-media-driver for Arch and CachyOS, built x86-64-v3. Enables OpenGL, Vulkan, and VA-API hardware video acceleration inside QEMU/KVM guests using Intel Xe GPUs, via virtio-gpu native context. No GPU passthrough required.

Two repos:

- `xe-virt-host-v3` for the **host** (the machine running QEMU)
- `xe-virt-guest-v3` for the **guest** (the Arch / CachyOS VM)

Both are needed for full functionality.

## Setup on the host

Add to `/etc/pacman.conf`, above the `[extra]` line:

```ini
[xe-virt-host-v3]
SigLevel = Optional
Server = https://github.com/cmspam/xe-virt-repo/releases/download/latest-host
```

Then:

```bash
sudo pacman -Syu virglrenderer
```

This installs a patched virglrenderer with `-Ddrm-renderers=xe-experimental` enabled. It's what handles guest GPU commands on the host side.

## Setup in the guest

Add to `/etc/pacman.conf` inside the VM, above `[extra]`:

```ini
[xe-virt-guest-v3]
SigLevel = Optional
Server = https://github.com/cmspam/xe-virt-repo/releases/download/latest-guest
```

Then:

```bash
sudo pacman -Syu mesa lib32-mesa intel-media-driver
```

Reboot the guest after install so all running apps pick up the new Mesa.

## What each package does

| Package | Side | Role |
|---------|------|------|
| `virglrenderer` | Host | Accepts virtio-gpu commands from the guest, replays them against the real Intel Xe driver. |
| `mesa`, `lib32-mesa` | Guest | OpenGL (iris) and Vulkan (ANV) inside the VM, talking to the host GPU over virtio. 32-bit Mesa is for Steam / Wine / 32-bit games. |
| `intel-media-driver` | Guest | Hardware-accelerated VA-API (video decode and encode) via the iHD driver. Used by browsers, ffmpeg, mpv, OBS, etc. |

## QEMU flags

Use the stock `qemu` package from Arch / CachyOS. The minimum flags to enable Xe native context:

```
-accel kvm,honor-guest-pat=on
-device virtio-vga-gl,blob=on,hostmem=4G,drm_native_context=on
```

`drm_native_context=on` is the key one. Adjust `hostmem` based on your workload (4G is fine for most desktop use; bump it for heavy graphics or VRAM-hungry games).

## VA-API in the guest

The `intel-media-driver` package automatically sets `LIBVA_DRIVER_NAME=iHD` for new shell sessions and graphical user sessions, only if you haven't already set it yourself. Files installed:

- `/etc/profile.d/xe-virt-vaapi.sh` (login shells)
- `/usr/lib/environment.d/90-xe-virt-vaapi.conf` (systemd user sessions)

Both are tracked by pacman and removed when you uninstall the package. Override in your dotfiles or `~/.config/environment.d/` if you need a different driver.

To verify VA-API is working in the guest:

```bash
vainfo
```

You should see entries for `iHD` and a list of supported codecs (H.264, HEVC, VP9, AV1 depending on your hardware).

## Things to watch out for

- **Pre-Broadwell Intel GPUs are not supported.** The iHD driver only supports Broadwell (5th gen) and newer. If you have an older Intel iGPU and need VA-API, do not use `xe-virt-guest-v3`. There is no fallback in this repo.
- **The guest must be running Wayland or X11 with the patched Mesa to get full GPU acceleration.** Headless / TTY use only benefits VA-API.
- **Use the same release channel on host and guest.** Mismatched virtio-gpu protocol versions between host virglrenderer and guest Mesa can cause crashes. Both repos rebuild together and stay in sync.
- **Updates are automatic.** New releases of Mesa, virglrenderer, and intel-media-driver from Arch's packaging are picked up within a day of upstream. Just `pacman -Syu` periodically.

## Source

Patches come from [cmspam/xe-native-context-enablement](https://github.com/cmspam/xe-native-context-enablement). PKGBUILDs come from [Arch GitLab packaging](https://gitlab.archlinux.org/archlinux/packaging/packages). Built in `cachyos/cachyos-v3` Docker.

## License

GPL-2.0 for the build scripts. Patches retain their upstream licenses.
