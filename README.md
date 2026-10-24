# FreeBSD UTM

## FreeBSD desktop graphics in UTM on Apple Silicon

The following Xorg configuration is a community-reported workaround from [BSDJedi's video](https://www.youtube.com/watch?v=__TrwS4u52I), “FreeBSD - how to do Xorg graphics on UTM on MacOS/arm64.” It selects the `scfb` framebuffer driver and requests a 1280x800 display mode. It is a troubleshooting lead for FreeBSD/arm64 under UTM on Apple Silicon, not a guarantee for every FreeBSD release or UTM display device.

### Instructions for the coding agent inside the VM

Give the VM-side coding agent this task: **“Use this README to get a working XFCE desktop displayed in this FreeBSD UTM VM. Inspect the guest first, then make and verify the needed changes. Continue through package installation and configuration instead of stopping after diagnosis. Preserve backups of files you replace, and report commands run, changes made, and any blocker.”**

The agent should:

1. Inspect the FreeBSD release/architecture, UTM display device and framebuffer, current user, installed packages, available disk space, Xorg configuration directories, and Xorg logs. Confirm this is the UTM/Apple Silicon setup this workaround targets.
2. If the VM uses UTM's `virtio-gpu-pci` display device, check whether the FreeBSD package `virtio-gpu-qemu-kmod` is available for this release and architecture. If it is, install it as root with `pkg install virtio-gpu-qemu-kmod`, then inspect its package message. Back up `/boot/loader.conf` and add `virtio_gpu_qemu_load="YES"` if it is not already present. Keep UTM's display device set to `virtio-gpu-pci` and reboot. The module must load at boot before the in-kernel VirtIO GPU driver claims the device; adding it to `rc.conf` or loading it after boot is not sufficient. The package supplies the framebuffer support and an Xorg `scfb` configuration. Do not apply this step to a different display device or guess a substitute kernel patch. If `pkg install` cannot find the package, capture the exact error and UTM version, check the package's current FreeBSD Ports status, and report the blocker instead of silently switching drivers.
3. Install missing Xorg, XFCE, D-Bus, and `scfb` packages with `pkg` if networking and package repositories work. Add the graphical user to the `video` group if needed and enable/start D-Bus as required by the installed XFCE packages.
4. Check which Xorg configuration directory this installation reads. The video comment specifies `/usr/local/share/X11/xorg.conf.d/00-monitor.conf`, while the current FreeBSD Handbook recommends `/usr/local/etc/X11/xorg.conf.d/`. The `virtio-gpu-qemu-kmod` package may also install a driver snippet under `share/X11/xorg.conf.d`; read its package message and Xorg logs to understand precedence. Put the configuration below in the active directory, first backing up any file it will replace and avoiding duplicate conflicting copies.
5. Set up the user's X session to launch XFCE (for example, a backed-up or newly created `~/.xinitrc` that runs `exec startxfce4`), then start the session from that user's console with `startx`. Do not configure automatic graphical login unless asked.
6. Verify from the Xorg log that the config loaded and `scfb` was selected, and verify that the XFCE desktop is visible. If it fails, diagnose the actual log errors and make the smallest reversible correction; report the exact remaining blocker rather than claiming success.

The guest may require root privileges for packages and system configuration. If the agent lacks them, it should identify the exact commands that need root and continue with all independent user-level setup and diagnostics.

Create the active directory if needed, then save this as `00-monitor.conf` inside it:

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

Once the VM has rebooted, inspect the Xorg log to confirm the framebuffer is available and `scfb` was selected, then continue with the desktop session. If the log reports `scfb_mmap: Invalid argument` and `/dev/fb0` is missing, check the VirtIO GPU kernel module and its loader setting before changing `.xinitrc` or reinstalling the desktop. A running display manager (such as SDDM) and D-Bus do not fix a missing framebuffer. The video describes a separate custom kernel patch for 2D framebuffer output; this Xorg file does not install that patch. A FreeBSD port named `graphics/virtio-gpu-qemu-kmod` was added for UTM/QEMU VirtIO GPU guests in July 2026 ([FreeBSD Ports commit](https://cgit.freebsd.org/ports/commit/?id=6b825bb84ccd0d0aee943776c21e250a6bf35c23)). Its module needs boot-time preloading and a reboot. Do not apply it to other display devices or guess patch instructions. The video notes that hardware acceleration is unavailable and XFCE may have monitor-detection or wallpaper issues.

The video comment also reports occasional redraw problems after switching virtual terminals (`Ctrl`+`Alt`+`F1`, then `Ctrl`+`Alt`+`F9`). If the X display does not redraw, run `xrefresh` from the X session; VT numbers may differ on your VM.

See [the detailed UTM and Xorg desktop guide](docs/utm-xorg-desktop.md) for diagnostic checks and a handoff prompt for Codex inside the VM. The [FreeBSD Handbook's X11 chapter](https://docs.freebsd.org/en/books/handbook/x11/) documents the `scfb` driver and Xorg configuration.

