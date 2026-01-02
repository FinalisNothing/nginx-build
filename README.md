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
- `nginx-root.tar.gz`: install tree (untar at `/`)
- `nginx`: standalone binary
- `nginx_build_info.txt`: output of `nginx -V`
- `stealth.patch`: generated unified diff applied to nginx source
- `SHA256SUMS.txt`: checksums for integrity

## Install on Debian 13 VPS

```bash
sudo tar -C / -xzf nginx-root.tar.gz
/nginx/sbin/nginx -V
sudo /nginx/sbin/nginx -t -c /nginx/conf/nginx.conf
```

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
