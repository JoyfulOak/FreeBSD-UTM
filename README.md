# FreeBSD UTM

## Hyprland on FreeBSD in UTM on Apple Silicon

Hyprland is a Wayland compositor. This guide focuses on getting Hyprland running; the `scfb` configuration from [BSDJedi's video](https://www.youtube.com/watch?v=__TrwS4u52I), “FreeBSD - how to do Xorg graphics on UTM on MacOS/arm64,” applies to Xorg and does not configure Hyprland.

### Instructions for the coding agent inside the VM

Give the VM-side coding agent this task: **“Use this README to install, configure, and launch Hyprland in this FreeBSD UTM VM. Inspect the guest first, then make and verify the needed changes. Continue through package installation and configuration instead of stopping after diagnosis. Preserve backups of files you replace, and report commands run, changes made, and any blocker.”**

The agent should:

1. Inspect the FreeBSD release/architecture, UTM display device, available GPU/render nodes (such as `/dev/dri/card0` and `/dev/dri/renderD128`), current user, installed packages, and available disk space. Check the FreeBSD Wayland and Hyprland instructions for this release.
2. Confirm that this GPU exposes the DRM/KMS device Hyprland needs. A framebuffer device such as `/dev/fb0` and the `scfb` Xorg driver alone do not establish Wayland/DRM support. Do not claim the Xorg workaround below enables Hyprland.
3. If the device is UTM's `virtio-gpu-pci`, check whether the FreeBSD package `virtio-gpu-qemu-kmod` is available for this release and architecture. If it is, install as root with `pkg install virtio-gpu-qemu-kmod` and inspect its package message. Back up `/boot/loader.conf` and add `virtio_gpu_qemu_load="YES"` if needed. Keep UTM's display set to `virtio-gpu-pci` and reboot. The module must load at boot before the in-kernel VirtIO GPU driver claims the device. This package documents framebuffer mmap and an Xorg `scfb` configuration; verify separately whether it provides the DRM/KMS interface Hyprland needs. If the package is unavailable, record the exact package error and UTM version, and check current FreeBSD Ports support.
4. If Hyprland and its dependencies are available for this guest, install the `hyprland`, `wayland`, and `seatd` packages (or use the package's dependency resolution), add the user to the `video` group if needed, and enable/start `seatd` as described in the FreeBSD Handbook. Configure D-Bus/runtime environment only as required by the installed Hyprland package and FreeBSD version.
5. Start Hyprland from the graphical user's local TTY using the command and session setup documented by the installed package. Do not configure a login manager unless required by the current FreeBSD instructions.
6. Verify Hyprland's startup output/logs, active Wayland session, and visible UTM display. If it fails, diagnose the specific DRM, seatd, permission, or package error and report the blocker; do not substitute Xorg configuration for missing Wayland graphics support.

The guest may require root privileges for packages and system configuration. If the agent lacks them, it should identify the exact commands that need root and continue with all independent user-level setup and diagnostics.

## Xorg-only resolution workaround from the video

The following configuration was posted to fix Xorg resolution in the video setup. It is for Xorg only and is not used by Hyprland. Use it only when separately configuring Xorg in this UTM guest. Save it as `/usr/local/share/X11/xorg.conf.d/00-monitor.conf` as the comment specifies, after checking Xorg's active config directory:

```conf
Section "Screen"
    Identifier "Screen0"
    Device "Card0"
    Monitor "Monitor0"
    DefaultDepth 24
    Option "ConnectedMonitor" "1280x800"
    SubSection "Display"
        Depth 24
        Modes "1280x800"
    EndSubSection
EndSection

Section "Device"
    Identifier "Card0"
    Driver "scfb"
    Option "RandR" "on"
    Option "ShadowFB" "on"
EndSection

Section "ServerLayout"
    Identifier "Layout0"
    Screen 0 "Screen0"
    Option "Xinerama" "on"
EndSection
```

The video describes a separate custom kernel patch for 2D framebuffer output. A FreeBSD port named `graphics/virtio-gpu-qemu-kmod` was added for UTM/QEMU VirtIO GPU guests in July 2026 ([FreeBSD Ports commit](https://cgit.freebsd.org/ports/commit/?id=6b825bb84ccd0d0aee943776c21e250a6bf35c23)). Its module needs boot-time preloading and a reboot. The port description refers to framebuffer mmap and Xorg; do not assume it provides the DRM/KMS interface needed by Hyprland. Check device nodes and current driver documentation.

The video comment reports occasional redraw problems after switching virtual terminals (`Ctrl`+`Alt`+`F1`, then `Ctrl`+`Alt`+`F9`), and suggests `xrefresh` from an X session. This is specific to Xorg and does not diagnose Hyprland.

See [the detailed UTM graphics guide](docs/utm-hyprland.md) for the video context. The [FreeBSD Handbook's Wayland chapter](https://docs.freebsd.org/en/books/handbook/wayland/) documents Wayland setup, the `video` group, and `seatd`; the [Hyprland installation guide](https://wiki.hypr.land/getting-started/installation/) lists the FreeBSD package option.

