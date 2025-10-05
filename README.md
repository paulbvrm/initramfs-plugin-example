<p align="center">
  <img src="https://raw.githubusercontent.com/volantvm/volant/main/banner.png" alt="VOLANT — The Intelligent Execution Cloud"/>
</p>

<p align="center">
  <a href="https://github.com/volantvm/initramfs-plugin-example/actions"><img src="https://img.shields.io/github/actions/workflow/status/volantvm/initramfs-plugin-example/build-plugin.yml?branch=main&style=flat-square&label=build"></a>
  <a href="https://github.com/volantvm/initramfs-plugin-example/releases"><img src="https://img.shields.io/github/v/release/volantvm/initramfs-plugin-example.svg?style=flat-square"></a>
  <img src="https://img.shields.io/badge/License-Apache_2.0-black.svg?style=flat-square">
</p>

---

# Initramfs Plugin Example

**Reference implementation for Volant initramfs plugins**

This is a working Caddy web server packaged as an initramfs plugin—boots in <100ms, runs in ~20MB.

Use it as a template for your own stateless services, API servers, or edge workloads.

---

## Quick Start

```bash
# Install the plugin
volar plugins install --manifest https://github.com/volantvm/initramfs-plugin-example/releases/latest/download/caddy.json

# Run it
volar vms create web --plugin caddy --cpu 1 --memory 512

# Test it
curl http://192.168.127.10
# → Hello from Caddy in a Volant microVM!
```

Boots in under 100ms. Runs in RAM. Zero persistence.

---

## What's Inside

| File | Purpose |
|------|---------|
| `fledge.toml` | Build configuration |
| `manifest/caddy.json` | Plugin manifest (install target) |
| `payload/caddy` | Static Caddy binary (downloaded in CI) |
| `payload/Caddyfile` | Web server config |
| `.github/workflows/` | Reproducible build pipeline |

**Build output**: `plugin.cpio.gz` (~20 MB compressed initramfs)

---

## Build It Yourself

### Prerequisites

```bash
# Install fledge
curl -LO https://github.com/volantvm/fledge/releases/latest/download/fledge-linux-amd64
chmod +x fledge-linux-amd64 && sudo mv fledge-linux-amd64 /usr/local/bin/fledge
```

### Build Steps

```bash
# Clone
git clone https://github.com/volantvm/initramfs-plugin-example
cd initramfs-plugin-example

# Download Caddy (static binary)
cd payload
CADDY_VERSION="2.10.2"
curl -fsSL "https://github.com/caddyserver/caddy/releases/download/v${CADDY_VERSION}/caddy_${CADDY_VERSION}_linux_amd64.tar.gz" | tar -xz caddy
chmod +x caddy
cd ..

# Build the initramfs
sudo fledge build

# Get checksum
sha256sum plugin.cpio.gz

# Update manifest with local path for testing
# (Edit manifest/caddy.json: set "url" to full path of plugin.cpio.gz)

# Install locally
volar plugins install --manifest manifest/caddy.json

# Test
volar vms create test --plugin caddy
curl http://192.168.127.10
```

---

## fledge.toml Breakdown

```toml
version = "1"
strategy = "initramfs"                    # RAM-based, fast boot

[agent]
source_strategy = "release"               # Download kestrel from GitHub
version = "latest"

[source]
busybox_url = "..."                       # Provides /bin/sh, ps, etc.
busybox_sha256 = "..."                    # Checksum verification

[mappings]
"payload/caddy" = "/usr/bin/caddy"        # Your app binary (755)
"payload/Caddyfile" = "/etc/caddy/Caddyfile"  # Config (644)
```

**How it works**:
1. Fledge downloads busybox + kestrel
2. Creates FHS structure (`/bin`, `/usr`, `/etc`)
3. Maps your files into the filesystem
4. Packages as CPIO + gzip
5. Generates manifest with checksum

---

## manifest.json Breakdown

```json
{
  "name": "caddy",
  "version": "0.1.0",
  "runtime": "caddy",

  "initramfs": {
    "url": "https://github.com/.../plugin.cpio.gz",  // Download URL
    "checksum": "sha256:..."                          // Verify integrity
  },

  "workload": {
    "type": "http",
    "entrypoint": ["/usr/bin/caddy", "run", "--config", "/etc/caddy/Caddyfile"],
    "base_url": "http://127.0.0.1:80"
  },

  "health_check": {
    "endpoint": "/",      // Polls this until 200 OK
    "timeout_ms": 10000
  }
}
```

**What volantd does**:
1. Downloads `plugin.cpio.gz` from `url`
2. Verifies `checksum`
3. Passes it to Cloud Hypervisor as `--initramfs`
4. Kernel unpacks it into RAM
5. Kestrel runs `entrypoint`
6. Polls `health_check.endpoint` until ready

---

## Customize for Your App

### 1. Replace the Binary

```bash
# Build your app (must be static)
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o myapp

# Update fledge.toml
[mappings]
"myapp" = "/usr/bin/myapp"
"config.yaml" = "/etc/myapp/config.yaml"
```

### 2. Update Manifest

```json
{
  "name": "myapp",
  "runtime": "myapp",
  "workload": {
    "entrypoint": ["/usr/bin/myapp", "--config", "/etc/myapp/config.yaml"],
    "base_url": "http://127.0.0.1:8080"
  }
}
```

### 3. Rebuild

```bash
sudo fledge build
volar plugins install --manifest manifest/myapp.json
volar vms create test --plugin myapp
```

---

## GitHub Actions CI/CD

`.github/workflows/build-plugin.yml`:

```yaml
- Download Caddy from official releases
- Verify SHA256 checksum
- Build plugin with fledge
- Calculate artifact checksum
- Create GitHub release with:
  - plugin.cpio.gz
  - manifest.json (updated with new checksum)
```

**Push a tag** → automatic release:

```bash
git tag v0.1.0
git push origin v0.1.0
```

Users install via manifest URL:

```bash
volar plugins install --manifest https://github.com/you/your-plugin/releases/download/v0.1.0/manifest.json
```

---

## Tips

| Tip | Why |
|-----|-----|
| **Static binaries only** | No dynamic linking in initramfs |
| **Keep it small** | Every MB affects boot time |
| **Verify checksums** | Security + reproducibility |
| **Test locally first** | Don't push broken builds |
| **Version your releases** | Users can pin versions |

### Verify Static Linking

```bash
ldd payload/caddy
# Should say: "not a dynamic executable"

file payload/caddy
# Should include: "statically linked"
```

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| VM won't boot | Check volantd logs: `journalctl -u volantd -f` |
| App crashes | Check VM logs: `volar vms logs <name>` |
| Health check fails | Verify entrypoint is correct + app binds to 0.0.0.0 |
| Build too large | Strip binary: `strip -s myapp` |
| Checksum mismatch | Recalculate: `sha256sum plugin.cpio.gz` |

### Debug Inside VM

```bash
volar vms console my-caddy

# Inside VM (minimal busybox):
ls /proc                  # See running processes
cat /proc/cmdline         # See manifest passed to VM
ls -la /usr/bin/caddy     # Check permissions
# Note: No curl, wget, or ps in minimal initramfs
# Test from host: curl http://192.168.127.10
```

---

## Why Initramfs?

| Metric | Initramfs (this) | OCI Rootfs |
|--------|------------------|------------|
| **Boot time** | 50-150ms | 2-5s |
| **Memory** | 15-20 MB | 50-80 MB |
| **Size** | 20 MB | 200+ MB |
| **Persistence** | None (RAM) | Disk-backed |
| **Best for** | Stateless services | Stateful apps |

Use initramfs when:
- Speed matters (edge computing, serverless)
- You don't need persistence
- You want minimal attack surface
- Your app is a single static binary

Use OCI rootfs when:
- You have an existing Docker image
- You need complex dependencies
- You want full filesystem access

---

## Further Reading

- [Volant Plugin Development](https://github.com/volantvm/volant/blob/main/docs/4_plugin-development/2_initramfs.md)
- [Fledge Build Tool](https://github.com/volantvm/fledge)
- [OCI Plugin Example](https://github.com/volantvm/oci-plugin-example)

---

## License

**Apache License 2.0**

Free to use, modify, and distribute. See [LICENSE](LICENSE) for details.

---

**© 2025 HYPR PTE. LTD.**
