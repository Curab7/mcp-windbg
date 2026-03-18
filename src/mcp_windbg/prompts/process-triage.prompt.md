Perform comprehensive analysis of a live running process by attaching with WinDbg/CDB, extracting diagnostic data, and generating a structured markdown report.

## IMPORTANT: PROCESS SAFETY
- The target process will be **suspended** while attached. Minimize attach duration.
- **Always detach** when analysis is complete to let the process resume.
- Never run destructive commands (e.g., `.kill`, `q`) on a live process.

## WORKFLOW - Execute in this exact sequence:

### Step 1: Process Identification
**If no PID provided:**
- Ask user to provide the PID of the target process.
- Suggest using `Get-Process` in PowerShell or Task Manager to find the PID.

### Step 2: Attach and Collect Diagnostic Data
**Attach to the process:**

**Tool:** `attach_windbg_process`
- **Parameters:**
  - `process_id`: Provided PID
  - `include_stack_trace`: true
  - `include_modules`: true
  - `include_threads`: true

If the attach times out, the most common reasons are:
- The process requires elevated (Administrator) privileges to attach
- Another debugger is already attached to the process
- The PID is invalid or the process has exited

**Then extract additional diagnostics with:** `run_windbg_cmd`
- **Command 1:** `|` (process info)
- **Command 2:** `vertarget` (OS version and platform details)
- **Command 3:** `~*k` (all thread call stacks)
- **Command 4:** `!peb` (process environment details)
- **Command 5:** `r` (registers of current thread)
- **Command 6:** `.time` (process uptime and current time)
- **Command 7:** `!locks` (critical section locks - useful for deadlock detection)
- **Command 8:** `!handle 0 0` (handle summary - useful for resource leak detection)

**Run additional commands based on the suspected issue:**
- **Hang/Deadlock:** `!locks`, `!cs -l`, `~*e !clrstack` (if .NET)
- **High CPU:** `!runaway` (thread CPU time)
- **Memory issues:** `!address -summary`, `!heap -s`

**Then cleanup:** `detach_windbg_process`

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
