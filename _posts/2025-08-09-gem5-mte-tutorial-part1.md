---
layout: post
title: "Extending gem5 for Arm MTE – Part 1"
description: Advertising Arm MTE Support in gem5
date: 2025-08-10 21:00:00
categories: [gem5 tutorial]
tags: [gem5, arm, arm_isa, arm_mte, memory_tagging, architectural_simulation]
author: saber
# image: /assets/img/figures/gem5_mte_tutorial_part1_mte_register.svg
---

This is the first post in a series on extending gem5 to support Arm’s Memory Tagging Extension (MTE), and we are not just going to do the “copy these steps and it works” routine. We will also look at the reasoning behind each step. Sometimes that means diving deep into gem5’s internals, like its ISA description language, decode logic, and register model. Other times we will zoom out to think like a CPU architect, exploring how you would approach modeling a new feature from scratch.

By the end of the series, MTE will not just be running in your gem5 build. You will also know how to approach *any* architectural extension, understand where it fits in the simulator, and feel confident navigating gem5’s framework without getting lost.

This short post focuses only on: (1) advertising Arm MTE in gem5 by adding `FEAT_MTE` and `FEAT_MTE2`, and (2) running a quick Full-System experiment to see how the kernel reacts, and to uncover what’s still missing.

## FEAT_* Codes in Arm

Arm evolves its ISA in modular chunks called **features**. Each feature has a short name, or mnemonic, such as `FEAT_SVE` for the Scalable Vector Extension, `FEAT_AES` for AES encryption instructions, or `FEAT_MTE` for the Memory Tagging Extension. Instead of making software guess which instructions a CPU understands (and risk a crash if it guesses wrong), the architecture provides **feature registers** that software can read to discover what is supported. In AArch64, one of the main places a CPU advertises these features is in the **Processor Feature Registers**.

We’re interested in `FEAT_MTE` and `FEAT_MTE2` because they're the architectural way of saying: “**this CPU can do memory tagging.**” In this post, we’ll wire up gem5 so it **advertises** MTE support.

## FEAT_MTE vs. FEAT_MTE2

The “vanilla” `FEAT_MTE` (introduced in Armv8.5-A, `MTE = 0b0001`) is what Arm calls *instruction-only* MTE. It gives you the tag instructions (`IRG`, `ADDG`, `STG`, ...), but doesn’t guarantee in-memory allocation tags or the full synchronous/asynchronous tag checking that Linux expects (we'll get into the details of these things in future posts). Modern Linux kernels only advertise MTE in `/proc/cpuinfo` and set the `HWCAP2_MTE` flag if the `MTE` field is at least `0b0010`, which means `FEAT_MTE2` or newer. `FEAT_MTE2` (Armv8.7-A) is basically MTE with the training wheels off: allocation tags are *mandatory*, *synchronous checking* is always there, and you get extra knobs for fault reporting and handling. So if our goal is to boot Linux in gem5 FS mode and see `mte` show up in `/proc/cpuinfo`, we’ll need to model (or at least advertise) `FEAT_MTE2`. That’s the point where the kernel says “**yep, MTE is really here**” and tells user space about it.

## ID_AA64PFR1_EL1 Register

`ID_AA64PFR1_EL1` is AArch64 Processor Feature Register 1. It is part of the **Processor Feature Register (PFR)** family, a set of read-only system registers that report the presence (and sometimes the version) of high-level architectural features to software. These registers are 64 bits wide and are divided into fixed-size bitfields, each assigned to a specific feature.

The PFR family currently includes `ID_AA64PFR0_EL1` and `ID_AA64PFR1_EL1`. This “numbered register” approach is common in AArch64 and also exists for other categories, such as the Instruction Set Attribute Registers (`ID_AA64ISARn_EL1`), Memory Model Feature Registers (`ID_AA64MMFRn_EL1`), and Debug Feature Registers (`ID_AA64DFRn_EL1`). Each category focuses on a different area of the architecture.

For MTE specifically, the **`mte` field** in `ID_AA64PFR1_EL1` occupies bits `[11:8]`. If this field is non-zero, software can assume that the core implements the basic MTE instruction set and state.

## Tell gem5 that FEAT_MTE exists

We start by modifying `src/arch/arm/ArmSystem.py`. This file lives in gem5’s **Python config layer** for Arm, which is the middle-layer between the low-level C++/ISA code and the higher-level simulation scripts. While most of the simulator’s CPU and ISA behavior lives in C++ and the ISA Domain Specific Language (DSL), `ArmSystem.py` is where we describe, in Python, what an Arm-based system looks like to the rest of gem5. It’s where you’ll find things like:

- The `ArmSystem` SimObject and its parameters (CPU type, memory ranges, etc.).
- The list of CPU “extensions” that control which architectural features the simulated CPU claims to support.
- Setup hooks for the memory map, GIC, and other platform components.

The important bit for us: this is where gem5 decides **what features your CPU will *say* it supports**. Adding `FEAT_MTE` and `FEAT_MTE2` to the default extension list here is the first step in making your simulated CPU advertise MTE. Without it, the guest software will never “see” MTE, even if you later add all the MTE instructions and registers.

Add `FEAT_MTE` and `FEAT_MTE2` to the `ArmExtension` enum. This enum is the list of default architectural features recognized by gem5’s Arm ISA model. Since MTE was introduced in *Armv8.5* and is present in later revisions, we’ll add it under the group of features for *Armv8.5* as well.

```python
class ArmExtension(ScopedEnum):
    vals = [
	...
        # Armv8.5
        "FEAT_FLAGM2",
        "FEAT_RNG",
        "FEAT_RNG_TRAP",
        "FEAT_EVT",
        "FEAT_MTE",  # <--
        "FEAT_MTE2", # <-- (is OPTIONAL from Armv8.5)
        ...
    ]

class Armv85(Armv84):
    extensions = Armv84.extensions + [
        "FEAT_FLAGM2",
        "FEAT_RNG",
        "FEAT_RNG_TRAP",
        "FEAT_EVT",
        "FEAT_MTE",  # <--
        "FEAT_MTE2"  # <--
    ]
```
{: file='src/arch/arm/ArmSystem.py'}

In gem5, the default feature set isn’t shared between simulation modes. **Full-System (FS)** mode uses `ArmDefaultRelease`, while **Syscall Emulation (SE)** mode uses `ArmDefaultSERelease`. In FS mode, the kernel runs at real exception levels, pokes at system registers like `ID_AA64PFR1_EL1`, and decides what features it can use. If `FEAT_MTE` isn’t in `ArmDefaultRelease`, the kernel simply won’t see it. SE mode is lighter weight, no kernel, just your program with gem5 faking the syscalls. But SE still has its own feature list (`ArmDefaultSERelease`), which it uses to fill in those same ID registers when user-space asks about them. If MTE’s not on that list, your SE test code will think it’s missing.

WE'll put `FEAT_MTE` and `FEAT_MTE2` in both places. That way your code sees MTE whether you’re running a tiny SE-mode test or a full OS in FS mode.

```python
class ArmDefaultRelease(Armv8):
    extensions = Armv8.extensions + [
        ...
        # Armv8.5
        "FEAT_FLAGM2",
        "FEAT_EVT",
        "FEAT_MTE",   # <--
        "FEAT_MTE2",  # <--
	...
    ]
```
{: file='src/arch/arm/ArmSystem.py'}

`ArmDefaultSERelease` lives in `ArmISA.py`, which sits closer to the CPU model. It defines ISA-specific SimObjects, parameters, and default extension sets for when you’re not simulating the full platform, exactly what SE mode does. No GIC, no firmware, just CPU, memory, and syscall emulation. Keeping it here lets the ISA model grab it without dragging in all the full-system baggage.

```python
class ArmDefaultSERelease(ArmRelease):
    extensions = [
	...
        # Armv8.5
        "FEAT_FLAGM2",
        "FEAT_MTE",   # <--
        "FEAT_MTE2",  # <--
	...
    ]
```
{: file='src/arch/arm/ArmISA.py'}

### Set the mte field in ID_AA64PFR1_EL1

According to the ARMv8.5 ISA spec, the `mte` field in `ID_AA64PFR1_EL1` register (see [this](https://developer.arm.com/documentation/ddi0601/2025-06/AArch64-Registers/ID-AA64PFR1-EL1--AArch64-Processor-Feature-Register-1)) tells you whether MTE is present on a system (and it's version). In gem5, this register is implemented as one of the “misc” registers, which you’ll find in `regs/misc.hh`.

```c++
namespace ArmISA
{
    enum MiscRegIndex {
        ...
        // AArch64 registers (Op0=1,3)
        ...
        MISCREG_ID_AA64PFR0_EL1,
        MISCREG_ID_AA64PFR1_EL1,
        ...
    }
    const char * const miscRegName[] = {
        ...
        // AArch64 registers (Op0=1,3)
        ...
        "id_aa64pfr0_el1",
        "id_aa64pfr1_el1",
        ...
    }
}
```
{: file='src/arch/arm/regs/misc.hh'}

The `ID_AA64PFR1_EL1` system register and its fields are already listed in `regs/misc_types.hh`, but the field definitions aren’t fully filled in yet.

```c++
BitUnion64(AA64PFR1)
    Bitfield<27, 24> sme;
    Bitfield<19, 16> mpamFrac;
EndBitUnion(AA64PFR1)
```
{: file='src/arch/arm/regs/misc_types.hh'}

Using the field descriptions from the Arm A-profile architecture documentation (see [this](https://developer.arm.com/documentation/ddi0601/2025-06/AArch64-Registers/ID-AA64PFR1-EL1--AArch64-Processor-Feature-Register-1)) shown in the figure below, we can see that the register type (`BitUnion64`) and its fields should be defined in `regs/misc_types.hh` like this:

![light mode only](/assets/img/figures/gem5_mte_tutorial_part1_mte_register.svg){: .light}
![dark mode only](/assets/img/figures/gem5_mte_tutorial_part1_mte_register_dark.svg){: .dark}
_Bit field layout of the ID_AA64PFR1_EL1 system register_

We’ll add the missing bit fields to the register definition based on Arm spec.

```c++
BitUnion64(AA64PFR1)
    Bitfield<63, 60> pear;
    Bitfield<59, 56> df2;
    Bitfield<55, 52> mtex;
    Bitfield<51, 48> the;
    Bitfield<47, 44> gcs;
    Bitfield<43, 40> mteFrac;
    Bitfield<39, 36> nmi;
    Bitfield<35, 32> csv2Frac;
    Bitfield<31, 28> rndrTrap;
    Bitfield<27, 24> sme;
    Bitfield<23, 20> res0;
    Bitfield<19, 16> mpamFrac;
    Bitfield<15, 12> rasFrac;
    Bitfield<11, 8> mte;  // <- this field is what we care about for now 
    Bitfield<7, 4> ssbs;
    Bitfield<3, 0> bt;
EndBitUnion(AA64PFR1)
```
{: file='src/arch/arm/regs/misc_types.hh'}

Now that we’ve defined the fields for `ID_AA64PFR1_EL1`, the next stop is `misc.cc` to actually initialize them. This is where we set the mte field so our simulated CPU advertises `FEAT_MTE`.

We could also set other related fields like `mtex` and `mteFrac` to enable newer MTE versions (`FEAT_MTE3`, `FEAT_MTE4`) or async tagging (`FEAT_MTE_ASYNC`), but we’ll keep that for later so we don’t bite off too much at once.

In the snippet below, we initialize `ID_AA64PFR1_EL1` based on the CPU’s extension list. If the list includes `FEAT_MTE` or `FEAT_MTE2`, we set the `mte` field to`0x1` or `0x2`, respectively; otherwise, it remains `0x0`. The same pattern is used for other features like `FEAT_SME` and `FEAT_MPAM`. This way, the register’s reported features always match what we’ve configured in the CPU’s extension set.

```c++
InitReg(MISCREG_ID_AA64PFR1_EL1)
  .reset([release=release](){
      AA64PFR1 pfr1_el1 = 0;
      uint8_t mte_level = 0;
      if (release->has(ArmExtension::FEAT_MTE2)) {
          mte_level = 0x2;
      } else if (release->has(ArmExtension::FEAT_MTE)) {
          mte_level = 0x1;
      }
      pfr1_el1.mte = mte_level;
      pfr1_el1.sme = release->has(ArmExtension::FEAT_SME) ? 0x1 : 0x0;
      pfr1_el1.mpamFrac = release->has(ArmExtension::FEAT_MPAM) ?
          0x1 : 0x0;
      return pfr1_el1;
  }())
  .unserialize(0)
  .faultRead(EL0, faultIdst)
  .faultRead(EL1, faultHcrEL1<&HCR::tid3>)
  .allPrivileges().writes(0);
```
{: file='src/arch/arm/regs/misc.cc'}

The figure below shows the path `FEAT_MTE` takes through gem5, from being enabled in your CPU config, to showing up in `ID_AA64PFR1_EL1`, and finally being visible to software.

![light mode only](/assets/img/figures/gem5_mte_tutorial_part1_mte_flow.svg){: .light}
![dark mode only](/assets/img/figures/gem5_mte_tutorial_part1_mte_flow_dark.svg){: .dark}
_Flow of FEAT_MTE through gem5’s CPU model to software_

## Time to See It in Action (and Why It Won’t Work… Yet)

Alright, so we’ve told gem5 “Hey, this CPU has MTE!”. Naturally, the next thing you might want to do is boot Linux and check `/proc/cpuinfo` for that sweet `mte` flag.
Here’s the catch: the moment the Linux kernel sees MTE in `ID_AA64PFR1_EL1`, it wastes no time putting it to work. Very early in boot, before you even get any outputs on the terminal, it tries to:

- Seed the random tag generator (`RGSR_EL1`)
- Configure the tag control policy (`GCR_EL1`)
- Ask the hardware about supported block sizes (`GMID_EL1`)

In our current gem5 build, none of these registers exist yet. The CPU model just shrugs when Linux pokes them, and you get an *unimplemented register* trap. Since this happens long before the console is ready, the boot just… stops. No error message, no panic, no dmesg; just a silent hang.

## Testing in FS Mode

Let’s confirm this issue when running the Linux kernel on gem5 FS mode. Even though we know this will fail, running it in FS mode is a good sanity check, and it’s the fastest way to confirm which registers Linux expects.

### Build gem5

From the gem5 root directory, build the ARM target:

```shell
scons -j$(nproc) build/ARM/gem5.opt
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> Building gem5 can take quite a while — patience required :'(
{: .prompt-info }
<!-- markdownlint-restore -->

### Get the Kernel and Disk Image

We’ll use a ready-to-go Linux v5.10 kernel ([link](https://resources.gem5.org/resources/arm64-linux-kernel-5.10.110/versions?database=gem5-resources&version=1.0.0)) and Ubuntu 18.04 disk image ([link](https://resources.gem5.org/resources/arm64-ubuntu-18.04-img/versions?database=gem5-resources&version=1.0.0)) from gem5’s public resources, but any kernel ≥ 5.10 should work.

```shell
mkdir -p fs_files
cd fs_files

# kernel
wget https://gem5dist.blob.core.windows.net/dist/develop/kernels/arm/static/arm64-vmlinux-5.10.110

# disk image
wget https://gem5dist.blob.core.windows.net/dist/develop/images/arm/ubuntu-18-04/arm64-ubuntu-20220425.img.gz
gunzip arm64-ubuntu-20220425.img.gz
```

### Run the FS simulation

Run gem5 with the FS config script and our resources. We'll run the simulation with debug flags enabled the to see the potential issues that may raise when accessing unimplemented registers.

```shell
build/ARM/gem5.opt \
    --debug-flags=Arm,MiscRegs,ExecAll \
    --debug-start=0 \
    --debug-end=100000000 \
    --debug-file=boot_debug.log \
    configs/example/arm/starter_fs.py \
    --cpu atomic \
    --num-cores 1 \
    --mem-size 1GB \
    --kernel fs_files/arm64-vmlinux-5.10.110 \
    --disk-image fs_files/arm64-ubuntu-20220425.img
```

As soon as gem5 starts the simulation, it prints something like:

```txt
txtsystem.terminal: Listening for connections on port 3456
```

You can connect to this port via telnet to watch the kernel boot in real time and then see the boot up process and later the actual terminal of the simulated kernel:

```shell
telnet localhost 3456
```

### Spot the Problem

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> The boot will appear frozen. That’s your cue to kill the simulation and go hunting in the debug log.
{: .prompt-warning }
<!-- markdownlint-restore -->

If you look at the `m5out/boot_debug.log` file and search for the *unimplemented* keyword, you can find one unimplemented register that `MSR` instrcution tries to read from.

```shell
grep -i "unimplemented" m5out/boot_debug.log
```

You’ll see something like:

```txt
1002250: system.cpu_cluster.cpus: A0 T0 : 0x80793668 @kernel_init.__cpu_setup+112    : msr op0:3 op1:0 crn:1 crm:0 op2:6 (unimplemented) : No_OpClass :   flags=(IsNonSpeculative|IsInvalid)
```

If you look at this opcode (`op0:3 op1:0 crn:1 crm:0 op2:6`) in the ARM documentation (see [this](https://developer.arm.com/documentation/ddi0601/2025-06/AArch64-Registers/GCR-EL1--Tag-Control-Register-)), you can see it encodes to `GCR_EL1` (Tag Control Register). The kernel reads from `GCR_EL1` in the very early `__cpu_setup` assembly code, immediately after detecting the feature. Missing it means the kernel can’t even finish CPU setup.

## Where We Go from Here

Part 2 will be all about giving Linux just enough MTE to survive boot (or at least get past the first few hurdles). That means implementing the key system registers (`GCR_EL1`, `RGSR_EL1`, `GMID_EL1`, ...) so the kernel’s setup code stops tripping over missing hardware. Once those are in place, the boot can finally move forward. Stay tuned!