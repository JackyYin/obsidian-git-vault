# UEFI and Bootloader Boot Chain

How firmware (UEFI), bootloaders (GRUB), and the kernel actually fit together — with the RISC-V / SiFive embedded variant and live-USB / hybrid-ISO mechanics covered explicitly.

Cross-references: [[SiFive - Computer Architecture Prep]], [[SiFive - Coding Interview Problems]], [[Manual cache control beyond DMA and JIT]].

---

## 1. The layered model

Four conceptual layers from power-on to user space:

```
[hardware] → [firmware: UEFI] → [bootloader: GRUB] → [kernel: Linux] → [init/userspace]
```

- **UEFI** is what runs on power-on. It lives in **SPI flash** on the motherboard / SoC. Always there, soldered down.
- **GRUB** is a regular program that UEFI loads from disk. It is a PE/COFF binary — `grubx64.efi`, `grubaa64.efi`, `grubriscv64.efi` — sitting on a filesystem.
- **Kernel** is loaded by GRUB, either as another UEFI application (EFISTUB) or via a hand-coded boot protocol.

UEFI doesn't know what an OS or a kernel is. It only knows how to load PE/COFF executables.

---

## 2. What UEFI provides

UEFI is a firmware *and* a runtime API. Three buckets of services:

### Boot Services
Valid only **before** `ExitBootServices()`. Torn down once the OS takes over.

- Filesystem access (FAT32 on the EFI System Partition)
- Block I/O, disk enumeration
- Network (PXE, HTTP boot, DHCPv4/v6)
- Graphics Output Protocol (GOP) — the framebuffer
- Memory map, page allocation
- `LoadImage()` / `StartImage()` — load and execute a PE/COFF binary
- Simple Text I/O (console)

### Runtime Services
Survive into the OS. Linux exposes some of these via `efivarfs`.

- NVRAM variables (where `BootOrder`, `BootXXXX`, Secure Boot keys live)
- Real-time clock
- Capsule update (firmware update mechanism)
- Reset / shutdown

### Protocols
A vtable-style interface registry. Drivers in firmware (NVMe, USB, NIC, GPU) register protocols; bootloaders consume them. Examples: `EFI_BLOCK_IO_PROTOCOL`, `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`, `EFI_GRAPHICS_OUTPUT_PROTOCOL`.

---

## 3. Where `grubx64.efi` lives

It must sit on the **EFI System Partition (ESP)**:
- FAT32 filesystem
- GPT type GUID: `C12A7328-F81F-11D2-BA4B-00A0C93EC93B`
- (Or MBR partition type `0xEF` on legacy disks)

UEFI has a built-in FAT driver and *only* knows how to read that filesystem at boot.

Typical ESP layout:

```
ESP (FAT32, mounted at /boot/efi on Linux)
├── EFI/
│   ├── BOOT/
│   │   └── BOOTX64.EFI         ← removable-media fallback
│   ├── debian/
│   │   ├── shimx64.efi
│   │   └── grubx64.efi         ← the "installed" path
│   └── Microsoft/
│       └── Boot/bootmgfw.efi
```

The underlying device can be SATA, NVMe, USB, SD, eMMC, or iSCSI — UEFI doesn't care, as long as it has a driver for the bus.

---

## 4. How UEFI knows *which* `.efi` to load

Three mechanisms, in order:

### (a) NVRAM Boot Manager entries — the installed case

NVRAM (still SPI flash, separate region from firmware code) holds variables like:

```
Boot0001* debian   HD(1,GPT,...)/File(\EFI\debian\shimx64.efi)
Boot0002* fedora   HD(1,GPT,...)/File(\EFI\fedora\shimx64.efi)
BootOrder: 0001,0002
```

On boot, UEFI Boot Manager:
1. Reads `BootOrder`.
2. Walks each `BootXXXX` entry.
3. Resolves the Device Path — (Disk, Partition, File).
4. Loads the first one that resolves successfully.

`efibootmgr` (Linux) is the tool that creates / edits these entries. Move a disk to a different machine and it often won't boot until either (1) `efibootmgr` is rerun on the new machine, or (2) the fallback path below kicks in.

For NVRAM entries, the partition's type GUID is **not** checked — the Device Path is explicit, so UEFI just resolves *(disk, partition, file)* and reads the FAT volume there.

### (b) Removable-media fallback path — USB / live ISO

UEFI spec defines a hardcoded fallback. If no NVRAM entry matches, the firmware looks for:

| Architecture | Fallback path |
|--------------|---------------|
| x86_64       | `\EFI\BOOT\BOOTX64.EFI` |
| AArch64      | `\EFI\BOOT\BOOTAA64.EFI` |
| RISC-V 64    | `\EFI\BOOT\BOOTRISCV64.EFI` |
| i386         | `\EFI\BOOT\BOOTIA32.EFI` |

…on every connected device's ESP.

The fallback path **strictly requires the partition to be tagged as an ESP** by its type GUID — see §5 below. Non-ESP partitions are ignored entirely, even if they happen to contain a valid FAT filesystem with `\EFI\BOOT\BOOTX64.EFI` inside.

Distro install ISOs put their bootloader at exactly that path so they "just work" on a machine that's never seen them before.

### (c) Network boot — no local disk at all

UEFI can fetch the `.efi` over **PXE** or **HTTP boot**. Firmware uses its NIC driver and `MTFTP` / `HTTP Boot` protocols to pull the image into RAM and `LoadImage()` it. Common in datacenters and netbooted appliances.

---

#### Look up boot order stored in NVRAM ?

 `sudo xxd  /sys/firmware/efi/efivars/BootOrder-8be4df61-93ca-11d2-aa0d-00e098032b8c`
```
00000000: 0700 0000 0100 0400 0200 0000 0300       ..............
```

 Bytes 0–3: attributes — 0x00000007 (little-endian 07 00 00 00)

| bit | mask | flag                                               | set? |
| --- | ---- | -------------------------------------------------- | ---- |
| 0   | 0x01 | EFI_VARIABLE_NON_VOLATILE                          |      |
| 1   | 0x02 | EFI_VARIABLE_BOOTSERVICE_ACCESS                    | Y    |
| 2   | 0x04 | EFI_VARIABLE_RUNTIME_ACCESS                        | Y    |
| 3   | 0x08 | EFI_VARIABLE_HARDWARE_ERROR_RECORD                 | Y    |
| 4   | 0x10 | EFI_VARIABLE_AUTHENTICATED_WRITE_ACCESS            |      |
| 5   | 0x20 | EFI_VARIABLE_TIME_BASED_AUTHENTICATED_WRITE_ACCESS |      |
| 6   | 0x40 | EFI_VARIABLE_APPEND_WRITE                          |      |

#### Verify with efibootmgr

command: `sudo efibootmgr (-v)`

```bash
BootCurrent: 0004
Timeout: 0 seconds
BootOrder: 0001,0004,0002,0000,0003

Boot0000* UiApp	FvVol(7cb8bdc9-f8eb-4f34-aaea-3ee4af6516a1)/FvFile(462caa21-7614-4503-836e-8ab6f4662331)

Boot0001* UEFI QEMU DVD-ROM QM00001 	PciRoot(0x0)/Pci(0x1f,0x2)/Sata(0,65535,0){auto_created_boot_option}

Boot0002* UEFI QEMU HARDDISK QM00003 	PciRoot(0x0)/Pci(0x1f,0x2)/Sata(1,65535,0){auto_created_boot_option}

Boot0003* EFI Internal Shell	FvVol(7cb8bdc9-f8eb-4f34-aaea-3ee4af6516a1)/FvFile(7c04a583-9e3e-4f1c-ad65-e05268d0b4d1)

Boot0004* Ubuntu	HD(1,GPT,398b324c-cd9b-4a98-9548-7c3b48526a21,0x800,0x219800)/File(\EFI\ubuntu\shimx64.efi)
```

---

## 5. Partition type GUIDs — how UEFI identifies the ESP

When a fresh USB stick is plugged in, UEFI has no NVRAM entry for it. The fallback rule says "find an ESP and load `\EFI\BOOT\BOOT<ARCH>.EFI`." But how does the firmware decide *which* of the partitions on the stick is the ESP?

**Answer: every GPT entry carries a Partition Type GUID. UEFI walks the GPT and picks the entry tagged with the well-known ESP GUID.** The other partitions are not even probed for FAT filesystems by the boot path — they're simply skipped.

### The ESP GUID

```
EFI System Partition:  C12A7328-F81F-11D2-BA4B-00A0C93EC93B
```

This is the only GUID UEFI needs to recognize for the fallback boot path. Memorize the leading `C12A7328` — it's the visual tell when scanning a GPT dump.

### MBR equivalent

Older / smaller systems use MBR partition tables. MBR uses 1-byte type codes instead of GUIDs:

| MBR type | Meaning |
|----------|---------|
| `0xEF` | EFI System Partition |
| `0x83` | Linux native |
| `0xEE` | GPT protective MBR (this disk is GPT, BIOS tools should not touch it) |
| `0x07` | NTFS / exFAT / Microsoft basic data |
| `0x0C` | FAT32 with LBA |

Modern hybrid ISOs typically expose a **GPT protective MBR** (single `0xEE` entry covering the whole disk) so legacy BIOS tools see a "full disk in use" and don't corrupt the GPT — while a UEFI firmware reads the GPT past it.

### Common GPT partition type GUIDs

Memorizing the ESP GUID is enough for boot-chain work, but here's a wider reference for when you're inspecting a GPT and want to know what each partition is supposed to be:

| GUID | Purpose |
|------|---------|
| `C12A7328-F81F-11D2-BA4B-00A0C93EC93B` | **EFI System Partition** |
| `024DEE41-33E7-11D3-9D69-0008C781F39F` | MBR partition scheme (legacy MBR-in-GPT) |
| `21686148-6449-6E6F-744E-656564454649` | BIOS boot partition (where GRUB stage 1.5 lives on BIOS+GPT systems) |
| `0FC63DAF-8483-4772-8E79-3D69D8477DE4` | Linux filesystem data |
| `4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709` | Linux x86-64 root (`/`) — discoverable by systemd-boot |
| `60D5A7FE-8E7D-435C-B714-3DD8162144E1` | Linux RISC-V 64 root — discoverable partition spec |
| `0657FD6D-A4AB-43C4-84E5-0933C84B4F4F` | Linux swap |
| `E6D6D379-F507-44C2-A23C-238F2A3DF928` | Linux LVM |
| `A19D880F-05FC-4D3B-A006-743F0F84911E` | Linux RAID |
| `933AC7E1-2EB4-4F13-B844-0E14E2AEF915` | Linux `/home` (discoverable) |
| `EBD0A0A2-B9E5-4433-87C0-68B6B72699C7` | Microsoft basic data (also: most installer ISOs' main payload partition) |
| `DE94BBA4-06D1-4D40-A16A-BFD50179D6AC` | Windows recovery environment |
| `9E1A2D38-C612-4316-AA26-8B49521E5A8B` | PowerPC PReP boot |
| `83BD6B9D-7F41-11DC-BE0B-001560B84F0F` | FreeBSD boot |
| `516E7CB6-6ECF-11D6-8FF8-00022D09712B` | OpenBSD data |

The full registry lives in the UEFI specification, Appendix A. Linux distributions also follow Lennart Poettering's [Discoverable Partitions Specification](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/) for the per-architecture root, `/home`, `/srv`, and similar GUIDs — those let `systemd` mount the right partitions without `/etc/fstab`.

---

### Inspecting partition type GUIDs on Linux

Tools that all show the same data, ranked by how directly they expose the GUID:

```bash
# lsblk: friendly label
lsblk -o NAME,PARTTYPENAME,FSTYPE,LABEL /dev/sda
# NAME                      PARTTYPENAME     FSTYPE      LABEL
# sda
# ├─sda1                    EFI System       vfat
# ├─sda2                    Linux filesystem ext4
# └─sda3                    Linux filesystem LVM2_member
#   └─ubuntu--vg-ubuntu--lv                  ext4
```

```bash
#  sfdisk: dump the partition table in machine-readable form
sudo sfdisk -d /dev/sda
# label: gpt
# label-id: F328F016-9546-4E56-A76C-F6207AFB0929
# device: /dev/sda
# unit: sectors
# first-lba: 34
# last-lba: 268435422
# sector-size: 512

# /dev/sda1 : start=        2048, size=     2201600, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=398B324C-CD9B-4A98-9548-7C3B48526A21
# /dev/sda2 : start=     2203648, size=     4194304, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=60E51D01-DC8A-4845-B80F-75CF9C491560
# /dev/sda3 : start=     6397952, size=   262035456, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=C132834A-D1E6-4858-BF34-8491CC15E71F
```

```bash
# Show the type GUID + name + size for every partition
sudo blkid -p /dev/sdb1
# /dev/sdb1: ... PART_ENTRY_TYPE="0fc63daf-8483-4772-8e79-3d69d8477de4"
#                PART_ENTRY_NAME="ISO9660"

# gdisk: human-readable code (EF00) plus the underlying GUID
sudo gdisk -l /dev/sdb
# Number  Start     End        Size       Code  Name
#    1    64        9709567    4.6 GiB    0700  ISO9660
#    2    9709568   9719807    5.0 MiB    EF00  EFI System Partition
```

`gdisk`'s 4-digit code is shorthand: `EF00` ↔ `C12A7328-...` (ESP), `0700` ↔ `EBD0A0A2-...` (Microsoft basic data / Linux data class), `8300` ↔ `0FC63DAF-...` (Linux filesystem data), `8E00` ↔ `E6D6D379-...` (Linux LVM).

---

### Setting partition type GUIDs

On an existing partition table:

```bash
# gdisk: interactive — t (change type) → enter code or GUID
sudo gdisk /dev/sdb

# sgdisk: scriptable, set partition 2 to ESP
sudo sgdisk --typecode=2:EF00 /dev/sdb
sudo sgdisk --typecode=2:C12A7328-F81F-11D2-BA4B-00A0C93EC93B /dev/sdb

# parted: set the "esp" flag (also sets "boot" automatically on GPT)
sudo parted /dev/sdb set 2 esp on
```


---

## 6. The full handoff sequence

```
0. Power on.
   UEFI firmware (already in SPI flash) initializes CPU, RAM,
   enumerates buses, loads its own drivers (NVMe, USB, NIC, GOP, FAT).

1. UEFI Boot Manager reads NVRAM:
   - Walks BootOrder → BootXXXX entries.
   - Each entry's Device Path resolves to (Disk, Partition, File).
   - If no entry matches, walks every device's GPT looking for partitions
     tagged with the ESP type GUID, falls back to
     \EFI\BOOT\BOOT<ARCH>.EFI on each. If still nothing, tries network
     boot (if enabled).

2. UEFI opens the ESP's FAT32 volume via Simple File System Protocol
   and reads grubx64.efi into memory.
   - LoadImage() parses PE/COFF header, relocates, verifies signature
     (under Secure Boot).
   - StartImage() jumps to GRUB's entry point with arguments
     (ImageHandle, SystemTable).

3. GRUB runs as a UEFI application:
   - Calls Boot Services to read grub.cfg, draw menu, read vmlinuz +
     initrd from /boot (which may or may not be on the ESP itself).
   - User picks a kernel (or default fires after timeout).

4. GRUB hands off to the kernel. One of two ways:
   (a) EFISTUB path (modern Linux): GRUB calls LoadImage(vmlinuz) +
       StartImage(). The kernel runs as a UEFI application, calls
       ExitBootServices() itself once it has its memory map.
   (b) Legacy hand-coded path: GRUB calls ExitBootServices(), sets up
       the kernel's expected boot protocol state (boot_params on x86,
       FDT pointer on RISC-V/ARM, etc.), then jumps to the kernel
       entry directly.

5. After ExitBootServices(): firmware Boot Services are gone. Only
   Runtime Services survive (efivarfs, RTC, reset). The kernel owns
   the machine — interrupts, MMU, peripherals, CPUs.
```

### `LoadImage` / `StartImage` — what they actually do

- **`LoadImage()`** — parses the PE/COFF header, allocates memory for the image, copies and applies relocations, runs Secure Boot signature verification (if enabled). Returns a handle but does not execute.
- **`StartImage()`** — invokes the entry point of a previously-loaded image with `(ImageHandle, SystemTable)` as arguments. The image runs as a UEFI application until it returns (or calls `ExitBootServices()` and never returns).

### `ExitBootServices()` — the Rubicon

This is the moment the firmware hands off ownership of memory, interrupts, and peripherals to the OS. After this call:

- Boot Services are gone (any pointer into them is invalid).
- Runtime Services remain available.
- The kernel's own page tables, IDT/IRQ controllers, drivers take over.

Forgetting to call it, or calling it with a stale memory map (the map version key is checked), is one of the canonical bring-up bugs.

---

## 7. EFISTUB — kernel as UEFI application

Modern Linux ships `vmlinuz` as a PE/COFF binary. The kernel's `arch/x86/boot/header.S` (and equivalents on AArch64, RISC-V) carries a PE header, so UEFI can `LoadImage` it directly.

This means:

- GRUB can `LoadImage` + `StartImage` on the kernel — same calls it would use on any UEFI app.
- You can also point an NVRAM `BootXXXX` entry directly at `vmlinuz` and skip GRUB entirely. This is "EFISTUB direct boot" — common on appliances and minimal systems.

Kernel options on EFISTUB boot are passed via the `LoadOptions` field in the `EFI_LOADED_IMAGE_PROTOCOL` — i.e., the same place a normal UEFI app would receive its argv.

---

## 8. Alternatives to GRUB

Once the kernel can be a UEFI app, GRUB's role is optional:

| Loader | What it is |
|--------|------------|
| **EFISTUB direct** | No bootloader. Kernel is the UEFI app. Fastest, simplest. |
| **systemd-boot** (`bootctl`) | Tiny UEFI app (~50 KB). Lists kernels on the ESP via simple `.conf` files, calls `LoadImage`. No scripting. |
| **shim** | Microsoft-signed UEFI app that trusts a distro CA and chain-loads a distro-signed `grubx64.efi`. Standard Secure Boot path for Linux distros. |
| **rEFInd** | Pretty graphical menu in front of LoadImage. Popular for multi-OS Mac users. |
| **GRUB** | Full scripting, BTRFS/ZFS/LVM root, encrypted /boot, multiboot, network boot, recovery shell. The Swiss Army knife. |

### Standard Secure Boot Linux chain

```
UEFI firmware
   → shim.efi               (signed by Microsoft, embeds distro CA)
      → grubx64.efi          (signed by distro)
         → vmlinuz             (signed by distro, EFISTUB)
            → kernel
```

Each link verifies the next via the EFI Image Authentication protocol. Distros embed their own CA in `shim` rather than asking Microsoft to sign GRUB on every release — that's the whole reason `shim` exists.

---

## 9. Live USB / hybrid ISO layout

A modern Linux ISO is already pre-arranged for both BIOS and UEFI before you flash it. `dd if=ubuntu.iso of=/dev/sdb` does a bit-for-bit copy — whatever layout was inside the ISO is now on the USB.

### Post-flash layout

```
NAME    PARTTYPENAME              FSTYPE   LABEL                   SIZE
sdb
├─sdb1  Linux filesystem          iso9660  Ubuntu 24.04 LTS amd64  4.7G
└─sdb2  EFI System                vfat     ESP                     5.0M
```

- **`sdb1`** — ISO9660 filesystem. Bulk of the install media (kernel, initrd, squashfs, packages, GRUB config).
- **`sdb2`** — tiny FAT32 ESP. Only `.efi` binaries needed for UEFI to bootstrap GRUB.

The two partitions overlap inside the ISO at the byte level — the FAT32 ESP is *embedded inside* the ISO9660. Same bytes, two filesystem views.

### `sdb2` (ESP) contents — what UEFI reads

```
/
└── EFI/
    └── BOOT/
        ├── BOOTX64.EFI        ← shim, signed by Microsoft
        ├── grubx64.efi        ← real GRUB, signed by Canonical
        └── mmx64.efi          ← MOK manager
```

UEFI walks `\EFI\BOOT\BOOTX64.EFI` (fallback path), launches `shim`, which chain-loads `grubx64.efi`.

### `sdb1` (ISO9660) contents — what GRUB reads after it starts

```
/
├── boot/grub/
│   ├── grub.cfg
│   ├── x86_64-efi/             ← GRUB modules
│   └── fonts/
├── casper/                      ← Ubuntu live system layer
│   ├── vmlinuz
│   ├── initrd
│   └── filesystem.squashfs
├── EFI/                         ← duplicate of ESP contents
├── isolinux/                    ← legacy BIOS boot path
├── dists/, pool/                ← apt repo for installation
└── md5sum.txt
```

### Hybrid ISO layout at the byte level

```
Offset 0     ┌─────────────────────────────────────────┐
             │ MBR (hybrid: also a GPT protective MBR) │  ← BIOS reads here
0x200        ├─────────────────────────────────────────┤
             │ GPT primary header + entries            │  ← UEFI reads here
~0x10000     ├─────────────────────────────────────────┤
             │ ISO9660 volume descriptors              │  ← optical drives
             ├─────────────────────────────────────────┤
             │ ISO9660 file data                       │
             ├─────────────────────────────────────────┤
             │ Embedded FAT32 ESP image                │  ← UEFI mounts this
             ├─────────────────────────────────────────┤
             │ GPT backup header (at end of disk)      │
End          └─────────────────────────────────────────┘
```

The GPT's "EFI System Partition" entry points at the offset where the embedded FAT32 image starts. The ISO9660 metadata at offset 0x10000 *also* indexes those same bytes as a regular file inside `/EFI/`. Both filesystem views resolve to the same underlying bytes.

---

## 10. RISC-V / embedded — the SiFive variant

The picture above is x86 / ARM server world. On a SiFive HiFive Unmatched (or any RISC-V SoC), the chain is layered differently but the *interface contract* between firmware and bootloader is still UEFI-shaped.

### The full RISC-V boot chain

```
ZSBL (mask ROM, soldered into the SoC)
   → FSBL / U-Boot SPL       (DRAM init, runs in M-mode)
      → OpenSBI                (M-mode runtime, services SBI ecalls)
         → U-Boot proper        (S-mode, optionally with EFI loader)
            → GRUB or kernel      (S-mode, EFISTUB or direct booti)
```

| Stage | Privilege | Job |
|-------|-----------|-----|
| ZSBL | M-mode | Bring up minimal CPU state, load next stage from QSPI / SD into SRAM |
| U-Boot SPL | M-mode | DRAM init, load OpenSBI + U-Boot proper |
| OpenSBI | M-mode | Stays resident as M-mode runtime; handles `ecall` from S-mode |
| U-Boot proper | S-mode | Main interactive bootloader; can implement UEFI Boot Services |
| GRUB | S-mode | Optional UEFI application; or skipped via direct `booti` |
| Linux | S-mode | The OS |

### U-Boot's UEFI subset (`CONFIG_EFI_LOADER`)

Build U-Boot with `CONFIG_EFI_LOADER=y` and it exposes enough of UEFI Boot Services that `grubriscv64.efi` runs unmodified. Distros (Fedora, Debian, openSUSE) ship RISC-V images that boot exactly this way.

So on a SiFive board you actually get **U-Boot in two roles**:

1. Native U-Boot bootloader (its own commands, environment, `bootcmd`).
2. UEFI firmware emulator (Boot Services for GRUB to call).

### EBBR — Embedded Base Boot Requirements

EBBR is a spec that says: *an embedded board is conformant if it presents a UEFI-ish interface to the OS, even if the underlying firmware is U-Boot.*

Key implications:
- The same `grubriscv64.efi` boots on both EDK2 (full Tianocore UEFI) and U-Boot.
- The OS image author doesn't need to know whether the firmware is UEFI or U-Boot.
- The interface is UEFI; the implementation differs.

There's also full **EDK2 (Tianocore) UEFI** for HiFive Unmatched and similar boards if you want the real thing instead of U-Boot's subset.

### RISC-V live USB layout

Same hybrid ISO shape as x86, different filenames:

```
sdb2 (FAT32 ESP, ESP type GUID):
  /EFI/BOOT/BOOTRISCV64.EFI    ← shim or GRUB for riscv64
  /EFI/BOOT/grubriscv64.efi

sdb1 (ISO9660, Linux data type GUID):
  /boot/grub/grub.cfg
  /boot/initrd.img-*
  /boot/vmlinuz-*
  /EFI/BOOT/...                ← duplicate
  /dtbs/                        ← device tree blobs per supported board
```

Notable RISC-V differences from x86:

- No `isolinux/` directory — there's no legacy BIOS path on RISC-V. UEFI/EBBR or it doesn't boot.
- `dtbs/` carries device tree blobs for the various SoCs — U-Boot picks the right one.
- Sometimes a U-Boot binary at the start of the image (instead of relying on the SoC's existing U-Boot), flashed at a known offset. Fedora's RISC-V images do this.

### Common SiFive bring-up failure modes

- **SD card flashed without an ESP**: U-Boot's UEFI path finds nothing. Drop to U-Boot prompt and `booti` the kernel image at a fixed offset by hand.
- **ESP exists but type GUID is wrong**: U-Boot's UEFI scan ignores it (same rule as full UEFI — only ESP-typed partitions are tried for the fallback path). Re-tag with `sgdisk --typecode=N:EF00`.
- **Wrong DTB**: kernel boots but probes wrong devices.
- **OpenSBI mismatch**: older OpenSBI without Zicbom / Zicboz support → kernel using `cbo.*` traps as illegal instruction. Match OpenSBI version to kernel.
- **NVMe enumeration timing**: disk not ready when U-Boot scans. Add `nvme scan` to `bootcmd` or wait for hotplug.

---

## 11. Quick reference — what goes where

| Item | Lives where | Format | When loaded |
|------|------------|--------|-------------|
| UEFI firmware | SPI flash (motherboard) | Vendor blob | Power-on |
| NVRAM Boot variables | SPI flash (separate region) | UEFI variable store | Power-on |
| `grubx64.efi` | ESP (FAT32, GUID `C12A7328-...`) | PE/COFF | UEFI Boot Manager |
| `grub.cfg`, `vmlinuz`, `initrd` | `/boot` (often ext4/btrfs) | Kernel binary, cpio | GRUB via Boot Services |
| Linux kernel | `/boot/vmlinuz-*` | bzImage / Image (PE/COFF if EFISTUB) | GRUB or UEFI directly |
| RISC-V SBI runtime | SPI flash + RAM | OpenSBI binary | U-Boot SPL → resident in M-mode |
| RISC-V device tree | RAM, passed via `a1` register | FDT blob | U-Boot → kernel |

---

## 12. Common confusions, resolved

> **Does grubx64.efi need to be on disk before UEFI runs?**

UEFI firmware itself: no, it's in SPI flash. But for UEFI to *load GRUB*, yes, `grubx64.efi` must already exist on a storage device the firmware can read (or be reachable via PXE/HTTP). The two escape hatches for fresh disks: removable-media fallback path (`\EFI\BOOT\BOOT<ARCH>.EFI`) and network boot.

> **How does UEFI know which partition is the ESP on a freshly-flashed USB?**

It reads the GPT (or MBR for older images) and inspects each partition's **type GUID**. Only the partition tagged with `C12A7328-F81F-11D2-BA4B-00A0C93EC93B` (or MBR type `0xEF`) is considered for the fallback path. Other partitions are ignored — even if they coincidentally contain a FAT filesystem with `\EFI\BOOT\BOOTX64.EFI` inside.

> **Why does GRUB exist on UEFI systems if the kernel can be a UEFI app?**

Features. Encrypted `/boot`, BTRFS/ZFS/LVM root, multiboot, dual-OS menu, scripting, recovery shell. If you don't need those, drop GRUB and use EFISTUB or systemd-boot.

> **What's the difference between UEFI and OpenSBI?**

UEFI is one *kind* of firmware (x86 / ARM / RISC-V server world). OpenSBI is RISC-V-specific M-mode runtime that handles `ecall` from S-mode. They're not alternatives — they're at different levels. On RISC-V you can have OpenSBI + UEFI together (EDK2 on top of OpenSBI), or OpenSBI + U-Boot-with-EFI-loader, or OpenSBI + U-Boot without EFI.

> **What does `ExitBootServices()` change?**

Before: every UEFI Boot Services function is callable. After: they're gone forever. Only Runtime Services (`efivarfs`, RTC, reset, capsule update) remain. The kernel must take over interrupt controllers, MMU, scheduling, drivers from this moment.

---

## 13. Sources / further reading

- **UEFI Specification** (latest at https://uefi.org/specifications) — Chapters on Boot Manager, Image Services, EFI System Partition, Device Paths. Appendix A lists all reserved Partition Type GUIDs.
- **Discoverable Partitions Specification** — https://uapi-group.org/specifications/specs/discoverable_partitions_specification/ — per-architecture root, `/home`, swap GUIDs used by systemd.
- **Linux kernel `Documentation/admin-guide/efi-stub.rst`** — EFISTUB details.
- **`efibootmgr`, `gdisk`, `sgdisk`, `parted`, `blkid`** — partition table inspection / manipulation.
- **EBBR** — https://github.com/ARM-software/ebbr — Embedded Base Boot Requirements.
- **U-Boot `doc/develop/uefi/`** — U-Boot's UEFI subset implementation notes.
- **OpenSBI documentation** — https://github.com/riscv-software-src/opensbi — RISC-V M-mode runtime.
- **EDK2 (Tianocore)** — https://www.tianocore.org/ — full open-source UEFI implementation, including RISC-V port.
- **`shim` README** — https://github.com/rhboot/shim — Secure Boot chain explanation.
- **`xorriso`** — https://www.gnu.org/software/xorriso/ — hybrid ISO build tool used by every major distro.

---

## 14. SiFive interview-grade summary

> UEFI is firmware in SPI flash; GRUB is a regular PE/COFF binary on the EFI System Partition (FAT32) that UEFI loads via `LoadImage`/`StartImage`. UEFI's NVRAM Boot Manager is the index — `BootOrder` → `BootXXXX` Device Paths — with `\EFI\BOOT\BOOTX64.EFI` as the removable-media fallback. The fallback path identifies the ESP by its partition type GUID `C12A7328-F81F-11D2-BA4B-00A0C93EC93B` (or MBR type `0xEF`); non-ESP partitions are skipped entirely. Hybrid ISOs are built with `xorriso --grub2-mbr -append_partition 2 0xef ...` to splice in a FAT32 ESP image alongside the ISO9660 payload, so a single `dd` produces a USB that satisfies BIOS, UEFI, and optical-drive boot paths simultaneously. GRUB runs *on top of* UEFI Boot Services, reads `grub.cfg` and the kernel via firmware FS drivers, then either `LoadImage`s the kernel (EFISTUB) or jumps directly after `ExitBootServices()`. On SiFive boards, U-Boot stands in for UEFI firmware via `CONFIG_EFI_LOADER` + EBBR — same `grubriscv64.efi` runs on both EDK2 Tianocore and U-Boot, because EBBR codifies UEFI as the *interface* even when the *implementation* is U-Boot. The full RISC-V chain is ZSBL → SPL → OpenSBI (M-mode resident) → U-Boot (S-mode, optionally with EFI loader) → GRUB → kernel.
