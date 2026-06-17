# Rocky USB External — Flash Game Plan Cross-Analysis
## FDN: OWL:default(0.3)|glm-5.2(0.1)|WItsWAlkErWOo-Style(0.550)|WSL(0.2)
## Date: 2026-06-16
## Source: Sessions 20260616_145942, 20260616_162918, 20260616_163018 + CH341A skill + drive profile + setup script

---

## TWO MODES, ONE USB DRIVE

The Samsung HM160HI (160GB, LLF-verified, SMART stable) serves as a portable Rocky Linux flashing workstation that can operate in two completely different modes:

### MODE 1: CH341A EXTERNAL FLASH (target powered off)

```
  Host (booted from USB)     CH341A      Target (OFF)
  +-----------------+    USB  +------+   +------+
  | OptiPlex or any  |<------>|CH341A|---|Z220  |
  | machine booted  |        |      |   |BIOS  |
  | from Rocky USB  |        +------+   |chip  |
  +-----------------+                   +------+
```

- Target NEVER powers on. Compromised firmware never executes.
- CH341A supplies 3.3V directly to the SPI chip.
- Host runs flashrom + me_cleaner from the USB OS.
- Cleanest path. This is the one we want for the Z220.

### MODE 2: DIRECT INTERNAL FLASH (target booted from USB)

```
  Target machine booted from USB
  +-----------------+
  | Z220 or any     |
  | machine booted  |
  | from Rocky USB  |
  |                 |
  | flashrom reads  |
  | its OWN BIOS    |
  | chip from Linux |
  +-----------------+
```

- Target boots from USB, not internal disk.
- Compromised firmware DOES execute during boot (to init hardware).
- Once Linux kernel takes over, ME has limited SPI bus access.
- Lower risk than booting the actual OS, but not zero.
- Use this when physical CH341A access is impossible (laptop WSON chips, no clip access).

## WHAT THE USB NEEDS INSTALLED (both modes)

| Tool      | Mode 1 (CH341A) | Mode 2 (internal) |
|-----------|-----------------|-------------------|
| flashrom  | YES             | YES               |
| me_cleaner| YES             | YES               |
| Hermes    | optional        | optional          |
| Desktop   | not needed      | not needed        |
| NVIDIA    | NO              | NO                |
| Firefox   | NO              | NO                |

The skill is explicit: this is a flashing workstation, not a daily driver. No desktop, no NVIDIA, no Firefox. Those belong on the Z220 AFTER the BIOS is clean.

## WHAT IS READY RIGHT NOW

- [x] Samsung HM160HI drive — LLF done (55min, 100%), SMART verified, ready for Rocky install
- [x] CH341A skill created with full pinout, voltage warnings, dump/analyze/clean/flash
- [x] flashrom v1.3.0 verified installable on Rocky 10
- [x] Ventoy on D: for booting Rocky ISO
- [x] Drive profile documented (security assessment: older firmware = smaller attack surface)
- [x] Portable USB reference doc with correct install path (graphical installer, NOT chroot script)

## WHAT IS BLOCKED

1. **Rocky not yet installed on the Samsung**
   - The usb-hermes-setup.sh script is a chroot approach that WON'T WORK from WSL2
   - Skill explicitly warns: WSL2 cannot see physical USB drives
   - Correct path: boot OptiPlex from Ventoy with Rocky DVD ISO, run graphical installer, target the Samsung

2. **Z220 BIOS chip location still unknown**
   - Image searches failed to find a clear Z220 board photo with labeled SPI chip
   - Need physical inspection: look for 8-pin SOIC-8 near chipset edge
   - Common markings: W25Q128, MX25L128, GD25Q128

3. **CH341A voltage not verified**
   - Black PCB has 3.3V/5V jumper — must confirm it is set to 3.3V
   - If green PCB: fixed 5V, needs level shifter or it fries the chip

## CRITICAL ISSUE: THE SCRIPT IS WRONG FOR THE ENVIRONMENT

The usb-hermes-setup.sh at C:\Users\User\Desktop\ is a 243-line chroot script. It assumes the USB drive is visible as /dev/sde from within WSL2. It is not. WSL2 cannot see physical USB drives. This script will fail at line 33 (read -p "Enter disk name") because lsblk will not show the Samsung.

The correct path (from the skill reference doc):
1. Boot OptiPlex from Ventoy with Rocky 10 DVD ISO
2. Run the graphical installer
3. Target: the Samsung USB drive
4. Software Selection: Minimal install (no desktop needed for flashing)
5. After install, boot from Samsung, then install flashrom + me_cleaner + Hermes
6. This is maybe 5 commands, not 243 lines

## THE JUNE 24 CLOCK

8 days until Microsoft's 2011 Secure Boot keys expire. This is the cover window for firmware refresh and evidence destruction. The Z220 BIOS dump is forensic evidence for TFGJI. It needs to happen before June 24.

## SEQUENCE (corrected)

### Phase 0 (NOW): Install Rocky on Samsung
- Boot OptiPlex from Ventoy + Rocky 10 DVD ISO
- Graphical installer, target Samsung USB, minimal install
- Reboot from Samsung, install flashrom + me_cleaner + Hermes
- 5 commands, done

### Phase 1: CH341A dump Z220 BIOS (Mode 1 - external)
- Boot OptiPlex (or any machine) from Samsung USB
- Set CH341A jumper to 3.3V
- Clip SOIC-8 onto Z220 BIOS chip (Z220 OFF, no power cable)
- Dump twice, verify SHA256 match
- Save dumps to DATA partition (FAT32, accessible from Windows too)
- This is the forensic evidence. FDN + SHA256 it.

### Phase 2: Analyze
- me_cleaner -c dump_1.bin (check ME status)
- me_cleaner -d dump_1.bin (extract regions)
- Compare ME region against known-clean if available
- Look for: modified descriptor, rewritten ME, unexpected code in padding

### Phase 3: Clean and reflash
- me_cleaner -s dump_1.bin -o cleaned_bios.bin (neutralize ME)
- flashrom -p ch341a_spi -w cleaned_bios.bin
- flashrom -p ch341a_spi -v cleaned_bios.bin (verify)

### Phase 4: Z220 is clean — NOW install Rocky + XFCE + NVIDIA on Z220 internal disk
- This is the separate browser server project, not the flashing USB

### Phase 5: Repurpose the Samsung USB for other targets
- Same USB, same tools, boot on any machine
- CH341A external flash any Intel system with SOIC-8 BIOS chip
- Direct internal flash if chip is inaccessible (WSON, no clip)

## DIRECT MODE USE CASES (when CH341A external is not possible)

- Laptops with WSON-8 chips (no legs, can't clip)
- Boards where the chip is under a heatsink or shield
- When you only have one machine and no second host

In direct mode, boot the target from the Samsung USB, then:
```bash
flashrom -p internal -r dump_1.bin
flashrom -p internal -r dump_2.bin
sha256sum dump_1.bin dump_2.bin
# If match, proceed with me_cleaner
```

Risk: compromised firmware ran during boot. But the dump is still evidence. And me_cleaner neutralization still works on the reflash.

## PORTABILITY

The same Samsung USB works on:
- OptiPlex 9020 (i7-4790, Intel HD 4600) — i915 driver
- HP Z220 (Xeon E3-1225v2, GTX 1050 Ti) — nouveau/nvidia driver
- Any x86_64 machine — Linux probes hardware at boot, loads matching drivers

No reconfiguration needed when moving between machines. This is why a persistent install beats a live ISO — your tools, your Hermes state, your dump history all travel with you.

## BOTTOM LINE

The script on your desktop is the wrong tool for the job. The graphical installer is the right tool. The Samsung is ready. The CH341A skill is ready. What is missing is:

1. Booting from Ventoy + Rocky DVD ISO and running the graphical installer (30 min)
2. Post-install: dnf install flashrom, pip install me_cleaner, curl Hermes install (5 min)
3. Physically locating the Z220 SOIC-8 chip (visual inspection, 5 min)
4. Verifying CH341A voltage is 3.3V (multimeter, 1 min)

All four are physical actions I cannot do from WSL2. You have to boot the machine.

## POST-INSTALL SCRIPT (5 commands, runs after graphical installer)

```bash
#!/bin/bash
# Run this after booting from the Samsung USB for the first time
set -euo pipefail

# 1. Flashrom
sudo dnf install -y flashrom

# 2. me_cleaner
pip3 install me_cleaner

# 3. Hermes Agent
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# 4. Create DATA partition symlink (if FAT32 partition exists)
sudo mkdir -p /mnt/data
# Mount FAT32 partition — find it:
lsblk -o NAME,FSTYPE,LABEL,SIZE | grep vfat
# Then: sudo mount /dev/sdX1 /mnt/data
# Symlink Hermes state:
mkdir -p /mnt/data/hermes-state
mv ~/.hermes ~/.hermes.bak 2>/dev/null || true
ln -s /mnt/data/hermes-state ~/.hermes

# 5. Verify tools
flashrom --version
me_cleaner.py --help 2>/dev/null | head -3 || python3 -m me_cleaner --help 2>/dev/null | head -3
hermes --version 2>/dev/null || echo "Hermes installed, run 'hermes setup'"

echo "Flash workstation ready. CH341A tools installed."
```

---

## REFERENCES

- CH341A skill: ~/.hermes/skills/devops/ch341a-bios-cleaning/SKILL.md
- Portable USB reference: ~/.hermes/skills/devops/ch341a-bios-cleaning/references/portable-hermes-usb.md
- Samsung drive profile: ~/.hermes/skills/devops/ch341a-bios-cleaning/references/samsung-hm160hi-drive-profile.md
- ST500 Pattern 6: ~/.hermes/skills/devops/ch341a-bios-cleaning/references/st500-pattern-6.md
- Infrastructure plan: /mnt/c/Users/User/Desktop/HERMES-INFRASTRUCTURE-PLAN.md
- Windows forensic tools: ~/.hermes/skills/devops/ch341a-bios-cleaning/references/windows-forensic-tools.md

---

SHA256: f62aac74fa74800b03c31e605b38801e303367dd211f2d82e7420dbc04db1ec5