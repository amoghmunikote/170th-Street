# VBIOS Flash Guide

**VBIOS Flash Guide**

Flashing the VBIOS on the CMP 170HX is a low-risk, reversible modification that updates the card's firmware to a newer version. It does not unlock FMA throttling or PCIe speed — those restrictions are enforced by the driver and cannot be removed via VBIOS on Ampere — but it can update power table entries, fix bugs, and potentially improve stability.

**What a VBIOS Flash Can and Cannot Do**

✅ Update from an older VBIOS version to a newer one (e.g. 92.00.67.00.01 → 92.00.6D.00.0A) ✅ Change the power limit table (our card's VBIOS supports up to 300W after the update) ✅ Fix driver compatibility issues with newer NVIDIA driver versions ✅ Potentially improve clocking behavior in some scenarios ❌ Remove FMA throttling ❌ Unlock PCIe Gen 1 speed restriction ❌ Enable NVLink or display outputs ❌ Change VRAM capacity

**Risk Level**

Moderate. A bad flash or power failure during flashing can brick the card. The CMP 170HX has no display output, which means if a bad flash prevents the card from booting, recovery requires a second system or the ability to reflash from a CLI environment without a display. Always back up your current VBIOS before proceeding.

**What You Need**

* nvflash for Linux (latest version from NVIDIA or [TechPowerUp](https://www.techpowerup.com/download/nvidia-nvflash/))
* A backup destination (a separate drive, network share, or another machine)
* The VBIOS file you want to flash — must match your card's Device ID `10DE:20C2` and Subsystem ID `10DE:1585`
* A second GPU driving your display (the CMP has no outputs)
* SSH access recommended so you don't lose your session during the process

**Step 1 — Download nvflash**

bash

```bash
cd ~/Downloads
wget https://developer.download.nvidia.com/compute/nvflash/nvflash_5.867_linux.zip
unzip nvflash_5.867_linux.zip
cd nvflash_5.867_linux/x64
chmod +x nvflash
```

**Step 2 — Identify Your Card**

bash

```bash
sudo ./nvflash --list
```

You will see something like:

```
<0> Graphics Device (10DE,1FB2,103C,1489) S:00,B:15,D:00,F:00   ← your display GPU
<1> Graphics Device (10DE,20C2,10DE,1585) S:00,B:21,D:00,F:00   ← CMP 170HX
```

The CMP 170HX is identified by Device ID `20C2` and Subsystem ID `10DE:1585`. Note its index number (1 in this example).

**Step 3 — Back Up Your Current VBIOS**

This is the most important step. Do not skip it.

bash

```bash
# Stop the display manager and nvidia services first
sudo systemctl stop gdm
sudo systemctl stop nvidia-persistenced
sudo rmmod nvidia_uvm nvidia_drm nvidia_modeset nvidia

# Back up
sudo ./nvflash --save cmp170hx_backup.rom
# Select your CMP index when prompted
```

Copy the backup to at least two locations — another drive and another machine if possible.

**Step 4 — Verify Your Current VBIOS Version**

bash

```bash
nvidia-smi --query-gpu=name,vbios_version --format=csv
```

Note the version before flashing so you can confirm the update succeeded.

**Step 5 — Find a Compatible VBIOS**

The VBIOS must match your card's exact subsystem ID. For the standard NVIDIA-manufactured CMP 170HX 8GB, look for VBIOSes with:

* Device ID: `10DE 20C2`
* Subsystem ID: `10DE 1585`

The [TechPowerUp VBIOS database](https://www.techpowerup.com/vgabios/?type=\&model=CMP+170HX) is the most reputable public source. Search for "CMP 170HX" and verify the subsystem IDs match before downloading.

Known good versions:

* `92.00.67.00.01` — original shipping VBIOS (May 2021)
* `92.00.6D.00.0A` — newer version (April 2022), raises power limit ceiling to 300W at [https://www.techpowerup.com/vgabios/268495/268495](https://www.techpowerup.com/vgabios/268495/268495)

**Step 6 — Flash**

bash

```bash
sudo ./nvflash 268495.rom
# Select your CMP index when prompted
# Verify the IDs match in the output before confirming with 'y'
```

nvflash will show you a confirmation screen like this before proceeding:

```
Current      - Version:92.00.67.00.01 ID:10DE:20C2:10DE:1585 (Normal Board)
Replace with - Version:92.00.6D.00.0A ID:10DE:20C2:10DE:1585 (Normal Board)
Update display adapter firmware?
Press 'y' to confirm ('s' to skip, 'a' to abort):
```

Only proceed if the Device IDs and Subsystem IDs match exactly. The board type should say "Normal Board" for both.

**Step 7 — Reboot and Verify**

bash

```bash
sudo reboot
# After reboot:
nvidia-smi --query-gpu=name,vbios_version --format=csv
```

Confirm the new version is shown. Then set the new power limit:

bash

```bash
sudo nvidia-smi -i [index] -pm 1
sudo nvidia-smi -i [index] -pl 300
```

**Recovery**

If something goes wrong and the card is no longer detected, boot your system using your display GPU only, then use nvflash with your backup ROM:

bash

```bash
sudo ./nvflash cmp170hx_backup.rom
```

If the card is completely unresponsive to nvflash, recovery requires a SPI flasher connected directly to the EEPROM and is significantly more difficult.

**Important Note on SPI Dumping:** Community attempts to dump VBIOS via external SPI programmers have revealed significant reliability issues:
- Signal integrity problems at high speeds (1.8V SPI bus, >50 MHz)
- Long probe wires (15+ cm) introduce noise that corrupts data
- Incorrect chip type assumptions result in partial or corrupted dumps
- Only ~64 KB successfully extracted from dumps that should be larger
- Recompiled images from partial dumps often fail to boot

**Recommendation:** Always use `nvflash --save` (via software) for backup, which is 100% reliable. Only resort to SPI dumping as a last resort for recovery, and use short probe connections with proper signal integrity practices if you attempt it.
