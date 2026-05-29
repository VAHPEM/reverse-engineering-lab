# x64dbg Workflow Cheatsheet

A practical workflow for dynamic analysis of protected Windows PE executables using x64dbg.

This cheatsheet focuses on runtime debugging, indirect control flow, anti-debugging behavior, memory protection checks, and unpacking-style analysis.

---

# 1. Initial Setup

## Recommended Layout

Open these panels before starting analysis:

- CPU
- Dump
- Stack
- Registers
- Memory Map
- Breakpoints
- Call Stack
- Threads
- Log

Useful views:

```text
View -> Memory Map
View -> Breakpoints
View -> Call Stack
View -> Threads
View -> Handles
```

---

# 2. First Run Strategy

Do not immediately patch everything.

Start with observation:

1. Run to entry point.
2. Step slowly through early code.
3. Watch for:
   - indirect calls
   - suspicious loops
   - memory protection changes
   - new thread creation
   - sudden process termination

Common suspicious patterns:

```asm
call rax
call rbx
call r9
```

These often indicate:

- dynamically resolved APIs
- unpacking stubs
- anti-debug checks
- transition into another execution stage

---

# 3. Step Over vs Step Into

## Step Over

Use step over when:

- the call target is known
- the function is likely a normal API
- you only care about return value

Example:

```asm
call kernel32.VirtualProtect
```

## Step Into

Use step into when:

- stepping over causes termination
- stepping over creates new threads
- the call target is indirect
- the call is part of a protection layer

Example:

```asm
call r9
call rax
call rbx
```

If step over causes the program to quit, restart and step into the call.

---

# 4. Handling Indirect Calls

When you see:

```asm
call r9
```

Check:

```text
R9
```

Then follow the target:

```text
Right click register -> Follow in Disassembler
```

or use the command bar:

```text
disasm <register_value>
```

Example:

```text
disasm r9
```

If the target looks like valid code, continue.

If the target looks like garbage:

- the stage may not be initialized yet
- a protection check may have failed
- memory permissions may be wrong
- a previous anti-debug branch may have corrupted execution

---

# 5. Memory Protection Checks

Protected binaries often check or modify section permissions.

Common APIs:

```text
VirtualProtect
VirtualAlloc
NtProtectVirtualMemory
NtAllocateVirtualMemory
```

Important registers for Windows x64 calling convention:

```text
RCX = arg1
RDX = arg2
R8  = arg3
R9  = arg4
[rsp+20] = arg5
```

For `VirtualProtect`:

```c
BOOL VirtualProtect(
  LPVOID lpAddress,     // RCX
  SIZE_T dwSize,        // RDX
  DWORD  flNewProtect,  // R8
  PDWORD lpflOldProtect // R9
);
```

Common protection values:

```text
0x01 = PAGE_NOACCESS
0x02 = PAGE_READONLY
0x04 = PAGE_READWRITE
0x08 = PAGE_WRITECOPY
0x10 = PAGE_EXECUTE
0x20 = PAGE_EXECUTE_READ
0x40 = PAGE_EXECUTE_READWRITE
0x80 = PAGE_EXECUTE_WRITECOPY
```

If a protection check fails and causes termination:

1. Check `LastError`.
2. Check the target section in Memory Map.
3. Confirm whether the target address is mapped.
4. Confirm section permissions.
5. Consider manually setting the relevant section to executable/readable/writable in Memory Map.

---

# 6. Memory Map Workflow

Use Memory Map to track runtime regions.

Look for:

- newly allocated memory
- executable private memory
- writable executable memory
- mapped payloads
- unpacked PE images

Important signs:

```text
MZ
PE
.text
.rdata
.data
.pdata
.reloc
```

If a new region appears during execution:

1. Note the base address.
2. Dump the first few pages.
3. Look for PE headers.
4. Check section table.
5. Find `AddressOfEntryPoint`.

Example:

```text
Base: 140000000
AEP : 4A08
OEP : 140004A08
```

Formula:

```text
OEP = ImageBase + AddressOfEntryPoint
```

---

# 7. Breakpoint Strategy

## Software breakpoint

Use for normal code addresses:

```text
bp <address>
```

Example:

```text
bp 140004A08
```

## Hardware breakpoint

Use when software breakpoints fail or code is self-modifying:

```text
bph <address>
```

## Memory breakpoint

Use to catch reads, writes, or execution on memory:

```text
bpm <address>, w
bpm <address>, r
bpm <address>, x
```

Examples:

```text
bpm 140020000, w
bpm 140020000, r
bpm 140004A08, x
```

Use memory write breakpoints to answer:

```text
Who writes to this buffer?
```

Use memory execute breakpoints to answer:

```text
When does this code execute?
```

---

# 8. Tracking Runtime Buffers

When a buffer becomes important:

1. Dump before the function call.
2. Step over the call.
3. Dump after the call.
4. Compare.

Useful targets:

```text
stack buffers
heap buffers
.data section
newly allocated executable memory
temporary decode buffers
```

If a buffer changes every run, suspect:

- entropy
- PRNG state
- anti-debug poison
- time-based randomization
- runtime scratch space

Instructions that indicate entropy:

```asm
rdtsc
rdtscp
QueryPerformanceCounter
GetTickCount
GetSystemTime
```

---

# 9. When a Call Causes Termination

If stepping over a call causes the program to quit:

1. Restart.
2. Step into the call.
3. Check if it:
   - creates threads
   - checks debugger state
   - checks memory protection
   - checks timing
   - validates checksums
4. Identify return value.
5. Test patching return value only.
6. If return-value patching breaks later code, do not skip the whole function.

Important lesson:

```text
Some checks initialize important runtime state.
Skipping them may corrupt later execution.
```

---

# 10. Common Anti-Debugging Indicators

Watch for:

```asm
rdtsc
rdtscp
int 3
icebp
pushfq / popfq
GetTickCount
QueryPerformanceCounter
IsDebuggerPresent
CheckRemoteDebuggerPresent
NtQueryInformationProcess
OutputDebugStringA
CreateThread
NtCreateThreadEx
```

Behavioral signs:

- step over creates new threads
- program exits only under debugger
- call target becomes garbage after skipping checks
- memory permissions differ from expected
- execution depends on timing

---

# 11. Patching Strategy

Patch minimally.

Preferred order:

1. Change register return value.
2. Patch conditional jump.
3. NOP small anti-debug call.
4. Patch memory protection manually.
5. Avoid deleting large functions unless confirmed safe.

Examples:

```asm
test eax,eax
je bad_path
```

Can be patched by:

```asm
jne good_path
```

or by setting:

```text
EAX = 1
```

or:

setting RIP to the next address

For calls:

```asm
call anti_debug_check
test al,al
je exit
```

Possible options:

```text
set AL = 1
NOP call
patch JE -> JNE
```

But always check if the call initializes state.

---

# 12. Tracing Section Reconstruction

When analyzing an unpacking routine, look for this pattern:

```asm
mov rcx, destination
mov rdx, source
mov r8, size
call copy_or_decrypt
```

Windows x64 convention:

```text
RCX = destination
RDX = source
R8  = size
```

If destination is inside mapped payload:

```text
140000000
140020000
140004A08
```

then the function may be:

- copying sections
- decompressing sections
- decrypting section data
- rebuilding PE memory layout

---

# 13. Identifying the Real Payload

Possible indicators of real payload logic:

- readable strings appear
- file names appear
- API arguments become meaningful
- code stops looking like loader stub
- fewer anti-debug branches
- stable control flow
- references to application-specific data

Examples:

```text
decryptme.txt
CreateFile
ReadFile
WriteFile
GetProcAddress
LoadLibrary
```

If a string appears only at runtime, it may be:

- stack string
- decrypted string
- constructed buffer
- temporary API argument

---

# 14. Useful x64dbg Commands

```text
bp <address>
```

---

# 15. Useful Questions During Analysis

Ask these repeatedly:

```text
What changed after this call?
What buffer did this function write?
Is this code protection logic or payload logic?
Is this value stable across runs?
Is this buffer plaintext, encrypted data, or entropy?
Did skipping this function corrupt later execution?
Where does this pointer come from?
Who writes to this address first?
Who executes this address first?
```

---

# 16. Practical Workflow Template

Use this workflow when facing an unknown protected binary:

```text
1. Run to entry point.
2. Identify suspicious indirect calls.
3. Step into indirect calls if step-over causes termination.
4. Track memory protections.
5. Watch Memory Map for new executable regions.
6. Identify mapped PE-like regions.
7. Calculate possible OEP.
8. Set breakpoints on OEP and key runtime regions.
9. Track memory writes to important buffers.
10. Compare memory before and after each major call.
11. Avoid over-patching.
12. Document each hypothesis and result.
```

---

# 17. Documentation Template

For every important finding, record:

```text
Address:
Instruction:
Registers:
Memory before:
Memory after:
Observed behavior:
Hypothesis:
Result:
```

Example:

```text
Address:
00007FF7586EB3B8

Instruction:
call elpack.7FF7586EB4A0

Observed behavior:
Populates runtime buffer around 0x140020000.

Hypothesis:
Section reconstruction or runtime buffer generation.

Result:
Buffer appears high-entropy and changes across runs.
Likely runtime state or randomized section data.
```

---

# 18. Key Lessons

- Do not trust static string references alone.
- Indirect calls often hide important transitions.
- Step-over can trigger anti-debugging.
- Some anti-debug functions also initialize required state.
- Runtime memory permissions matter.
- A buffer that changes every run may be entropy, not plaintext.
- Write down failed hypotheses; they are part of the analysis.
- A good reversing writeup explains reasoning, not just final results.
