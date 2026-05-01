---
tags:
  - linux-kernel
  - gpio
  - device-driver
  - testing
created: '2026-04-29'
---
# GPIO Emulation — gpio-mockup and gpio-sim

## Overview

|                  | gpio-mockup        | gpio-sim |
| ---------------- | ------------------ | -------- |
| Introduced       | ~4.x               | 5.17     |
| Status           | Deprecated         | Active   |
| IRQ simulation   | Unreliable on 6.x+ | Reliable |
| Per-line control | debugfs            | configfs |
| Recommended      | No                 | Yes      |

---

## gpio-mockup

### Load

```bash
# Check availability
modinfo gpio-mockup

# Load with one chip of 8 pins (global numbers 512–519)
sudo modprobe gpio-mockup gpio_mockup_ranges=0,8

# Verify
cat /sys/kernel/debug/gpio | grep mockup
gpiodetect
```

### Toggle a pin (may not trigger IRQ on newer kernels)

```bash
echo 1 > /sys/kernel/debug/gpio-mockup/gpiochip0/0
echo 0 > /sys/kernel/debug/gpio-mockup/gpiochip0/0
```

### Unload

```bash
sudo rmmod gpio-mockup
```

---

## gpio-sim (recommended)

### Load

```bash
modinfo gpio-sim
sudo modprobe gpio-sim

# Mount configfs if not already mounted
mount -t configfs configfs /sys/kernel/config
```

### Create a simulated chip via configfs

```bash
# 1. Create chip directory
mkdir /sys/kernel/config/gpio-sim/my-chip

# 2. Create a bank — the directory name becomes part of the chip label
mkdir /sys/kernel/config/gpio-sim/my-chip/bank0

# 3. Set number of lines
echo 8 > /sys/kernel/config/gpio-sim/my-chip/bank0/num_lines

# 4. Create per-line directories BEFORE going live
mkdir /sys/kernel/config/gpio-sim/my-chip/bank0/line0
mkdir /sys/kernel/config/gpio-sim/my-chip/bank0/line1

# 5. Activate the chip
echo 1 > /sys/kernel/config/gpio-sim/my-chip/live
```

### Verify

```bash
gpiodetect
# gpiochip0 [gpio-sim.0-node0] (8 lines)
#            ^^^^^^^^^^^^^^^^^^
#            actual label format: gpio-sim.<id>-node<bank-number>

gpioinfo gpiochip0
cat /sys/kernel/debug/gpio | grep sim
```

> The chip label is **not** just the bank directory name. gpio-sim generates it as `gpio-sim.<id>-<bank>`, e.g. `gpio-sim.0-node0`. Always verify with `gpiodetect` before using in `GPIO_LOOKUP`.

### Trigger IRQ (pull a line high/low)

Signal injection is done via **sysfs**, not configfs. configfs is only for chip configuration.

```bash
# Find the correct path first
ls /sys/devices/platform/gpio-sim.0/gpiochip0/

# Rising edge (triggers IRQF_TRIGGER_RISING)
echo 0 > /sys/devices/platform/gpio-sim.0/gpiochip0/sim_gpio0/pull
echo 1 > /sys/devices/platform/gpio-sim.0/gpiochip0/sim_gpio0/pull

# Falling edge (triggers IRQF_TRIGGER_FALLING)
echo 0 > /sys/devices/platform/gpio-sim.0/gpiochip0/sim_gpio0/pull
```

> `sim_gpioX` where X is the **line offset** (0, 1, 2…), not the global GPIO number.
> The `.0` suffix and `gpiochipY` number may differ — check with `ls /sys/devices/platform/gpio-sim.*/`.

### Tear down

```bash
echo 0 > /sys/kernel/config/gpio-sim/my-chip/live
rmdir /sys/kernel/config/gpio-sim/my-chip/bank0/line0
rmdir /sys/kernel/config/gpio-sim/my-chip/bank0/line1
rmdir /sys/kernel/config/gpio-sim/my-chip/bank0
rmdir /sys/kernel/config/gpio-sim/my-chip
sudo rmmod gpio-sim
```

---

## Using gpio-sim in a kernel driver

### Lookup table

```c
static struct gpiod_lookup_table my_gpio_table = {
    .dev_id = "my-gpio-dev",
    .table  = {
        // key must match the chip name shown in brackets by gpiodetect
        GPIO_LOOKUP("gpio-sim.0-node0", 0, "my-pin", GPIO_ACTIVE_HIGH),
        { }
    },
};
```

### Verify the chip label

The first parameter of `GPIO_LOOKUP` must match the chip name shown **in brackets** by `gpiodetect`:

```bash
gpiodetect
# gpiochip0 [gpio-sim.0-node0] (8 lines)
#            ^^^^^^^^^^^^^^^^^^
#            use exactly this string as the key
```

gpio-sim generates the label as `gpio-sim.<instance>-<bank-directory-name>` — it is **not** simply the bank directory name. Always confirm with `gpiodetect` after creation.

The `@key` field in `struct gpiod_lookup` is documented as:
> either the name of the chip the GPIO belongs to, or the GPIO line name

### Common errors

| Error | Cause | Fix |
|---|---|---|
| `-ENOENT` (−2) | Chip label mismatch | Check `gpiodetect`, fix label in `GPIO_LOOKUP` |
| `-EBUSY` (−16) | Pin already claimed | Check `gpioinfo`, remove hog directory or `rmmod gpio-mockup` |
| IRQ never fires | Pin direction wrong | Must use `GPIOD_IN`, not `GPIOD_OUT_*` |
| IRQ never fires | Trigger flag mismatch | `IRQF_TRIGGER_RISING` needs 0→1 transition |

### Check who owns a pin

```bash
gpioinfo                          # shows "used" / "unused" per pin
cat /sys/kernel/debug/gpio        # shows driver label of claimed pins
```

---

## Verify IRQ from userspace first

Before testing in a driver, confirm gpio-sim IRQ simulation works:

```bash
# Terminal 1 — watch for events
gpiomon --num-events=2 --both-edges gpiochip0 0

# Terminal 2 — trigger via sysfs (not configfs)
echo 1 > /sys/devices/platform/gpio-sim.0/gpiochip0/sim_gpio0/pull
echo 0 > /sys/devices/platform/gpio-sim.0/gpiochip0/sim_gpio0/pull
```

If `gpiomon` receives events, IRQ simulation is working correctly.

---

## References

- `Documentation/driver-api/gpio/consumer.rst`
- `tools/testing/selftests/gpio/`
- `drivers/gpio/gpio-sim.c`
- `drivers/gpio/gpio-mockup.c`
