Perform comprehensive analysis of a live running process by attaching with WinDbg/CDB, extracting diagnostic data, and generating a structured markdown report.

## IMPORTANT: PROCESS SAFETY
- The target process will be **suspended** while attached. Minimize attach duration.
- **Always detach** when analysis is complete to let the process resume.
- Never run destructive commands (e.g., `.kill`, `q`) on a live process.

---

## CRITICAL: LARGE BINARY SAFETY (UE5, Game Engines, AAA Titles)

Monolithic builds (UE5 Test/Shipping, large game executables) often have **300MB+ binaries with massive PDBs**. Standard WinDbg workflows can cause catastrophic timeouts. Follow these rules strictly:

### Forbidden Commands on Large Binaries
| Command | Risk | Alternative |
|---------|------|-------------|
| `x *!symbol*` | CDB hangs loading all symbols, session becomes unrecoverable | `x ModuleName!ExactClass::Method` |
| `x Module*!*pattern*` with wildcards | May still timeout on huge PDBs | Use exact match first, broaden cautiously |
| `!analyze -v` on live process | Extremely slow for large binaries | Collect targeted data instead |
| `~*k` (all thread stacks) | Can timeout with 200+ threads | `~*k 5` (limit depth) or check specific threads |
| `lm` (all modules) | Can timeout with hundreds of modules | `lm m <specific_module>` |

### Safe Command Patterns
1. **Always check module name first:** `lm m <module_name>` to confirm exact name and verify deferred symbols
2. **Use exact symbol lookup:** `x Module!Namespace::Class::Method` — no wildcards
3. **Source-level breakpoints:** `` bp `filename.cpp:linenum` `` — fast and precise
4. **Type dumps:** `dt Module!TypeName address` — works well with deferred symbols
5. **Raw memory reads:** `dq address L<count>` — zero symbol dependency

### Session Recovery
If a command hangs (30s timeout) and session becomes unresponsive:
1. **Do NOT try more commands** — they will all queue behind the stuck one
2. **Detach immediately** — but be aware: detach from a corrupted session may terminate the process
3. **Wait 5+ seconds** before re-attaching to let CDB fully cleanup
4. If re-attach fails with "initialization timed out", wait longer or restart the target process

---

## WORKFLOW - Execute in this exact sequence:

### Step 1: Process Identification
**If no PID provided:**
- Ask user to provide the PID of the target process.
- Use PowerShell: `Get-Process | Where-Object { $_.ProcessName -like "*keyword*" } | Format-Table Id, ProcessName, MainWindowTitle -AutoSize`

### Step 2: Attach and Collect Diagnostic Data

**Pre-attach checklist:**
- Identify process size: large game/engine binaries need the "Large Binary Safety" rules above
- Plan your commands before attaching to minimize suspension time

**Attach to the process:**

**Tool:** `attach_windbg_process`
- **Parameters:**
  - `process_id`: Provided PID
  - `include_stack_trace`: **false** for large binaries (collect manually with targeted commands)
  - `include_modules`: **false** for large binaries
  - `include_threads`: **false** for large binaries

If the attach times out, the most common reasons are:
- The process requires elevated (Administrator) privileges to attach
- Another debugger is already attached to the process
- The PID is invalid or the process has exited
- **PDB is extremely large** — increase timeout or use `.symopt+0x4` (deferred loads)

**Then extract diagnostics with:** `run_windbg_cmd`

**For normal-sized processes:**
- **Command 1:** `|` (process info)
- **Command 2:** `vertarget` (OS version and platform details)
- **Command 3:** `~*k` (all thread call stacks)
- **Command 4:** `!peb` (process environment details)
- **Command 5:** `r` (registers of current thread)
- **Command 6:** `.time` (process uptime and current time)
- **Command 7:** `!locks` (critical section locks - useful for deadlock detection)
- **Command 8:** `!handle 0 0` (handle summary - useful for resource leak detection)

**For large binaries (UE5, game engines):**
- Start with `lm m <exe_module_name>` to get module bounds and confirm symbol status
- Use exact symbol lookups: `x Module!GlobalVar` for specific globals
- Set breakpoints on known functions, `g` to hit them, then `dv /t` for locals
- Use `dt Module!ClassName address` to dump object fields
- Use `dq address L<count>` for raw memory inspection

**Run additional commands based on the suspected issue:**
- **Hang/Deadlock:** `!locks`, `!cs -l`, `~*e !clrstack` (if .NET)
- **High CPU:** `!runaway` (thread CPU time)
- **Memory issues:** `!address -summary`, `!heap -s`

**Then cleanup:** `detach_windbg_process`

---

## UE5/GAME ENGINE SPECIFIC PATTERNS

### Finding Globals
```
x Module!GEngine                    # UE GEngine pointer
x Module!GWorld                     # UE GWorld pointer
x Module!CVarName                   # Console variable (TAutoConsoleVariable)
```

### Inspecting UE Objects
```
dt Module!UGameEngine <address> GameViewport    # dump specific field
dt Module!FD3D12Viewport <address>              # dump D3D12 viewport
dt Module!UGameViewportClient <address> Viewport
```

### UE Container Inspection (TArray/TMap)
TArray layout: `[pointer(8)] [ArrayNum(4)] [ArrayMax(4)]`
TMap layout: TSparseArray with `[TArray Data(16)] [BitArray(32)] [FirstFreeIndex(4)] [NumFreeIndices(4)]`
```
dq <container_address> L4    # read TArray header: ptr, num|max
```

### Setting Breakpoints on UE Functions
```
x Module!ClassName::MethodName              # find address
bp <address>                                # set breakpoint
bp `SourceFile.cpp:linenum`                 # source-level breakpoint
g                                           # run until breakpoint hit
dv /t                                       # dump typed local variables
```

### vtable Identification
```
dps <object_address> <object_address+8>     # read vtable pointer with symbol
```

---

### Step 3: Generate Structured Analysis Report

## REQUIRED OUTPUT FORMAT:

```markdown
# Live Process Analysis Report
**Analysis Date:** [Current Date]
**Process ID:** [PID]
**Process Name:** [Process name]

## Executive Summary
- **Issue Type:** [Hang/High CPU/Memory Leak/Crash/General Inspection]
- **Severity:** [Critical/High/Medium/Low]
- **Finding:** [Brief description of the identified issue]
- **Recommended Action:** [Immediate next steps]

## Process Metadata
- **Process Name:** [Name and PID]
- **Process Path:** [Full executable path from !peb]
- **Command Line:** [Process command line arguments]
- **Working Directory:** [Process working directory]
- **Process Uptime:** [From .time]
- **OS Build:** [Windows version and build from vertarget]
- **Platform:** [x86/x64/ARM64]

## Thread Analysis
- **Total Thread Count:** [Count]
- **Running Threads:** [Count]
- **Waiting Threads:** [Count]

### Notable Threads
| Thread ID | State | Current Function | CPU Time |
|-----------|-------|-----------------|----------|
| [tid]     | [Running/Waiting/Suspended] | [Top of stack] | [If available] |

### Thread Call Stacks
**Thread [ID] - [Description]:**
```
Frame Module!Function+Offset
===== ==================================================
[0]   module!function+0x123
[1]   module!function+0x456
[Continue...]
```

## Lock Analysis
- **Held Locks:** [Count and details]
- **Contended Locks:** [Count and details]
- **Potential Deadlocks:** [Yes/No with explanation]

| Lock Address | Owning Thread | Recursion Count | Contention Count |
|-------------|---------------|-----------------|------------------|
| [address]   | [thread id]   | [count]         | [count]          |

## Resource Summary
- **Handle Count:** [Total handles]
- **Handle Breakdown:** [File/Event/Semaphore/etc. counts]
- **Virtual Memory:** [Usage summary if available]

## Loaded Modules Summary
| Module | Base Address | Size | Path |
|--------|--------------|------|------|
| [Primary modules of interest] | | | |

## Root Cause Analysis
[Detailed explanation of the findings, including:]
- **What was observed:** [Technical description of the process state]
- **Why it matters:** [Impact analysis]
- **Contributing factors:** [Analysis of locks, thread states, resource usage]

## Recommendations

### Immediate Actions
1. [Specific action item 1]
2. [Specific action item 2]
3. [Specific action item 3]

### Investigation Steps
1. [Follow-up analysis steps]
2. [Code review recommendations]
3. [Monitoring suggestions]

### Prevention Measures
1. [Code changes to prevent recurrence]
2. [Additional validation/checks needed]
3. [Process improvements]

## Additional Notes
[Any other relevant observations or context]
```

## ANALYSIS DEPTH:
Provide comprehensive technical details suitable for developers to:
- Understand the current process state and any anomalies
- Identify root causes of hangs, high CPU, or resource issues
- Implement appropriate fixes
- Set up monitoring to detect similar issues early

## ISSUE-SPECIFIC GUIDANCE:

### For Hang/Deadlock Analysis:
- Focus on `!locks` output to identify held and contended locks
- Check `~*k` for threads waiting on synchronization primitives (WaitForSingleObject, EnterCriticalSection, etc.)
- Look for circular wait patterns in lock ownership

### For High CPU Analysis:
- Use `!runaway` to identify threads consuming the most CPU time
- Examine call stacks of high-CPU threads for tight loops or expensive operations

### For Memory Analysis:
- Use `!address -summary` for overall memory layout
- Use `!heap -s` for heap statistics
- Look for unusually large allocations or fragmentation

**Always ask follow-up questions if:**
- PID is not provided or unclear
- User wants to focus on a specific issue type
- Additional investigation commands are needed
- The process state suggests a different analysis approach
