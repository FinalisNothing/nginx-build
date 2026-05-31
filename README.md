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
| `cpu_baseline` | Target CPU baseline for `-march` (e.g. `x86-64`, `x86-64-v2`, `x86-64-v3`) | `x86-64-v2` |

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

## Automatic rebuilds

Besides manual runs, the workflow runs on a **weekly schedule** (Mondays 04:00 UTC) so new nginx releases and security fixes are picked up without anyone clicking a button. Each scheduled run:

1. Resolves the latest `mainline` nginx version from nginx.org.
2. Checks whether a GitHub Release already exists for that version.
3. Builds and publishes a Release **only if the version is new** — so weeks with no upstream release are no-ops (no duplicate releases).

Scheduled runs use the defaults (`mainline`, prefix `/nginx`, empty `SERVER_SOFTWARE`, `x86-64-v2`). Manual `workflow_dispatch` runs always build, regardless of existing releases. To also track `stable` automatically, add a second `cron` entry (or a channel matrix) in `.github/workflows/build.yml`.

## Install on Debian 13 VPS

```bash
sudo tar -C / -xzf nginx-root.tar.gz
/nginx/sbin/nginx -V
sudo /nginx/sbin/nginx -t -c /nginx/conf/nginx.conf
```

> **CPU requirement:** the binary is compiled with `-march=x86-64-v2` by default (≈ Nehalem / 2009+ x86-64 with SSE4.2). It will not run on older CPUs or on non-x86-64 architectures (`Illegal instruction`). Build with the `cpu_baseline` input set to `x86-64` for maximum compatibility, or `x86-64-v3` for newer CPUs (AVX2).

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

## License

The build tooling, workflow, and documentation in this repository are released under the [MIT License](LICENSE). nginx itself is distributed under the [2-clause BSD license](https://nginx.org/LICENSE), and the Brotli module ([google/ngx_brotli](https://github.com/google/ngx_brotli)) under its own license — both apply to the binaries produced by this pipeline.
