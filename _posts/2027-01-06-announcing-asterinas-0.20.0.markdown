---
layout: post
title:  "Announcing 0.20.0: Linux's drivers, Rust's core"
date:   2027-01-06 09:00:00 +0800
author: "Hongliang Tian"
---

<!--
Editor's note (delete before publishing):
- "Linuxlet" and the component name `aster-linuxlet` are proposed names for the Linux driver sandbox; the draft paper calls it SYSNAME/LFR. Rename as needed.
- The kernelet section is qualitative by design: the kernelet book has no end-to-end measurement yet.
- The driver-reuse numbers are the headline figures of the draft paper's abstract and introduction. The paper's evaluation section is not written yet; check the numbers against the final paper before publishing.
- The driver-reuse paper is an anonymous submission; the post does not name or link it. Link the paper from "a forthcoming paper" once a public version exists, and confirm with the authors what "natively" means for the 98 to 114% figures (native Linux or native Asterinas drivers) and whether the NVIDIA driver is the open kernel modules or the proprietary one.
- The kernelet section keeps two design facts from the kernelet book (a patch of about 200 lines, a 25-function table); it has no performance numbers.
- Name the Linux versions the kernelet patch supports (the prototype targets Linux 6.12) once that is settled.
- Replace the bracketed placeholders at the end.
-->

The [Asterinas](https://github.com/asterinas/asterinas) community is happy to announce Asterinas [0.20.0](https://github.com/asterinas/asterinas/releases/tag/v0.20.0)!

Asterinas 0.20.0 lets a Rust kernel and Linux share one machine, in kernel mode, in one address space, with either one as the host. Linux can host Rust kernelets that take over workloads one sandbox at a time. Asterinas can host Linux as a driver sandbox and inherit its drivers. No driver is rewritten, and nothing in Linux's core is rewritten in Rust. Rust arrives beside Linux's code, not inside it.

Rust for Linux brings Rust to drivers and modules, while the core of Linux stays in C. The core is unlikely to follow: it is one tightly coupled body of code, and every piece rewritten in Rust must still serve the C code that calls it. Asterinas took the other road, a whole kernel in Rust, and met the problem every new kernel meets: it lacks Linux's drivers and Linux's years of hardening.

Two features in 0.20.0 put the two kernels side by side:

- [Kernelets](#kernelets-rust-kernels-hosted-by-linux): a Rust kernel for each workload, hosted by Linux
- [Linuxlets](#linuxlets-linux-drivers-inside-asterinas): Linux's drivers inside Asterinas, in a sandbox

## Kernelets: Rust kernels hosted by Linux

A kernelet is a complete instance of the Asterinas kernel that serves the processes of one tenant, such as a container or an AI agent. It runs in kernel mode inside the host kernel, and all of its code above [OSTD](https://asterinas.github.io/book/ostd/), the small framework that holds Asterinas's `unsafe` code, is safe Rust. Each kernelet has a private copy of the kernel's writable state. Its build of OSTD leaves out every operation a tenant must never have, such as masking interrupts or touching the kernel page table. Code that reaches for one of those does not compile. Nothing in hardware separates kernelets from each other or from the host. The boundary rests on the Rust compiler and on OSTD being correct. In return, each workload gets a kernel of its own, which a container does not, and calling the host is an ordinary function call, without the VM exits a microVM pays for.

0.20.0 ships the first kernelets, with Asterinas as the host and with Linux as the host. An operator keeps the Linux kernel they already run, applies a small patch of about 200 lines that only adds code, and loads one module, the endovisor, which creates, runs and kills kernelets. The patch adds a gate on the system call path. Every system call from a tenant process goes to its kernelet before Linux sees it, so the tenant can reach only its own kernel and none of Linux's several hundred system calls. Everything else a kernelet needs (memory, processors, timers, virtual devices) comes through a table of 25 functions that the endovisor implements on what Linux already offers its modules, plus a few helpers the patch adds. The kernelet runtime implements the standard OCI interface for container runtimes, so containerd, Podman and Kubernetes can move a workload onto a kernelet by switching its runtime, and move it back.

<figure class="aster-fig" id="fig-kernelets">
<div class="head">
<div class="tag">Linux as the host</div>
<div class="title">Linux keeps running, and selected workloads get a kernel of their own</div>
</div>
<svg class="d" viewBox="0 0 860 346" role="img" aria-label="One machine running one Linux kernel. Above it, on the left, today's apps and containers continue unchanged on Linux's own subsystems, which carry one small gate on the system call path. On the right, agents A, B and C each get a kernelet, a safe-Rust kernel of their own, running inside the same Linux kernel and managed by the endovisor, one loadable module that serves them through a table of 25 functions. Both halves sit on the same hardware.">
<defs>
<linearGradient id="kernelets-dg" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" class="g0"/><stop offset="100%" class="g1"/></linearGradient>
</defs>
<text class="cap faint" x="24" y="16" font-size="12">TODAY'S WORKLOADS, UNCHANGED</text>
<text class="cap oh end" x="836" y="16" font-size="12">WORKLOADS THAT GET THEIR OWN KERNEL</text>
<rect x="24" y="26" width="114" height="32" rx="5"/><rect x="154" y="26" width="114" height="32" rx="5"/><rect x="284" y="26" width="114" height="32" rx="5"/>
<g class="lbl mid"><text x="81" y="47">app</text><text x="211" y="47">app</text><text x="341" y="47">container</text></g>
<g class="tint"><rect x="462" y="26" width="114" height="32" rx="5"/><rect x="592" y="26" width="114" height="32" rx="5"/><rect x="722" y="26" width="114" height="32" rx="5"/></g>
<g class="lbl mid"><text x="519" y="47">agent A</text><text x="649" y="47">agent B</text><text x="779" y="47">agent C</text></g>
<line class="sep dash" x1="16" y1="74" x2="844" y2="74"/>
<text class="ghost" x="16" y="69" font-size="12">user mode</text>
<text class="ghost" x="16" y="88" font-size="12">kernel mode</text>
<rect class="region" x="16" y="96" width="828" height="174" rx="10"/>
<rect x="36" y="114" width="362" height="110" rx="6"/>
<text class="lbl mid" x="217" y="164">Linux's own subsystems</text>
<text class="faint mid" x="217" y="186" font-size="12">patched: one small gate, ~200 lines</text>
<g class="o hi"><rect x="462" y="114" width="114" height="64" rx="6" style="fill:url(#kernelets-dg)"/><rect x="592" y="114" width="114" height="64" rx="6" style="fill:url(#kernelets-dg)"/><rect x="722" y="114" width="114" height="64" rx="6" style="fill:url(#kernelets-dg)"/></g>
<g class="ot mid"><text x="519" y="141">Kernelet A</text><text x="649" y="141">Kernelet B</text><text x="779" y="141">Kernelet C</text></g>
<g class="note mid" font-size="12"><text x="519" y="161">safe Rust</text><text x="649" y="161">safe Rust</text><text x="779" y="161">safe Rust</text></g>
<rect class="o" x="462" y="192" width="374" height="32" rx="5"/>
<text class="oh mid" x="649" y="213" font-size="12">endovisor: one module · a 25-function table</text>
<text class="cap mid" x="430" y="254" font-size="12">the Linux kernel the operator already runs, patches and trusts</text>
<rect class="base" x="16" y="282" width="828" height="30" rx="5"/>
<text class="faint mid" x="430" y="302" font-size="12">hardware</text>
<text class="faint mid" x="430" y="336" font-size="12">same machine · same kernel · no hypervisor · adopt one workload at a time</text>
</svg>
<svg class="m" viewBox="0 0 320 376" role="img" aria-label="One machine running one Linux kernel. On the left, today's apps and containers continue unchanged on Linux's own subsystems, which carry one small gate. On the right, agents A, B and C each get a kernelet, a safe-Rust kernel of their own, inside the same Linux kernel and managed by the endovisor, one loadable module. Both sides sit on the same hardware.">
<defs>
<linearGradient id="kernelets-mg" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" class="g0"/><stop offset="100%" class="g1"/></linearGradient>
</defs>
<text class="cap faint" x="10" y="14" font-size="12">UNCHANGED</text>
<text class="cap oh end" x="310" y="14" font-size="12">OWN KERNEL</text>
<rect x="10" y="22" width="140" height="28" rx="5"/>
<text class="lbl mid" x="80" y="41" font-size="12">apps, containers</text>
<rect class="tint" x="170" y="22" width="140" height="28" rx="5"/>
<text class="lbl mid" x="240" y="41" font-size="12">agents A · B · C</text>
<line class="sep dash" x1="4" y1="66" x2="316" y2="66"/>
<text class="ghost" x="6" y="62" font-size="12">user mode</text>
<text class="ghost" x="6" y="80" font-size="12">kernel mode</text>
<rect class="region" x="4" y="88" width="312" height="206" rx="10"/>
<rect x="14" y="104" width="136" height="136" rx="6"/>
<g class="lbl mid" font-size="12"><text x="82" y="152">Linux's own</text><text x="82" y="168">subsystems</text></g>
<g class="faint mid" font-size="12"><text x="82" y="194">patched: one</text><text x="82" y="210">small gate</text></g>
<g class="o hi"><rect x="170" y="104" width="136" height="26" rx="5" style="fill:url(#kernelets-mg)"/><rect x="170" y="136" width="136" height="26" rx="5" style="fill:url(#kernelets-mg)"/><rect x="170" y="168" width="136" height="26" rx="5" style="fill:url(#kernelets-mg)"/></g>
<g class="ot mid" font-size="12"><text x="238" y="121">Kernelet A</text><text x="238" y="153">Kernelet B</text><text x="238" y="185">Kernelet C</text></g>
<rect class="o" x="170" y="200" width="136" height="40" rx="5"/>
<g class="oh mid" font-size="12"><text x="238" y="217">endovisor:</text><text x="238" y="232">one module</text></g>
<g class="cap mid" font-size="12"><text x="160" y="264">the Linux kernel the operator</text><text x="160" y="280">already runs, patches, trusts</text></g>
<rect class="base" x="4" y="304" width="312" height="28" rx="5"/>
<text class="faint mid" x="160" y="323" font-size="12">hardware</text>
<g class="faint mid" font-size="12"><text x="160" y="352">same machine · same kernel</text><text x="160" y="368">no hypervisor · one at a time</text></g>
</svg>
<figcaption>Nothing in Linux is replaced. A kernelet is added beside it, and the only way out that its code can name is the endovisor's table of 25 functions.</figcaption>
</figure>

## Linuxlets: Linux drivers inside Asterinas

Linux's `drivers/` directory holds more than 17 million lines of C and years of workarounds for misbehaving hardware; the NVMe driver alone has more than 20 of them. No new kernel can afford to rewrite that. Re-implementing Linux's driver API underneath the drivers does not scale either. The kernel functions that four common Ethernet drivers use are enough for only 6 of 189 Ethernet drivers surveyed, and the API keeps changing. 0.20.0 takes the drivers as they are. A linuxlet, from the new `aster-linuxlet` component, is a Linux kernel built as a library with the [Linux Kernel Library (LKL)](https://github.com/lkl/linux) port and run in kernel mode inside a sandbox in Asterinas. Asterinas hands devices to the linuxlet, and Linux's own drivers take them over as they would on Linux. Asterinas then uses each device through the interface Linux gives its own programs, such as a device file's `ioctl` calls or Wi-Fi's netlink messages, by way of a small adapter per kind of device. Network packets and disk requests are too frequent for that route, so they get dedicated zero-copy paths straight to the driver, and Asterinas keeps its own network stack and file systems, serving unmodified Linux programs as before.

Asterinas treats all of this C code as untrusted, and the sandbox keeps it out of the code the Rust kernel has to trust. Asterinas still trusts its own framework, the Rust compiler, the sandbox's enforcement code, the CPU and the IOMMU (the hardware that limits which memory a device can reach). Before Linux's code is loaded, Asterinas checks it for any instruction that could switch the sandbox off. On x86-64, memory protection keys (MPK) confine what memory it can touch, and address masking together with Intel CET checks every indirect jump and return it makes. Interrupt paths are hardened so that an interrupt arriving during a check cannot skip it. Asterinas calls a driver directly, with no switch of privilege level and no VM exit. Each call has an execution budget: short operations finish inline, and longer ones are paused and moved to a kernel thread, so Asterinas regains control from a slow or stuck driver within a bounded time.

In 0.20.0, linuxlets drive more than 20 devices, including Wi-Fi cards and GPUs. With no driver-specific changes, network throughput in iperf and storage I/O rates in fio reach 98 to 114% of native performance. NVIDIA's official GPU driver needed fewer than 100 changed lines to run CUDA and LLM inference, at 104% of native token throughput in llama-bench. The design and the measurements will be described in a forthcoming paper.

<figure class="aster-fig" id="fig-linuxlets">
<div class="head">
<div class="tag">Asterinas as the host</div>
<div class="title">Linuxlets: Linux drivers run inside Asterinas, in a sandbox</div>
</div>
<svg class="d" viewBox="0 0 860 350" role="img" aria-label="One machine running Asterinas. Above the user-mode line, unmodified Linux apps make system calls to Asterinas. Below it, in one kernel-mode address space, the Asterinas kernel in safe Rust sits on the left with its own network stack and file systems. On the right, a linuxlet, Linux built as a library, holds Linux's own drivers for the NIC, storage, Wi-Fi and GPU and is marked as untrusted C. The linuxlet sits in an in-kernel sandbox of code validation, memory protection keys and CET. Two arrows cross into it: control through ioctl and netlink, and data for packets and blocks with zero-copy DMA. Both sit on hardware whose devices are assigned to the linuxlet.">
<defs>
<linearGradient id="linuxlets-dg" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" class="g0"/><stop offset="100%" class="g1"/></linearGradient>
<marker id="linuxlets-da" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path class="mk o" d="M0 0 L8 4 L0 8 z"/></marker>
</defs>
<rect x="24" y="26" width="114" height="32" rx="5"/><rect x="154" y="26" width="114" height="32" rx="5"/><rect x="284" y="26" width="114" height="32" rx="5"/>
<g class="lbl mid"><text x="81" y="47">app</text><text x="211" y="47">app</text><text x="341" y="47">container</text></g>
<text class="faint" x="420" y="47" font-size="12">unmodified Linux apps, served by Asterinas</text>
<line class="sep dash" x1="16" y1="74" x2="844" y2="74"/>
<text class="ghost" x="16" y="69" font-size="12">user mode</text>
<text class="ghost" x="16" y="88" font-size="12">kernel mode</text>
<rect class="region" x="16" y="96" width="828" height="180" rx="10"/>
<rect class="o hi" x="36" y="114" width="290" height="130" rx="6" style="fill:url(#linuxlets-dg)"/>
<text class="ot mid" x="181" y="150">Asterinas kernel</text>
<g class="note mid" font-size="12"><text x="181" y="172">safe Rust</text><text x="181" y="188">system calls · scheduling</text><text x="181" y="204">network stack · file systems</text><text x="181" y="226">device adapters</text></g>
<text class="rn mid" x="679" y="108" font-size="12">SANDBOX: VALIDATION + MPK + CET</text>
<path class="lo" d="M328 162 H532" marker-start="url(#linuxlets-da)" marker-end="url(#linuxlets-da)"/>
<text class="note mid" x="430" y="153" font-size="12">control: ioctl · netlink</text>
<path class="lo" d="M328 212 H532" marker-start="url(#linuxlets-da)" marker-end="url(#linuxlets-da)"/>
<g class="note mid" font-size="12"><text x="430" y="203">data: packets · blocks</text><text x="430" y="232">zero-copy DMA</text></g>
<rect class="risk" x="534" y="114" width="290" height="130" rx="6"/>
<text class="rt mid" x="679" y="138">Linuxlet: Linux as a library</text>
<text class="rn mid" x="679" y="156" font-size="12">untrusted C, contained</text>
<g class="inset"><rect x="542" y="172" width="62" height="24" rx="5"/><rect x="612" y="172" width="62" height="24" rx="5"/><rect x="682" y="172" width="62" height="24" rx="5"/><rect x="752" y="172" width="62" height="24" rx="5"/></g>
<g class="lbl mid" font-size="12"><text x="573" y="188">NIC</text><text x="643" y="188">storage</text><text x="713" y="188">Wi-Fi</text><text x="783" y="188">GPU</text></g>
<text class="rn mid" x="679" y="222" font-size="12">Linux's own drivers, 20+ devices</text>
<text class="cap mid" x="430" y="262" font-size="12">one address space · no privilege switch · no VM exit</text>
<rect class="base" x="16" y="288" width="828" height="30" rx="5"/>
<text class="faint mid" x="430" y="308" font-size="12">hardware · devices assigned to the linuxlet</text>
<text class="faint mid" x="430" y="342" font-size="12">every call has an execution budget · Asterinas regains control within a bounded time</text>
</svg>
<svg class="m" viewBox="0 0 320 456" role="img" aria-label="One machine running Asterinas. Above the user-mode line, unmodified Linux apps make system calls to Asterinas. Below it, in one kernel-mode address space, the Asterinas kernel in safe Rust sits at the top with its own network stack and file systems. A two-way arrow carries control through ioctl and netlink and data for packets and blocks with zero-copy DMA across a dashed red sandbox boundary of code validation, memory protection keys and CET. Below the boundary a linuxlet, Linux built as a library, holds Linux's own drivers for the NIC, storage, Wi-Fi and GPU and is marked as untrusted C. Hardware at the bottom has its devices assigned to the linuxlet.">
<defs>
<linearGradient id="linuxlets-mg" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" class="g0"/><stop offset="100%" class="g1"/></linearGradient>
<marker id="linuxlets-ma" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path class="mk o" d="M0 0 L8 4 L0 8 z"/></marker>
</defs>
<rect x="10" y="22" width="300" height="28" rx="5"/>
<text class="lbl mid" x="160" y="41" font-size="12">unmodified Linux apps</text>
<line class="sep dash" x1="4" y1="66" x2="316" y2="66"/>
<text class="ghost" x="6" y="62" font-size="12">user mode</text>
<text class="ghost" x="6" y="80" font-size="12">kernel mode</text>
<rect class="region" x="4" y="88" width="312" height="296" rx="10"/>
<rect class="o hi" x="14" y="102" width="292" height="70" rx="6" style="fill:url(#linuxlets-mg)"/>
<text class="ot mid" x="160" y="124" font-size="12">Asterinas kernel · safe Rust</text>
<g class="note mid" font-size="12"><text x="160" y="142">system calls · scheduling</text><text x="160" y="158">network stack · file systems</text></g>
<path class="lo" d="M60 174 V234" marker-start="url(#linuxlets-ma)" marker-end="url(#linuxlets-ma)"/>
<g class="note" font-size="12"><text x="76" y="190">control: ioctl · netlink</text><text x="76" y="206">data: packets · blocks,</text><text x="76" y="222">zero-copy DMA</text></g>
<line class="lr dash" x1="14" y1="240" x2="306" y2="240"/>
<text class="rn mid" x="160" y="258" font-size="12">sandbox: validation + MPK + CET</text>
<rect class="risk" x="14" y="266" width="292" height="92" rx="6"/>
<text class="rt mid" x="160" y="286" font-size="12">Linuxlet: Linux as a library</text>
<text class="rn mid" x="160" y="302" font-size="12">untrusted C, contained</text>
<g class="inset"><rect x="24" y="312" width="62" height="24" rx="5"/><rect x="94" y="312" width="62" height="24" rx="5"/><rect x="164" y="312" width="62" height="24" rx="5"/><rect x="234" y="312" width="62" height="24" rx="5"/></g>
<g class="lbl mid" font-size="12"><text x="55" y="328">NIC</text><text x="125" y="328">storage</text><text x="195" y="328">Wi-Fi</text><text x="265" y="328">GPU</text></g>
<text class="rn mid" x="160" y="352" font-size="12">Linux's own drivers, 20+ devices</text>
<text class="cap mid" x="160" y="376" font-size="12">one address space · no VM exit</text>
<rect class="base" x="4" y="394" width="312" height="28" rx="5"/>
<text class="faint mid" x="160" y="413" font-size="12">hardware · devices assigned</text>
<g class="faint mid" font-size="12"><text x="160" y="438">every call has a budget;</text><text x="160" y="453">Asterinas regains control in bounded time</text></g>
</svg>
<figcaption>The drivers stay C and stay in Linux. The sandbox keeps them out of the code the Rust kernel has to trust, and the zero-copy data paths keep them fast.</figcaption>
</figure>

## Where this leads

On a machine that runs Linux, kernelets let Rust take over the workloads that want a kernel of their own while Linux runs the rest. On a machine that runs Asterinas, a linuxlet keeps Linux only for its drivers. Either way, more of the machine runs on a memory-safe kernel, and nobody has to rewrite Linux to get there.

## Also in this release

[Placeholder: two or three bullets on other notable changes in 0.20.0, in the format of the 0.19.0 post, with PR links. Everything else is in the [full changelog](https://github.com/asterinas/asterinas/blob/main/RELEASES.md).]

## Contributors

[Placeholder: "N people contributed to this release, M for the first time", counted from git with aliases merged, followed by the contributor list in the format of previous posts.]
