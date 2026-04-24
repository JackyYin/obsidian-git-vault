---
tags:
  - linux
  - kernel
  - driver-model
---
# Linux Driver Model: Structure Relationships

## Ownership / Registration

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                        Kernel Driver Core                       │
 │                                                                 │
 │   bus_register()          class_register()    driver_register() │
 │         │                       │                    │          │
 │         ▼                       ▼                    │          │
 │  ┌─────────────┐        ┌─────────────┐              │          │
 │  │  bus_type   │        │    class    │              │          │
 │  │─────────────│        │─────────────│              │          │
 │  │ name        │        │ name        │              │          │
 │  │ match()     │        │ dev_uevent()│              │          │
 │  │ probe()     │        │ dev_release()│             │          │
 │  │ remove()    │        │ devnode()   │              │          │
 │  │ uevent()    │        │ pm          │              │          │
 │  │ pm          │        └──────┬──────┘              │          │
 │  └──────┬──────┘               │                     │          │
 │         │                      │                     ▼          │
 │         │              ┌───────────────┐   ┌────────────────┐   │
 │         │              │  device_type  │   │ device_driver  │   │
 │         │              │───────────────│   │────────────────│   │
 │         │              │ name          │   │ name           │   │
 │         │              │ uevent()      │   │ bus ───────────┼───┼──► bus_type
 │         │              │ devnode()     │   │ probe()        │   │
 │         │              │ release()     │   │ remove()       │   │
 │         │              │ pm            │   │ shutdown()     │   │
 │         │              └───────┬───────┘   │ pm             │   │
 │         │                      │           └───────┬────────┘   │
 │         │                      │                   │            │
 │         └──────────┐           └──────┐   ┌────────┘            │
 │                    │                  │   │                     │
 │                    ▼                  ▼   ▼                     │
 │              ┌───────────────────────────────────────┐          │
 │              │              device                   │          │
 │              │───────────────────────────────────────│          │
 │              │ kobj                                  │          │
 │              │ parent          ──────► device        │          │
 │              │ bus             ──────► bus_type      │          │
 │              │ driver          ──────► device_driver │          │
 │              │ type            ──────► device_type   │          │
 │              │ class           ──────► class         │          │
 │              │ platform_data                         │          │
 │              │ driver_data                           │          │
 │              │ of_node / fwnode                      │          │
 │              │ devt / id                             │          │
 │              │ pm_domain                             │          │
 │              └───────────────────────────────────────┘          │
 └─────────────────────────────────────────────────────────────────┘
```

## Binding Flow (match → probe)

```
  New device registered          New driver registered
  on bus "foo"                   on bus "foo"
         │                              │
         ▼                              ▼
  bus_type.match(dev, drv) ◄────────────┘
         │
    match? ──── No ──► try next driver/device
         │
        Yes
         │
         ▼
  bus_type.probe(dev)
    └─► device_driver.probe(dev)
              │
         success?
         │        │
        Yes       No ──► -EPROBE_DEFER → retry later
         │
         ▼
  device.driver = drv   ← bound!
```

## Sysfs Layout (reflecting the model)

```
  /sys/
  ├── bus/
  │   └── <bus_type.name>/          ← one dir per registered bus_type
  │       ├── devices/
  │       │   └── <dev> ──────────────────────────────────────────────┐
  │       └── drivers/                                                │
  │           └── <device_driver.name>/                               │
  │               └── <dev> (symlink) ◄── device.driver points here   │
  │                                                                   │
  ├── class/                                                          │
  │   └── <class.name>/             ← one dir per registered class    │
  │       └── <dev> (symlink) ◄──────── device.class points here      │
  │                                                                   │
  └── devices/                                                        │
      └── <dev>/  ◄───────────────────────────────────────────────────┘
          ├── uevent          ← DEVTYPE=<device_type.name>
          ├── subsystem ──────────────────► /sys/bus/<bus> or /sys/class/<class>
          ├── driver    ──────────────────► /sys/bus/<bus>/drivers/<drv>
          └── <attribute_groups from driver, bus, class, device_type>
```

## Responsibility Summary

```
  ┌──────────────┬─────────────────────────────────────────────────────────┐
  │  Structure   │  Role                                                   │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │  bus_type    │  Topology + binding. Owns match/probe pipeline.         │
  │              │  Knows all devices AND drivers on the bus.              │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │  device_     │  Belongs to one bus. Implements probe/remove for a      │
  │  driver      │  specific set of devices. Bound to device at runtime.   │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │  device      │  A concrete hardware instance. References bus, driver,  │
  │              │  type, and class simultaneously.                        │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │  device_type │  Sub-taxonomy within a bus (e.g. "disk" vs "partition"  │
  │              │  on SCSI). Affects uevent DEVTYPE and devnode naming.   │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │  class       │  Functional view for userspace (what it does, not how   │
  │              │  it connects). No role in driver binding at all.        │
  └──────────────┴─────────────────────────────────────────────────────────┘
```


Refs:
- https://www.youtube.com/watch?v=AdPxeGHIZ74
- https://static.lwn.net/images/pdf/LDD3/ch14.pdf
- https://ithelp.ithome.com.tw/articles/10243519