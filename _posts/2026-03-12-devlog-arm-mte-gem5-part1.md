---
layout: post
title: "Arm MTE in gem5 | Part 1 : Advertizing Arm MTE"
description: Enabling Arm Memory Tagging Extension in gem5 by advertising CPU support and exposing it through feature registers.
date: 2026-03-12 22:00:00
categories: [dev_log]
tags: [gem5, arm, arm_isa, arm_mte, mte, feat_mte, memory_tagging, architectural_simulation]
author: saber
---

For much of my PhD, I’ve lived deep in the weeds of gem5. While I’ve worked with other simulators, gem5 has become my primary tool for exploring architectural ideas. This journey documents the development challenges and lessons that grew from a specific research need: bringing Arm Memory Tagging Extension (MTE) support to gem5.

It all started with a *deceptively* simple question: ***What would it take to get Arm MTE running in a full-system simulation?***

I didn't want a hack that just appeared to work. I wanted to understand how gem5’s internals, from the ISA description language and decode logic to the CPU pipeline and memory subsystem, actually interact to support a modern hardware feature. Whether you are a fellow researcher or a simulator enthusiast, I hope this journey helps you navigate the complex maze of architectural extensions in gem5.

## Arm ISA Feature Model and FEAT_MTE

One of the first things I had to wrap my head around was how Arm evolved its ISA. It’s not a single monolithic specification; it’s a collection of modular features like `FEAT_SVE` or `FEAT_AES`. Here, we’re interested in `FEAT_MTE` and `FEAT_MTE2` because they're the architectural way of saying: “**this CPU can do memory tagging.**”

`FEAT_MTE` (introduced in Armv8.5-A) is what Arm calls instruction-only MTE. It gives you the tag instructions (`IRG`, `ADDG`, `STG`, ...), but doesn’t guarantee in-memory allocation tags or the full synchronous/asynchronous tag checking that Linux expects. `FEAT_MTE2` (Armv8.7-A) is basically MTE with the training wheels off: allocation tags are *mandatory*, *synchronous checking* is always there, and you get extra knobs for fault reporting and handling. 

## How Arm Advertises Features

Instead of making software guess which instructions a CPU understands (and risk a crash if it guesses wrong), the architecture provides **feature registers** that software can read to discover what is supported. In AArch64, one of the main places a CPU advertises these features is in the **Processor Feature Registers (PFR)**, a set of read-only system registers that report the presence (and sometimes the version) of high-level architectural features to software. These registers are 64 bits wide and are divided into fixed-size bitfields, each assigned to a specific feature.

`ID_AA64PFR1_EL1`, the AArch64 Processor Feature Register 1, is the one we care about because bits `[11:8]` define the `mte` field. If this field is non-zero, software can assume that the CPU implements the basic MTE instruction set and state. Modern Linux kernels advertise MTE in `/proc/cpuinfo` and set the `HWCAP2_MTE` flag if the `mte` field is at least `0b0010`, which means `FEAT_MTE2` or newer. So if our goal is to boot Linux in gem5 FS mode and see `mte` show up in `/proc/cpuinfo`, we’ll need to model (or at least advertise) `FEAT_MTE2`. That’s the point where the kernel says “**yep, MTE is really here**” and tells user space about it.

## Tell gem5 that FEAT_MTE exists

First up, we edit `src/arch/arm/ArmSystem.py`, gem5’s Python config layer that defines what an Arm system looks like to the simulator. This is where CPU parameters, memory setup, and the list of “extensions” are defined, showing what features the CPU claims to support. To get the guest to see MTE, we need to add `FEAT_MTE` and `FEAT_MTE2` here; Without it, the guest software will never “see” MTE, even if you later add all the MTE instructions and registers.

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

In gem5, FS and SE modes have separate default feature sets. **Full-System (FS)** mode uses `ArmDefaultRelease`, which lets the kernel run at real exception levels and query system registers like `ID_AA64PFR1_EL1`. **Syscall Emulation (SE)** mode is lighter, with no kernel, just your program and syscall stubs, and it uses `ArmDefaultSERelease`, which lives in `ArmISA.py`. If MTE is missing from the respective list, the kernel or SE program will not see it. We will add `FEAT_MTE` and `FEAT_MTE2` to both lists so your code can see MTE whether it is a small SE test or a full OS running in FS mode.

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

Looking at the field descriptions from the Arm A-profile architecture docs (see [this](https://developer.arm.com/documentation/ddi0601/2025-06/AArch64-Registers/ID-AA64PFR1-EL1--AArch64-Processor-Feature-Register-1)) and the figure below, we can see that the register type (`BitUnion64`) and its fields should be defined like this:

![ID_AA64_PFR1_EL1 Register Encoding](/assets/img/figures/ID_AA64_PFR1_EL1.svg){: .dark-invert}
<em>Bit field layout of the ID_AA64PFR1_EL1 system register</em>

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

With `ID_AA64PFR1_EL1` defined, we move to `misc.cc` to initialize it. The mte field is set based on the CPU’s extension list: `0x1` for `FEAT_MTE`, `0x2` for `FEAT_MTE2`, and `0x0` if neither is present. The same pattern applies to features like `FEAT_SME` and `FEAT_MPAM`, keeping the register’s reported capabilities in sync with the CPU configuration. We could also handle newer MTE versions or async tagging, but let's keep it for later.

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
  .serializing(false)
  .faultRead(EL0, faultIdst)
  .faultRead(EL1, faultHcrEL1<&HCR::tid3>)
  .allPrivileges().writes(0);
```
{: file='src/arch/arm/regs/misc.cc'}

## Time to See It in Action (and What Went Wrong)

Alright, now that we’ve told gem5 “Hey, this CPU has MTE,” I wanted to check it out in Linux. I booted and arm64 kernel in FS mode (any kernel ≥ 5.10 should work, I used [this](https://resources.gem5.org/resources/arm64-linux-kernel-5.10.110/versions?database=gem5-resources&version=1.0.0)), expecting to spot `mte` in `/proc/cpuinfo`. 

Almost immediately, the boot froze. No messages, no panic, nothing. I checked the boot log and searched for potential issues and found this (debuged using `--debug-flags=Arm,MiscRegs,ExecAll,Faults` gem5 flags):

```
1001000: system.cpu_cluster.cpus: A0 T0 : 0x8079364c @kernel_init.__cpu_setup+84    :   mrs   x10, id_aa64pfr1_el1 : System :  D=0x0000000001000200  flags=(IsInteger)
1001250: system.cpu_cluster.cpus: A0 T0 : 0x80793650 @kernel_init.__cpu_setup+88    :   ubfm   x10, x10, #8, #11 : IntAlu :  D=0x0000000000000002  flags=(IsInteger)
1001500: global: (in, iz) = (0, 1)
1001500: global: (iv) = (0)
1001500: global: (ic) = (1)
1001500: system.cpu_cluster.cpus: A0 T0 : 0x80793654 @kernel_init.__cpu_setup+92    :   subs   x10, #2           : IntAlu :  D=0x0000000000000001  flags=(IsInteger)
1001750: system.cpu_cluster.cpus: A0 T0 : 0x80793658 @kernel_init.__cpu_setup+96    :   b.lt   <kernel_init.__cpu_setup+144> : IntAlu :   flags=(IsControl|IsDirectControl|IsCondControl)
1002000: system.cpu_cluster.cpus: A0 T0 : 0x8079365c @kernel_init.__cpu_setup+100    :   movz   x10, #240, #0     : IntAlu :  D=0x00000000000000f0  flags=(IsInteger)
1002250: system.cpu_cluster.cpus: A0 T0 : 0x80793660 @kernel_init.__cpu_setup+104    :   bfm   x5, x10, #56, #7   : IntAlu :  D=0x000c0400bb44f0ff  flags=(IsInteger)
1002500: system.cpu_cluster.cpus: A0 T0 : 0x80793664 @kernel_init.__cpu_setup+108    :   orr   x10, xzr, #131071  : IntAlu :  D=0x000000000001ffff  flags=(IsInteger)
1002750: system.cpu_cluster.cpus.isa: Reading MiscReg sctlr_el1 with clear res1 bits: 0x8800000
1002750: system.cpu_cluster.cpus.isa: Writing MiscReg spsr_el1 (559 4) : 0x600001c5
1002750: system.cpu_cluster.cpus.isa: Writing MiscReg elr_el1 (561 561) : 0x80793668
1002750: system.cpu_cluster.cpus.isa: Updating CPSR from 0x1c5 to 0x4003c5 f:1 i:1 a:1 d:1 mode:0x5
1002750: system.cpu_cluster.cpus.isa: Writing MiscReg cpsr (0 0) : 0x4003c5
1002750: system.cpu_cluster.cpus.isa: Writing MiscReg esr_el1 (586 586) : 0x2000000
1002750: Undefined Instruction: Invoking Fault (AArch64 target EL):Undefined Instruction cpsr:0x4003c5 PC:0x80793668 elr:0x80793668 esr_el1: 0x2000000 inst: 0xd51810ca
```

First of all, the kernel is clearly reading `ID_AA64PFR1_EL1` and extracting the MTE field (`ubfm   x10, x10, #8, #11`).

This isolates bits `<11:8>`, which contain the MTE support level.

```
ID_AA64PFR1_EL1 = 0x0000000001000200
bits[11:8] = 0b0010 → FEAT_MTE2
```

So, at this point we can say the CPU is correctly advertizing MTE feature using the `ID_AA64PFR1_EL1` feature register. Hooray!!

But, it was still not clear to me what is causing the boot process to fail. So, I looked for the faulting instruction and I realized that this is a system register access instruction and based on the fault location in the source code and the instruction encoding (`inst: 0xd51810ca`), we can say it is `msr tcr_el1, x5` instruction. So the kernel wasn’t crashing randomly, it was trying to write to `TCR_EL1`. 

Looking back at the surrounding code in `__cpu_setup`, this instruction tries to write the MTE-related fields of the **Translation Control Register** (`TCR_EL1`), enabling **Top Byte Ignore (TBI)** and the tag-aware address translation behavior required for tagged pointers (see [this](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/TCR-EL1--Translation-Control-Register--EL1-)). In other words, Linux had reached the point where it assumes tagging is usable and begins configuring the MMU accordingly. 

**Here’s the catch:** the moment the Linux kernel sees MTE in `ID_AA64PFR1_EL1`, it wastes no time putting it to work. Very early in boot (in `__cpu_setup` assembly code), before you even get any outputs on the terminal, it tries to: 

- Enable Top-Byte Ignore for address translation and configuring MMU (`TCR_EL1`)
- Seed the random tag generator (`RGSR_EL1`) 
- Configure the tag control policy (`GCR_EL1`) 
- Ask the hardware about supported block sizes (`GMID_EL1`) 

In our current gem5 build, either these registers do not exist at all or their bitfields are not complete yet to support MTE and finish the its setup. The CPU model just shrugs when Linux pokes them, and you get an *unimplemented register* trap or a fault like what we saw. Since this happens long before the console is ready, the boot stops. No error message, no panic, no dmesg; just a silent hang. This shows that just marking MTE in the CPU isn’t enough. Linux expects certain system registers to exist before it can use memory tagging.

In the next post, we’ll add these registers so Linux can boot cleanly and actually see MTE in `/proc/cpuinfo`.
