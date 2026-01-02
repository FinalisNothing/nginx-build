# Stealth notes: what we changed and why

## Goals (threat model)

Reduce easy nginx fingerprinting from:
- `Server:` response header (HTTP/1.1, HTTP/2, HTTP/3)
- Version strings displayed by `nginx -V`
- FastCGI template default `SERVER_SOFTWARE` leaking `nginx/$nginx_version`

This does NOT try to hide nginx from advanced active probing or TLS/JARM/HTTP2 behavior fingerprinting.

## Implementation overview

### 1) `Server:` header emission (H1/H2/H3)

We gate the code blocks that emit the Server header using a compile-time macro:

- Define: `-DNGX_STEALTH_NO_SERVER_HEADER`
- Wrap **both** length accounting and output emission blocks in:
  `#if !defined(NGX_STEALTH_NO_SERVER_HEADER) ... #endif`

Files touched:
| File | Protocol |
|------|----------|
| `src/http/ngx_http_header_filter_module.c` | HTTP/1.1 |
| `src/http/v2/ngx_http_v2_filter_module.c` | HTTP/2 |
| `src/http/v3/ngx_http_v3_filter_module.c` | HTTP/3 |

### 2) `nginx -V` / version strings

These macros are made empty in `src/core/nginx.h`:
- `NGINX_VERSION`
- `NGINX_VER`
- `NGINX_VER_BUILD`

This affects display strings and header literals, but does not mutate internal numeric `nginx_version`.

### 3) Error page footer

The hardcoded `nginx` string in error page footer (`server_tokens off` case) is removed:

```c
// Before: "<hr><center>nginx</center>"
// After:  "<hr><center></center>"
```

File: `src/http/ngx_http_special_response.c`
### 4) FastCGI templates

Upstream nginx ships:
- `conf/fastcgi.conf`
- `conf/fastcgi_params`

Default `SERVER_SOFTWARE` includes `nginx/$nginx_version`. We rewrite it to:
- `""` (empty) by default, or
- User provided string from workflow input `fastcgi_server_software`

## What is `stealth.patch`?

`stealth.patch` is a unified diff **generated during CI** by:
1. Editing upstream nginx source files inside the build container
2. Running `diff -u` against the originals
3. Applying the generated patch back with `patch --fuzz=0`

Keeping it provides:
- Auditable record of exact source deltas used for that build
- Maintenance tool: if upstream source changes, `patch --fuzz=0` will fail early
- Fast way to review/compare changes between versions

`stealth.patch` is not required on the VPS at runtime. It is a build-time evidence + reproducibility artifact.

## Verification

### HTTP/1.1

```bash
curl -I http://127.0.0.1/ | grep -i '^server:' && echo BAD || echo OK
```

### HTTP/2 (temporary test instance)

```bash
# Start temp instance
RUN=/tmp/nginx-stealth-h2
sudo mkdir -p "$RUN"/{conf,logs}
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 1 -subj "/CN=localhost" \
  -keyout "$RUN/conf/key.pem" -out "$RUN/conf/cert.pem" 2>/dev/null

sudo tee "$RUN/conf/nginx.conf" >/dev/null <<CONF
worker_processes 1;
error_log $RUN/logs/error.log;
pid $RUN/logs/nginx.pid;
events { worker_connections 1024; }
http {
  server {
    listen 8443 ssl;
    http2 on;
    ssl_certificate $RUN/conf/cert.pem;
    ssl_certificate_key $RUN/conf/key.pem;
    location / { return 200 "ok\n"; }
  }
}
CONF

sudo /nginx/sbin/nginx -c "$RUN/conf/nginx.conf"

# Verify
curl -skI --http2 https://127.0.0.1:8443/ | grep -i '^server:' && echo BAD || echo OK

# Cleanup
sudo kill $(cat "$RUN/logs/nginx.pid") 2>/dev/null
sudo rm -rf "$RUN"
```

### HTTP/3

Only if `nginx -V` includes `--with-http_v3_module` and curl supports `--http3-only`:

```bash
curl -skI --http3-only https://127.0.0.1:8443/ | grep -i '^server:' && echo BAD || echo OK
```

## Known risks / boundaries

- Dynamic modules compiled against non-stealth headers can re-introduce `Server:` behavior
- In-place hot upgrade can mix binaries with different macro behavior if not carefully managed
- HTTP/3 support depends on nginx build flags and crypto stack capabilities
- This does not prevent fingerprinting via timing, TLS fingerprint (JA3/JA4), or HTTP/2 frame behavior
