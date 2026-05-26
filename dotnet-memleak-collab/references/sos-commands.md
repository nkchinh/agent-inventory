# SOS Command Reference — Practical Guide

Quick-reference for the most useful SOS commands in memory leak and deadlock investigation.
Commands are grouped by use case, not alphabetically.

---

## Table of Contents

1. [Heap Overview](#heap-overview)
2. [Object Inspection](#object-inspection)
3. [GC Root Analysis](#gc-root-analysis)
4. [Thread & Stack Analysis](#thread--stack-analysis)
5. [Lock & Deadlock Analysis](#lock--deadlock-analysis)
6. [Async & Task Analysis](#async--task-analysis)
7. [Finalizer Queue](#finalizer-queue)
8. [GC & Runtime Info](#gc--runtime-info)
9. [Specialized Commands](#specialized-commands)

---

## Heap Overview

### `dumpheap -stat`
Summary of all objects on the managed heap, grouped by type.
```
> dumpheap -stat
Statistics:
              MT    Count    TotalSize Class Name
00007f...    206770     19494060 System.String
00007f...    200000      4800000 MyApp.Controllers.Customer
```
**Use**: First command in every investigation. Sort by TotalSize mentally.

### `dumpheap -stat -type <partial-name>`
Filter to a specific namespace or type:
```
> dumpheap -stat -type MyApp.Services
> dumpheap -stat -type Customer
```

### `dumpheap -type <partial-name>`
List all instances of a type with addresses and sizes:
```
> dumpheap -type Customer
         Address               MT     Size
00007f6ad09421f8 00007f6c20a67498       24
00007f6ad0942210 00007f6c20a67498       24
```
Pick an address for `dumpobj` or `gcroot`.

### `dumpheap -mt <MethodTable>`
List all instances by MethodTable address (from -stat output):
```
> dumpheap -mt 00007f6c20a67498
```
More precise than `-type` — matches exactly one class.

### `dumpheap -min <bytes>` / `dumpheap -max <bytes>`
Filter by object size:
```
> dumpheap -min 85000 -stat      # Large Object Heap candidates
> dumpheap -min 1000000 -stat    # Objects > 1 MB
```

### `dumpgen <generation>`
Show objects in a specific GC generation:
```
> dumpgen 2        # Gen 2 = long-lived objects, main leak candidate
> dumpgen 3        # LOH (Large Object Heap)
```

---

## Object Inspection

### `dumpobj <address>`
Full details of an object: type, size, fields, field values:
```
> dumpobj 00007f6ad09421f8
Name:        MyApp.Controllers.Customer
MethodTable: 00007f6c20a67498
Size:        48(0x30) bytes
Fields:
              MT    Field   Offset   Type VT  Attr  Value  Name
00007f...   400001       8   String   0  inst  00007f6a...  Name
00007f...   400002      10    Int32   1  inst          42   Id
```
**Use**: After finding a suspicious address, inspect its field values and sub-references.

### `dumpvc <MethodTable> <address>`
Inspect a value type (struct). Use when `dumpobj` can't resolve a value type field.

### `dumparray <address>`
Inspect a managed array, showing element count and addresses:
```
> dumparray 00007f6a00148a00
```

### `objsize <address>`
Recursively calculate total memory retained by an object (including all references):
```
> objsize 00007f6ad09421f8
sizeof(00007f6ad09421f8) = 2097152 (0x200000) bytes (MyApp.Services.Cache)
```
**Use**: When a small object header hides a large retained graph.

### `listnearobj <address>`
Show the object immediately before and after an address on the heap. Useful for
understanding heap layout and corruption.

---

## GC Root Analysis

### `gcroot <address>`
**Most important command.** Find all paths from GC roots to this object:
```
> gcroot 00007f6ad09421f8

Thread 3f68:
  00007F67... MyApp.Services.Cache.GetAll()
    rbx: (interior)
      -> 00007F6B... System.Object[]
      -> 00007F69... MyApp.Services.CustomerCache
      -> 00007F69... System.Collections.Generic.List`1[[Customer]]
      -> 00007F6C... MyApp.Controllers.Customer[]
      -> 00007F6A... MyApp.Controllers.Customer
Found 1 root.
```

**Interpreting roots**:
- `Thread N` → on an active thread stack (expected during execution)
- `Static` → static field holds reference (leak candidate)
- `Handle` → explicit GCHandle not freed
- `Finalizer` → in finalizer queue (Dispose not called)

### `gcroot -nofields <address>`
Faster variant — skips field inspection. Use on large heaps when `gcroot` is slow.

### `pathto <root-address> <target-address>`
Find the shortest path between two specific objects. Useful when you already know the
suspected root object.

### `gchandles`
List all GC handles (pinned, strong, weak, etc.) and statistics:
```
> gchandles
```
High number of Pinned handles can cause GC fragmentation.

---

## Thread & Stack Analysis

### `clrthreads`
List all managed threads with OS thread ID, state, and managed thread ID:
```
> clrthreads
ThreadCount:      7
UnstartedThread:  0
BackgroundThread: 6
PendingThread:    0
DeadThread:       0

   ID OSID        State   GC Mode     GC Alloc Context Lock Count    Apt Exception
   1  1234     202a020 Preemptive  ...                  0 Ukn
   6  5678     8009220 Preemptive  ...                  1 Ukn (Threadpool Worker)
```

### `clrstack`
Stack trace of the current thread (managed frames only):
```
> clrstack
```

### `clrstack -all`
Stack traces of **all** threads:
```
> clrstack -all
```
**Use**: Deadlock investigation — scan all stacks for Monitor/lock patterns.

### `clrstack -p`
Include method parameters in stack trace. Helps understand what data the method was
processing.

### `threads <thread-id>` or `setthread <thread-id>`
Switch focus to a specific thread (by managed thread ID from `clrthreads`):
```
> setthread 6
> clrstack
```

### `dso` or `dumpstackobjects`
List all managed objects on the current thread's stack:
```
> dso
```

### `parallelstacks`
Visual merged view of all thread stacks — similar to Visual Studio's Parallel Stacks panel.
Helps find groups of threads stuck at the same point.

---

## Lock & Deadlock Analysis

### `syncblk`
Show SyncBlock table — which objects have Monitor locks, who owns them, who is waiting:
```
> syncblk
Index  SyncBlock  MonitorHeld  Recursion  Owning Thread
   43  00000246...      603         1      0000024B... 5634
```
- `MonitorHeld` = 1 means lock held; > 1 means additional threads waiting
- Cross-reference `Owning Thread` address with `clrthreads` output

### `eestack`
Run `clrstack` on all threads including unmanaged frames. More detailed than `clrstack -all`.

### `threadpool`
Show thread pool state: min/max threads, active workers, completion port threads:
```
> threadpool
```

### `threadpoolqueue`
List queued work items in the thread pool queue:
```
> threadpoolqueue
```
Large queue = CPU-bound saturation or work items blocking.

---

## Async & Task Analysis

### `dumpasync`
**Critical for async leak investigation.** Lists all async state machines on the heap
with their current await point:
```
> dumpasync
```
Shows which `await` point each state machine is suspended at. Many state machines stuck
at the same point indicates a bottleneck or leak.

### `dumpheap -type Task -stat`
Count Task objects — large numbers indicate fire-and-forget accumulation:
```
> dumpheap -type System.Threading.Tasks.Task -stat
> dumpheap -type System.Threading.Tasks.Task`1 -stat
```

### `taskstate <address>`
Human-readable Task state (Running, WaitingForActivation, Faulted, Canceled, etc.):
```
> taskstate 00007f6a...
```

---

## Finalizer Queue

### `finalizequeue`
List all objects registered for finalization (have finalizers / `~Destructor`):
```
> finalizequeue
SyncBlocks to be cleaned up: 0
Free-Threaded Interfaces to be released: 0
MTA Interfaces to be released: 0

Statistics for all finalizable objects (including all objects ready for finalization):
              MT    Count    TotalSize Class Name
00007f...        5          600 System.IO.FileStream
00007f...       12         2880 System.Data.SqlClient.SqlConnection
```

**Use**: Large counts of `SqlConnection`, `FileStream`, `HttpClient` etc. = IDisposable
objects with finalizers that were never `Dispose()`d.

---

## GC & Runtime Info

### `eeheap -gc`
Detailed GC heap layout — SOH generations, LOH, allocated vs committed:
```
> eeheap -gc
Number of GC Heaps: 1
generation 0 starts at 0x...
generation 1 starts at 0x...
generation 2 starts at 0x...
Large object heap starts at 0x...
         segment     begin allocated      size
large    00007f...  00007f...  00007f...  0x...
Total Size:         Size: 0x... (524288000) bytes.
GC Heap Size:       Size: 0x... (524288000) bytes.
```

### `gcheapstat`
Statistics per GC heap (useful for Server GC with multiple heaps):
```
> gcheapstat
```

### `analyzeoom`
If app crashed with OOM, show details of the last failed allocation:
```
> analyzeoom
```

### `eeversion`
Show .NET runtime and SOS versions. Always run first to confirm dump is valid:
```
> eeversion
Microsoft (R) .NET Framework. Version  8.0.10
```

---

## Specialized Commands

### `timerinfo`
List all active `System.Threading.Timer` instances:
```
> timerinfo
```
Each active timer holds a delegate (and thus `this`) alive.

### `dumpdomain`
Show all AppDomains and loaded assemblies:
```
> dumpdomain
```

### `dumpheap -type WeakReference -stat`
Check if `WeakReference` objects are being used (should be collectible).

### `dumpheap -type ConditionalWeakTable -stat`
Find ConditionalWeakTable instances — used in caches, can still leak if keys aren't freed.

### `verifyheap`
Check for GC heap corruption:
```
> verifyheap
```
Any output here means heap corruption — different problem from a memory leak.

### `dumpconcurrentdictionary <address>`
Inspect contents of a `ConcurrentDictionary`:
```
> dumpconcurrentdictionary 00007f6a...
```

### `dumpdelegate <address>`
Show what a delegate points to (useful for event handler investigation):
```
> dumpdelegate 00007f6a...
```
