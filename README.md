# Reverse Engineering Lab

A personal collection of reverse engineering notes, runtime unpacking workflows, anti-debugging analysis, and protected binary investigations using x64dbg.

This repository documents my learning process while practicing reverse engineering techniques commonly found in protected Windows PE executables and malware-like loaders.

The primary goal of this repository is to improve low-level debugging skills, dynamic analysis workflows, and practical understanding of packers, anti-debugging mechanisms, and staged execution.

---

# Focus Areas

- Runtime PE unpacking
- Dynamic analysis with x64dbg
- Indirect control flow tracing
- Anti-debugging bypass techniques
- Memory protection analysis
- Thread-based debugger detection
- Staged loader analysis
- Runtime-generated buffers and data regions
- Windows API tracing
- Control-flow reconstruction

---

# Tools

- x64dbg
- PE-bear
- Detect It Easy (DIE)
- Process Hacker
- Windows API knowledge
- Manual runtime tracing

---

# Repository Structure

```text
reverse-engineering-lab/
│
├── README.md
│
├── writeups/
│   └── elpack-runtime-analysis/
│       ├── README.md
│       ├── images/
│       ├── notes/
│       └── patches/
│
├── cheatsheets/
│   ├── x64dbg-workflow.md
│   ├── anti-debugging-patterns.md
│   └── pe-unpacking-notes.md
│
└── scripts/
```

---

# Current Writeups

## Elpack Runtime Analysis

Dynamic analysis of an Elpack-protected executable involving:

- Indirect `call r9` tracing
- Runtime memory allocation
- VirtualProtect-based integrity checks
- Multi-stage anti-debugging layers
- Thread-based debugger detection
- Runtime-generated memory regions
- Section protection validation
- Runtime execution flow reconstruction

Main observations include:

- Multiple indirect-call stages (`call r9`, `call rax`)
- Runtime-generated executable regions
- Protection checks using `VirtualProtect`
- Anti-debugging routines triggered during step-over operations
- Dynamically generated buffers at `0x140020000`
- Staged execution that changes behavior depending on memory permissions

The writeup focuses on reasoning, workflow, and debugging methodology rather than simply patching the binary.

---

# Current Analysis Progress

The current Elpack analysis workflow includes:

## Stage 1 — Initial Indirect Control Flow

- Traced the first `call r9`
- Step-over caused immediate thread creation and program termination
- Suspected debugger detection during indirect execution
- Stepped into the call instead of stepping over

---

## Stage 2 — Runtime Anti-Debugging Layer

Inside the first `call r9`:

- Identified multiple loops and runtime-generated logic
- Encountered `call rax` that caused immediate termination when stepped over
- Determined that the indirect target was located inside the `.edata` region
- Observed that execution depended on memory protection state

Key observations:

- `call rbx` resolved to `VirtualProtect`
- The protection change failed during debugging
- Failed protection redirected execution toward anti-debugging logic

To bypass:

- Changed all sections in `elpack.exe` to full access permissions (`ERWC`)
- Forced `RAX = 1` after the protection check
- Prevented immediate debugger-triggered termination

---

## Stage 3 — Protection Validation Layer

After bypassing the first anti-debugging routine:

- Encountered `call elpack.7FF7FC8F50B0`
- Suspected this function validated section protections
- Bypassing the function directly caused execution corruption
- The next `call r9` resolved to garbage code unless protections were fixed properly

Conclusion:

- The binary validated runtime memory protections before continuing execution
- Correct memory permissions were required for valid staged execution

---

## Stage 4 — Second Indirect Execution Stage

Inside the second `call r9`:

Identified two anti-debugging functions:

- `call elpack.7FF7FC8BBD50`
- `call elpack.7FF7FC8BBD90`

Behavior observed:

- Stepping over triggered multiple thread creations
- Program terminated shortly afterward

Bypass approach:

- NOPed both anti-debugging calls
- Continued analysis by stepping into `call elpack.7FF7FC8B1000`

---

## Stage 5 — Runtime Loader / Core Logic

Inside `call elpack.7FF7FC8B1000`:

Observed:

- Strings such as `decryptme.txt`
- Multiple runtime-generated regions
- Large loop structures
- Dynamically generated buffers
- Runtime memory writes into `0x140020000`

Additional findings:

- A loop executed approximately 32 times
- After loop completion, `0x140020000` contained randomized data
- During loop execution, the memory region initially remained mostly empty
- Runtime-generated data appeared only after staged completion

Hypotheses explored:

- Runtime key generation
- Encrypted payload construction
- Staged buffer population
- Runtime plaintext reconstruction

The exact role of `0x140020000` remains partially unresolved.

---

# Learning Goals

This repository exists mainly for skill development in:

- Reverse engineering
- Binary analysis
- Runtime debugging
- Malware-analysis-style workflows
- Understanding Windows internals
- Preparing for advanced reversing certifications such as GIAC GREM

---

# Future Work

Potential future improvements include:

- Runtime dumping
- OEP recovery
- Import reconstruction
- Automated tracing scripts
- Memory snapshot comparison
- Runtime decryption analysis
- Thread creation tracing
- Hardware breakpoint workflows
- Anti-debugging pattern cataloging

---

# Disclaimer

All binaries analyzed in this repository are used strictly for educational and research purposes inside isolated environments.

No malicious deployment or unauthorized use is intended.
