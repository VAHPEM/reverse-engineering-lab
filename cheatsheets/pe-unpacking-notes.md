# PE Unpacking Notes — Elpack.exe

## Overview

This document contains notes and observations gathered while dynamically analyzing and partially unpacking Elpack.exe.

The goal of the analysis was not only to recover the final payload, but also to understand:

- execution stages
- runtime memory behavior
- anti-debugging mechanisms
- unpacking flow
- protection validation logic

The reversing process focused heavily on dynamic analysis using x64dbg.

---

# Initial Entry Analysis

## First Suspicious Behavior

At the initial execution stage, several indirect calls and loops appeared almost immediately.

One instruction became especially important:

call r9 

Stepping over this instruction caused:

- new thread creation
- debugger redirection into ntdll.dll
- eventual process termination

However:

- stepping INTO the call allowed execution to continue further

This strongly suggested anti-debugging behavior.

---

# First Major Anti-Debugging Stage

## Runtime Setup Page

Inside the first call r9, another execution stage appeared containing:

- loops
- indirect calls
- memory manipulation
- protection-related logic

One critical instruction:

call rax 

Stepping over this call caused immediate termination.

The target address appeared related to the .edata section:

base(.edata) + 17D7 

---

## Hypothesis

At this stage, the function was suspected to perform:

- anti-debugging checks
- execution environment validation
- protection verification

Possibly also runtime initialization.

---

# VirtualProtect-Related Logic

## Important Observation

Another nearby instruction:

call rbx 

resolved to a VirtualProtect-related operation.

This function attempted to modify memory protections but failed initially.

Afterward:

call rax 

caused immediate process termination.

---

## Bypass Strategy

To continue execution:

- memory protections of sections inside elpack.exe were manually changed
- sections were set to:
  
text id="eyq5kx" ERWC (Execute / Read / Write / Copy) 

After changing protections:

- call rax no longer terminated execution
- later execution stages became reachable

---

# Protection Validation Layer

## Additional Validation Function

Even after bypassing the first anti-debugging routine, another function still terminated execution:

call elpack.7FF7FC8F50B0 

Initially, forcing execution flow manually did not work.

---

## Important Discovery

Simply setting:

rax = 1 

was NOT enough.

The binary still failed later.

Only after properly correcting section protections did execution continue normally.

---

## Reverse Engineering Insight

This demonstrated an important concept:

text id="i3m0m7" Some anti-debugging routines also initialize required runtime state. 

Blindly bypassing checks may break later execution.

---

# Second call R9 Stage

## New Execution Layer

After fixing protection issues, the next:

call r9 

became valid executable code instead of garbage.

This indicated that earlier execution stages successfully initialized runtime memory.

---

# Hidden Anti-Debugging Functions

Inside this second-stage execution flow, two functions repeatedly caused crashes or thread creation:

asm id="snwmnp" call elpack.7FF7FC8BBD50 call elpack.7FF7FC8BBD90 

Behavior observed:

- stepping over caused thread creation
- debugger instability occurred
- process terminated afterward

---

## Bypass Method

These calls were patched using:

NOP instructions 

After patching them, execution could continue deeper into the binary.

---

# Entering the Final Large Function

Execution eventually reached:

call elpack.7FF7FC8B1000 

This function appeared to be one of the major runtime stages because:

- execution complexity increased significantly
- memory activity increased
- important strings appeared
- runtime-generated regions became visible

Strings observed included:

decryptme.txt 

This strongly suggested that the unpacked or decrypted payload logic was nearby.

---

# Runtime Memory Region — 0x140020000

## Memory Generation Behavior

A runtime-generated memory region appeared around:

140020000 

Initially:

- region contained mostly 00
- no readable plaintext appeared

However, repeated function calls gradually filled the region.

---

## Relevant Functions

Functions involved:

00007FF7586EB39F 
00007FF7586EB3B8 
00007FF7586EB4A0 

Observed behavior:

- approximately 32 loop iterations
- memory gradually populated
- resulting bytes appeared randomized

---

# Important Observation

During loop execution:

140020000 did not visibly change immediately. 

Only after multiple iterations did meaningful data appear.

This suggested:

- staged runtime generation
- buffered writes
- delayed commits
- chunked memory transformation

---

# Insert Function Analysis

One important function:

00007FF7586EB4A0 

appeared responsible for inserting/generated data into runtime memory.

Notable behavior inside the function:

- heavy loop usage
- XOR operations
- shifting operations
- pseudo-random transformations
- repeated writes into generated memory regions

Example patterns:

xor r10, r15 
xor r10, [rsp+68] 
shl rcx, D 
shr rax, 7 

---

# Possible Interpretations

The generated region may represent:

- encrypted intermediate state
- runtime unpacked payload
- transformed data blocks
- key schedule
- decoded executable content

At the current analysis stage, it was NOT confirmed to contain final plaintext.

---

# Key Reverse Engineering Lessons

## 1. Runtime behavior matters more than imports

Traditional anti-debug APIs were mostly absent.

Behavioral analysis became more useful than static import inspection.

---

## 2. Indirect calls are critical

Many important execution paths relied on:

call r9 
call rax 
call rbx 

instead of normal APIs.

---

## 3. Protection bypasses can break initialization

Incorrectly bypassing checks caused:

- garbage code
- invalid execution paths
- broken runtime state

---

## 4. Memory protections can control execution flow

Correct section permissions were essential for stable execution.

---

## 5. Runtime-generated memory is important

The generated memory region around:

140020000 

became one of the most important analysis targets.

---

# Current Analysis Status

Successfully identified:

- multiple anti-debugging stages
- debugger-sensitive execution
- thread-based anti-analysis behavior
- protection validation logic
- runtime-generated memory regions
- staged unpacking behavior

Partially analyzed:

- runtime insertion/generation routines
- generated memory buffers
- execution stage transitions

Not yet fully resolved:

- final plaintext recovery
- full payload reconstruction
- exact decryption algorithm
- complete unpacking chain

---

# Potential Future Work

Future analysis directions may include:

- hardware breakpoints on generated regions
- memory access tracing
- write-watch analysis
- dump reconstruction
- syscall tracing
- dynamic instrumentation
- emulation-assisted tracing

---

# Final Notes

Although the full payload was not completely recovered, the reversing process provided significant practical experience in:

- x64dbg workflow
- PE unpacking methodology
- anti-debugging analysis
- runtime memory tracing
- staged execution analysis
- debugger-aware control flow analysis

The challenge demonstrated how modern protectors can combine:

- anti-debugging
- runtime initialization
- staged execution
- memory protection logic
- indirect control flow

into a tightly connected protection system.
