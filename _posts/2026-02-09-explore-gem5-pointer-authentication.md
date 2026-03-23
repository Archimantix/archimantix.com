---
layout: post
title: "Arm Pointer Authentication in gem5"
description: A deep dive into gem5's FEAT_PAUTH implementation, what works, what doesn't, and how to fix the missing fault behavior.
date: 2026-02-9 22:00:00
categories: [Explorations]
tags: [gem5, arm, arm_isa, arm_pa, pauth, feat_pauth, pointer_authentication, architectural_simulation]
author: saber
---

I knew gem5 had some **Arm Pointer Authentication (PA)** code buried somewhere (`src/arch/arm/pauth_helpers.cc|hh`), but I'd never actually seen it move. So, I decided to give it a shot to see if the implementation is working (or to what extent). Instead of staring at the source and guessing, I figured we’d poke at it with a small test in a full-system simulation. Before diving into the test, here's a quick refresher on the PA instructions we'll be using, just enough to follow along without having the Arm Architecture Reference Manual open in another tab.

## A Quick Primer on the Arm PA Instructions

Pointer Authentication in ARM works by calculating a cryptographic signature (PAC) over a pointer, optionally using a context value (also called the modifier). This PAC is then embedded in the pointer’s unused high bits. When you later want to use that pointer, you can authenticate it, which either restores the original pointer or causes a fault if it’s been tampered with.

Here are the main instructions in our experiment:

### PACIA
Pointer Authentication Code using `key A`, with instruction address context. \
Signs a pointer using the specified context, producing a “PAC-signed” pointer.

Original pointer: `0x0000007ffee4a8c0` \
After PACIA with context `0x42`: `0xffab007ffee4a8c0`

### AUTIA
Authenticate Pointer using `key A`, with instruction address context. \
Checks the PAC on a pointer and, if valid, restores the original pointer. \
If invalid, sets an architecturally-defined error value (and using it will likely fault).

PAC-signed pointer: `0xffab007ffee4a8c0` \
After `AUTIA` with correct context: `0x0000007ffee4a8c0` \
After `AUTIA` with incorrect context: `0x0020007ffee4a8c0` (error pattern in high bits)

### XPACI
eXclude PAC for `key A` with instruction address context. \
Strips the PAC bits from a pointer without authenticating. \
(Handy for debugging, but don’t use this for security.) \

PAC-signed pointer: `0xffab007ffee4a8c0` \
After XPACI: `0x0000007ffee4a8c0` (original low bits restored, but validity not checked)

In the code, we’ll also trigger signals like `SIGILL` and `SIGSEGV` to detect when authentication fails.

## Test Program

Now, it's time to run a small C program to see these instructions in action. The code starts by signing a test pointer, checking if any PAC bits were added, then stripping them away and authenticating the pointer with the correct context to make sure it works. After that, it tries authentication with the wrong context to see the error pattern, and finally attempts to use the bad pointer just to confirm it triggers a fault.

```c
#include <stdio.h>
#include <stdint.h>
#include <inttypes.h>
#include <signal.h>
#include <setjmp.h>

static jmp_buf jmpbuf;

/* Catch SIGILL/SIGSEGV during PAC experiments */
static void sigill_handler(int sig) {
    printf("caught signal %d (PAC fault)\n", sig);
    longjmp(jmpbuf, 1);
}

/* Sign pointer with PACIA */
static inline void* pac_sign_ia(void* p, uint64_t context) {
    void* q = p;
    __asm__ volatile("pacia %0, %1" : "+r"(q) : "r"(context));
    return q;
}

/* Authenticate pointer with AUTIA */
static inline void* pac_auth_ia(void* p, uint64_t context) {
    void* q = p;
    __asm__ volatile("autia %0, %1" : "+r"(q) : "r"(context));
    return q;
}

/* Authenticate and then “use” the pointer */
static inline void pac_auth_and_use(void* p, uint64_t context) {
    void* q = p;
    __asm__ volatile(
        "autia %0, %1\n"
        "cbz %0, 1f\n"
        "1:\n"
        : "+r"(q)
        : "r"(context)
    );
}

/* Strip PAC without authentication (XPACI) */
static inline void* pac_strip(void* p) {
    void* q = p;
    __asm__ volatile("xpaci %0" : "+r"(q));
    return q;
}

int main(void) {
    printf("== PAC debug test ==\n\n");

    signal(SIGILL,  sigill_handler);
    signal(SIGSEGV, sigill_handler);

    uint64_t original_ptr = 0x12345678ULL;
    uint64_t modifier     = 0x42ULL;

    printf("1) original: 0x%016" PRIx64 "\n", original_ptr);

    void* signed_ptr = pac_sign_ia((void*)original_ptr, modifier);
    printf("2) signed  : %p\n", signed_ptr);

    if ((uint64_t)signed_ptr == original_ptr) {
        printf("   note: pointer unchanged after signing\n");
    } else {
        printf("   pac-hi : 0x%04" PRIx64 "\n",
               (((uint64_t)signed_ptr & 0xFFFF000000000000ULL) >> 48));
    }

    void* stripped_ptr = pac_strip(signed_ptr);
    printf("3) stripped: %p\n", stripped_ptr);

    printf("\n4) good auth\n");
    void* auth_good = pac_auth_ia(signed_ptr, modifier);
    printf("   result  : %p\n", auth_good);

    uint64_t auth_val = (uint64_t)auth_good;
    if ((auth_val & 0xFF00000000000000ULL) == 0) {
        printf("   status  : looks valid (PAC removed)\n");
    } else if ((auth_val & 0xFF00000000000000ULL) == 0x0020000000000000ULL) {
        printf("   status  : error pattern 0x%016" PRIx64 "\n",
               (auth_val & 0xFF00000000000000ULL));
    }

    printf("\n5) bad auth\n");
    if (setjmp(jmpbuf) == 0) {
        void* auth_bad = pac_auth_ia(signed_ptr, modifier + 1);
        uint64_t bad_val = (uint64_t)auth_bad;

        printf("   result  : %p\n", auth_bad);

        if ((bad_val & 0xFF00000000000000ULL) != 0) {
            printf("   status  : error bits 0x%016" PRIx64 "\n",
                   (bad_val & 0xFF00000000000000ULL));
        } else {
            printf("   status  : unexpected success (bad auth)\n");
        }

        printf("\n6) branch/use with bad auth\n");
        if (setjmp(jmpbuf) == 0) {
            pac_auth_and_use(signed_ptr, modifier + 1);
            printf("   note    : used bad pointer without fault\n");
        }
    } else {
        printf("   handler : PAC fault observed\n");
    }

    printf("\n== done ==\n");
    return 0;
}
```
{: file='test_pa.c'}

Once we have our test program, the next step is to compile it for an ARMv8.3-A target so we get the PA instructions we want. I’m using clang here with `--target=aarch64-linux-gnu` and the right `-march` flag:

```bash
clang --target=aarch64-linux-gnu -march=armv8.3-a -static -o test_pa test_pa.c
```

## Running the Test in gem5 FS Mode

Once the binary is built, we need to get it into our disk image so gem5 can run it inside the simulated system. Let's first get our kernel and disk image.

```shell
mkdir -p fs_files
cd fs_files

# kernel
wget https://gem5dist.blob.core.windows.net/dist/develop/kernels/arm/static/arm64-vmlinux-5.10.110

# disk image
wget https://gem5dist.blob.core.windows.net/dist/develop/images/arm/ubuntu-18-04/arm64-ubuntu-20220425.img.gz
gunzip arm64-ubuntu-20220425.img.gz
```

The easiest way to put the binary into the disk image is to mount it with losetup and copy the file. There are probably quicker ways using gem5 tools, but this method works just fine for me.

```bash
LOOP_DEVICE=$(sudo losetup -f --show -P fs_files/arm64-ubuntu-20220425.img)
sudo mkdir -p /mnt/gem5-disk
sudo mount ${LOOP_DEVICE}p1 /mnt/gem5-disk # or p2 if p1 is not available
sudo cp path/to/test_pa /mnt/gem5-disk/
ls -la /mnt/gem5-disk/test_pa # make sure the binary is copied correctly
sudo umount /mnt/gem5-disk
sudo losetup -d $LOOP_DEVICE
```

Now our binary is sitting in the root directory of the disk image, ready to run. To make it execute automatically right after Linux boots, I’ll drop in a small boot script called `boot.rcS` that calls the program and then exits gem5:

```shell
# Run the test
echo "Running pointer authentication test..."
/test_pa

/sbin/m5 exit
```
{: file='fs_files/boot.rcS'}

With that in place, we can build gem5 and fire up a full-system simulation. At this point, we’ll be able to see exactly how pointer authentication behaves inside gem5.

```shell
scons -j$(nproc) build/ARM/gem5.opt
build/ARM/gem5.opt configs/example/arm/starter_fs.py \
		--cpu atomic \
		--num-cores 1 \
		--mem-size 2GB \
		--kernel fs_files/arm64-vmlinux-5.10.110 \
		--disk-image fs_files/arm64-ubuntu-20220425.img
		--script fs_files/boot.rcS
```

## Expected Output

This is the Linux kernel boot log from the simulation:

```txt
$ telnet localhost 3456
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
==== m5 terminal: Terminal 0 ====
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd070]
[    0.000000] Linux version 5.10.110 (kaustavg@citra) (aarch64-linux-gnu-gcc (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #3 SMP PREEMPT Wed Apr 27 17:07:00 PDT 2022
[    0.000000] Machine model: V2P-CA15
[    0.000000] Memory limited to 1024MB
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000080000000-0x00000000bfffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x00000000bfffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x00000000bfffffff]
[    0.000000] cma: Reserved 16 MiB at 0x00000000bd000000
[    0.000000] percpu: Embedded 22 pages/cpu s52568 r8192 d29352 u90112
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: ARM erratum 832075
[    0.000000] CPU features: detected: ARM erratum 834220
[    0.000000] CPU features: detected: Spectre-v2
[    0.000000] CPU features: detected: Spectre-v4
[    0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923
[    0.000000] CPU features: detected: Address authentication (architected algorithm)
[    0.000000] CPU features: detected: Spectre-BHB
[    0.000000] alternatives: patching kernel code
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 258048
[    0.000000] Kernel command line: console=ttyAMA0 lpj=19988480 norandmaps root=/dev/vda1 rw mem=1GB
[    0.000000] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
[    0.000000] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
...
```

This line `CPU features: detected: Address authentication (architected algorithm)` means Pointer Authentication (PA) is implemented and detected on your system. It means the CPU is advertising `FEAT_PAUTH` with the standard QARMA algorithm, not some custom implementation. Good so far.

Then, the test itself runs:

```bash
=== PAC Debug Test ===

1. Original Pointer: 0x12345678
2. Signed Pointer: 0x4b5c0012345678
   PAC bits added: 0x4b
3. Stripped (XPACI): 0x12345678

4. Testing GOOD authentication:
   Authenticated pointer: 0x12345678
   Authentication appears successful (PAC removed)

5. Testing BAD authentication:
   Setting up fault handler...
   BAD AUTH RESULT: 0x20000012345678
   ERROR: Bad authentication succeeded - PAC validation not working!

6. Attempting to use bad pointer:
   ERROR: Using bad pointer succeeded!

=== Test Complete ===
```

Steps 1 through 4 look exactly right. But steps 5 and 6 are the problem. When we authenticate with the wrong modifier, instead of faulting we get back `0x20000012345678` — the error bits got set, but the program just kept running. And then in step 6, using that corrupted pointer as a branch target also didn't fault. On real hardware, both of those should have crashed.

So what's working and what isn't?

Working:

- Signing with `PACIA` correctly embeds PAC bits (added PAC bits `0x4b5c` to make `0x4b5c0012345678`)
- `XPACI` correctly strips them without checking (strips to original `0x12345678`)
- `AUTIA` with the correct modifier correctly removes the PAC and restores the original pointer

**NOT** Working:

- `AUTIA` with a wrong modifier sets error bit 53 (`0x20000...`) but doesn't fault
- Using the corrupted pointer afterward also **doesn't** fault

The `0x20000...` pattern is the clue. Let's look at why gem5 produces it and why it's wrong.

**Does this mean that gem5's PAC implementation is incomplete/incorrect** for authentication failures?

To see the exact behavior, let’s take a look at the gem5 source code.

## gem5's Implementation of PA

The core of gem5's PA logic lives in `auth()`, called by the instruction implementations for `AUTIA`, `AUTIB`, `AUTDA`, and so on. Here's the function in full implemeted in `src/arch/arm/pauth_helpers.cc`:

```cpp
uint64_t
ArmISA::auth(ThreadContext *tc, ExceptionLevel el, uint64_t ptr,
        uint64_t modifier, uint64_t k1, uint64_t k0, bool data,
        uint8_t errorcode)
{
    uint64_t PAC;
    uint64_t result;
    uint64_t original_ptr;
    // Reconstruct the extension field used of adding the PAC to the pointer
    bool tbi = calculateTBI(tc, el, ptr, data);
    bool selbit = (bool) bits(ptr, 55);

    int bottom_PAC_bit = calculateBottomPACBit(tc, el, selbit);

    uint32_t top_tbi = tbi? 56: 64;
    uint32_t nbits = top_tbi - bottom_PAC_bit;
    uint64_t pacbits = ((uint64_t)0x1 << nbits) -1; // 2^n -1;
    uint64_t mask = (pacbits << bottom_PAC_bit); // creates mask

    if (selbit) {
        original_ptr = ptr | mask;
    } else {
        original_ptr = ptr & ~mask;
    }


    PAC = QARMA::computePAC(original_ptr,  modifier, k1, k0);
    // Check pointer authentication code

    // <bottom_PAC_bit:0>
    uint64_t low_mask = ((uint64_t)0x1 << bottom_PAC_bit) -1;
    // <54:bottom_PAC_bit>
    uint64_t pac_mask = 0x007FFFFFFFFFFFFF & ~low_mask;

    uint64_t masked_pac = PAC & pac_mask;
    uint64_t masked_ptr = ptr & pac_mask;

    if (tbi) {
        if (masked_pac == masked_ptr) {
            result = original_ptr;
        } else {
            uint64_t mask2= ~((uint64_t)0x3 << 53);
            result = original_ptr & mask2;
            result |= (uint64_t)errorcode << 53;    // <-- places error at bits [54:53]
        }
    } else {
        if ((masked_pac == masked_ptr) && ((PAC >>56)==(ptr >> 56))) {
            result = original_ptr;
        } else {
            uint64_t mask2 = ~((uint64_t)0x3 << 61);
            result = original_ptr & mask2;
            result |= (uint64_t)errorcode << 61;   // <-- places error at bits [62:61]
        }
    }
    return result;
}
```
{: file='src/arch/arm/pauth_helpers.cc'}

There is one clear problem visible here, before we even look at the callers.

### The Problem — auth() Never Raises a Fault

Look at the function signature: it returns `uint64_t`. The `ThreadContext*` is passed in, but only used to look up TBI and PAC key width, nothing on it is ever called to raise an exception. Without `FEAT_FPAC`, the spec says `AUT*` instructions don't fault immediately. Instead they write the corrupted pointer back and let the fault come later, when the CPU actually uses it as a branch target or memory address. At that point the MMU is supposed to see the error bits and raise an **Authentication Fault** (`EC=0x21` or `0x25`). Step 6 of our test was exactly this scenario; authenticate badly, then branch through the result. On hardware it crashes. **In gem5 nothing happens**, because `FEAT_FPAC` hasn't been implemented and the MMU has no knowledge of PAC error bits.

So `auth()` itself is doing what it should: it detects the mismatch and produces a corrupted pointer. The hole is downstream.

### Going Deeper

Let's follow how `auth()` gets called from an actual instruction. The high-level helpers, `authIA`, `authDA`, `authIB`, `authDB`, all look something like this:

```cpp
Fault
ArmISA::authIA(ThreadContext * tc, uint64_t X, uint64_t Y, uint64_t* out)
{
    bool trapEL2 = false;
    bool trapEL3 = false;
    bool enable = false;

    uint64_t hi_key= tc->readMiscReg(MISCREG_APIAKeyHi_EL1);
    uint64_t lo_key= tc->readMiscReg(MISCREG_APIAKeyLo_EL1);

    ExceptionLevel el = currEL(tc);
    ...

    switch (el)
    {
        case EL0: { ... }
        case EL1: { ... }
        case EL2: { ... }
        case EL3: { ... }
        default:
            //unreachable
            break;
    }
    if (!enable)
        *out = X;
    else if (trapEL2)
        return trapPACUse(tc, EL2);
    else if (trapEL3)
        return trapPACUse(tc, EL3);
    else
        *out = auth(tc, el, X, Y, hi_key, lo_key, false, 0x1);

    return NoFault;
}
```
{: file='src/arch/arm/pauth_helpers.cc'}

And in `src/arch/arm/isa/insts/pauth.isa`, the `AUTIA` instruction is wired up roughly as:

```cpp
    uint64_t res;
    // op1 is XDest
    // op2 is Src
    fault = %(op)s(xc->tcBase(), %(op1)s, %(op2)s, &res);
    XDest = res;
```
{: file='src/arch/arm/isa/insts/pauth.isa'}


That's it. The instruction writes the (possibly poisoned) result to the destination register and moves on. Even though `authIA` does return a `Fault`, that return value is only ever non-null for the trapping cases (EL2/EL3 trap). Authentication failure itself always returns `NoFault`. Nothing further down the pipeline checks whether `XDest` now holds a corrupted pointer.

## The Full Picture

gem5's PA implementation is a genuine partial implementation, not a stub. Signing works correctly, `PACIA` produces a properly computed QARMA-based PAC and embeds it in the right bits. Stripping works too, `XPACI` cleanly restores the original pointer. Successful authentication also does the right thing, `AUTIA` with a matching modifier correctly strips the PAC and returns the original pointer. Even mismatch detection is there, gem5 sees the bad PAC and writes an error value back to the register. What gem5 has implemented is `FEAT_PAUTH`, the baseline ARMv8.3-A feature, which is exactly what the kernel detects and reports at boot. What it hasn't gotten to yet is `FEAT_FPAC`, the extension that adds faulting on authentication failure.

## Possible Fixes

There are two paths to getting fault behavior working, and which one you want depends on what you're trying to simulate.

### Option 1 — Implement FEAT_FPAC

The architecturally correct way to get immediate faults on authentication failure is to implement `FEAT_FPAC`. This means going into the `authIA`, `authDA`, `authIB`, and `authDB` wrappers and checking whether `auth()` returned a poisoned pointer. If it did, instead of returning `NoFault`, raise an exception right there:

```cpp
Fault
ArmISA::authIA(ThreadContext *tc, uint64_t X, uint64_t Y, uint64_t *out)
{
    // ... trapping checks as before ...

    uint64_t result = auth(tc, el, X, Y, k1, k0, false, errorcode);
    *out = result;

    // FEAT_FPAC: fault immediately if authentication failed
    if (haveFEAT_FPAC(tc) && isPACPoisoned(result)) {
        return faultOnPACFailure(tc, el, ...);
    }

    return NoFault;
}
```
{: file='src/arch/arm/pauth_helpers.cc'}

This keeps the existing non-`FEAT_FPAC` path untouched and adds the immediate-fault behavior gated behind the feature flag, which is exactly how the ARM spec structures it.

### Option 2 — Implement the Deferred Fault in the MMU

If you want the non-`FEAT_FPAC` baseline behavior where the fault fires on use of the corrupted pointer rather than at the `AUT*` instruction itself, the fix lives in the address translation path. The MMU needs to recognize the PAC error bit pattern and raise an **Authentication Fault** before proceeding with the translation. I belive `src/arch/arm/mmu.cc` is the right place, and the function to look at is `MMU::translateFs()`. That's where all translation requests go through regardless of whether they're atomic, timing, or functional (`translateAtomic`, `translateTiming`, `translateFunctional` all eventually call into `translateFs`). Within `translateFs`, the very first thing that happens to the virtual address is a call to `purifyTaggedAddr()`, which strips the tag bits if TBI is active. That's exactly where a PAC poison check belongs (right after `purifyTaggedAddr()` and before the actual TLB lookup begins), since at that point you have the raw virtual address the CPU is trying to use and can call out early with the right fault.

```cpp
Fault
MMU::translateFs(const RequestPtr &req, ThreadContext *tc, Mode mode, ...)
{
    updateMiscReg(tc, NormalTran, state.isStage2);
    Addr vaddr_tainted = req->getVaddr();
    Addr vaddr = 0;
    if (state.aarch64) {
        vaddr = purifyTaggedAddr(vaddr_tainted, tc, state.exceptionLevel,
            static_cast<TCR>(state.ttbcr), mode==Execute, state);

        // PAC deferred fault check: if authentication previously failed,
        // the pointer carries an error code in bits [62:61]
        uint64_t err_bits = (vaddr >> 61) & 0x3;
        if (err_bits == 0x1 || err_bits == 0x2) {
            // EC = 0x21 for instruction fetch, 0x25 for data
            return std::make_shared<PACFault>(vaddr_tainted, mode == Execute);
        }
    } else {
        vaddr = vaddr_tainted;
    }

    // ... rest of translation continues normally
}
```
{: file='src/arch/arm/mmu.cc'}

`PACFault` doesn't exist in gem5's fault hierarchy yet. Before this compiles you'll need to add it to `src/arch/arm/faults.hh` and `faults.cc` as a new `ArmFault` subclass with the correct EC value (`0x21` for instruction fetch, `0x25` for data access).

### Wrapping Up

For most gem5 users this will never surface. If you're just booting Linux or running workloads that don't deliberately tamper with signed pointers, everything works fine. But if you're using gem5 to simulate PAC-aware security code, test exploit mitigations, or study how the failure path behaves, **the simulator will silently let corrupted pointers through where real hardware would have crashed**.

The two fix options we sketched out, implementing `FEAT_FPAC` in the `authIA`/`authDA` wrappers, or adding a deferred PAC poison check in `MMU::translateFs()`, each have their place. Option 1 is the architecturally correct path and the right long-term fix per the ARM spec. Option 2 is a reasonable shortcut if you just need the fault to fire without going through a full `FEAT_FPAC` implementation (a pain). If you end up going down either route and want to contribute a patch upstream, I'd love to hear about it. Drop a comment and I'll update this post accordingly.