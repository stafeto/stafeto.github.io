+++
title = "stafeto"
description = "A small capability microkernel in Rust for AArch64, with a POSIX layer that runs ready-made programs."
template = "index.html"
+++

stafeto is a small capability microkernel written in Rust for AArch64, built for real-time work: every path inside the kernel is bounded in time, and every driver and service runs as an ordinary isolated process. A POSIX layer on top of it runs ready-made programs such as BusyBox, nginx, Doom and htop. It runs in QEMU on any host, with hardware acceleration (HVF) on a Mac with Apple silicon, under Apple Virtualization.framework, and on AWS EC2 Graviton. The project is experimental. The name is Esperanto for "messenger" and "relay".

## Latest release

[stafeto 0.6.1](https://github.com/stafeto/stafeto/releases/tag/v0.6.1) repairs the data volume after a power cut in seconds. It is a patch release of [0.6.0](https://github.com/stafeto/stafeto/releases/tag/v0.6.0), which connects POSIX programs to the data volume, which they see as `/data`, and starts the graphics stack of the composition service (a frame clock, OpenWF Display and the scene session of OpenWF Composition).

## Where to go

- [Try it in five minutes](@/try.md): run the release archive on your machine, or launch the public image on AWS.
- [The blog](@/blog/_index.md): posts about the design and the progress of the project.
- [The source code](https://github.com/stafeto/stafeto) on GitHub.
- [All releases](https://github.com/stafeto/stafeto/releases), with archives and release notes.
- [The documentation](https://github.com/stafeto/stafeto/tree/main/docs), which lives in the repository: status, timing, POSIX coverage, packages and releases.
