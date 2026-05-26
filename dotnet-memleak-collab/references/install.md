# Installing dotnet-dump — All Platforms

## With .NET SDK (easiest)

```bash
dotnet tool install --global dotnet-dump
# Verify
dotnet-dump --version
```

If the tool is already installed but outdated:
```bash
dotnet tool update --global dotnet-dump
```

---

## Without .NET SDK — Direct Binary Download

No SDK, no package manager needed. Download and run directly.

### Linux x64
```bash
curl -Lo dotnet-dump https://aka.ms/dotnet-dump/linux-x64
chmod +x dotnet-dump
./dotnet-dump --version
```

### Linux ARM64
```bash
curl -Lo dotnet-dump https://aka.ms/dotnet-dump/linux-arm64
chmod +x dotnet-dump
```

### Linux ARM (32-bit)
```bash
curl -Lo dotnet-dump https://aka.ms/dotnet-dump/linux-arm
chmod +x dotnet-dump
```

### Linux musl x64 (Alpine)
```bash
curl -Lo dotnet-dump https://aka.ms/dotnet-dump/linux-musl-x64
chmod +x dotnet-dump
```

### Linux musl ARM64 (Alpine ARM)
```bash
curl -Lo dotnet-dump https://aka.ms/dotnet-dump/linux-musl-arm64
chmod +x dotnet-dump
```

### Windows x64
Download from: https://aka.ms/dotnet-dump/win-x64
Run as: `dotnet-dump.exe`

### Windows x86 (32-bit app)
Download from: https://aka.ms/dotnet-dump/win-x86

> **Important**: To analyze an x86 app, you must use the x86 version of dotnet-dump.
> The x64 version cannot analyze x86 process dumps.

### Windows ARM64
Download from: https://aka.ms/dotnet-dump/win-arm64

---

## After download — verify it works

```bash
./dotnet-dump ps
```

Should list running dotnet processes. If it shows an empty list but dotnet apps are
running, you may need to run as the same user as the target process (or as root).

---

## Add to PATH (optional, for convenience)

```bash
# Linux/macOS — add to ~/.bashrc or ~/.zshrc
export PATH="$PATH:/path/to/directory/containing/dotnet-dump"

# Or move to a directory already on PATH
sudo mv dotnet-dump /usr/local/bin/
```
