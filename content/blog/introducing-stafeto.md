+++
title = "Introducing stafeto"
description = "What stafeto is, why its kernel holds every step to a bound, what runs on it today, how to try it and what comes in 0.7."
date = 2026-11-21
draft = true
+++

stafeto is a POSIX operating system on its own capability microkernel, written in Rust for AArch64. This post says why the kernel is built around bounded steps, what runs on it today, how to try it in five minutes, and what the next release brings.

## Why a microkernel with bounded steps

Real-time, embedded and server systems need predictable timing, and a fault in a driver that ends only that driver. stafeto aims at both. Every path inside the kernel is bounded in time, and every driver and service runs as an ordinary isolated process. A kernel small enough to reason about is the base for a system that runs ordinary POSIX software on top of it.

The kernel is about 210 KiB and has 38 system calls over handles with rights: processes, threads, channels, timers, memory objects, device windows and interrupts. Messages are synchronous and carry priority inheritance. Scheduling uses fixed priorities with 64 levels and sporadic-server time budgets. Every process has quotas for kernel memory, and memory is never writable and executable at once.
<!-- SOURCE: README.md (tag v0.6.0), section "What works today", item Kernel: "about 210 KiB (212,997 bytes as measured by cargo xtask kernel-size)", "38 system calls", "64 levels". -->

### The gate of 20,410 ticks

Services run in user space, so a service can hold up a more urgent thread just as the kernel can. stafeto limits that with one number. Every step of a service stays under a gate of 20,410 ticks. The same number, called B, is the blocking term in the response-time analysis of the system.
<!-- SOURCE: docs/non-preemptible-paths.md (tag v0.6.0): "TERM_B (20,410), the budget of the blocking term for the response-time analysis and for the steps of the services" (about line 285); docs/status.md line 162: "Every step of the service stays below the services' budget of 20,410 ticks". -->

A tick here is one instruction. The measurements run in QEMU with its instruction counter (`-icount`), where one tick of the counter `CNTVCT_EL0` is one instruction. The counts do not see barriers, cache misses or other costs that show up only on hardware, so they compare versions of the kernel with each other, and the bounds in time on hardware come from measurements on hardware.
<!-- SOURCE: docs/non-preemptible-paths.md (tag v0.6.0), the paragraph above the table of paths: "under -icount shift=4,sleep=off, where one tick of CNTVCT_EL0 is one instruction" and "they compare versions of the kernel, and bounds in time come from hardware" (about lines 317 to 324). -->

The check has three layers, and the continuous integration (CI) runs them on every change:

- **The kernel.** The longest stretch the kernel runs with interrupts off is measured and enforced in CI: 8,870 of a limit of 18,000 instructions. Long operations run in bounded portions, such as 32 handles at a time when a process is torn down. The test `kernel-test icount` fails when the kernel's longest row passes the bound of 18,000.
- **The services.** Each service has a command that boots it under `-icount` and drives it to its longest steps, for example `cargo xtask process-steps` for the process service and `cargo xtask posix-sock-steps` for the socket calls. The tool compares the own part of every kind of step of every service with 20,410, and checks the waits on other services separately.
- **Preemptions.** A step is judged by two charges for each preemption it counts: 487 ticks against the hard bound B of 20,410 and 1,300 ticks against 0.9 B, which is 18,369.
<!-- SOURCE for the three items: README.md "Bounded timing" (8,870 of 18,000); docs/non-preemptible-paths.md "KERNEL_B_MAX (18,000) ... which kernel-test icount fails above" (about line 286), "Own part and wait of a step" (about line 636), the 32-entry portions (about line 277); docs/releases/0.6.0.md, "Kernel and services", item "The step gate": "487 ticks against the hard bound B (20,410) and 1,300 ticks against 0.9 B (18,369)". -->

A bound that is only written down drifts. In 0.6.0 the number found a real defect: one unlock of a whole file in the RAM file service released 64 pages in a single step, and measured 21,157 and 21,444 ticks against the 20,410 of the gate. A request now parks the page it empties, and a maintenance turn releases 16 pages at a time.
<!-- SOURCE: docs/releases/0.6.0.md, "Kernel and services", item "Parked lock pages in the RAM file service": "21,157 and 21,444 ticks against the term B of 20,410 ... a maintenance turn releases 16 pages at a time". -->

## What runs on it today

stafeto 0.6.0 is the latest release. It is experimental, and the ABI of its series is frozen.

- **POSIX.** All 1,035 interfaces that POSIX.1-2024 requires are present. The os-test `basic` suite passes 1,184 of 1,187 tests and the Open POSIX Test Suite passes 1,986 of 2,036. That covers `fork`, `exec`, signals, threads, timers, `mmap`, file locks, local sockets and `/proc`.
- **Programs.** relibc is the C library. BusyBox 1.37 and the ready-made utilities linked into it give 103 of the 112 POSIX utilities, and GNU bash, GNU nano, htop and ncdu run on it. Every third-party program is a signed package.
- **Network.** A TCP/IP service in user space with sockets and epoll, DHCP, `wget` and `curl` over TLS, nginx (also on a user-space FreeBSD stack, F-Stack) and Dropbear SSH. On EC2 the stack runs over the ENA device.
- **Several CPUs.** Up to 8 CPUs are supported and 4 are tested, with partitioned scheduling, so a real-time task keeps its CPU and its bounds.
- **Files that last.** POSIX programs see the data volume as `/data`, with permissions by the rules of POSIX. A cut of the power leaves the old or the new generation of a file, and BusyBox `vi` saves there.
- **Graphics.** The composition service has a frame clock at 60 Hz by default. OpenWF Display is implemented over the display driver, and the service draws the elements of an OpenWF Composition scene. Doom is playable.
- **Machines.** QEMU `virt` on any host, QEMU with HVF on Apple silicon, Apple Virtualization.framework, and AWS EC2 Graviton.
<!-- SOURCE: README.md (tag v0.6.0), "What works today": 1,035 interfaces; os-test basic 1,184 of 1,187; Open POSIX 1,986 of 2,036; BusyBox 1.37; 103 of 112 utilities; up to 8 CPUs and 4 tested; Machines. docs/releases/0.6.0.md: /data, frame clock 60 Hz by default, OpenWF Display and scene session; docs/doom.md: Doom. "Every third-party program is a signed package": README.md "Programs" says every program is a package, and "Security" says packages are signed. -->

## Try it

Download `stafeto-0.6.0-aarch64-virt-full.tar.xz` from the [0.6.0 release](https://github.com/stafeto/stafeto/releases/tag/v0.6.0), unpack it and run `./run.sh` (add `--hvf` on a Mac with Apple silicon). At the `ash` prompt, `doom -iwad /games/freedoom1.wad` starts Doom in the QEMU window. A public image for AWS EC2 Graviton is also available. The [Try it](@/try.md) page has the steps, the AMI id and the costs.

## What comes in 0.7

The next release is planned around these parts. Plans move, so read this list as the direction and the release notes as the record.

- **Ordinary user login.** Logging in as an ordinary user, with the rights of that user.
- **A faster `/data`.** In 0.6.0 a call moves at most 1 KiB, so a large file is thousands of calls: 64 MiB took 31 s to write and 48 s to read under QEMU. The planned fix moves the data through a memory object.
- **A layered file system with snapshots.** Read-only layers (the signed packages are the first of them) with a writable layer on top, and a snapshot that freezes the top layer so that new writes start a new one.
- **The graphics track.** OpenWF Composition and Display are in 0.6.0. Next come EGL and OpenGL ES through Mesa.
<!-- SOURCE: the items "ordinary user login" and "layered file system with snapshots" come from the owner's plan for 0.7 and from the design note on layers (stafeto-design, tasks/2026-11-21-layered-fs-design.md); they are not yet in a public file of the stafeto repository. CONFIRM before publishing. The "faster /data" numbers: docs/releases/0.6.0.md, Limits: "Under QEMU 64 MiB were written in 31 s and read in 48 s ... The fix (a transfer through a memory object) is planned after 0.6." Graphics: README.md "Later subprojects" and docs/releases/0.6.0.md "Not in this release" (libWFC is not part of it). The EGL, OpenGL ES and Mesa step is from the owner's graphics roadmap; CONFIRM. -->

The source is at [github.com/stafeto/stafeto](https://github.com/stafeto/stafeto), and the documentation is in its `docs` directory.
