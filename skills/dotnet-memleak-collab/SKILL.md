---
name: dotnet-memleak-collab
description: >
  Collaborative human-agent workflow for diagnosing .NET memory leaks and deadlocks using
  dotnet-dump. Use this skill whenever a developer mentions memory leak, high memory usage,
  OOM, OutOfMemoryException, heap growth, "memory never goes down", "GC isn't collecting",
  threads hung, deadlock, or wants to analyze a .dmp / core dump file. The agent guides
  the investigation step by step while the developer executes commands and reports results —
  neither can do it alone. Trigger even if the developer hasn't tried any tools yet.
---

# .NET Memory Leak — Human-Agent Collaborative Investigation

## Role Split — Read This First

The agent and the app run on the same machine, in the developer's local environment.
The agent has CLI access, can read source code in the working directory, and can install
and run dotnet-dump directly — no need to ask the developer to do these things.

**Agent does autonomously — never ask the developer:**
- Check whether dotnet-dump is installed; install it if missing
- Run `dotnet-dump ps` to find the target PID
- Run `dotnet-dump collect` to capture the `.dmp` file
- Decide where to place the `.dmp` file (inspect project structure first — use
  `diagnostics/` or `dumps/` if present, otherwise `/tmp` on Linux/macOS or
  `%TEMP%` on Windows)
- Run `dotnet-dump analyze` and all SOS commands to analyze the dump
- Read source code to understand context before asking the developer anything

**Must ask the developer — agent cannot do these:**
- **Trigger behavior in the app**: call an endpoint, perform a UI action, run a specific
  workflow to reproduce the leak — only the developer can operate the app
- **Answer business questions** that cannot be inferred from code: "is this cache meant
  to persist for the entire app lifetime?", "which flow typically causes this?"

**Production scenario (minimal support):**
The app runs on a separate server the agent cannot access. The developer collects the
dump on the server, copies it to their local machine, and provides the file path. The
agent analyzes from that file — all subsequent steps proceed as normal.

---

## When to Trigger This Skill

**Trigger on:**
- "memory keeps growing", "OOM crash", "heap too large", "GC not collecting"
- "threads are stuck", "app hangs", "deadlock"
- "I have a .dmp file", "can you analyze this dump"
- "how do I find a memory leak in .NET"

**Opening:**

Briefly confirm the plan and outline three stages:
1. **Setup & Collect** — install dotnet-dump if needed, capture a dump while the symptom
   is active
2. **Investigate** — analyze the dump, trace the root cause
3. **Fix** — produce a specific code fix and a way to verify it worked

If the developer already has a `.dmp` file: ask for the path and jump to Stage 2.

---

## Stage 1: Setup & Collect

**Goal:** Install dotnet-dump if needed, identify the correct process, capture a dump,
and determine whether the leak is already visible — before asking the developer to do
anything in the app.

### Step 1: Triage

Ask the developer only what cannot be determined from code or the environment:

1. What exactly is the symptom? (memory grows without bound / OOM crash / app hangs /
   abnormally high stable baseline)
2. Is the app running right now, or does the developer need to start it first?

Do not ask how to reproduce the leak yet. The agent collects a dump first and checks
whether the leak is already visible. Only if the heap looks clean does it make sense to
ask the developer to trigger load.

While waiting for the answer, the agent proactively reads the project structure, identifies
the app type (.NET version, hosting model), and checks whether dotnet-dump is installed.

### Step 2: Check and install dotnet-dump

Agent runs:
```bash
dotnet tool list --global
```

- `dotnet-dump` listed → already installed, proceed
- Not listed → install immediately:
  ```bash
  dotnet tool install --global dotnet-dump
  ```
- No .NET SDK available → use the direct binary download for the platform.
  See `references/install.md`.

### Step 3: Identify the target process

Agent runs:
```bash
dotnet-dump ps
```

`dotnet-dump ps` only lists .NET Core / .NET 5+ processes. It does not show .NET
Framework processes — but `dotnet-dump collect` still works on them if given the PID
directly.

**Interpret the output:**

- Expected process is listed → use that PID, proceed
- Multiple processes listed → match by command-line args. If still ambiguous, ask the
  developer which one is the target app
- Process is missing or list is empty → do not assume the app is not running. It may be
  a .NET Framework process. Find the PID using OS tools:

  On Windows:
  ```bash
  tasklist /FI "IMAGENAME eq <AppName>.exe"
  # or for all dotnet-related processes:
  tasklist | findstr -i "dotnet\|<AppName>"
  ```

  On Linux/macOS:
  ```bash
  ps aux | grep -i "<AppName>"
  ```

  If the process is found this way, use that PID directly with `dotnet-dump collect`.
  Only ask the developer if the process genuinely cannot be found by any of the above.

### Step 4: Collect a first dump immediately

Do not ask the developer to trigger any load yet. Collect a dump right away to establish
a baseline and check whether the leak is already visible.

Choose dump type based on the reported symptom:

| Symptom | Type | Reason |
|---|---|---|
| Memory leak / OOM | `Heap` | Full object graph, no native module images |
| Crash / unhandled exception | `Full` | Complete process state |
| Hang / deadlock | `Mini` | Thread stacks only, fast to collect |
| Production with PII concern | `Triage` | Like Mini but with PII stripped |

Inspect the project structure to choose a sensible output path:
- If `diagnostics/`, `dumps/`, or similar exists → use it
- Otherwise → `/tmp` (Linux/macOS) or `%TEMP%` (Windows)

**Notify the developer before running:**
> *"I'm about to collect a Heap dump. Memory will spike briefly during collection —
> if the app is in a container with a tight memory limit it may get killed. Let me know
> if you need to adjust anything first."*

Agent collects:
```bash
dotnet-dump collect -p <PID> --type Heap -o <chosen-path>/memleak_1.dmp
```

Confirm the file exists and is non-empty:
```bash
ls -lh <chosen-path>/memleak_1.dmp
```

**If collect fails:**
- Permission error → try with `sudo`, or check the process owner: `ps aux | grep dotnet`
- Timeout on Linux/macOS → `TMPDIR` mismatch. See `references/troubleshooting.md`
- Running in a container → see `references/docker.md`

### Step 5: Quick heap check — decide whether to ask for load

Immediately run a quick heap check on the first dump. Note: always end analyze commands
with `-c "exit"` so dotnet-dump exits instead of waiting for interactive input.

```bash
dotnet-dump analyze <path>/memleak_1.dmp -c "dumpheap -stat" -c "exit"
```

**If the leak is already visible** (suspicious app types with high Count or TotalSize):
→ Proceed directly to Stage 2 with this dump. No need to ask the developer to do anything.

**If the heap looks clean** (no suspicious accumulation, memory appears normal):
→ The leak has not materialized yet. Now ask the developer to trigger the relevant flow:
> *"The heap looks clean at this point — the leak hasn't built up enough to be visible
> yet. Could you [perform action X / navigate to screen Y / run workflow Z] several
> times to reproduce the memory growth? Let me know when done and I'll collect another
> dump immediately."*

After the developer confirms, collect a second dump and proceed to Stage 2:
```bash
dotnet-dump collect -p <PID> --type Heap -o <chosen-path>/memleak_2.dmp
```

---

## Stage 2: Investigate

**Goal:** Identify which type is consuming memory and why the GC cannot reclaim it.

The agent runs all SOS commands autonomously using non-interactive mode:
```bash
dotnet-dump analyze <path>/memleak.dmp -c "<command>" -c "exit"
```

**Always end with `-c "exit"`** — without it, dotnet-dump drops into an interactive
session waiting for console input, which the agent cannot handle. Multiple commands can
be chained: `-c "cmd1" -c "cmd2" -c "exit"`.

The developer is only needed when the agent has a business question or needs additional
behavior triggered in the app.

### Step 1: Runtime sanity check

```bash
dotnet-dump analyze <path>/memleak.dmp -c "eeversion" -c "exit"
```

Confirms the .NET runtime version and that SOS loaded correctly. If SOS errors appear,
see `references/troubleshooting.md` before continuing.

### Step 2: Baseline heap — most important command

```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpheap -stat" -c "exit"
```

Analyze the full output before reporting anything:

1. Sort mentally by `TotalSize` (rightmost column)
2. Find the top 3–5 types by total memory consumed
3. Separate app types (app namespace) from framework types (`System.*`)
4. Flag any type with Count > 50,000 or TotalSize > 50 MB
5. Note names containing: Cache, Session, Handler, Context, Manager, Repository

Cross-reference with source code: find the corresponding class, read the implementation,
and form an initial hypothesis before running the next command.

Report to the developer:
- What the top consumers are
- Which types look suspicious and why
- Current hypothesis in plain language
- What the next command will test

### Step 3: Drill into the suspect type

```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpheap -type <PartialTypeName> -stat" -c "exit"
```

Get instance addresses:
```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpheap -type <PartialTypeName>" -c "exit"
```

Pick 2–3 addresses to continue with.

### Step 4: Find why the GC cannot collect it

```bash
dotnet-dump analyze <path>/memleak.dmp -c "gcroot <address>" -c "exit"
```

This is the core diagnostic question: *why is this object still alive?*

Interpret the root chain and branch to the matching investigation path:

| Root chain shows | Interpretation | Path |
|---|---|---|
| `static` field | Static collection holding references | → Path A |
| `delegate` / event | Publisher keeping subscriber alive | → Path B |
| Finalizer queue | IDisposable never called | → Path C |
| `Timer` callback | Timer holding outer object alive | → Path D |
| Async state machine / Task | Incomplete async chain | → Path E |
| Thread → active local var | Object currently in use — expected | Try another address |
| No roots found | Object IS collectible — creation rate is the issue | → Path F |

Before branching: cross-reference the gcroot chain with source code. The class and field
responsible for holding the reference are usually visible immediately.

---

## Investigation Paths

### Path A — Static Collection

**Hypothesis:** A static field holds a collection that accumulates objects and is never
cleared.

```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpobj <root-owner-address>" -c "exit"
```

Look for fields of type `List<T>`, `Dictionary<K,V>`, `ConcurrentDictionary`, `Queue<T>`.
Read the source code of that class: is there an eviction or expiry policy? Is this
collection ever cleared?

Drill deeper into the collection if needed:
```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpobj <collection-address>" -c "exit"
```

Ask the developer only when intent cannot be inferred from code:
> *"Class `X` has a static field `_cache` of type `Dictionary<string, Y>`. I don't see
> anywhere in the code that clears it. Is this cache meant to live for the entire app
> lifetime, or should it be released per request or session?"*

**Fix pattern:**
- Add an eviction policy (size limit, time-based expiry)
- Replace with `IMemoryCache` or `IDistributedCache` instead of a raw static dictionary
- If key lifetime is tied to another object's lifetime, use `ConditionalWeakTable<K,V>`

### Path B — Event Handler

**Hypothesis:** A subscriber registered for an event but never unsubscribed. The publisher
holds a delegate pointing to the subscriber, preventing GC from collecting it.

```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpdelegate <delegate-address>" -c "exit"
```

Identify the subscriber type and method. Read the subscriber's source code: does it
implement `IDisposable`? Does `Dispose()` call `-=`?

Special attention: `SystemEvents.*` and `Application.*` are process-wide static events.
Subscribing without unsubscribing leaks for the entire process lifetime.

Ask the developer only if the lifecycle is unclear from code:
> *"`X` subscribes to event `Y` in its constructor. I don't see an unsubscribe anywhere.
> What is the intended lifetime of `X` — is it ever explicitly disposed?"*

**Fix pattern:**
```csharp
public void Dispose()
{
    publisher.SomeEvent -= OnSomeEvent;
    SystemEvents.DisplaySettingsChanged -= OnDisplayChanged;
}
```

### Path C — IDisposable / Finalizer Leak

**Hypothesis:** Objects with finalizers were created but `Dispose()` was never called.
They queue up for finalization instead of being released immediately.

```bash
dotnet-dump analyze <path>/memleak.dmp -c "finalizequeue" -c "exit"
```

Large counts of `SqlConnection`, `FileStream`, `HttpClient`, `StreamReader`, or `Timer`
confirm the hypothesis. Follow up with a count for the specific type:

```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpheap -type <TypeFromFinalizeQueue> -stat" -c "exit"
```

Read the source code to find where that type is created without a `using` block or
explicit `Dispose()` call.

**Fix pattern:**
```csharp
// Always use 'using' — guarantees Dispose() even when an exception is thrown
using var conn = new SqlConnection(connectionString);
using var cmd = new SqlCommand(query, conn);
```

### Path D — Timer Leak

**Hypothesis:** A `Timer` holds a delegate that captures `this`, preventing the outer
object from being collected.

```bash
dotnet-dump analyze <path>/memleak.dmp -c "timerinfo" -c "exit"
```

A large timer count, or timers pointing to types that should be short-lived, confirms the
hypothesis. Read the source code: does the owning class's `Dispose()` call
`_timer.Dispose()`?

**Fix pattern:**
```csharp
public void Dispose()
{
    _timer?.Dispose();
    _timer = null;
}
```

### Path E — Async / Task Leak

**Hypothesis:** Fire-and-forget tasks accumulate, or async state machines are stuck at
an await that never completes.

```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpasync" -c "exit"
```
```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpheap -type System.Threading.Tasks.Task -stat" -c "exit"
```

Many state machines stuck at the same await point indicates resource exhaustion blocking
completion. Read the code around that await: is there a timeout? A `CancellationToken`?

If more data is needed: ask the developer to trigger the flow that creates async work,
then collect a fresh dump.

**Fix pattern:**
- Replace `_ = DoWorkAsync()` with `await DoWorkAsync()`
- Use a bounded `Channel<T>` for intentional background work
- Add `CancellationToken` and timeouts to long-running operations

### Path F — High Object Creation Rate

**Hypothesis:** Objects are collectible but are being created faster than the GC can
reclaim them, causing sustained memory pressure.

```bash
dotnet-dump analyze <path>/memleak.dmp -c "dumpgen 2" -c "exit"
```
```bash
dotnet-dump analyze <path>/memleak.dmp -c "gcheapstat" -c "exit"
```

High Gen 2 count for a type that should be short-lived means objects are being promoted
through GC generations too quickly. A large LOH size indicates fragmentation from large
allocations (objects > 85 KB go directly to the LOH and are rarely compacted).

**Fix pattern:** `ArrayPool<T>`, `ObjectPool<T>`, `Span<T>`/`Memory<T>` to reduce heap
allocations in hot paths.

---

## Stage 3: Root Cause & Fix

When the investigation converges on a root cause, produce this report:

```
## Memory Leak Root Cause

**Symptom**: [what the developer observed]
**Leaking type**: [fully qualified name]
**Instance count**: [from dumpheap]
**Memory retained**: [TotalSize]
**Why GC cannot collect it**: [gcroot chain in plain English]

**Root cause**: [one specific sentence]

**Fix**:
[before/after code]

**How to verify the fix**:
[e.g., "Collect a new dump after 30 minutes of load — the count of X should
stay stable below 10"]

**Pattern to prevent recurrence**:
[rule to apply across the codebase]
```

Ask the developer whether the fix fits their design — they may have context that changes
the approach (e.g., the static collection is intentional but needs an eviction policy).

---

## When the Investigation Stalls

After 3+ commands with no new information:

1. **Check dump timing** — was the dump collected while memory was actually elevated? If
   it was collected too early, the evidence is not there. Ask the developer to trigger load,
   then collect a new dump at peak.
2. **Recheck the baseline** — return to `dumpheap -stat`. The type with the highest
   TotalSize is almost always the right starting point.
3. **Compare two dumps** — collect a second dump after more time or load. The delta in
   `dumpheap -stat` output directly reveals the leaking type.
4. **Read source code** — for the suspect type, reading the implementation often makes
   the leak immediately obvious.
5. **Ask the developer** — if still unclear after reading the code, ask precisely:
   *"Where is `X` created, and what is responsible for disposing it?"*
   Never leave the developer without a clear next step.

---

## Tips

- Read source code before asking the developer — most business questions answer themselves
  from the implementation.
- Notify the developer before any action with side effects: collecting a dump (memory
  spike), collecting a Full dump (very large file, may slow the app).
- When asking the developer to trigger behavior, be specific:
  *"Please [perform action X in the UI / call endpoint Y with payload Z] about 10 times
  to build up enough pressure for the leak to be visible in the dump."*
- Summarize after every 4–5 commands: what has been confirmed, what has been ruled out,
  and what the current focus is.
- Never leave the developer without knowing what the next step is.

---

## Reference Files

- `references/docker.md` — Collecting dumps from Docker/Kubernetes containers
- `references/troubleshooting.md` — SOS errors, ptrace issues, symbol problems
- `references/sos-commands.md` — Full SOS command reference
- `references/install.md` — Installing dotnet-dump without the .NET SDK
