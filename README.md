# hidpi

Sharp text on non-Retina external monitors on Apple Silicon Macs, without a virtual display.

macOS dropped subpixel antialiasing years ago, so text on a 1440p or 1080p monitor running at 1x looks soft next to a Retina screen. The fix is to render at 2x and downscale, which is what macOS does on Retina displays. On Apple Silicon, macOS doesn't offer those HiDPI modes for most sub-4K monitors.

`hidpi` writes a display override file for each external monitor below 4K. The override adds HiDPI modes, and the tool sets the WindowServer flag that lets macOS use them. There's no virtual screen and no mirroring, so there's no added latency and refresh rate and VRR are unaffected.

## Get it

`hidpi` is a single zsh script that only uses tools that ship with macOS (`ioreg`, `plutil`, `defaults`, `xxd`). There's nothing to build or install.

```
curl -O https://raw.githubusercontent.com/kirakirag/hidpi/main/hidpi && chmod +x hidpi
```

Read it before running it with sudo. It's about 130 lines.

## Use

```
./hidpi list                       # external displays, native resolution, override state
./hidpi install --dry-run          # show what would be written
sudo ./hidpi install               # write overrides, enable HiDPI modes
sudo ./hidpi uninstall             # remove them again
```

Reboot after `install` or `uninstall`. Then open System Settings > Displays, turn on "Show all resolutions", and pick one of the new sizes. The HiDPI ones are listed by their plain size; the 1x versions are labelled "(low resolution)".

## What it writes

For each external display narrower than 3840 pixels:

- `/Library/Displays/Contents/Resources/Overrides/DisplayVendorID-<vendor>/DisplayProductID-<product>`, a plist with:
  - `scale-resolutions`: HiDPI sizes from native width minus 64 down to 75% of native, in steps of 64, at the display's aspect ratio. Each size gets three entries: the HiDPI mode (`2w, 2h, 0x9, 0x200000`), its 2x backing size, and the 1x size.
  - `default-resolution`: the largest of those sizes, at the display's highest refresh rate.
  - `DisplayProductName`: the display's name with ` (HiDPI)` appended. The tool uses this suffix to recognise its own files.
- `DisplayResolutionEnabled = YES` in `/Library/Preferences/com.apple.windowserver`.

If an override file from somewhere else is already there, it's moved to `.bak` first and put back on `uninstall`.

## Limits

macOS ignores a HiDPI mode that looks like the panel's native resolution, so a 2560x1440 monitor tops out at 2496x1404 HiDPI, which is 97.5% of the space.

Base M1/M2 chips cap the backing store at 6K, and the Pro, Max and Ultra chips at 8K. macOS silently drops sizes past the cap, so big ultrawides on base chips get fewer options.

I haven't confirmed that `default-resolution` stops macOS from switching back to native when the monitor reconnects. If it resets, pick the HiDPI size again in Settings.

The only tested setup is an LG UltraGear 27" 1440p at 165 Hz on an M2 Max. Other monitors should behave the same way, but nobody has tried them yet.

## Undo by hand

If the tool isn't around:

```
sudo rm -r /Library/Displays/Contents/Resources/Overrides/DisplayVendorID-<vendor>
sudo defaults delete /Library/Preferences/com.apple.windowserver DisplayResolutionEnabled
```

`./hidpi list` prints the vendor ID. Reboot afterwards.

The tool never writes to the monitor itself. At worst, the external monitor shows no picture until you remove the file, and the built-in screen keeps working the whole time.
