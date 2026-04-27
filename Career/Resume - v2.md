---
tags:
  - career
---

# 殷豪
**Senior Systems & Kernel Engineer** 

jjyyg1123@gmail.com | 0937150335 | Hsinchu  
[Linkedin](https://www.linkedin.com/in/hao-yin-9178bb13b) | [GitHub](https://github.com/JackyYin)

---

## Professional Summary
Performance-oriented **Senior Systems Engineer** with extensive experience in **Linux Kernel internals, eBPF development, and high-performance C/C++ security engines.** 
Proven expertise in cross-platform porting (PowerPC, s390x), exploit mitigation (Intel CET), and low-level debugging. 
Passionate about low-level hardware-software co-design and MCU-class platform enablement.

---

## Technical Skills
* **Languages:** C, C++, Assembly (x86, arm64, PowerPC, s390), Shell (Bash), Python, Node.js, PHP (Laravel).
* **Kernel:** Linux Kernel Modules, eBPF, ELF/Linkers, Page Table Walking, Livepatch, Memory Management.
* **MCU Protocols:** SPI, USB, UART
* **Security & Hardware:** Intel CET (Shadow Stack/IBT), Retpoline, Rethunk, Big-Endian/Little-Endian optimization
* **Tools:** Kdump/Crash, LLVM, Git, CMake, GDB, Valgrind, Perf.

---

## Side Projects

### **Game Boy Emulator**
[Link To Repository](https://github.com/JackyYin/gb-emu)
* **Core Architecture:** Developed a full-featured Game Boy (LR35902/SM83) emulator from scratch using C/C++, focusing on cycle-accurate instruction timing.
* **DMA & Memory Architecture:** Implemented **DMA-driven pixel transfers** between VRAM and the display framebuffer, and designed a **Memory Bank Controller (MBC1)** supporting bank switching for large ROMs (up to 2MB) and external RAM.
* **Graphics:** Engineered a **Pixel Processing Unit (PPU)** — implementing a tile-based rendering pipeline, **VRAM and OAM (Object Attribute Memory) buffer management**, and memory-mapped graphics registers.
* **Hardware & I/O**: Simulated hardware peripheral interfaces including timer interrupts, serial I/O (analogous to SPI), and memory-mapped I/O — mirroring real MCU peripheral driver design patterns.

---


## Experience

### **Trend Micro — ORCA (Linux Kernel & Security)**
*Senior Software Engineer | 2023 – 2026*

* **Architecture Porting:** Engineered Linux kernel module support for **PowerPC** and **s390x** (Mainframe) architectures. Resolved complex architecture-specific hurdles, including **PowerPC TOC (Table of Contents)** and manual **page table walking** logic.
* **Hardware-Level Security:** Integrated support for **Intel Control-flow Enforcement Technology (CET)**. Managed critical mitigations for speculative execution vulnerabilities, including **Retpoline** and **Rethunk** within live-patching frameworks.
* **eBPF Engineering:** Developed high-performance EPP/EDR features using **eBPF**. Optimized **LLVM/ELF relocation** and bypassed kernel verifier constraints to ensure stable, low-overhead system monitoring.
* **Root Cause Analysis:** Led deep-dive investigations into kernel panics and system hangs using **crash dump analysis**, improved product stability for customers.

### **Trend Micro — ERS (Email Reputation Services)**
*Senior Software Engineer | 2021 – 2023*

* **Engine Optimization:** Developed and maintained a core email scanning engine in **C/C++**, optimizing for high throughput and low latency under massive concurrent loads.
* **Stability & Debugging:** Utilized **Valgrind and GDB** on a daily basis to eliminate memory leaks, race conditions, and segmentation faults.
* **ML Integration:** Successfully integrated pre-trained machine learning models into the C++ scanning pipeline to support multi-language spam detection.

### **Earlier Experience**
*Backend Engineer, Botbonnie Inc. (2019–2021) | DevOps Engineer, STARLUX Airlines (2019) | Software Engineer, Larvata (2017–2018)*

* Built serverless backend systems using **AWS Lambda, SQS, and DynamoDB** to support high-traffic production workloads.
* Orchestrated **OpenShift** clusters and built monitoring systems using **Prometheus and EFK stack**.
* Containerized legacy projects with **Docker** and established CI/CD pipelines.

---

## Education
**Bachelor of Information Management** - National Taiwan University, Taipei
**Science Track**  - National Experimental High School, Hsinchu

---

## Languages
* **English:** B2
* **Chinese:** Native