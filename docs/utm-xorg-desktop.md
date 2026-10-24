# FreeBSD desktop graphics in UTM on Apple Silicon

This note records a community-reported way to get Xorg displaying through UTM's framebuffer on an Apple Silicon Mac. It is a troubleshooting lead, not a guarantee that every FreeBSD release, UTM display device, or desktop environment will behave the same way.

## Video and reported workaround

The source is [“FreeBSD - how to do Xorg graphics on UTM on MacOS/arm64” by BSDJedi](https://www.youtube.com/watch?v=__TrwS4u52I). The video description says it uses the author's kernel patch to enable 2D video in FreeBSD under UTM. It says the setup has no hardware acceleration and notes that XFCE may fail to detect the monitor or set a wallpaper.

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

## Important checks before applying it

- Confirm the VM is FreeBSD/arm64 under UTM and identify the FreeBSD release and UTM display device. The video is specific to this setup.
- The video describes a custom kernel patch. The comment above only provides the Xorg configuration; it does not provide kernel-patch installation steps. Do not invent or apply kernel changes based on this note. Find and review the patch source and instructions first, and explain any kernel rebuild or boot changes before making them.
- FreeBSD's current Handbook documents Xorg configuration snippets under `/usr/local/etc/X11/xorg.conf.d/`, while this video comment specifically says `/usr/local/share/X11/xorg.conf.d/`. Check the guest's Xorg log for the active configuration directories and use the path actually read by that Xorg installation. Do not create duplicate conflicting copies.
- Check whether the `scfb` Xorg driver is installed and whether Xorg is actually selecting it. The Handbook lists `scfb` as the System Console Framebuffer driver and documents configuration files in its [X11 chapter](https://docs.freebsd.org/en/books/handbook/x11/).
- Preserve and inspect existing Xorg configuration before editing. The `ConnectedMonitor` option and `Xinerama` setting are reproduced from the video comment; verify their effect in this guest rather than assuming they are required or supported by its driver setup.
- A successful Xorg session only establishes display output. It does not provide accelerated graphics, and a desktop environment can still have issues such as monitor detection or wallpaper controls.

## Handoff prompt for Codex inside the VM

Have the VM-side Codex read the repository README and this guide, then complete the desktop setup in the live guest. It should inspect the FreeBSD release and architecture, UTM display adapter/framebuffer, current user, available disk space, installed Xorg/XFCE/`scfb` packages, active Xorg configuration directory, and relevant log errors. For a `virtio-gpu-pci` device, it should check whether `virtio-gpu-qemu-kmod` supports the guest, install it if available, add `virtio_gpu_qemu_load="YES"` to `/boot/loader.conf` after backing up the file, keep UTM's display set to `virtio-gpu-pci`, and reboot. The module must load before the built-in driver claims the device; `rc.conf` or loading it after boot is too late. If `pkg install` cannot find it, capture the exact error and UTM version and report the blocker. After reboot, if needed, install Xorg, XFCE, D-Bus, and the `scfb` driver; add the desktop user to the `video` group; configure D-Bus and that user's X startup file; apply the 1280x800 config with a backup; and run `startx` from the user's console to launch XFCE. If Xorg reports `scfb_mmap: Invalid argument` or `/dev/fb0` is missing, check the kernel module and boot setting before changing the desktop session. Verify the Xorg log and visible desktop, then report changes and any remaining blocker. Do not guess custom kernel patch instructions or apply an unreviewed kernel change. If root privileges are unavailable, finish available diagnostics and user-level setup and list the exact privileged commands still needed. The video reports that VT switching may require `xrefresh` to redraw.
