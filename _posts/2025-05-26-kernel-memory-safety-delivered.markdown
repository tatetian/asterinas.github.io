---
layout: post
title:  "Kernel Memory Safety, Delivered"
date:   2025-05-26 17:00:00 +0800
author: "Hongliang Tian"
---

Less than one year ago, on July 19, 2024, millions of Windows desktops, servers, and devices suddenly crashed with the infamous "blue screen of death". This global incident—now known as [the CrowdStrike outage](https://en.wikipedia.org/wiki/2024_CrowdStrike-related_IT_outages)—was triggered by a memory-safety bug in a Windows driver: an out-of-bounds access. It was a sobering reminder that even mature, commercial off-the-shelf operating systems (OSes) like Windows and Linux remain vulnerable to memory-safety flaws.

Two years before the CrowdStrike outage, around mid-2022, we began asking: could Rust finally solve the problem of kernel memory safety? Not partially. Not with guardrails. But completely.

We came up with this new OS organization principle called the **framekernel** architecture, designed to unlock the full potential of Rust by confining `unsafe` code to a minimal, auditable scope within the kernel. The idea was bold but simple:

> What if we could build an entire OS kernel where **everything but a small trusted core is written in safe Rust**?

From this idea, [**the Asterinas project**](https://github.com/asterinas/asterinas) was born—the first real-world implementation of a framekernel. We didn't know if it would work. 

1. How *small* can the TCB be?
2. Is this too restrictive for _rich_ OS features?
3. Would *performance* suffer?
4. Are memory-safety _bugs_ really gone?

After three years of development and over 100,000 lines of Rust, Asterinas has grown from a research prototype into a maturing system — with real-world use on the horizon. We've seen enough, built enough, and tested enough to say this with confidence:

> Kernel memory safety has been delivered — with no compromises.

We proved that a kernel with rich functionality, Linux compatibility, and competitive performance can be built entirely in safe Rust, atop a small and sound OS framework. We named this framework **OSTD**.

We reported the design and implementation of Asterinas and OSTD in a research paper, which has been accepted to [USENIX ATC 2025](https://www.usenix.org/conference/atc25/technical-sessions). You can find the preprint [here]().

In this post, we'll give you a lightning tour of the framekernel architecture—and show how Asterinas brings it to life.

## Framekernel 101

The framekernel combines the best of both worlds:

> The performance of a monolithic kernel and the security of a microkernel.

A framekernel lives in a single address space (like a monolithic kernel), but is **logically partitioned**:

* The **OS framework** is _privileged_: it contains a small, rigorously verified amount of `unsafe` Rust and encapsulates them into high-level, safe abstractions.
* The **OS services** are _de-privileged_: they are written entirely in safe Rust using those abstractions.

This **language-based, intra-kernel privilege separation** means that memory safety only depends on the correctness of the tiny privileged OS framework. Communication between different parts of the OS remains fast—via ordinary function calls and shared memory—preserving monolithic speed while enabling microkernel-like safety.

![Kernel Architecture Comparison](/assets/images/monolithic-kernel-vs-microkernel-vs-framekernel.png)
(TCB is short for Trusted Computing Base, which is the code responsible for kernel memory safety.)

## Your "Hello, World" Framekernel

Let's see a real example: handling system calls.

In Asterinas, the OS framework is called OSTD. We position it as Rust's (unofficial) standard library for OS development and has published it to [crates.io](https://crates.io/crates/ostd). OSTD has been used for developing Asterinas and may be used for more Rust OSes. The diagram below shows how system call loop may be written entirely in safe Rust using the API of OSTD.

![Handling System Calls with OSTD](/assets/images/handling-system-calls-with-ostd.png)

This example uses four safe abstractions from OSTD:

1. **`Task`** – Handles context switching and kernel stacks.
2. **`UserMode`** – Allows the CPU to execute in the user mode until an event occurs.
3. **`UserContext`** – Manipulates the user-mode CPU state such as general-purpose registers.
4. **`VmSpace`** – Manages user virtual memory safely.

All these abstractions wrap `unsafe` Rust code behind safe and sound APIs. The result: developing a Rust kernel can be done as safe and productive as doing Rust applications.

For a fully working example, check out this sample project: [Write a Hello World kernel in 100 lines of safe Rust](https://asterinas.github.io/book/ostd/a-100-line-kernel.html).

## Soundness Is Hard—But Possible

Rust's promise of memory safety means **no [undefined behaviors (UBs)](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)**. But UB in a kernel isn't just a language issue—it can stem from:

- Invalid memory mappings
- Stack corruption
- Bad page tables
- Faulty DMA devices
- And more...

To uphold Rust's guarantees, the TCB (i.e., OSTD) must defend against all of these. Even with hostile user code, misbehaving drivers, or malicious peripherals, OSTD must ensure the absence of UBs; this is called **soundness** in Rust.

We achieve soundness by:
- Defining strict safety invariants for `ostd`'s APIs
- Using MMU/IOMMU hardware enforcement
- Reviewing `unsafe` code rigorously
- Running Miri (a Rust UB detector) on kernel code, with kernel-specific adaptations

More details can be found in the paper. An on-going work is to formally verify critical unsafe code (e.g., page tables) inside OSTD with [Verus](https://github.com/verus-lang/verus). Our current progress has been reported in [a previous blog post](https://asterinas.github.io/2025/02/13/towards-practical-formal-verification-for-a-general-purpose-os-in-rust.html).

## A Vision Becomes Reality

After three years of development, we can now answer our original questions:

> How *small* can the TCB be?

Our TCB (`ostd`) is ~15K LoC—just 14% of the Asterinas kernel. The absolute size is comparable to verified microkernels like seL4 (~10K LoC) and the relative size is smaller than other Rust-based OSes such as Tock (43%), RedLeaf (66%), and Theseus (62%).

> Is this too restrictive for _rich_ OS features?

No. Currently, Asterinas supports:

- 210+ Linux system calls
- x86-64 & RISC-V CPU architectures
- File systems: Ext2, RamFS, SysFS, and OverlayFS
- Networking: TCP, UDP, Unix, and Netlink
- Devices: Virtio, NVMe, and USB

We hit rough edges in safe Rust, but never roadblocks.

> Would *performance* suffer?

Minimal. Asterinas is highly optimized, matching Linux's performance on the syscall-intensive LMbench microbenchmarks in the single-core setting. Multi-core scalability is actively being improved. Perhaps the biggest performance challenge we faced is efficient IOMMU management—a hurdle we overcame with a static mapping strategy.

> Are memory-safety _bugs_ really gone?

Almost. Asterinas contributors build features in **100% safe Rust**. Only OSTD developers touch `unsafe`, which is thoroughly reviewed and verified. Combined with **Miri** and **Verus**, the result is a kernel in which we have a great confidence.

## Wrap Up

The framekernel architecture proves that memory-safe OSes aren’t just possible—they're practical. **Asterinas is living proof.**

But this is just the beginning.

Asterinas is being built in the open, and we're looking for contributors of all kinds to help push it forward. Whether you're writing code, improving documentation, building drivers, or testing edge cases, your contributions matter.

Let’s make memory safety the new standard—together.