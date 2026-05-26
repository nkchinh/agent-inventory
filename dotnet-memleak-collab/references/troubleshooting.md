# dotnet-dump Troubleshooting Guide

## SOS / DAC Errors

### "Failed to load data access DLL, 0x80004005"

The SOS extension cannot find the correct `mscordaccore.dll` (DAC) for the runtime version
in the dump.

**Fix**:
```
> setsymbolserver -ms
```
This points SOS at Microsoft's public symbol server to download the correct DAC.

Or manually:
```
> setclrpath /path/to/shared/dotnet/runtime/<version>
```

### "SOS does not support the current target architecture"

Dump was collected on x64 but analyzed on x86 (or vice versa). Use the matching
architecture of `dotnet-dump`.

### "No managed threads" or "Unable to find runtime"

The dump may be a native dump without managed state, or collected from a non-.NET process.
Verify the dump came from a .NET process:
```bash
file leak.dmp
strings leak.dmp | grep -i "microsoft.netcore"
```

---

## Collection Errors

### "Failed to establish connection with target process"

- **Cause 1**: Wrong user. Run as the same OS user as the dotnet process, or as root.
- **Cause 2**: ptrace blocked (Docker/K8s). See `docker.md` for capability grants.
- **Cause 3**: Process exited between `ps` and `collect`.

### "Timed out waiting for dump"

- On Linux/macOS: `TMPDIR` mismatch between agent and target process. Export the same
  `TMPDIR` before running dotnet-dump.
- Target process may be frozen/deadlocked — try with `--type Mini` first.

### "Access denied / permission denied on output path"

The dotnet process user does not have write permission to the output directory.

```bash
dotnet-dump collect -p <PID> -o /tmp/dump.dmp
```
`/tmp` is almost always writable. Move the file afterwards.

### OOM / Container killed during collection

The OS paged in too much memory. Options:
1. Increase container memory limit temporarily
2. Use `--type Mini` instead of `Heap`
3. Enable swap on the host

---

## Analysis Errors

### `gcroot` takes very long or hangs

Normal for large heaps (>4 GB). `gcroot` traverses the entire object graph. Options:
- Wait — it will complete (may take minutes on large heaps)
- Use `-nofields` flag to speed up: `gcroot -nofields <addr>`
- Sample: only run gcroot on 2-3 representative addresses of the leaking type

### `dumpheap` shows many `<error>` type names

Symbol resolution failed for some types. Run:
```
setsymbolserver -ms
```
Then retry. Private types will still show as `<error>` if PDB is not on the symbol server —
this is expected for internal app code in some configurations.

### `clrstack` shows no managed frames

The thread may be in native code. Use `eestack` to see all threads including native frames:
```
eestack
```

### "Object not found at address"

Address is stale (dump collected at a different GC cycle, or address reused). Re-run
`dumpheap -type <TypeName>` to get fresh addresses from the current dump session.

---

## Symbol Issues

For best results with `dumpobj` and `clrstack`:

```
setsymbolserver -ms
```

For private symbols (your own code):
```
setsymbolserver -directory /path/to/symbols
```

Or if symbols are on a private symbol server:
```
setsymbolserver -server http://your-symbol-server
```

---

## macOS Limitations

- `dotnet-dump` on macOS requires .NET 5 or later
- macOS System Integrity Protection (SIP) may block ptrace even with root
- Temporarily disable SIP in Recovery Mode if ptrace is blocked (not recommended for
  production machines)

---

## x86 Applications

`dotnet-dump` for x86 apps requires the x86 version of the tool. Download directly:
```
https://aka.ms/dotnet-dump/win-x86
```

The global tool (`dotnet tool install`) installs the architecture matching the SDK, not the
target app. If your app is x86 but SDK is x64, use the direct download.
