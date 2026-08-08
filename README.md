# kGate

**kGate** is a self-hosted API Gateway and Messenger, built to be simple to install, run anywhere from a laptop to an air-gapped internal network, and license-managed without phoning home constantly.

- **Gateway** — request routing, service registry, referer-based access control, health checks
- **Messenger** — lightweight WebSocket pub/sub with delivery tracking and webhook fallback for offline subscribers

Both components are optional and selected during installation — run either one, or both, from the same binary.

---

## Quick start

### Docker

```bash
docker pull ghcr.io/klivolks/kgate:latest
docker run -d --name kgate --restart=always \
  -v kgate-data:/app/data \
  -p 8080:8080 \
  ghcr.io/klivolks/kgate:latest
```

Then visit `http://localhost:8080/install` to complete setup.

kGate does not configure a reverse proxy for you — put nginx, Apache, OpenLiteSpeed, Traefik, or your platform's own ingress in front of it for TLS termination and public access.

### Ubuntu / Debian

```bash
curl -fsSL https://klivolks.github.io/kgate-releases/kgate-archive-keyring.gpg.asc | sudo gpg --dearmor -o /usr/share/keyrings/kgate.gpg
echo "deb [signed-by=/usr/share/keyrings/kgate.gpg] https://klivolks.github.io/kgate-releases/apt stable main" | sudo tee /etc/apt/sources.list.d/kgate.list
sudo apt-get update
sudo apt-get install kgate

sudo systemctl enable --now kgate
```

Visit `http://localhost:8080/install` to complete setup. Reverse proxy config examples (nginx, Apache, OpenLiteSpeed) are installed to `/usr/share/doc/kgate/reverse-proxy/`.

### RHEL / CentOS / Fedora

```bash
sudo tee /etc/yum.repos.d/kgate.repo <<EOF
[kgate]
name=kGate
baseurl=https://klivolks.github.io/kgate-releases/yum
enabled=1
gpgcheck=1
gpgkey=https://klivolks.github.io/kgate-releases/kgate-archive-keyring.gpg.asc
EOF

sudo dnf install kgate
sudo systemctl enable --now kgate
```

### Windows

1. Download `kgate-setup.exe` from the [latest release](https://github.com/klivolks/kgate-releases/releases/latest).
2. Run it. You'll be asked for the local port kGate should listen on, and whether to auto-configure an IIS reverse-proxy site.
3. The installer registers kGate as a native Windows service (`kGate`) and starts it.

If you skip IIS auto-configuration, or want to front kGate with a different setup, configure your own reverse proxy pointing at `127.0.0.1:<port>`.

Manage the service from an elevated PowerShell/CMD prompt:
```powershell
sc start kGate
sc stop kGate
sc query kGate
```

To uninstall, run the uninstaller from **Add or Remove Programs** — this stops and removes the Windows service automatically.

---

## First-time setup

Every install, regardless of platform, goes through the same setup wizard at `/install`:

1. **Components** — choose Gateway, Messenger, or both
2. **Database** — MongoDB, MySQL, Microsoft SQL Server, or an internal embedded store (zero external dependencies)
3. **Administrator account** — created only if the chosen database has no existing users
4. **License activation** — start a trial, enter a license key, or (for offline/air-gapped installs) request manual activation

The application restarts itself once setup completes, so kGate must run under a process supervisor with an always-restart policy — this is handled automatically by the systemd unit, Windows service, and `--restart=always` in the Docker example above.

---

## Licensing

kGate is license-gated at runtime, not at install time — the binary and packages are freely downloadable.

- **Trial** — 30 days, no license key required, started directly from the install wizard
- **Licensed** — a signed license token verified against Klivolks' licensing server; verification is cryptographically signed (RSA) so a downloaded license file can't be edited to extend its own validity
- **Offline / air-gapped installs** — the admin panel's **About** page shows this installation's ID; send it to Klivolks to receive a signed license token back, which is applied entirely locally with no network call required

When a license lapses, the admin dashboard and About page remain reachable so you can reactivate — only Gateway/Messenger traffic and the rest of the admin panel are paused until a valid license is restored.

---

## Configuration

kGate reads environment variables (via `.env` in the working directory, or your platform's native environment):

| Variable | Default | Purpose |
|---|---|---|
| `HOST` | `0.0.0.0` | Bind address |
| `PORT` | `8080` | Bind port |
| `MESSENGER_DATA_PATH` | `./data/messenger` | Path for the Messenger component's local storage |

Database connection details, selected components, and license state are configured through the `/install` wizard and stored under `./data/` — back this directory up as part of your normal backup routine.

---

## CLI

The same binary doubles as an admin CLI for recovery scenarios (see [`RECOVERY.md`](./RECOVERY.md) for full detail):

```bash
kgate -list-users
kgate -create-user
kgate -reset-password
```

On Windows, install/uninstall the service directly:
```powershell
kgate.exe install-service
kgate.exe uninstall-service
```

---

## Development

```bash
git clone <this repo>
cd kgate
go mod tidy
go run .
```

Requires Go 1.25+. See [`RECOVERY.md`](./RECOVERY.md) for local `.env` setup notes and debugging tips.

### Building release artifacts locally

Release packaging (binaries, `.deb`/`.rpm`, Docker image, checksums, signing) is handled by [GoReleaser](https://goreleaser.com/) via `.goreleaser.yaml`:

```bash
goreleaser release --snapshot --clean
```

The Windows installer (`packaging/windows/kgate.iss`) requires [Inno Setup](https://jrsoftware.org/isinfo.php) and is built separately from the GoReleaser pass — see `.github/workflows/release.yml` for the full release pipeline.

---

## Architecture at a glance

- **Router**: [gorilla/mux](https://github.com/gorilla/mux)
- **Database**: pluggable via a `Store` interface — MongoDB, MySQL, MSSQL, or an embedded [BadgerDB](https://github.com/dgraph-io/badger)-backed internal store
- **Messenger transport**: WebSocket, with a signed webhook fallback for subscribers that are offline past a configurable read-timeout
- **Auth**: `X-Client-Id` header plus Origin/Referer matching for both gateway API calls and Messenger connections
- **UI**: server-rendered admin panel, self-hosted assets only (no CDN dependencies), so kGate can be installed and administered entirely offline

---

## Support

- Documentation: `/admin/docs` within your running instance
- Issues / questions: [klivolks.com](https://klivolks.com)

## License

kGate is proprietary software. See your license agreement for terms. This repository's source is private; distributable binaries and packages are published to [klivolks/kgate-releases](https://github.com/klivolks/kgate-releases).