# Collecting dotnet-dump in Docker / Kubernetes

## Why containers are different

`dotnet-dump collect` uses `ptrace` syscall to attach to the target process. Containers
restrict this by default for security reasons.

---

## Docker — Grant ptrace capability

```bash
# When starting the container
docker run --cap-add=SYS_PTRACE <image>

# Or for a running container via docker update (limited support)
# Better: restart with the flag
```

If `--cap-add=SYS_PTRACE` is not possible, try:
```bash
docker run --security-opt seccomp=unconfined <image>
```

---

## Kubernetes — Allow ptrace in Pod spec

```yaml
securityContext:
  capabilities:
    add:
    - SYS_PTRACE
```

Or disable seccomp entirely (less secure, for debugging only):
```yaml
securityContext:
  seccompProfile:
    type: Unconfined
```

---

## Installing dotnet-dump inside a container

### Option A — Install at runtime (debug session)

```bash
# Exec into container
kubectl exec -it <pod> -- /bin/bash
# or
docker exec -it <container> /bin/bash

# Install tool
dotnet tool install --global dotnet-dump
export PATH="$PATH:/root/.dotnet/tools"

# Collect
dotnet-dump ps
dotnet-dump collect -p <PID> --type Heap -o /tmp/leak.dmp
```

### Option B — Multi-stage Dockerfile (pre-installed)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS diagnostics
RUN dotnet tool install --global dotnet-dump
RUN dotnet tool install --global dotnet-counters

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
COPY --from=diagnostics /root/.dotnet/tools /root/.dotnet/tools
ENV PATH="$PATH:/root/.dotnet/tools"
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Option C — Direct binary download (no SDK)

```bash
# x64 Linux
curl -Lo /usr/local/bin/dotnet-dump https://aka.ms/dotnet-dump/linux-x64
chmod +x /usr/local/bin/dotnet-dump
```

---

## Exfiltrating the dump file

```bash
# From Kubernetes pod to local machine
kubectl cp <pod>:/tmp/leak.dmp ./leak.dmp

# From Docker container
docker cp <container>:/tmp/leak.dmp ./leak.dmp
```

---

## TMPDIR requirement (Linux/macOS)

`dotnet-dump` and the target process must share the same `TMPDIR`. If they differ, the
collect command will time out.

```bash
# Check TMPDIR of running process
cat /proc/<PID>/environ | tr '\0' '\n' | grep TMPDIR

# Set before running dotnet-dump
TMPDIR=/proc/<PID>/environ_value dotnet-dump collect -p <PID> ...
```

---

## Memory limit warnings

Collecting a Heap or Full dump may temporarily double the process's virtual memory footprint
as the OS pages in data. For containers:

1. Temporarily increase memory limit before collecting
2. Use `--type Mini` if a Heap dump cannot be safely collected
3. Collect during off-peak hours

---

## Permission requirements

`dotnet-dump` must run as the same user as the target process, or as root.

```bash
# Check target process user
ps aux | grep dotnet

# Run as same user
sudo -u <username> dotnet-dump collect -p <PID> -o /tmp/dump.dmp

# Or as root
sudo dotnet-dump collect -p <PID> -o /tmp/dump.dmp
```
