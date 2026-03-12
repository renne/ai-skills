---
name: usb-c-display-alt-mode-troubleshooting
description: Troubleshoot Linux laptops that lose high-resolution external display modes over USB-C docks, especially when Windows still gets 4K60. Use when xrandr only offers ~30 Hz modes, a USB Billboard device appears, Thunderbolt devices are missing, or the kernel logs DPCD / DP bridge errors through a dock, KVM, or TV chain.
---
# USB-C Display Alt Mode Troubleshooting on Linux

## When to use this skill

Use this skill when all or most of the following are true:

- The display path is `Linux laptop -> USB-C/TB dock -> KVM/adapter -> display`
- Windows on similar hardware still gets `3840x2160@60`
- Linux only offers `1920x1080@29.95` or other ~30 Hz modes
- `xrandr` shows a real DRM connector like `DP-2`, but the available mode list is heavily reduced
- A **USB Billboard** device appears in `lsusb`
- `/sys/class/typec` is empty or UCSI/Type-C drivers do not bind

This pattern often indicates **USB-C DisplayPort Alternate Mode negotiation failure** or a dock/bridge compatibility problem under Linux, not just a missing mode line.

---

## Key signs and what they mean

### 1. `xrandr` only shows ~30 Hz modes

Example:

```bash
xrandr --query
```

Symptoms:
- `DP-2 connected`
- `1920x1080 29.95*+`
- nearly every listed mode is around 30 Hz

Interpretation:
- The display link is connected, but Linux is using a degraded transport path or limited link bandwidth
- Under Wayland, `xrandr` is emulated, so treat kernel/DRM state as more authoritative than `xrandr --newmode`

### 2. USB Billboard device is present

Check:

```bash
lsusb
lsusb -v | sed -n '/Billboard/,/^$/p'
```

Typical result:

```text
2109:8818 VIA Labs, Inc. USB Billboard Device
```

Interpretation:
- A Billboard device usually means **USB-C Alternate Mode negotiation failed**
- On docks, this is a strong sign that DisplayPort alt mode was not established cleanly

### 3. Type-C modules load, but no Type-C devices appear

Check:

```bash
sudo modprobe typec
sudo modprobe typec_ucsi
sudo modprobe ucsi_acpi
sudo modprobe typec_displayport

lsmod | grep -E 'thunderbolt|typec|ucsi'
find /sys/class/typec -maxdepth 3 -type f
```

Interpretation:
- If modules load but `/sys/class/typec` stays empty, the runtime stack exists but **nothing is binding to a usable USB-C controller**
- This points toward firmware/ACPI exposure problems or platform-specific compatibility gaps

### 4. ACPI shows a USB-C device, but it is inactive

Check:

```bash
ls /sys/bus/acpi/devices | grep -i 'UCSI\|USBC'
for f in /sys/bus/acpi/devices/USBC000:00/{hid,path,status,uid,adr}; do
  [ -e "$f" ] && echo "--- $f ---" && cat "$f"
done
```

Important clue:

```text
status = 0
```

Interpretation:
- Firmware knows about a USB-C ACPI node, but Linux is **not being given an active controller/device**
- In this situation, `ucsi_acpi` cannot bind even when the module is present

### 5. Thunderbolt loads, but the dock never appears as a TB device

Check:

```bash
sudo modprobe thunderbolt
boltctl list
boltctl domains
find /sys/bus/thunderbolt/devices -maxdepth 2
```

Interpretation:
- If only the host domain appears and the dock does not, the dock is likely functioning as a **USB-C alt-mode device**, not as an active Thunderbolt peripheral from Linux’s perspective
- Do not assume “Thunderbolt 3 dock” means Linux will enumerate it as a Thunderbolt device

### 6. Kernel logs show DPCD read failures

Check:

```bash
journalctl -k --no-pager | grep -iE 'i915|drm|dpcd|DP-2|typec|thunderbolt|billboard'
```

Important clue:

```text
i915 ... [drm] *ERROR* Failed to read DPCD register 0x92
```

Interpretation:
- This often points to a **DP bridge / PCON / AUX communication issue**
- It does **not** automatically mean the i915 driver alone is broken; it may reflect a broken bridge negotiation path through the dock/KVM chain

---

## Case pattern this skill captures

Observed working diagnosis on:

- Acer Aspire VN7-593G
- Intel HD Graphics 630 (Kaby Lake)
- Ubuntu 24.04.4
- HWE kernel `6.17.0-14`
- SSK SC220 dock
- InLine 4K60 Multiviewer KVM

Observed facts:

- Windows could still reach `2160p60`
- Linux stayed at `1920x1080@29.95`
- A **VIA Labs USB Billboard** device appeared
- Type-C and UCSI modules loaded, but `/sys/class/typec` stayed empty
- ACPI exposed `USBC000:00`, but with `status=0`
- Thunderbolt host domain existed, but the connected SSK dock did not enumerate as a Thunderbolt device
- The platform BIOS was very old (`Insyde V1.11`, `2018-08-01`)

Most likely interpretation:

- Linux is not being given a working USB-C DisplayPort alt-mode path on this platform/dock chain
- The failure is below user-space mode setting and is unlikely to be fixed by `xrandr`, simple GRUB changes, or just installing a newer kernel

---

## Recommended diagnostic order

1. Confirm the session is actually using the intended kernel:

```bash
uname -r
```

2. Capture current display state:

```bash
xrandr --verbose | sed -n '1,120p'
journalctl -k --no-pager | grep -iE 'i915|drm|DP-2|dpcd|typec|thunderbolt|billboard'
```

3. Check for Billboard / dock USB identity:

```bash
lsusb
lsusb -t
```

4. Load the USB-C runtime stack and re-check:

```bash
sudo modprobe typec
sudo modprobe typec_ucsi
sudo modprobe ucsi_acpi
sudo modprobe typec_displayport
sudo modprobe thunderbolt
```

5. Check whether any Type-C controller actually bound:

```bash
lsmod | grep -E 'thunderbolt|typec|ucsi'
find /sys/class/typec -maxdepth 3 -type f
ls /sys/bus/acpi/devices | grep -i 'UCSI\|USBC'
```

6. Inspect firmware-visible platform state:

```bash
fwupdmgr get-devices
hostnamectl
for f in /sys/class/dmi/id/{bios_vendor,bios_version,bios_date}; do
  [ -r "$f" ] && echo "$f" && cat "$f"
done
```

---

## What usually does NOT fix it

- Removing invalid `video=` GRUB overrides
- Manually adding modes with `xrandr --newmode` under Wayland
- Loading Type-C modules when the firmware never exposes a bindable controller
- Assuming the dock must appear as a Thunderbolt peripheral just because the hardware marketing says “Thunderbolt 3”

These may be worth checking, but they are not the main fix when Billboard + empty `/sys/class/typec` are present.

---

## Most realistic remediation paths

### Path A: BIOS / firmware inspection

Look for BIOS options such as:

- `Thunderbolt Support`
- `Thunderbolt Security`
- `USB-C`
- `DisplayPort Alternate Mode`
- `DMA protection`
- `PCIe tunneling`
- `Assist mode` / dock support

Older Insyde BIOS versions may expose few or none of these.

### Path B: Firmware update

If the notebook vendor offers a newer BIOS than the Linux-visible one, update it outside Linux if necessary.

### Path C: Hardware compatibility confirmation

If the same Linux notebook fails only through one dock/KVM chain, treat the dock chain as a Linux compatibility suspect even if Windows works.

### Path D: File a bug with evidence

Include:

- `uname -r`
- `xrandr --verbose`
- `journalctl -k` excerpts showing DPCD/Billboard/typec state
- `lsusb` and `lsusb -t`
- `/sys/class/typec` contents
- BIOS version and date

---

## Practical conclusion

If you see **all three** of these together:

- USB Billboard device
- empty `/sys/class/typec`
- all external modes limited to ~30 Hz

then the problem is very likely **USB-C alt-mode exposure/negotiation failure under Linux**, not simply “Ubuntu forgot the 4K mode.”
