# FreeBSD Hyprland graphics in UTM on Apple Silicon

This note records the UTM framebuffer information from a community video and how it relates to the goal of running Hyprland, a Wayland compositor, on FreeBSD/arm64 under UTM on Apple Silicon. The Xorg workaround below is not a Hyprland configuration.

## Video and reported workaround

The source is [“FreeBSD - how to do Xorg graphics on UTM on MacOS/arm64” by BSDJedi](https://www.youtube.com/watch?v=__TrwS4u52I). The video description says it uses the author's kernel patch to enable 2D video in FreeBSD under UTM, without hardware acceleration.

The video's update comment says to save the following as `/usr/local/share/X11/xorg.conf.d/00-monitor.conf`:

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

The comment reports that switching away from X and back with `Ctrl`+`Alt`+`F1`, then `Ctrl`+`Alt`+`F9`, may leave the display undrawn; it suggests running `xrefresh` from the X session. The VT numbers can vary by system.

## Important distinction for Hyprland

Hyprland uses Wayland and needs a working DRM/KMS graphics path. A framebuffer such as `/dev/fb0`, the `scfb` Xorg driver, and the 1280x800 file below are not by themselves evidence that Wayland compositing will work. Check for the DRM device nodes and confirm driver support for the exact FreeBSD version and UTM display adapter before treating the graphics setup as complete.

The FreeBSD port `graphics/virtio-gpu-qemu-kmod` was added for UTM/QEMU VirtIO GPU guests in July 2026 ([FreeBSD Ports commit](https://cgit.freebsd.org/ports/commit/?id=6b825bb84ccd0d0aee943776c21e250a6bf35c23)). Its package message says the module must be loaded at boot using `virtio_gpu_qemu_load="YES"` in `/boot/loader.conf` and requires a reboot. The port description discusses framebuffer mmap and its installed Xorg `scfb` configuration; do not assume it supplies the DRM/KMS interface Hyprland needs. If `pkg install virtio-gpu-qemu-kmod` cannot find a compatible package, record the exact error and UTM version and check current Ports support rather than silently switching display devices.

For the Wayland session, follow the current [FreeBSD Handbook Wayland chapter](https://docs.freebsd.org/en/books/handbook/wayland/) and [Hyprland installation guide](https://wiki.hypr.land/getting-started/installation/). The Handbook documents `wayland`, `seatd`, the `video` group, and starting `seatd` before a compositor. Check that `hyprland` is available for the guest release and architecture, then start it from a local TTY using the package's current instructions.

## Xorg-only information from the video

The following checks apply only to Xorg. They do not configure Hyprland.

- Confirm the VM is FreeBSD/arm64 under UTM and identify the FreeBSD release and UTM display device. The video is specific to this setup.
- The video describes a custom kernel patch. The comment above only provides the Xorg configuration; it does not provide kernel-patch installation steps. Do not invent or apply kernel changes based on this note. Find and review the patch source and instructions first, and explain any kernel rebuild or boot changes before making them.
- FreeBSD's current Handbook documents Xorg configuration snippets under `/usr/local/etc/X11/xorg.conf.d/`, while this video comment specifically says `/usr/local/share/X11/xorg.conf.d/`. Check the guest's Xorg log for the active configuration directories and use the path actually read by that Xorg installation. Do not create duplicate conflicting copies.
- Check whether the `scfb` Xorg driver is installed and whether Xorg is actually selecting it. The Handbook lists `scfb` as the System Console Framebuffer driver and documents configuration files in its [X11 chapter](https://docs.freebsd.org/en/books/handbook/x11/).
- Preserve and inspect existing Xorg configuration before editing. The `ConnectedMonitor` option and `Xinerama` setting are reproduced from the video comment; verify their effect in this guest rather than assuming they are required or supported by its driver setup.
- A successful Xorg session only establishes Xorg display output. It does not demonstrate that a Wayland/DRM compositor can start.

## Handoff prompt for Codex inside the VM

Have the VM-side Codex read the repository README and this guide, then complete the Hyprland setup in the live guest. It should inspect the FreeBSD release/architecture, UTM display adapter, current user, available disk space, available DRM/render nodes, installed Hyprland/Wayland/seatd packages, and relevant kernel logs. Confirm that the graphics stack provides the DRM/KMS interface Hyprland needs; do not treat `/dev/fb0` or an Xorg `scfb` config as sufficient. For a `virtio-gpu-pci` device, check package support for `virtio-gpu-qemu-kmod`; if available, install it, back up `/boot/loader.conf`, add `virtio_gpu_qemu_load="YES"`, keep UTM's display set to `virtio-gpu-pci`, and reboot, while verifying separately whether this driver supports Hyprland's DRM/KMS requirements. If the package cannot be found, capture the exact error and UTM version and report the blocker. If the graphics path is supported, install Hyprland and its needed Wayland/seatd dependencies, add the user to the `video` group if necessary, enable/start `seatd`, and launch Hyprland from the local TTY using the installed package's current instructions. Verify the active Wayland session and visible output. Do not configure another compositor or Xorg session as a substitute. Do not guess custom kernel patch instructions or apply an unreviewed kernel change. If root privileges are unavailable, finish diagnostics and user-level setup and list the exact privileged commands still needed. The video's `xrefresh` workaround applies only to Xorg VT redraw problems.
