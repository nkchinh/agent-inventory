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

This skill structures a joint investigation session where the agent provides expert
direction and interpretation, and the developer executes commands and reports back.
Neither party can do it alone: the agent can't run tools on the developer's machine;
the developer doesn't need to know which commands to run or what the output means.

## When to Offer This Workflow

**Trigger on phrases like:**
- "memory keeps growing", "OOM crash", "heap too large", "GC not collecting"
- "threads are stuck", "app hangs", "deadlock"
- "I have a .dmp file", "can you analyze this dump"
- "how do I find a memory leak in .NET"

**Opening offer:**

Offer the developer a structured investigation session. Explain there are three stages:

1. **Setup & Collect** — understand the environment, install dotnet-dump if needed,
   capture a memory dump while the problem is active
2. **Investigate Together** — run targeted commands inside the dump, with the agent
   interpreting each result and deciding what to look at next
3. **Root Cause & Fix** — identify the exact source of the leak or deadlock and produce
   a specific fix

Explain the ground rules: the developer runs commands and pastes output; the agent
interprets and decides the next step. The developer doesn't need to understand the
output — that's the agent's job.

Ask if they want to try this structured approach, or prefer to work freeform.

If they decline, work freeform using the reference files as needed.
If they accept, proceed to Stage 1.

---

## Stage 1: Setup & Collect

**Goal:** Understand the environment and obtain a usable memory dump while the symptom
is active.

### Step 1: Triage questions

Ask these questions together in one message — the developer can answer in shorthand:

1. What's the symptom? (memory grows without bound / OOM crash / app hangs / high
   baseline memory that never drops)
2. What type of app? (ASP.NET Core, Worker Service, WinForms, console, other)
3. Where is it running? (local machine, Docker, Kubernetes, Windows service, VM)
4. .NET version?
5. Is the problem happening right now, or intermittent?
6. Do they already have a `.dmp` or `core` dump file?

**If they already have a dump file:** Skip to Stage 2 immediately.

**If the problem is intermittent:** Ask what triggers it and how long before the symptom
is visible. The dump must be collected while memory is elevated — not before or after.

### Step 2: Check dotnet-dump

Ask the developer to run:

```
dotnet tool list --global
```

Paste the output. If `dotnet-dump` is not listed, provide the install command appropriate
for their situation:

- **Has .NET SDK:**
  ```
  dotnet tool install --global dotnet-dump
  ```

- **No SDK (binary download):** Give the direct download link for their platform.
  See `references/install.md` for per-platform commands.

- **Docker/Kubernetes:** The tool must be installed inside the container.
  Read `references/docker.md` and follow the container guide before continuing.

Confirm installation by asking them to run `dotnet-dump --version`.

### Step 3: Identify the target process

```
dotnet-dump ps
```

Ask them to paste the full output. From the output, identify the correct process together:
- Match by process name and command-line args
- If multiple dotnet processes, ask the developer to confirm which one is the app in question

Tell them the PID to use before moving on.

### Step 4: Choose dump type and collect

Choose the dump type based on the symptom:

| Symptom | Type | Reason |
|---|---|---|
| Memory leak / OOM | `Heap` | Contains full object graph, excludes native module images |
| Crash / unhandled exception | `Full` | Needed for complete state reconstruction |
| Hang / deadlock | `Mini` | Thread stacks only — fast to collect and analyze |
| Production (PII concern) | `Triage` | Like Mini but with PII stripped |

Give the developer the exact command with the PID filled in:

```
dotnet-dump collect -p <PID> --type Heap -o memleak.dmp
```

**Before they run it, warn them:**
- Full/Heap dumps may temporarily spike memory — in containers, this can trigger OOM
  termination. Recommend temporarily raising the container memory limit if applicable.
- On Linux/macOS: `dotnet-dump` and the target process must share the same `TMPDIR`.
  If collection times out, see `references/troubleshooting.md`.
- `dotnet-dump` must run as the same OS user as the dotnet process, or as root.

Ask them to paste the last 3 lines of output to confirm success, and to run:
```
ls -lh memleak.dmp
```
to confirm the file is non-zero size.

**Exit condition for Stage 1:**
A `.dmp` or `core` file is confirmed on disk and non-empty. Proceed to Stage 2.

---

## Stage 2: Investigate Together

**Goal:** Systematically identify what is consuming memory and why the GC cannot
reclaim it.

**Instructions to developer at the start of this stage:**

Explain the ground rules for this stage:

> *"For each step I'll tell you: (1) what command to run, (2) what it does in one sentence,
> and (3) what I'm looking for. You run the command and paste the full output — don't
> truncate even if it's long. I'll interpret it and tell you what's next.
> You don't need to understand the output yourself."*

### Step 1: Open the dump

```
dotnet-dump analyze memleak.dmp
```

Ask them to paste the prompt that appears (should be `>`). This confirms the session opened
successfully.

Then run the first diagnostic command:

```
eeversion
```

This confirms the .NET runtime version and that SOS loaded correctly. If SOS errors appear
here, read `references/troubleshooting.md` — fix before continuing.

### Step 2: Baseline — the most important command

```
dumpheap -stat
```

Ask for **full output**. This is the foundation of the entire investigation.

**How to interpret — do this analysis before responding:**

1. Sort mentally by `TotalSize` (rightmost column)
2. Find the top 3-5 types by total memory consumed
3. Separate application types (app namespace) from framework types (`System.*`)
4. Note any type with Count > 50,000 or TotalSize > 50 MB
5. Look for patterns: arrays growing large, collections, anything with "Cache", "Session",
   "Handler", "Context" in the name

**Tell the developer:**
- What the top consumers are
- Which ones look suspicious and why
- Your current hypothesis (e.g., *"200,000 Customer objects consuming 4.8 MB suggests
   something is holding a collection of customers that should have been released"*)
- What the next command will test

### Step 3: Drill into the suspect type

For the most suspicious type from Step 2:

```
dumpheap -type <PartialTypeName> -stat
```

Then get instance addresses:

```
dumpheap -type <PartialTypeName>
```

Pick **2-3 addresses** from the output. Tell the developer which addresses to use for the
next step.

### Step 4: Find why objects aren't being collected

This is the core diagnostic question. For each address:

```
gcroot <address>
```

Ask them to paste the full output. `gcroot` traces the reference chain from the GC root
to this object — it answers *"why can't the GC collect this?"*

**Interpret the root type and branch to the matching investigation path:**

| Root chain shows... | Interpretation | Investigation path |
|---|---|---|
| `static` field anywhere in chain | Static collection holding references | → Path A |
| `delegate` / event handler in chain | Publisher holding subscriber alive | → Path B |
| `Finalizer` / finalizer queue | Disposable not being disposed | → Path C |
| `Timer` callback | Timer holding outer object alive | → Path D |
| async state machine / `Task` | Incomplete async chain | → Path E |
| `Thread` → local var (top frame active) | Expected — object in use right now | Run on a different instance |
| No roots found | Object IS collectible — leak is creation rate, not retention | → Path F |

State the interpretation and hypothesis to the developer before running the next command.

---

## Investigation Paths

### Path A — Static Collection

**Hypothesis:** A static field holds a collection that accumulates objects and is never
cleared.

Ask the developer to run:
```
dumpobj <root-owner-address>
```
(use the address of the static owner from the gcroot chain)

Look at its fields for `List<T>`, `Dictionary<K,V>`, `ConcurrentDictionary`, `Queue<T>`.
If found, check the item count in those collection fields.

Then ask:
```
dumpobj <collection-address>
```

Walk the chain until the static field is identified. Report the full path:
`MyApp.SomeClass._staticField → Dictionary<string,T> → T[]`

**Fix pattern to suggest:**
- Add expiry/eviction to the collection
- Use `WeakReference<T>` or `ConditionalWeakTable<K,V>` if ownership is unclear
- Consider if the static field should exist at all — maybe it should be scoped to a request
  or service lifetime

### Path B — Event Handler

**Hypothesis:** A subscriber object registered for an event but never unsubscribed.
The publisher holds a delegate pointing to the subscriber, keeping it alive.

Ask the developer to run:
```
dumpdelegate <delegate-address>
```
(use the delegate address from the gcroot chain)

This shows the target object (subscriber) and method. Identify the subscriber type.

Then ask about the code: where does this type subscribe to events? Is there a `Dispose()`
method? Does it call `-=` on those events?

**Fix pattern to suggest:**
```csharp
// In the subscriber class
public void Dispose()
{
    publisher.SomeEvent -= OnSomeEvent;
    // Also check: Application.Idle, SystemEvents.*, static events
}
```

Special attention: `SystemEvents.*` and `Application.*` are process-wide static events —
anything subscribing to these and not unsubscribing will leak for the lifetime of the process.

### Path C — IDisposable / Finalizer Leak

**Hypothesis:** Objects with finalizers (SqlConnection, FileStream, HttpClient, etc.)
were created but never `Dispose()`d. They sit in the finalizer queue, waiting.

Ask the developer to run:
```
finalizequeue
```

Large counts of `SqlConnection`, `FileStream`, `HttpClient`, `StreamReader`, `Timer` in
this output confirm the hypothesis.

Ask them to also run:
```
dumpheap -type System.Data.SqlClient.SqlConnection -stat
dumpheap -type System.IO.FileStream -stat
```
(adjust type names to match what appeared in finalizequeue)

**Fix pattern to suggest:**
```csharp
// Use 'using' — this guarantees Dispose() even on exceptions
using var conn = new SqlConnection(connectionString);
using var cmd = new SqlCommand(query, conn);
// ... work

// Or explicit try/finally if you need to control lifetime across methods
SqlConnection conn = null;
try {
    conn = new SqlConnection(connectionString);
    // ... work
} finally {
    conn?.Dispose();
}
```

### Path D — Timer Leak

**Hypothesis:** A `System.Threading.Timer` (or `System.Timers.Timer`) is holding
a delegate that captures `this`, keeping the outer object alive indefinitely.

Ask the developer to run:
```
timerinfo
```

Large number of timers, or timers pointing to objects that should be short-lived, confirms.

**Fix pattern to suggest:**
```csharp
public class MyService : IDisposable
{
    private Timer _timer;

    public MyService()
    {
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));
    }

    public void Dispose()
    {
        _timer?.Dispose(); // Without this, _timer holds 'this' alive forever
        _timer = null;
    }
}
```

### Path E — Async / Task Leak

**Hypothesis:** Tasks are created but never awaited ("fire and forget"), causing them
to accumulate. Or async state machines are stuck at an await that never completes.

Ask the developer to run:
```
dumpasync
```

Then:
```
dumpheap -type System.Threading.Tasks.Task -stat
```

If Task count is very high, look at the await points in `dumpasync` output. Many state
machines stuck at the same await point indicates a bottleneck or resource exhaustion
preventing completion.

**Fix pattern to suggest:**
- Avoid fire-and-forget: `_ = DoWorkAsync()` → replace with `await DoWorkAsync()`
- Add `CancellationToken` to allow cleanup of stuck tasks
- Use bounded channels or `SemaphoreSlim` to limit concurrent async work

### Path F — High Object Creation Rate

**Hypothesis:** Objects are collectible (no GC roots holding them), but they're being
created so fast that GC can't keep up, causing temporary memory pressure.

Ask the developer to run:
```
dumpgen 0
dumpgen 1
dumpgen 2
```

Gen 2 is the critical one — objects here survived multiple GC cycles. High Gen 2 count
of short-lived types indicates they're being promoted too fast.

Also check:
```
gcheapstat
```

Look at Gen 2 size and LOH size relative to total heap.

**Fix pattern:** Object pooling (`ArrayPool<T>`, `ObjectPool<T>`), reducing allocations
in hot paths, using `Span<T>` and `Memory<T>` to avoid heap allocations.

---

## Stage 3: Root Cause & Fix

**Goal:** Produce a clear, actionable finding that the developer can act on immediately.

When investigation converges on a root cause, produce this report:

```
## Memory Leak Root Cause

**Symptom**: [what the developer observed]
**Leaking type**: [fully qualified type name]
**Instance count**: [from dumpheap]
**Memory retained**: [TotalSize from dumpheap]
**Why GC can't collect it**: [the gcroot chain in plain English]

**Root cause**: [one clear sentence — e.g., "CustomerCache subscribes to
ApplicationEvents.OnRequest but never unsubscribes, so every CustomerCache
instance created during the app's lifetime remains alive."]

**Fix**:
[before/after code showing exactly what to change]

**How to verify the fix worked**:
[e.g., "After deploying, collect a new dump after 30 minutes of load and
run dumpheap -stat — CustomerCache count should stay below 10."]

**Pattern to prevent recurrence**:
[e.g., "Any class that subscribes to events in its constructor must implement
IDisposable and unsubscribe in Dispose()."]
```

Ask the developer if the fix makes sense given their codebase. They may have context that
changes the approach (e.g., the static collection is intentional, but needs an eviction
policy).

---

## Handling Blockers — What to Do When the Developer Can't Proceed

These are common situations where the developer is stuck. Handle each one directly.

### "I don't have access to the server to collect a dump"

Ask:
1. Is the app in Docker, Kubernetes, or a remote VM?
2. Do they have SSH, `kubectl exec`, or RDP access?
3. Can they ask someone who does have access to run the collect command?

For Docker/Kubernetes specifics, read `references/docker.md` and walk them through it.
For remote VM: give them a self-contained script they can hand to the server team.

### "dotnet-dump collect fails — permission denied / ptrace error"

Ask them to run:
```
ps aux | grep dotnet
```
to see which OS user owns the process. Then give them the exact command:
```
sudo -u <process-owner> dotnet-dump collect -p <PID> --type Heap -o /tmp/dump.dmp
```
If sudo isn't available, ask them to try as root. If ptrace is blocked by container
policy, read `references/docker.md`.

### "The output is really long / terminal cut it off"

Ask them to save to a file first:
```
logopen /tmp/dumplog.txt
<the command>
logclose
```
Then paste the file contents:
```
cat /tmp/dumplog.txt
```

### "gcroot is hanging / taking forever"

Normal on heaps > 2 GB. Options:
- Wait — it will finish (may take several minutes)
- Try the faster variant: `gcroot -nofields <address>`
- Sample instead: run gcroot on just 2-3 addresses of the leaking type — the chain will
  be identical for all instances of the same leak pattern

### "The app isn't showing the leak right now"

Two options — ask the developer which fits:

**Option A — Wait and collect:** Does their monitoring (Prometheus, Datadog, Task Manager)
show when memory peaks? Plan to collect a dump at that time. Walk them through what to run
in advance so they're ready when the moment comes.

**Option B — Code review mode:** Ask them to share the relevant C# source files. Switch
to static analysis using the known leak patterns. Less definitive, but can narrow the
investigation significantly.

### "dotnet-dump analyze opens but I get SOS errors"

Ask them to run inside the session:
```
setsymbolserver -ms
```
This downloads the correct DAC/SOS from Microsoft's public symbol server. Requires
internet access from the machine running dotnet-dump.

For offline environments or persistent errors, read `references/troubleshooting.md`.

---

## If the Investigation Stalls

If 3+ commands in a row produce no new information:

1. **Check dump timing**: Was the dump collected while memory was actually elevated?
   If collected before the leak manifested, the evidence won't be there.
   → Collect a new dump at peak memory.

2. **Recheck the baseline**: Go back to `dumpheap -stat`. Are we investigating the
   right type? The top TotalSize type is almost always the right starting point.

3. **Compare two dumps**: Collect a second dump after more time/load and compare the
   `dumpheap -stat` delta. The types that grew are the leak.

4. **Switch to code review**: Ask the developer to share source files for the suspicious
   types. The code often makes the leak obvious once you know which class to look at.

5. **State the impasse clearly**: Tell the developer exactly what additional information
   would unblock the investigation, and what their options are.
   Never leave them with "I don't know" and no next step.

---

## Tips for This Workflow

**Tone:**
- Be direct and procedural — this is a diagnostic session, not a tutorial
- Explain the *why* briefly when it affects what the developer should do
- Don't try to teach SOS internals — focus on finding the leak

**Pacing:**
- One command at a time. Wait for output before giving the next command.
- After every output, state: (1) what it shows, (2) what it rules out,
  (3) what the next command will check
- Summarize progress every 4-5 commands

**If the developer wants to move faster:**
Acknowledge it. Offer to give 2-3 commands at once if they're comfortable running them
in sequence and pasting all output together. Adjust to their preference.

**If the developer seems lost:**
Remind them they don't need to understand the output — just run and paste. Their job is
execution; interpretation is the agent's job.

---

## Reference Files

- `references/docker.md` — Collecting dumps from Docker/Kubernetes containers
- `references/troubleshooting.md` — SOS errors, ptrace issues, symbol problems
- `references/sos-commands.md` — Full SOS command reference for when deeper
  investigation is needed beyond the standard paths above
- `references/install.md` — Platform-specific install commands (no SDK required)
