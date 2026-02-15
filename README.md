# nginx stealth build (Debian 13 / trixie)

This repo builds a custom nginx from upstream source inside a Debian 13 (trixie) container using GitHub Actions, and removes common nginx fingerprint surfaces:

- HTTP/1.1/2/3 `Server:` response header emission (compile-time gated)
- `nginx -V` displayed version strings (`NGINX_VERSION` / `NGINX_VER*` made empty at build time)
- Error page footer hardcoded `nginx` string (removed)
- FastCGI templates no longer leak `nginx/$nginx_version` via `SERVER_SOFTWARE`

## How to build

GitHub Actions -> workflow `build-nginx-stealth` -> Run workflow

Inputs:
| Input | Description | Default |
|-------|-------------|---------|
| `install_prefix` | Install prefix in the tarball | `/nginx` |
| `channel` | `mainline` (odd minor) or `stable` (even minor) | `mainline` |
| `fastcgi_server_software` | Value for `fastcgi_param SERVER_SOFTWARE` | `""` (empty) |

Artifacts produced:
- `nginx-root.tar.gz`: install tree (untar at `/`), includes pre-created `html/errors/` and `acme_challenge/` directories
- `nginx`: standalone binary
- `nginx_build_info.txt`: output of `nginx -V` (version strings are empty due to stealth patch)
- `nginx_version.txt`: actual nginx version number used in the build
- `stealth.patch`: generated unified diff applied to nginx source
- `SHA256SUMS.txt`: checksums for integrity

Each successful build also creates a **GitHub Release** with all artifacts attached. The latest build can be downloaded directly:

```
https://github.com/FinalisNothing/nginx-build/releases/latest/download/nginx-root.tar.gz
```

## Install on Debian 13 VPS

```bash
sudo tar -C / -xzf nginx-root.tar.gz
/nginx/sbin/nginx -V
sudo /nginx/sbin/nginx -t -c /nginx/conf/nginx.conf
```

The tarball includes these pre-created directories:
- `/nginx/html/errors/` — custom error pages (502.html, 504.html); served only via `error_page` internal redirect
- `/nginx/acme_challenge/.well-known/acme-challenge/` — webroot for ACME certificate verification

To add or update error pages, place `.html` files in `html/errors/` in this repo and re-run the workflow.

## systemd service

Create `/etc/systemd/system/nginx.service`:

```ini
[Unit]
Description=nginx
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/nginx/logs/nginx.pid
ExecStartPre=/nginx/sbin/nginx -t -c /nginx/conf/nginx.conf
ExecStart=/nginx/sbin/nginx -c /nginx/conf/nginx.conf
ExecReload=/nginx/sbin/nginx -s reload
ExecStop=/nginx/sbin/nginx -s quit
TimeoutStopSec=5
KillMode=mixed
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

## Verification

```bash
# HTTP/1.1
curl -I http://127.0.0.1/ | grep -i '^server:' && echo BAD || echo OK

# HTTP/2 (requires TLS config)
curl -skI --http2 https://127.0.0.1:443/ | grep -i '^server:' && echo BAD || echo OK
```

See [docs/STEALTH_NOTES.md](docs/STEALTH_NOTES.md) for detailed verification examples.
