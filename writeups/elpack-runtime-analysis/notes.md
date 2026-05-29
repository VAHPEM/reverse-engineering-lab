# Elpack.exe Reversing Notes

## Goal

Originally wanted to solve the challenge completely and recover the plaintext.

After several days of analysis, the goal shifted toward:

- understanding the protector
- documenting anti-debugging techniques
- building reverse engineering experience
- producing a public writeup for learning purposes

---

# Day 1

Opened the binary in x64dbg.

Immediately noticed:

- many loops
- many indirect calls
- very little obvious logic
- execution repeatedly jumped through registers

Most suspicious instruction:

asm 
call r9 

Stepping over it:

- created a new thread
- entered ntdll
- eventually terminated

Stepping into it:

- execution continued

First indication that debugger interaction affected execution.

---

# First Wrong Assumption

Initially thought:

text call r9 -> anti-debugging 

and planned to simply bypass it.

Later realized:

text Some protection routines also initialize runtime state. 

Blindly skipping them breaks execution.

---

# VirtualProtect Discovery

Inside the first call r9:

Found:

asm 
call rbx 

which resolved to VirtualProtect-related logic.

The protection change failed.

Immediately afterward:

asm 
call rax 

terminated the process.

At first I only patched:

text rax = 1 

Result:

Program survived slightly longer.

Then crashed later.

---

# Important Lesson

Passing one check is not enough.

The protector appeared to verify:

- memory permissions
- runtime state
- execution environment

Multiple layers were linked together.

---

# Protection Fix

Changed all sections of elpack.exe to:

text ERWC 

After doing this:

- call rax stopped killing the process
- call elpack.7FF7FC8F50B0 stopped killing the process
- execution progressed much further

This was probably the first major breakthrough.

---

# Garbage Code Problem

At one point I bypassed a check incorrectly.

The next:

asm 
call r9 

led to garbage instructions.

Initially thought:

text Wrong code path. 

Later realized:

text Earlier routines probably initialized something. 

After fixing memory protections correctly:

the same call produced valid code.

---

# Second call R9

Eventually reached another:

asm call r9 

This one led into much larger execution logic.

Inside it:

asm call elpack.7FF7FC8BBD50 call elpack.7FF7FC8BBD90 

caused:

- thread creation
- process exit
- debugger instability

NOP'd both calls.

Execution continued.

---

# Entering Large Function

Reached:

asm call elpack.7FF7FC8B1000 

This looked important because:

- much larger code
- many loops
- memory allocations
- interesting strings

Observed:

text decryptme.txt 

which was the first string that felt related to challenge logic.

---

# 140020000 Region

One of the most interesting findings.

Memory region:

text 140020000 

was created dynamically.

Initially:

text 00 00 00 00 ... 

Everywhere.

---

# Runtime Generation

Found:

asm 00007FF7586EB39F 00007FF7586EB3B8 00007FF7586EB4A0 

related to this region.

Observed:

- around 32 iterations
- repeated writes
- region eventually filled with random-looking bytes

Important observation:

During execution I often saw:

text 140020000 unchanged 

and assumed nothing was happening.

Later discovered the changes only became visible after many iterations.

---

# Unanswered Question

Still not sure what:

text 140020000 

actually contains.

Possibilities:

- encrypted plaintext
- runtime key schedule
- intermediate buffer
- unpacked payload
- decryption workspace

Current evidence is insufficient.

---

# About the Key

At some point suspected:

text 140000000 

might contain a key.

No strong proof.

Current status:

Hypothesis only.

Need write tracing to confirm.

---

# About Plaintext

Originally assumed:

If plaintext exists,
it should appear in string references.

After spending more time on the challenge:

Not necessarily true.

The protector may:

- build strings dynamically
- decrypt only when needed
- destroy plaintext immediately
- keep plaintext entirely in memory

---

# Biggest Mistake

For a long time I focused on:

text What API is being called? 

Later realized:

The protector barely exposed obvious APIs.

Much more useful questions were:

text What changed in memory?  What caused process termination?  What state must exist for execution to continue? 

---

# Current Status

Solved:

- multiple anti-debugging layers
- protection validation logic
- thread-based detection behavior
- execution flow past several protected stages

Partially solved:

- runtime memory generation
- 140020000 region
- large function around FC8B1000

Not solved:

- final plaintext
- exact decryption algorithm
- full unpacking chain

---

# Personal Reflection

Spent around 3-4 days on this challenge.

Probably the hardest reversing task I have attempted so far.

Even though I did not fully solve it, I learned much more than from easier crackmes:

- x64dbg workflow
- memory tracing
- anti-debugging recognition
- indirect control flow analysis
- runtime protection mechanisms

This challenge taught me that reverse engineering is often about building evidence and hypotheses rather than immediately finding the final answer.
