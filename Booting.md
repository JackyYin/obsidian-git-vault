---
tags:
  - computer_architecture
---
```
                          HARDWARE POWER ON
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         FIRMWARE LAYER                              │
  │                                                                     │
  │      Platform with UEFI                  Bare-metal (no UEFI)       │
  │                                                                     │
  │   ┌───────────────────────┐           ┌───────────────────────┐     │
  │   │     BIOS (legacy)     │           │        U-Boot         │     │
  │   │    or UEFI (modern)   │           │                       │     │
  │   │                       │           │ - hardware init       │     │
  │   │  e.g.                 │           │ - can load kernel     │     │
  │   │  x86 desktop/server   │           │   directly, OR        │     │
  │   │  ARM64 server         │           │   emulate UEFI        │     │
  │   │  (Ampere, Graviton)   │           │   interface           │     │
  │   │  RISC-V server        │           │                       │     │
  │   │  (with EDK2)          │           │  e.g.                 │     │
  │   │  RPi4 (with EDK2)     │           │  RPi4 (bare metal)    │     │
  │   │                       │           │  BeagleBone           │     │
  │   │ - POST / hardware init│           │  SiFive HiFive        │     │
  │   │ - Secure Boot         │           │  STM32 / ESP32        │     │
  │   │ - finds bootloader    │           │                       │     │
  │   └──────────┬────────────┘           └──────────┬────────────┘     │
  └──────────────┼─────────────────────────────────--┼──────────────────┘
                 │                                   │
                 │                        ┌──────────┴───────────┐
                 │                        │                      │
                 │                  emulate UEFI            load kernel
                 │                  (e.g. EDK2)              directly
                 │                        │                      │
                 │                        ▼                      │
                 │              (acts as UEFI provider)          │
                 │                        │                      │
                 └───────────────┬────────┘                      │
                                 │                               │
                                 ▼                               │
  ┌──────────────────────────────────────────┐                   │
  │           BOOTLOADER LAYER               │                   │
  │                                          │                   │
  │  ┌────────────────────────────────────┐  │                   │
  │  │               GRUB                 │  │                   │
  │  │                                    │  │                   │
  │  │  Platforms:                        │  │                   │
  │  │  - x86 (BIOS or UEFI)              │  │                   │
  │  │  - ARM64 server (UEFI)             │  │                   │
  │  │  - RISC-V server (UEFI)            │  │                   │
  │  │  - RPi4 (via U-Boot UEFI)          │  │                   │
  │  │                                    │  │                   │
  │  │  - shows OS selection menu         │  │                   │
  │  │  - loads kernel (vmlinuz)          │  │                   │
  │  │  - loads initramfs                 │  │                   │
  │  │  - passes cmdline args to kernel   │  │                   │
  │  └──────────────────┬─────────────────┘  │                   │
  └─────────────────────┼────────────────────┘                   │
                        │                                        │
                        └────────────────┬───────────────────────┘
                                         │
                                         ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         KERNEL LAYER                                │
  │                                                                     │
  │   ┌──────────────────────────────────────────────────────────┐      │
  │   │               Linux Kernel (vmlinuz)                     │      │
  │   │                                                          │      │
  │   │  - decompresses itself                                   │      │
  │   │  - initializes CPU, memory, interrupts                   │      │
  │   │  - mounts initramfs as temporary root filesystem         │      │
  │   │  - runs /init inside initramfs                           │      │
  │   └──────────────────────────┬───────────────────────────────┘      │
  │                              │                                      │
  │   ┌──────────────────────────▼───────────────────────────────┐      │
  │   │                      initramfs                           │      │
  │   │                 (initial RAM filesystem)                 │      │
  │   │                                                          │      │
  │   │  - loads essential drivers (disk, crypto, LVM, RAID)     │      │
  │   │  - unlocks encrypted volumes (LUKS)                      │      │
  │   │  - assembles RAID arrays                                 │      │
  │   │  - mounts the REAL root filesystem                       │      │
  │   │  - hands off via switch_root                             │      │
  │   └──────────────────────────┬───────────────────────────────┘      │
  └──────────────────────────────┼──────────────────────────────────────┘
                                 │  switch_root
                                 ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                        USERSPACE LAYER                              │
  │                                                                     │
  │   ┌──────────────────────────────────────────────────────────┐      │
  │   │                    systemd (PID 1)                       │      │
  │   │                                                          │      │
  │   │  - first real userspace process                          │      │
  │   │  - starts services in dependency order                   │      │
  │   │  - mounts filesystems (/etc/fstab)                       │      │
  │   │  - brings up networking                                  │      │
  │   │  - launches login / SSH / daemons                        │      │
  │   └──────────────────────────────────────────────────────────┘      │
  │                              │                                      │
  │            ┌─────────────────┼──────────────────┐                   │
  │            ▼                 ▼                  ▼                   │
  │       networking           sshd            getty (login)            │
  └─────────────────────────────────────────────────────────────────────┘