---
tags:
  - linux-kernel
  - device-driver
  - platform
  - driver-model
created: '2026-04-29'
---
# Linux Platform Device and Driver API

## What is a Platform Device?

A **platform device** is a device that cannot be auto-discovered by the hardware — it doesn't sit on a self-enumerating bus like USB or PCI. The kernel must be told it exists via:

- Device Tree (on embedded systems like Raspberry Pi)
- ACPI tables (on x86)
- Board files / manual registration (for testing/development)

Examples: GPIO controllers, UARTs, timers, SPI/I2C host controllers.

---

## The Big Picture — Bus Model

The platform bus is the glue between `platform_device` and `platform_driver`. When both are registered, the kernel matches them by name and calls `probe()`.

```
platform_device_register("my-dev")
        ↓
  [platform bus]  ←→  match by name
        ↓
platform_driver_register(.name = "my-dev")
        ↓
  probe(&pdev->dev) called automatically
```

Both sides must use the **same name** for the match to succeed.

---

## Core Structs

### `platform_device`

Represents the hardware (or virtual) device:

```c
struct platform_device {
    const char      *name;       // must match platform_driver.driver.name
    int              id;         // -1 = only one instance, 0/1/2 = multiple
    struct device    dev;        // embedded generic device (use &pdev->dev)
    struct resource *resource;   // IRQ, memory regions, etc.
    unsigned int     num_resources;
};
```

### `platform_driver`

Represents the driver that handles the device:

```c
struct platform_driver {
    int  (*probe)(struct platform_device *);    // called on match
    void (*remove)(struct platform_device *);   // called on unbind
    struct device_driver driver;
        // .name          — must match platform_device.name
        // .of_match_table — Device Tree match table
        // .pm            — power management ops
};
```

---

## Lifecycle

```
   platform_device_alloc()        allocate struct, set name + id
          ↓
   platform_device_add()          register with bus, triggers match
          ↓
   [kernel matches device + driver by name]
          ↓
   probe(&pdev)                   driver initializes hardware
          ↓
   [driver in use]
          ↓
   platform_device_unregister()   triggers remove(), frees device
```

---

## Creating a Device Manually (no Device Tree)

Used in modules and tests where there is no DT/ACPI:

```c
static struct platform_device *my_pdev;

static int __init my_init(void)
{
    int ret;

    // 1. Allocate
    my_pdev = platform_device_alloc("my-dev", -1);
    if (!my_pdev)
        return -ENOMEM;

    // 2. Optionally add resources (IRQ, MMIO regions)
    // platform_device_add_resources(my_pdev, resources, ARRAY_SIZE(resources));

    // 3. Register — triggers probe() if driver is already registered
    ret = platform_device_add(my_pdev);
    if (ret) {
        platform_device_put(my_pdev);   // release ref on failure
        return ret;
    }

    return 0;
}

static void __exit my_exit(void)
{
    platform_device_unregister(my_pdev);   // triggers remove(), frees memory
}
```

> `platform_device_put()` decrements the reference count. Only call it on failure before `platform_device_add()` — after registration, `platform_device_unregister()` handles cleanup.

---

## Writing a Platform Driver

```c
static int my_probe(struct platform_device *pdev)
{
    struct my_priv *priv;

    // Allocate private state — tied to device lifetime via devm
    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    // Store for use in remove(), IRQ handlers, etc.
    platform_set_drvdata(pdev, priv);

    // All devm_* allocations auto-freed when device is unbound
    priv->gpio = devm_gpiod_get(&pdev->dev, "my-pin", GPIOD_IN);
    if (IS_ERR(priv->gpio))
        return PTR_ERR(priv->gpio);

    dev_info(&pdev->dev, "probe successful\n");
    return 0;
}

static void my_remove(struct platform_device *pdev)
{
    // With devm_* — usually nothing to do here
    struct my_priv *priv = platform_get_drvdata(pdev);
    dev_info(&pdev->dev, "removed\n");
}

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-dev",               // must match platform_device name
        // .of_match_table = my_of_match, // for Device Tree matching
    },
};
```

---

## Registering Device and Driver Together

A common pattern using the helper macro:

```c
// Registers both device and driver in one call
// Only use when there is exactly one instance
module_platform_driver(my_driver);

// Equivalent to:
static int __init my_init(void) { return platform_driver_register(&my_driver); }
static void __exit my_exit(void) { platform_driver_unregister(&my_driver); }
module_init(my_init);
module_exit(my_exit);
```

`module_platform_driver()` assumes the device is already registered (e.g. via DT). When manually creating a `platform_device` too, you need the explicit init/exit pattern shown above.

---

## `devm_*` — Automatic Resource Management

All `devm_*` functions tie the resource lifetime to `struct device`. When the device is unbound (removed), resources are freed automatically in **reverse order** of allocation:

```c
// All of these auto-release on driver detach — no cleanup needed in remove()
priv = devm_kzalloc(&pdev->dev, size, GFP_KERNEL);
gpio = devm_gpiod_get(&pdev->dev, "name", GPIOD_IN);
ret  = devm_request_irq(&pdev->dev, irq, handler, flags, "name", priv);
base = devm_ioremap_resource(&pdev->dev, res);
clk  = devm_clk_get(&pdev->dev, "name");
```

Without `devm_*`, you must manually free everything in `remove()` and handle partial-failure cleanup in `probe()` — error-prone and verbose.

---

## Passing State Between probe(), remove(), and IRQ Handlers

```c
// In probe() — store private struct
platform_set_drvdata(pdev, priv);

// In remove() or IRQ handler — retrieve it
struct my_priv *priv = platform_get_drvdata(pdev);
```

Internally this calls `dev_set_drvdata()` / `dev_get_drvdata()` on `&pdev->dev`.

---

## Resources

Platform devices can describe their hardware resources (MMIO, IRQ) so the driver can retrieve them portably:

```c
// Define resources on the device side
static struct resource my_resources[] = {
    {
        .start = 0x10000000,
        .end   = 0x10000FFF,
        .flags = IORESOURCE_MEM,
    },
    {
        .start = 42,
        .end   = 42,
        .flags = IORESOURCE_IRQ,
    },
};

// Retrieve in probe()
struct resource *mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);
int irq             = platform_get_irq(pdev, 0);

void __iomem *base  = devm_ioremap_resource(&pdev->dev, mem);
```

When using Device Tree, resources are populated automatically from the DT node — no manual `struct resource` needed.

---

## Device Tree Matching

On real hardware (e.g. Raspberry Pi), the driver matches via `compatible` string instead of name:

```c
static const struct of_device_id my_of_match[] = {
    { .compatible = "myvendor,my-device" },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name           = "my-dev",
        .of_match_table = my_of_match,   // ← DT match
    },
};
```

DT node in the board file:
```
my_device: my-device@10000000 {
    compatible = "myvendor,my-device";
    reg = <0x10000000 0x1000>;
    interrupts = <0 42 4>;
    gpios = <&gpio 17 GPIO_ACTIVE_HIGH>;
};
```

---

## Sysfs Visibility

Once registered, the device appears in sysfs:

```
/sys/bus/platform/devices/my-dev.0/    ← device
/sys/bus/platform/drivers/my-dev/      ← driver

# Manual bind/unbind without rmmod
echo my-dev.0 > /sys/bus/platform/drivers/my-dev/unbind
echo my-dev.0 > /sys/bus/platform/drivers/my-dev/bind
```

---

## Summary

```
Use platform_device + platform_driver when:
  ✓ Device cannot self-enumerate (no USB/PCI)
  ✓ You want devm_* auto-cleanup
  ✓ You want DT/ACPI matching on real hardware
  ✓ You want bind/unbind via sysfs
  ✓ Production driver

Use a plain module (no platform API) when:
  ✓ Quick prototype or learning exercise
  ✓ No real hardware, no DT, no resources needed
  ✓ Fewer lines of boilerplate is the priority
```

---

## GPIO Lookup Table and `.dev_id`

The `gpiod_lookup_table` wires a GPIO chip pin to a specific device by name, so the driver can request it generically via `devm_gpiod_get()` without knowing chip internals.

### How matching works

When `devm_gpiod_get(&pdev->dev, "my-pin", ...)` is called, the kernel searches all registered lookup tables for an entry where:

1. `.dev_id` matches `dev_name(&pdev->dev)`
2. `con_id` matches the requested connection name (`"my-pin"`)

```c
static struct gpiod_lookup_table my_gpio_table = {
    .dev_id = "my-gpio-dev",        // must match dev_name(&pdev->dev)
    .table  = {
        GPIO_LOOKUP("gpio-sim.0-node0", 0, "my-pin", GPIO_ACTIVE_HIGH),
        { }
    },
};
```

```
devm_gpiod_get(&pdev->dev, "my-pin", GPIOD_IN)
        ↓
  dev_name(&pdev->dev) == "my-gpio-dev"
        ↓
  scan all lookup tables for .dev_id == "my-gpio-dev"
        ↓
  found → scan entries for con_id == "my-pin"
        ↓
  found → resolve to gpio-sim.0-node0, offset 0
```

### `dev_name()` depends on platform device id

The device name is constructed from the platform device name and `id`:

```c
platform_device_alloc("my-gpio-dev", -1);
// id = -1  → dev_name = "my-gpio-dev"
// id =  0  → dev_name = "my-gpio-dev.0"
// id =  1  → dev_name = "my-gpio-dev.1"
```

If `id = 0` is used instead of `-1`, the `.dev_id` must be `"my-gpio-dev.0"` — mismatch here causes `-ENOENT` in `devm_gpiod_get()`.

### Registration order

The lookup table must be registered **before** `devm_gpiod_get()` is called (i.e. before or during `probe()`):

```c
static int __init my_init(void)
{
    gpiod_add_lookup_table(&my_gpio_table);   // register wiring first
    platform_device_add(my_pdev);             // triggers probe()
    platform_driver_register(&my_driver);     //   which calls devm_gpiod_get()
}

static void __exit my_exit(void)
{
    platform_driver_unregister(&my_driver);
    platform_device_unregister(my_pdev);
    gpiod_remove_lookup_table(&my_gpio_table);
}
```

### Why decouple it this way?

The lookup table acts as **board-level wiring** — the same role Device Tree plays on real hardware. The driver only asks for `"my-pin"` by name and gets a descriptor back, with no knowledge of which GPIO chip or offset backs it:

```
gpiod_add_lookup_table()    "wire" the GPIO at board level
        +
platform_device_add()       create the device
        +
devm_gpiod_get("my-pin")    driver asks by name → gets descriptor
```

---

## References

- `Documentation/driver-api/driver-model/platform.rst`
- `Documentation/driver-api/driver-model/device.rst`
- `include/linux/platform_device.h`
- `drivers/base/platform.c`
