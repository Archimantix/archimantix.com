---
layout: post
title: "Arm MTE in gem5 | Part 2 : Understanding and Adding the System Registers"
description: Exploring the Linux kernel to identify MTE system registers and implement them in gem5 for early boot support.
date: 2026-03-15 22:00:00
categories: [dev_log]
tags: [gem5, arm, arm_isa, arm_mte, mte, feat_mte, memory_tagging, architectural_simulation]
author: saber
---

In the last post, we discovered that simply advertising MTE through the feature registers isn’t enough to make it usable. The moment Linux sees `FEAT_MTE`/`FEAT_MTE2`, it assumes the hardware is ready and immediately starts enabling memory tagging. That early boot code reaches for a handful of system registers to enable tag-aware address translation, seed the tag generator, define the tag control policy, and query MTE implementation details. In our current gem5 model, those registers don’t exist yet or they are not complete. So, everything stops before the console even wakes up.

This post is my deep dive into the Linux bootup procedure and Arm documentation to track down system registers that MTE depends on, find their encodings and bitfields, and wire them up in gem5 (or complete the stubs if they’re already there). But, first let's find and look at `__cpu_setup` in linux to get a better sense of what's going on.

## Reading __cpu_setup to see what MTE really needs

To understand the silent boot failure, I dug into `linux/arch/arm64/mm/proc.S` (see [this](https://github.com/torvalds/linux/blob/master/arch/arm64/mm/proc.S)) and looked at `__cpu_setup`, the early boot routine that runs before the console is alive. Its job is to reset CPU control state, configure translation controls, enable architectural features based on what the CPU advertises and prepare the MMU. If the kernel traps here, the boot appears to “freeze”, which is exactly what we saw.

When memory tagging is enabled, the kernel folds MTE support directly into the translation setup:

```S
#ifdef CONFIG_ARM64_MTE
#define TCR_MTE_FLAGS TCR_EL1_TCMA1 | TCR_EL1_TBI1 | TCR_EL1_TBID1
#else
#define TCR_MTE_FLAGS 0
#endif

mov_q tcr, ... | TCR_MTE_FLAGS

msr mair_el1, mair
msr tcr_el1, tcr
```
{: file='linux/arch/arm64/mm/proc.S'}

This does four important things for MTE:
- **Enables Top Byte Ignore (TBI)** which allows tagged pointers by ignoring the top byte during address translation.
- **Enable Tag Check Mask (TCMA1)** that defines how tag checks behave for certain accesses.
- **Prepare MAIR for tagged memory** that reserves an attribute slot for "Normal Tagged" memory mappings.
- **Set translation controls for kernel mapping** which ensures tagged memory is handled correctly in early page tables.

Once these bits are set, the kernel assumes tagged pointers and tagged memory are safe to use.

## Setting Up the MTE Hardware in Linux

After `__cpu_setup`, the kernel calls `cpu_enable_mte()` as part of CPU feature initialization (see [this](https://github.com/torvalds/linux/blob/master/arch/arm64/kernel/cpufeature.c)). As far as I understand it (and I might be oversimplifying here), when the kernel boots, it builds a table of supported CPU features by reading system registers like `ID_AA64PFR1_EL1` to check for `MTE` and `MTE2`. If the CPU claims support for these features, the kernel schedules a callback, such as `cpu_enable_mte()` to run as part of its CPU capability setup.

```c
static const struct arm64_cpu_capabilities arm64_features[] = {
  ...
  #ifdef CONFIG_ARM64_MTE
	{
		.desc = "Memory Tagging Extension",
		.capability = ARM64_MTE,
		.type = ARM64_CPUCAP_STRICT_BOOT_CPU_FEATURE,
		.matches = has_cpuid_feature,
		.cpu_enable = cpu_enable_mte,
		ARM64_CPUID_FIELDS(ID_AA64PFR1_EL1, MTE, MTE2)
	},
  ...
#endif /* CONFIG_ARM64_MTE */
  ...
};
```
{: file='linux/arch/arm64/arm64/kernel/cpufeature.c'}

Inside `cpu_enable_mte()`, the `mte_cpu_setup()` function is where the kernel does the actual hardware initialization needed for memory tagging and it touches the MTE hardware registers (see [this](https://github.com/torvalds/linux/blob/master/arch/arm64/kernel/mte.c)). The function performs a series of operations to prepare the MTE system registers so that tagging works correctly:

```c
#ifdef CONFIG_ARM64_MTE
static void cpu_enable_mte(struct arm64_cpu_capabilities const *cap)
{
  ...
  mte_cpu_setup();
  ...
}
#endif /* CONFIG_ARM64_MTE */
```
{: file='linux/arch/arm64/arm64/kernel/cpufeature.c'}

The first thing it does is to update the memory type for tagged memory in `MAIR_EL1`:

```c
sysreg_clear_set(mair_el1,
                 MAIR_ATTRIDX(MAIR_ATTR_MASK, MT_NORMAL_TAGGED),
                 MAIR_ATTRIDX(MAIR_ATTR_NORMAL_TAGGED, MT_NORMAL_TAGGED));
```
{: file='linux/arch/arm64/arm64/kernel/mte.c'}

Next, Linux configures tag checking in the **Tag Control Register** `GCR_EL1`:

```c
write_sysreg_s(KERNEL_GCR_EL1, SYS_GCR_EL1);
```
{: file='linux/arch/arm64/arm64/kernel/mte.c'}

This sets up `TCMA1`, enables top-byte ignore for kernel memory, and defines how tag faults behave. Without it, the CPU cannot interpret tags, and Linux traps immediately.

Then it seeds the **Random Tag Seed Register** `RGSR_EL1` so memory tags are pseudo-random:

```c
rgsr = (read_sysreg(CNTVCT_EL0) & SYS_RGSR_EL1_SEED_MASK) << SYS_RGSR_EL1_SEED_SHIFT;
if (rgsr == 0)
    rgsr = 1 << SYS_RGSR_EL1_SEED_SHIFT;
write_sysreg_s(rgsr, SYS_RGSR_EL1);
```
{: file='linux/arch/arm64/arm64/kernel/mte.c'}

Finally, Linux clears any pending tag-check faults:

```c
write_sysreg_s(0, SYS_TFSR_EL1);
write_sysreg_s(0, SYS_TFSRE0_EL1);
local_flush_tlb_all();
```
{: file='linux/arch/arm64/arm64/kernel/mte.c'}

There’s also another register, `GMID_EL1`, which Linux reads to determine the supported tag granule size (16B, 64B, 128B) and any implementation-specific constraints. Unlike the registers we write in `cpu_enable_mte()`, `GMID_EL1` is only read, not configured, and it’s used in other parts of the kernel after `cpu_enable_mte()` has run. The definition for this register lives in `linux/arch/arm64/kernel/mte.c`, but its role is mostly for feature detection and memory allocation decisions rather than hardware initialization.

Once these pieces are in place, the CPU is finally ready to support memory tagging. Linux first enables tag-aware address translation by programming `TCR_EL1`, `TCRMASK_EL1`, and `MAIR_EL1` so tagged pointers and tagged memory types are interpreted correctly. It then configures the tagging policy through `GCR_EL1`, seeds the random tag generator via `RGSR_EL1`, and clears any stale tag fault state in `TFSR_EL1` and `TFSRE0_EL1`. Finally, the kernel expects to be able to read `GMID_EL1` to learn the supported tag granule size and implementation limits. If any of these pieces are missing or incomplete in gem5, the kernel hits an exception during early boot, just like we saw in the previous post.

## Adding MTE system Registers to gem5

I'm gonna try to explain what I *think* I finally understood after a lot of digging and jumping between the ARM documentation and gem5 source code and connecting the dots. My goal was to find, in general, how gem5 defines Arm system (misc) registers, how they get initialized, and how access permissions are enforced.

I think the easiest way to make sense of it is to walk through a concrete example. I’ll use the **Tag Control Register** (`GCR_EL1`) and break down each step needed to add it. Once this flow makes sense, the same pattern applies to any other system register you need to implement or update.

First of all, we need to know all the register definitions for Arm live in `src/arch/arm/regs/`. The following steps are required to add a new ARM system register. Each step is illustrated with the actual `GCR_EL1` implementation.

### Step 1: Add the Register Enum Entry

We first need to add an entry to the `MiscRegIndex` enum. The entry **must** be placed before `NUM_PHYS_MISCREGS` (which counts the number of physical registers, everything after it is considered a pseudo-register). I just put it right before the `NUM_PHYS_MISCREGS`.

```cpp
enum MiscRegIndex
{
    ...
    // FEAT_MTE and FEAT_MTE2 (ARMv8.5)
    MISCREG_GCR_EL1,

    NUM_PHYS_MISCREGS,
    ...
};
```
{: file='src/arch/arm/regs/misc.hh'}

### Step 2: Add the String Name

In the same file, there is a parallel `miscRegName[]` array. We then need to add the string name at the **exact same relative position** as the enum entry. The string name is used for debug output, tracing, and checkpoint serialization.

```cpp
const char * const miscRegName[] = {
		...
    // FEAT_MTE and FEAT_MTE2 (ARMv8.5)
    "gcr_el1",

    "num_phys_regs",
    ...
};
```
{: file='src/arch/arm/regs/misc.hh'}

### Step 3: Add the Instruction Encoding Map

ARM system registers are accessed via MRS/MSR instructions with a 5-tuple encoding: `(op0, op1, CRn, CRm, op2)`. I looked for the encoding and the definition of this register in Arm doc (see [this](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/GCR-EL1--Tag-Control-Register-)) and I found the following for `GCR_EL1`:

| op0   | op1   | CRn    | CRm    | op2
| :---- | :---- | :----- | :----- | :------
| 0b11  | 0b000 | 0b0001 | 0b0000 | 0b110

So, we need to add the mapping for this encoding to `miscRegNumToIdx` map in `src/arch/arm/regs/misc.cc`. The `MiscRegNum64` constructor takes `(op0, op1, CRn, CRm, op2)` and these values come directly from the ARM Architecture Reference Manual. The map is sorted by encoding value; insert at the correct position.

```cpp
std::unordered_map<MiscRegNum64, MiscRegIndex> miscRegNumToIdx{
    ...
    { MiscRegNum64(3, 0, 1, 0, 3), MISCREG_SCTLR2_EL1 },
    { MiscRegNum64(3, 0, 1, 0, 6), MISCREG_GCR_EL1 }, // we add this
    { MiscRegNum64(3, 0, 1, 2, 0), MISCREG_ZCR_EL1 },
    ...
};
```
{: file='src/arch/arm/regs/misc.cc'}

### Step 4: Define the Bitfield Type

Next is to define a `BitUnion64` (or `BitUnion32`) type in `src/arch/arm/regs/misc_types.hh` so the register's fields can be accessed by name in C++ code. For `GCR_EL1`, the ARM specification defines the bitfields as follows (see [this](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/GCR-EL1--Tag-Control-Register-)):

![](/assets/img/figures/GCRL_EL1_reg_encoding.svg){: .dark-invert}
<em>Bit field layout of the GCR_EL1 system register</em>

So, we can define a `BitUnion64` type for it called `GCR`. 

```cpp
BitUnion64(GCR)
    Bitfield<16>    rrnd;       // Random tag generation control
    Bitfield<15, 0> exclude;    // Allocation tag exclusion mask
EndBitUnion(GCR)
```
{: file='src/arch/arm/regs/misc_types.hh'}

`Bitfield<hi, lo>` defines a multi-bit field (inclusive range) and `Bitfield<bit>` defines a single-bit field. Bits not listed are implicitly reserved/unused (`RES0` in the figure). Now, after this declaration, the type is assignable from integers or the fields are accessible as members such as:

```cpp
GCR gcr = 0;
gcr.rrnd = 1;
```

### Step 5: Initialize the Register

Now, one of the most important steps is to add an `InitReg()` call in the `ISA::initializeMiscRegMetadata()` function in `src/arch/arm/regs/misc.cc`. This sets the reset value, reserved-bit masks, access permissions, and fault/trap behavior.

The `InitReg()` function returns a `MiscRegLUTEntryInitializer` that provides a method-chaining API. All methods return a reference to allow chaining: `InitReg(X).method1().method2().method3();` For example code below sets the reset/initial value for `MISCREG_FOO` register:

```cpp
InitReg(MISCREG_FOO).reset(0x00000001);
```
Or, instead of specifying the whole bits using `.reset()`, you can use `BitUnion` fields that we defined in the previous step.

```cpp
InitReg(MISCREG_FOO).reset([](){
    FOO foo = 0;
    foo.enable = 1;
    foo.mode = 0x3;
    return foo;
}());
```

Also, you can do conditional assignments based on features (`release->has(ArmExtension::FEAT_X) ? 1 : 0`) by capturing the capture `release` pointer:

```cpp
InitReg(MISCREG_FOO).reset([release=release](){
    FOO foo = 0;
    foo.feature_x = release->has(ArmExtension::FEAT_X) ? 1 : 0;
    return foo;
}());
```

The next important thing is the **access permissions**. Access is controlled per exception level (EL0-EL3) and per security state (Secure/Non-Secure). The permission model uses a bitset with these flags:

```cpp
MISCREG_USR_NS_RD, MISCREG_USR_NS_WR    // EL0, Non-Secure
MISCREG_USR_S_RD,  MISCREG_USR_S_WR     // EL0, Secure
MISCREG_PRI_NS_RD, MISCREG_PRI_NS_WR    // EL1, Non-Secure
MISCREG_PRI_S_RD,  MISCREG_PRI_S_WR     // EL1, Secure
MISCREG_HYP_NS_RD, MISCREG_HYP_NS_WR   // EL2, Non-Secure
MISCREG_HYP_S_RD,  MISCREG_HYP_S_WR    // EL2, Secure
MISCREG_MON_NS0_RD, MISCREG_MON_NS0_WR  // EL3 (SCR.NS==0)
MISCREG_MON_NS1_RD, MISCREG_MON_NS1_WR  // EL3 (SCR.NS==1)
```
{: file='src/arch/arm/regs/misc_info.hh'}

However, there are some methods to access them conviniently such as `.allPrivileges()` which gives full access at all ELs or `.exceptUserMode()` which removes EL0 access (and is commonly chained after `.allPrivileges()`). You can find more information about these methods and functions in `src/arch/arm/regs/misc_info.hh`.

The last thing I want to talk about which I had a hard time to understand is the **Fault Handlers**. They implement conditional trapping behavior where an access at a given EL should generate a trap to a higher EL. The table below summarizes what I learned about different types of fault handlers:

| Method | Description |
|--------|-------------|
| `.faultRead(EL, callback)` | Trap read accesses at the specified EL |
| `.faultWrite(EL, callback)` | Trap write accesses at the specified EL |
| `.fault(EL, callback)` | Trap both reads and writes at the specified EL |
| `.fault(callback)` | Trap at all ELs (EL0 through EL3) |

The signature for a fault callback is like this, which eturns `NoFault` to allow the access, or calls `inst.generateTrap(target_EL)` to trap.

```cpp
Fault callback(const MiscRegLUTEntry &entry,
               ThreadContext *tc,
               const MiscRegOp64 &inst);
```

If you look at the existing fault handlers, you'll see the most common fault callback is `faultHcrFgtEL1`, which checks an `HCR_EL2` bit and an `HFGTR` bit for fine-grained traps. For example:

```cpp
InitReg(MISCREG_SCTLR_EL1)
    .allPrivileges().exceptUserMode()
    .faultRead(EL1, faultHcrFgtEL1<true, &HCR::trvm, &HFGTR::sctlrEL1>)
    .faultWrite(EL1, faultHcrFgtEL1<false, &HCR::tvm, &HFGTR::sctlrEL1>)
    .res0(...)
    .res1(...)
    .mapsTo(MISCREG_SCTLR_NS);
```
{: file='src/arch/arm/regs/misc.cc'}

I spent more time than I'd care to admit staring at the `InitReg()` chains before it finally clicked how they work. The way gem5 handles bitsets for permissions and fault handlers feels like a secret handshake between the ISA and the CPU models and if you don't get the sequence of methods like `.allPrivileges().exceptUserMode()` or the fault handling exactly right, you'll find yourself wasting hours to figure out why your code is trapping. My only suggestion is to look at what the existing code does and try to make sense out of it to use the same patterns for yourself instead of trying to learn every bit and bytes and re-invent the wheel.

Nevermind! Let's get back to our `GCR_EL1` register example and apply all of these to initialize this register. I'm gonna start by adding two MTE-specific fault handler functions following the same pattern as the existing `faultPieEL1` / `faultPieEL2`
handlers which check `SCR_EL3.piEn` for `FEAT_S1PIE` registers. I found defining my own custom fault handlers more convenient as it was a little bit tricky to cover different fault handling situations for this register based on the Arm documentation (see the pseudocode [here](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/GCR-EL1--Tag-Control-Register-)). We'll add them to `src/arch/arm/regs/misc.cc` somewhere around the other feature-specific handlers.

We need to handle two different cases:

- **MTE fault handler for EL1** where it traps to EL2 if `HCR_EL2.ATA==0`, then to EL3 if `SCR_EL3.ATA==0`, and returns `UNDEFINED` if `FEAT_MTE2` is not implemented.

```cpp
Fault
faultMteEL1(const MiscRegLUTEntry &entry,
    ThreadContext *tc, const MiscRegOp64 &inst)
{
    if (!HaveExt(tc, ArmExtension::FEAT_MTE2)) {
        return inst.undefined();
    }
    const HCR hcr = tc->readMiscReg(MISCREG_HCR_EL2);
    if (EL2Enabled(tc) && !hcr.ata) {
        return inst.generateTrap(EL2);
    }
    if (ArmSystem::haveEL(tc, EL3)) {
        const SCR scr = tc->readMiscReg(MISCREG_SCR_EL3);
        if (!scr.ata) {
            return inst.generateTrap(EL3);
        }
    }
    return NoFault;
}
```
{: file='src/arch/arm/regs/misc.cc'}

- **MTE fault handler for EL2** where it traps to EL3 if `SCR_EL3.ATA==0`, and returns `UNDEFINED` if `FEAT_MTE2` is not implemented.

```cpp
Fault
faultMteEL2(const MiscRegLUTEntry &entry,
    ThreadContext *tc, const MiscRegOp64 &inst)
{
    if (!HaveExt(tc, ArmExtension::FEAT_MTE2)) {
        return inst.undefined();
    }
    if (ArmSystem::haveEL(tc, EL3)) {
        const SCR scr = tc->readMiscReg(MISCREG_SCR_EL3);
        if (!scr.ata) {
            return inst.generateTrap(EL3);
        }
    }
    return NoFault;
}
```
{: file='src/arch/arm/regs/misc.cc'}

At this point, we're ready to add the `InitReg()` function for `GCR_EL1` as follows:

```cpp
InitReg(MISCREG_GCR_EL1).reset([](){
      GCR gcr_el1 = 0;
      gcr_el1.rrnd = 0x0;
      gcr_el1.exclude = 0x0;
      return gcr_el1 ;
  }())
  .allPrivileges().exceptUserMode()
  .res0(0xFFFFFFFFFFFE0000ULL)  // Bits [63:17]
  .faultRead(EL1, faultMteEL1)
  .faultWrite(EL1, faultMteEL1)
  .faultRead(EL2, faultMteEL2)
  .faultWrite(EL2, faultMteEL2)
  .reset(0);
```
{: file='src/arch/arm/regs/misc.cc'}

The final thing to note is that we must be careful to check any existing gem5 system registers for their fields if we're accessing them in our code like accessing `hcr.ata` or `scr.ata` in our new fault handlers. We need to make sure the `ata` field exist in both registers. After checking the Arm documentation (see [this](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/HCR-EL2--Hypervisor-Configuration-Register)), I found that the `HCR_EL2` register must have the `ata` bit as its 56th bit, however, the gem5 does not have this bit. The same goes for `SCR_EL3` and its 26th bit as `ata` bit. So, we need to add these fields to these registers.

![](/assets/img/figures/HCR_EL2.svg){: .dark-invert}
<em>Bit field layout of the HCR_EL2 system register</em>

![](/assets/img/figures/SCR_EL3.svg){: .dark-invert}
<em>Bit field layout of the SCR_EL3 system register</em>

```cpp
BitUnion64(HCR)
    Bitfield<56>     ata;   // ARMv8.5 FEAT_MTE2
    Bitfield<55>     ttlbos;
    Bitfield<54>     ttlbis;
    Bitfield<52>     tocu;
    ...
EndBitUnion(HCR)

BitUnion64(SCR)
    ...
    Bitfield<27> fgten;
    Bitfield<26> ata;   // ARMv8.5 FEAT_MTE2
    Bitfield<21> fien;
    ...
EndBitUnion(SCR)
```
{: file='src/arch/arm/regs/misc_types.cc'}

There are still some other system registers to add which I'm not gonna add to this post to not make this post longer than it is right now, but the principles and the steps to follow are the same as the one mentioned above. The most important thing is to follow the Arm documentation to define the register encodings, bitfields, initializations and permissions exactly the same as specifications.

Just to wrap up and summarize all the register additions/updates required for MTE support, take a look at the following table:

| Register | Full Name | Addition/Modification | Reference
|---------------|-----------------------| ----------| ----------|
| `GCR_EL1` | Tag Control Register | Newly added | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/GCR-EL1--Tag-Control-Register-) |
| `RGSR_EL1` | Random Allocation Tag Seed Register | Newly added | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/RGSR-EL1--Random-Allocation-Tag-Seed-Register-) |
| `TFSR_EL1` | Tag Fault Status Register (EL1) | Newly added | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/TFSR-EL1--Tag-Fault-Status-Register--EL1-) |
| `TFSRE0_EL1` | Tag Fault Status Register (EL0) | Newly added | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/TFSRE0-EL1--Tag-Fault-Status-Register--EL0--) |
| `GMID_EL1` | Multiple tag transfer ID Register | Newly added | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/GMID-EL1--Multiple-tag-transfer-ID-Register) |
| `HCR_EL2` | Hypervisor Configuration Register | Add `ata` field | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/HCR-EL2--Hypervisor-Configuration-Register) |
| `SCR_EL3` | Secure Configuration Register | Add `ata` field | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/SCR-EL3--Secure-Configuration-Register) |
| `TCR_EL1` | Translation Control Register | Add `tcma0` and `tcma1` fields | [Link](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/TCR-EL1--Translation-Control-Register--EL1-) |
| `SCTLR_EL1` | System Control Register | Add `ata` and `ata0` fields | [Link](https://developer.arm.com/documentation/111107/2025-12/AArch64-Registers/SCTLR-EL1--System-Control-Register--EL1-) |

### One More Thing: The Bootloader Needs to Know About MTE Too!

After wiring up all the registers above, I rebuilt gem5, booted the kernel, and... it still crashed in the exact same place. But this time, I've got `Secure Monitor Trap`. I stared at the trace for a while before it hit me: the problem wasn't in gem5's register model at all. It was in the *bootloader*.

Here's what was happening. Our `faultMteEL1` handler checks `SCR_EL3.ATA` (bit 26) before allowing access to MTE registers from EL1. If that bit is zero, the access traps to EL3. And it *was* zero, because the gem5 bootloader (`system/arm/bootloader/arm64/boot.S`) constructs the `SCR_EL3` value from scratch during EL3 initialization and never sets the `ATA` bit. So no matter what reset value you put in `InitReg(MISCREG_SCR_EL3)`, the bootloader overwrites it with its own value before dropping to EL2. By the time the kernel reaches `__cpu_setup` and tries to write `GCR_EL1`, `SCR_EL3.ATA` is zero, the fault handler traps to EL3, and since there's no exception handler there anymore (the bootloader already `eret`'d out), the CPU falls into the EL3 exception vector at address `0x400`, hits an undefined instruction, faults to `0x200`, and loops forever.

Before dropping to a lower exception level, the firmware must explicitly grant that level permission to use MTE by setting `SCR_EL3.ATA = 1`. The gem5 bootloader is a minimal stub, not a full firmware stack, so it simply never did this. Note that `HCR_EL2.ATA` (bit 56) doesn't need the same treatment here because the kernel is entered at EL2 and sets that bit itself in `head.S` via `init_el2_hcr HCR_HOST_NVHE_FLAGS | HCR_ATA`.

The fix is a single line in the bootloader's EL3 setup:

```S
        mov	x0, #0x30	    // RES1
        orr	x0, x0, #(1 << 0)   // Non-secure EL1
        orr	x0, x0, #(1 << 8)   // HVC enable
        orr	x0, x0, #(1 << 10)  // 64-bit EL2
        orr	x0, x0, #(1 << 16)  // Disable FEAT_PAuth traps (APK)
        orr	x0, x0, #(1 << 17)  // Disable FEAT_PAuth traps (API)
        orr     x0, x0, #(1 << 38)  // Enable HXEn
        orr     x0, x0, #(1 << 26)  // Allow MTE at lower ELs (ata)
        msr	scr_el3, x0
```
{: file='system/arm/bootloader/arm64/boot.S'}

After editing, rebuild the bootloader with `make` in `system/arm/bootloader/arm64/` (you'll need the `aarch64-linux-gnu-gcc` cross-compiler). This produces a new `boot.arm64` binary that gem5 uses at simulation startup. It's a small change, but without it none of the register work above matters because the kernel never gets a chance to touch them.

### Time to See It in Action (Second try)

Now, after adding all the MTE-related system registers, the boot procedure passes the initial step and you'll be able to see the following kernel dmesg log:

```
[0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd070]
[0.000000] Linux version 5.10.110 (kaustavg@citra) (aarch64-linux-gnu-gcc (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #3 SMP PREEMPT Wed Apr 27 17:07:00 PDT 2022
[0.000000] Machine model: V2P-CA15
[0.000000] Memory limited to 1024MB
[0.000000] Zone ranges:
[0.000000]   DMA      [mem 0x0000000080000000-0x00000000bfffffff]
[0.000000]   DMA32    empty
[0.000000]   Normal   empty
[0.000000] Movable zone start for each node
[0.000000] Early memory node ranges
[0.000000]   node   0: [mem 0x0000000080000000-0x00000000bfffffff]
[0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x00000000bfffffff]
[0.000000] On node 0 totalpages: 262144
[0.000000]   DMA zone: 4096 pages used for memmap
[0.000000]   DMA zone: 0 pages reserved
[0.000000]   DMA zone: 262144 pages, LIFO batch:63
[0.000000] cma: Reserved 16 MiB at 0x00000000bd000000
[0.000000] percpu: Embedded 22 pages/cpu s52568 r8192 d29352 u90112
[0.000000] pcpu-alloc: s52568 r8192 d29352 u90112 alloc=22*4096
[0.000000] pcpu-alloc: [0] 0 
[0.000000] Detected PIPT I-cache on CPU0
[0.000000] CPU features: detected: ARM erratum 832075
[0.000000] CPU features: detected: ARM erratum 834220
[0.000000] CPU features: detected: Spectre-v2
[0.000000] CPU features: detected: Spectre-v4
[0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923
[0.000000] CPU features: detected: Address authentication (architected algorithm)
[0.000000] CPU features: detected: Memory Tagging Extension
[0.000000] CPU features: detected: Spectre-BHB
[0.000000] ------------[ cut here ]------------
[0.000000] kernel BUG at arch/arm64/kernel/traps.c:407!
[0.000000] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
[0.000000] Modules linked in:
[0.000000] CPU: 0 PID: 0 Comm: swapper Not tainted 5.10.110 #3
[0.000000] Hardware name: V2P-CA15 (DT)
[0.000000] pstate: 00000085 (nzcv daIf -PAN -UAO -TCO BTYPE=--)
[0.000000] pc : do_undefinstr+0x2f4/0x318
[0.000000] lr : do_undefinstr+0x300/0x318
[0.000000] sp : ffffffc010b93cb0
[0.000000] x29: ffffffc010b93cb0 x28: ffffffc010b9f680 
[0.000000] x27: 0000000000000020 x26: 0000000000000001 
[0.000000] x25: ffffffc010c1f508 x24: 0000000000000004 
[0.000000] x23: 0000000000000085 x22: ffffffc010350454 
[0.000000] x21: ffffffc010b93e90 x20: ffffffc010b9f680 
[0.000000] x19: ffffffc010b93d40 x18: 0000000000000010 
[0.000000] x17: 0000000000016000 x16: 0000000000000000 
[0.000000] x15: ffffffc010350454 x14: 000000000000001d 
[0.000000] x13: ffffffc010b9faf8 x12: 00000000ffffffea 
[0.000000] x11: ffffffc010bb8470 x10: ffffffc010bb5430 
[0.000000] x9 : ffffffc010bb5488 x8 : 0000000000002fe8 
[0.000000] x7 : 0000000000000000 x6 : ffffffc010b93d08 
[0.000000] x5 : ffffffc010b93d08 x4 : ffffffc010ba19c8 
[0.000000] x3 : 0000000000000000 x2 : 0000000000000002 
[0.000000] x1 : ffffffc010b9f680 x0 : 0000000000000085 
[0.000000] Call trace:
[0.000000]  do_undefinstr+0x2f4/0x318
[0.000000]  el1_undef+0x30/0x50
[0.000000]  el1_sync_handler+0x8c/0xc8
[0.000000]  el1_sync+0x88/0x140
[0.000000]  mte_clear_page_tags+0x10/0x24
[0.000000]  enable_cpu_capabilities+0x8c/0xcc
[0.000000]  init_cpu_features+0x388/0x3a8
[0.000000]  cpuinfo_store_boot_cpu+0x4c/0x5c
[0.000000]  smp_prepare_boot_cpu+0x24/0x34
[0.000000]  start_kernel+0x17c/0x4c0
[0.000000] Code: f9401bf7 17ffff7c a9025bf5 f9001bf7 (d4210000)
```

You can see the CPU correctly advertizes the Arm MTE feature and Linux kernel correctly sees it (`CPU features: detected: Memory Tagging Extension`). However, again it crashes when it tries to *actually* use MTE instructions. The key line in the call trace is: 

```
[0.000000]  mte_clear_page_tags+0x10/0x24
```

This function lives in `arch/arm64/lib/mte.S` in the kernel and uses actual MTE instructions like `STZG` (Store Allocation Tag, Zeroing) or `STZGM` (Store Tag Multiple) to clear memory tags. gem5 doesn't implement these instructions yet. When the kernel executes one of these MTE instructions, gem5's decoder doesn't recognize it, generates an undefined instruction exception, and the kernel panics.

At this stage, we've completed the feature advertisement layer (system registers), but now comes the real fun: the instruction implementation layer. In the next few posts I'll dive into how to define and add these new Arm MTE instructions so the CPU decode stage actually knows what to do with them, and hopefully, the Linux kernel finally survives the boot procedure. Stay tuned!