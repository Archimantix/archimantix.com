---
title: "Arm MTE in gem5 | Part 3 : Adding the First Instruction"  
description: Implementing IRG, the first Arm MTE instruction  
date: 2026-03-18 22:00:00  
categories: [dev_log]  
tags: [gem5, arm, arm_isa, arm_mte, mte, feat_mte, memory_tagging, architectural_simulation]  
author: saber
---

In the last post, the Linux kernel boot hit another wall, this time because it tries to execute MTE instructions during early setup, like `STZG` and `STZ2G` (for zeroing allocation tags), and gem5 simply didn't know what to do with them.. So, in this post and the next few, I'll dive into the guts of gem5 to gradually add these missing MTE instructions.

This part of gem5 is where things get really interesting. It gives us the power to extend an ISA with custom instructions, a "must-have" capability in computer architecture research when you're prototyping hardware support for new ideas. So follow along for a real-world ISA extension in action, a process I personally learned a lot from.

## The Low-Hanging Fruit

I’m starting with the simplest Arm MTE instruction: `IRG`. This is a the perfect "Hello World" for gem5’s ISA layer and its Domain Specific Language (DSL) because it's strictly a **tag manipulation** instruction. Since it only touches registers and stay far away from memory access, it's a great way to get hands-on without things getting too messy—at least, not yet :)

```
// Generate a random tag for our base pointer in x0
irg  x0, x1
```

To actually teach gem5 how to handle `IRG`, we have to stop looking at it like programmers and start looking at it like hardware. So, let’s look at the Arm ISA bible to see how these instructions are actually constructed. I’ll zoom in on the `IRG` instruction to show you how to read these encodings. If we want gem5 to simulate this, we need to know exactly which bits tell the decoder 'this is an `IRG`' and which bits identify our registers. The following information are from Arm A-profile A64 ISA Base Instructions description section (see [this](https://developer.arm.com/documentation/ddi0602/2025-12/Base-Instructions/IRG--Insert-random-tag-)):

### Step 1: Read Instruction Specification

> This instruction inserts a random Logical Address Tag into the address in the first source register, and writes the result to the destination register. Any tags specified in the optional second source register or in `GCR_EL1.Exclude` are excluded from the selection of the random Logical Address Tag

Also, the encoding of this instruction is shown in the figure below:

`IRG <Xd|SP>, <Xn|SP>{, <Xm>}`

- `Xd|SP`  Is the 64-bit name of the destination general-purpose register or stack pointer
- `Xn|SP` Is the 64-bit name of the first source general-purpose register or stack pointer
- `Xm`	Is the 64-bit name of the second general-purpose source register, Defaults to XZR if absent.

{: .dark-invert}
*The IRG instruction's encoding.*

This bit pattern is our roadmap. In gem5, we’ll use this encoding to tell the simulator: 'When you see this specific sequence of ones and zeros, trigger the MTE logic for IRG instruction.' 

The pseudocode below is the bridge. It defines the behavior we need to replicate in gem5's DSL, specifically how we extract the register indices and apply the tag. We’ll use this logic later to ensure that when the gem5 decode stage identifies an IRG bit pattern, it knows exactly how to transform the source address into a tagged result.

```
if !IsFeatureImplemented(FEAT_MTE) then EndOfDecode(Decode_UNDEF); end;
let d : integer{} = UInt(Xd);
let n : integer{} = UInt(Xn);
let m : integer{} = UInt(Xm);

let operand : bits(64) = if n == 31 then SP{64}() else X{64}(n);
let exclude_reg : bits(64) = X{}(m);
let exclude : bits(16) = exclude_reg[15:0] OR GCR_EL1().Exclude;

let rtag : bits(4) = AArch64_ChooseTagOrZero(exclude);
let result : bits(64) = AArch64_AddressWithAllocationTag(operand, rtag);

if d == 31 then
    SP{64}() = result;
else
    X{64}(d) = result;
end;
```

With the encoding and operation pseudocode in hand, we have our **blueprint**. Now comes the fun part: diving into the gem5 source code to actually implement it. There's probably a more 'correct' way to order these steps, but I found this sequence to be the most intuitive for me.

### Step 2: Add the ISA Decoder Entry

Before we can implement the logic, we have to tell gem5 that this specific bit pattern actually belongs to a real instruction and this is done by gem5 ISA decoder as part of its **ISA Parser** (see [this](https://www.gem5.org/documentation/general_docs/architecture_support/isa_parser/)).

The gem5 decoder is essentially a massive, nested tree structure defined in a custom DSL. When the simulator fetches a 32-bit chunk of memory, it walks down this tree, checking specific bit fields, until it reaches a "leaf" that identifies the instruction. For `IRG`, we need to find the right "branch" of the A64 ISA and plug in our encoding. While the main Arm ISA decoder tree is defined in `src/arch/arm/isa/decoder`, the actual aarch64 decoder logic for aarch64 instructions is redirected to C++ functions in `src/arch/arm/isa/formats/aarch64.isa` or other files in `src/arch/arm/isa/formats` based on the instruction type. Since `IRG` is a data processing instruction with two source registers (based on the encoding above), the closest target match is `decodeDataProcTwoS` object in `src/arch/arm/isa/formats/aarch64.isa`. Honestly, I'm not sure if there's a more "official" way to find the correct class in the decoder, but it does not actually matters when you can always confirm you're in the right place by matching the encoding bit patterns.

In decodeDataProcTwoS, the decoder first verifies that bit `[29]` is 0 (as required by the ISA for this group). It then extracts bits `[4:0]`, `[9:5]`, and `[20:16]` to identify `Xd`, `Xn`, and `Xm`. The critical part is the switch statement on bits `[15:10]`, which represent the Opcode. The next switch statement and its various case statements instantiate a different instruction object based on the encoded `Opcode`. In our case, if these 6 bits represent binary sequence `0b000100` or hex value of `0x4`, it maps to `IRG` instruction. For `IRG`, these 6 bits are `0b000100` (hex `0x4`). Gem5 doesn't have this case by default, so we have to add it. Note that we use `makeSP()` for the destination and first source register, as the `IRG` instruction allows these to be the Stack Pointer (SP).

```cpp
output decoder { {
namespace Aarch64
{
    StaticInstPtr decodeLogical(ExtMachInst machInst){/* ... */}
    StaticInstPtr decodeAddSub(ExtMachInst machInst) {/* ... */}
    StaticInstPtr decodeAddSubWithCarry(ExtMachInst machInst) {/* ... */}
    StaticInstPtr decodeRotIntoFlags(ExtMachInst machInst) {/* ... */}
    StaticInstPtr decodeEvalIntoFlags(ExtMachInst machInst) {/* ... */}
    StaticInstPtr decodeCondCompare(ExtMachInst machInst) {/* ... */}
    StaticInstPtr decodeCondSelect(ExtMachInst machInst) {/* ... */}
    StaticInstPtr decodeDataProcTwoS(ExtMachInst machInst) {
        if (bits(machInst, 29) != 0)
            return new Unknown64(machInst);

        RegIndex rd = (RegIndex)(uint8_t)bits(machInst, 4, 0);
        RegIndex rdzr = makeZero(rd);
        RegIndex rn = (RegIndex)(uint8_t)bits(machInst, 9, 5);
        RegIndex rm = (RegIndex)(uint8_t)bits(machInst, 20, 16);

        uint8_t switch_val = bits(machInst, 15, 10);
        switch (switch_val) {
          // ...
          case 0xb:
            return new Rorv64(machInst, rdzr, rn, rm);
          case 0x4:   // we add this for IRG instruction
            return new Irg(machInst, makeSP(rd), makeSP(rn), rm);
          case 0xc:
            return new Pacga(machInst, rd, rn, makeSP(rm));
          // ...
          default:
            return new Unknown64(machInst);
        }
    }
    StaticInstPtr decodeDataProcOneS(ExtMachInst machInst) {/* ... */}
    StaticInstPtr decodeDataProcReg(ExtMachInst machInst) {/* ... */}
}
}};
```

{: file='src/arch/arm/isa/formats/aarch64.isa'}

### Step 2: Define Instruction Behavior in the ISA DSL

gem5 uses a custom ISA DSL to define instructions. This DSL is a mix of C++-like syntax and Python code, and honestly, it’s the steepest part of the learning curve in the gem5 ISA layer. It took me quite a bit of trial and error to figure it out, mainly because there isn't any document for this specific part of the gem5's ISA layer (maybe there is one now which I'm not aware of). But after banging my head against it for a while, I came up with a simple intuition: **think of the ISA DSL as an automatic code-generating factory.**

Instead of manually writing thousands of repetitive C++classes for every single instruction, you write a high-level "recipe" in the DSL. The gem5 ISA Parser then reads these recipes and automatically churns out the complex C++ code needed for the simulator to function. It’s essentially a clever mix of Python (for the logic of building the code) and C++ templates (for the actual hardware logic).

The DSL uses three main components to build an instruction:

- **Templates**: These are C++ "skeletons." They define what an instruction class looks like but leave blanks for things like the mnemonic (e.g., `irg`) and the actual execution math.
- **Code Blobs**: This is the "meat" of the instruction. It's the small snippet of C++ that describes exactly what the hardware does (e.g., `res = op1 + op2`).
- **Python Builders**: These are the automatic "generators". They take a template, plug in a code blob, and assign it a name.

Before we look at the `IRG` code, let’s look at how we would define a simple `XOR` instruction. In the DSL, you don't write a C++ class; you write a recipe that tells gem5 how to build one. 

Imagine we want to implement `XOR X0, X1, X2`:

First, we write the C++ snippet for the actual math. We use DSL keywords like `Op164` (Source 1), `Op264` (Source 2), and XDest (Destination) so gem5 knows which registers to "wire" up (they're defined in `src/arch/arm/isa/operands.isa`).

```cpp
xor_logic = "XDest = Op164 ^ Op264;"
```

Next, we tell gem5: "Create a new instruction called `xor`. Use the `DataX2RegOp` skeleton, which provides two source registers and one destination."

```cpp
iop = ArmInstObjParams("xor", "Xor", "DataX2RegOp", xor_logic)
```

Finally, we trigger the factory to inject that logic into the standard AArch64 C++ templates. This generates the header, the decoder, and the execution code all at once.

```cpp
header_output  += DataX2RegDeclare.subst(iop)
decoder_output += DataX2RegConstructor.subst(iop)
exec_output    += BasicExecute.subst(iop)
```

I followed this same pattern for `IRG`. I don't have to write a whole C++ class for it from scratch. Instead, I tell the DSL:
"Hey, use the standard 'Two-Register' skeleton, call it 'irg', and here is the snippet of logic to put inside it."

To keep things clean, I put all MTE-related definitions in a new file: `src/arch/arm/isa/insts/mte.isa` (don't forget to add it to `src/arch/arm/isa/insts/insts.isa`). Here is how I defined the `IRG` instruction in ISA DSL.

```cpp
let { {
    header_output = ""
    decoder_output = ""
    exec_output = ""

    def mteEnabledCode():
        return """
            if (!HaveExt(xc->tcBase(), ArmExtension::FEAT_MTE)) {
                return std::make_shared<UndefinedInstruction>(machInst, true);
            }
            """

    irgCode = mteEnabledCode() + """
        uint64_t res;
        fault = opIRG(xc->tcBase(), Op164, Op264, &res);
        XDest = res;
        """

    iop = ArmInstObjParams("irg", "Irg", "DataX2RegOp", irgCode, [])
    header_output += DataX2RegDeclare.subst(iop)
    decoder_output += DataX2RegConstructor.subst(iop)
    exec_output += BasicExecute.subst(iop)
}};
```

{: file='src/arch/arm/isa/insts/mte.isa'}

Building on the `XOR` intuition, the `IRG` implementation follows the same flow. The primary difference is the use of the `DataX2Reg` template, which handles the wiring for three register operands (two sources and one destination). The DSL generators then simply swap in our `irgCode` as the execution logic.

Because the MTE logic is too complex to fit into a single string, we use a hybrid approach: the DSL defines the instruction's interface and register mapping, while dedicated C++helper functions (`src/arch/arm/mte_helpers.cc|.hh`) handle the architectural logic. This keeps the DSL files readable and provides the full control of a standard C++ environment for the heavy lifting (following the the same practice from the existing `pauth_helper.cc|.hh` files related to the Arm Pointer Authentication logic).

The `irgCode` consists of two distinct components:

- **A Feature Gatekeeper**: the `mteEnabledCode()` snippet ensures the CPU throws an `UndefinedInstruction` fault if `FEAT_MTE` is missing. This is a direct translation of the ARM pseudocode:

```
if !IsFeatureImplemented(FEAT_MTE) then EndOfDecode(Decode_UNDEF); end;
```

- **The Architectural Logic**: `opIRG()` function which maps to the core operation of the instruction.

```
let d : integer{} = UInt(Xd);
let n : integer{} = UInt(Xn);
let m : integer{} = UInt(Xm);

let operand : bits(64) = if n == 31 then SP{64}() else X{64}(n);
let exclude_reg : bits(64) = X{}(m);
let exclude : bits(16) = exclude_reg[15:0] OR GCR_EL1().Exclude;

let rtag : bits(4) = AArch64_ChooseTagOrZero(exclude);
let result : bits(64) = AArch64_AddressWithAllocationTag(operand, rtag);

if d == 31 then
    SP{64}() = result;
else
    X{64}(d) = result;
end;
```

`opIRG()` function gets two 64-bit source registers `Op164` and `Op264` and a 64-bit destination register `res`. Also the function returns a fault if anything unexpected happens during its exeuction. Also, `xc->tcBase()` passes the current "Thread Context" (the CPU state) so the helper function can read system registers like `GCR_EL1`.

```cpp
fault = opIRG(xc->tcBase(), Op164, Op264, &res);
```

Now, this is the actual code for `opIRG` in `src/arch/arm/mte_helpers.cc`:

```cpp
namespace gem5
{
using namespace ArmISA;

Fault
ArmISA::opIRG(ThreadContext *tc, uint64_t operand, uint64_t excludeReg,
              uint64_t *result)
{
    // read system registers `GCR_EL1`
    GCR gcr_el1 = tc->readMiscReg(MISCREG_GCR_EL1);

    // ARM spec: 
    // exclude = GCR_EL1.Exclude OR operand2<15:0>
    uint16_t exclude = gcr_el1.exclude | bits(excludeReg, 15, 0);

    // ARM spec: 
    // let rtag : bits(4) = AArch64_ChooseTagOrZero(exclude);
    uint8_t rtag = chooseTagOrZero(tc, exclude);

    // ARM spec: 
    // let result : bits(64) = AArch64_AddressWithAllocationTag(operand, rtag);
    *result = addressWithAllocationTag(operand, rtag);

    return NoFault;
}
}
```

{: file='src/arch/arm/mte_helpers.cc'}

My goal was to make the C++ an almost line-by-line translation of the Arm A64 ISA pseudocode. It reads the current exception level `currEL(tc)` of the thread context, reads system registers, `GCR_EL1` for configuration and `RGSR_EL1` for the current tag state, calculates the exclusion mask, and coordinates whether to generate a tag via the internal seed or a truly random source. Finally, it stores the randomly tagged address in the destination register.

Arm ISA specification has a clear definition for each of the functions used in the pseudocode like `AArch64_ChooseTagOrZero()` which is defined like this (see [this](https://developer.arm.com/documentation/ddi0602/2025-12/Shared-Pseudocode/aarch64-functions-system?lang=en#func_AArch64_ChooseTagOrZero_1)):

```
// AArch64_ChooseTagOrZero()
// =========================
// Return a tag, excluding any tags in the given mask, , or zero if Allocation
// Tag access is not enabled.

func AArch64_ChooseTagOrZero(exclude : bits(16)) => bits(4)
begin
    if IsMTEEnabled(PSTATE.EL) then
        if GCR_EL1().RRND == '1' then
            if IsOnes(exclude) then
                return '0000';
            else
                return ChooseRandomNonExcludedTag(exclude);
            end;
        else
            if IsFeatureImplemented(FEAT_MTE_EIRG) then
                return AArch64_ChooseEIRGNonExcludedTag(exclude);
            end;
            let start_tag : bits(4) = RGSR_EL1().TAG;
            let offset : bits(4)    = AArch64_RandomTag();

            RGSR_EL1().TAG = AArch64_ChooseNonExcludedTag(start_tag, offset, exclude);

            return RGSR_EL1().TAG;
        end;
    else
        return '0000';
    end;
end;
```

To implement these, you just need to hunt down the functions in the Arm ISA spec and map them to C++as closely as possible. gem5 actually makes this pretty easy with built-in utilities in the ArmISA and gem5 namespaces like `readMiscReg()` for reading system registers or `currEL()` for checking the Exception Level. I found that digging through existing code like `src/arch/arm/pauth_helpers.cc|.hh` was the best "cheat sheet" for finding these utility functions and implementation patterns. As you can see below, the final C++ ends up being almost a line-by-line match with the Arm pseudocode. I ignored `FEAT_MTE_EIRG` (Enhanced Insert Random taG algorithm) for now since it’s for newer MTE revisions and isn't needed for a base implementation. 

```cpp
namespace gem5
{
using namespace ArmISA;

uint8_t
ArmISA::chooseTagOrZero(ThreadContext *tc, uint16_t exclude)
{
    ExceptionLevel el = currEL(tc);
    GCR gcr_el1 = tc->readMiscReg(MISCREG_GCR_EL1);
    RGSR rgsr_el1 = tc->readMiscReg(MISCREG_RGSR_EL1);

    uint8_t rtag;

    if (IsMTEEnabled(tc, el)) {
        if (gcr_el1.rrnd == 1) {
            if (exclude == 0xFFFF)
                rtag = 0;
            else
                rtag = chooseRandomNonExcludedTag(tc, exclude);
        } else {
            uint8_t startTag = rgsr_el1.tag;
            uint8_t offset = randomTag(tc);
            rtag = chooseNonExcludedTag(tc, startTag, offset, exclude);
            rgsr_el1.tag = rtag;
            tc->setMiscReg(MISCREG_RGSR_EL1, rgsr_el1);
        }
    } else {
        rtag = 0;
    }

    return rtag;
}
}
```

{: file='src/arch/arm/mte_helpers.cc'}

From here, it’s just a matter of following the spec and ticking off each required function in `mte_helpers.cc|.hh`. Once those supporting functions are finished, the `opIRG()` logic is complete and ready to use.

For the final touch; don't forget to add `mte_helpers.hh` to the `output exec` block in `src/arch/arm/isa/includes.isa`:

```cpp
output exec { {
...
#include "arch/arm/mte_helpers.hh"
#include "arch/arm/pauth_helpers.hh"
...

namespace gem5::ArmISAInst
{
using namespace ArmISA;
} // namespace gem5::ArmISAInst

}};
```

### Step 3: Build and Test

Once the `IRG` instruction is fully implemented, we can give it a try to see if we can run a small test to make sure it works as expected. Let's write a simple c code:

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define insert_random_tag(ptr) ({               \
    uint64_t __val;                             \
    asm volatile("IRG %0, %1"                   \
        : "=r" (__val)                          \
        : "r" (ptr));                           \
    __val;                                      \
})

int main(int argc, const char **argv) {

  const char* ptr = "SECRET";
  char* tagged_ptr;

  tagged_ptr = (char *) insert_random_tag(ptr);

  printf(" Pointer is tagged with MTE Address Tag 0x%x: Addr=%p\n", 
        (uint8_t)(((uint64_t)tagged_ptr >> 56) & 0xFF), tagged_ptr);

  tagged_ptr = (char *) insert_random_tag(ptr+1);

  printf("Pointer is tagged with MTE Address Tag 0x%x: Addr=%p\n", 
        (uint8_t)(((uint64_t)tagged_ptr >> 56) & 0xFF), tagged_ptr);

  return 0;
}
```

{: file='irg_test.c'}

Then compile it for aarch64 + MTE:

```bash
aarch64-linux-gnu-gcc -march=armv8.5-a+memtag -O0 -g -static irg_test.c -o irg_test
```

Now, you can run your code in the SE mode (our FS mode is still not passing the early boot process) and now you should not get any errors for an unimplemented instruction and verify if the instruction adds random tags to the pointers correctly.

```bash
./build/ARM/gem5.opt configs/example/arm/starter_se.py --cpu atomic --num-cores 1 ./irg_test
```

The output looks like:

```
Pointer is tagged with MTE Address Tag 0x0: Addr=0x453230
Pointer is tagged with MTE Address Tag 0x0: Addr=0x453231
```

That’s a wrap on our first instruction! Extending the Arm A64 ISA by adding `IRG` is a huge milestone, but we’re just getting started. In the next few posts, I’ll keep walking through the rest of the MTE instructions to dig into even more advanced gem5 ISA DSL features and templates. Next up, I'll be adding the rest of the tag manipulation instructions. Stay tuned!