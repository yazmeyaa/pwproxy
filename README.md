# pwproxy

DNS + TLS passthrough proxy in Docker. Intercepts configured domains at the DNS level and transparently proxies their TLS traffic through HAProxy without termination.

## How it works

```
Client ─── DNS query (53) ──► CoreDNS
                                 ├─ intercepted domain → returns HAProxy IP (172.20.0.10)
                                 └─ everything else    → forwards to 1.1.1.1 / 8.8.8.8

Client ─── TLS (443) ──► HAProxy ─── reads SNI ──► forwards TCP stream to real upstream
                          (no TLS termination, L4 proxy)
```

1. CoreDNS resolves intercepted domains to the HAProxy container IP.
2. The client connects to HAProxy on port 443, thinking it is the real server.
3. HAProxy inspects the TLS ClientHello, extracts the SNI hostname, resolves it to the real server IP via public DNS (`do-resolve`), and proxies the raw TCP stream — no decryption, no modification.

## Intercepted domains

| Pattern | Type |
|---|---|
| `pw.game` | exact |
| `ns.photonengine.io` | exact |
| `*.exitgames.com` | wildcard |
| `notpix-3d-content.*.digitaloceanspaces.com` | wildcard |
| `not-platform.*.cdn.digitaloceanspaces.com` | wildcard |

## Project structure

```
├── docker-compose.yml
├── coredns/
│   └── Corefile
└── haproxy/
    └── haproxy.cfg
```

## Requirements

- Docker Engine 20.10+
- Docker Compose v2+
- Ports 53 (TCP/UDP) and 443 (TCP) available on the host

## Quick start

```bash
docker compose up -d
```

Verify both containers are running:

```bash
docker compose ps
```

## Testing

### DNS resolution — intercepted domain

```bash
dig @127.0.0.1 pw.game A
```

Expected: `pw.game. 60 IN A 172.20.0.10`

### DNS resolution — wildcard domain

```bash
dig @127.0.0.1 ns1.exitgames.com A
```

Expected: `ns1.exitgames.com. 60 IN A 172.20.0.10`

### DNS resolution — non-intercepted domain

```bash
dig @127.0.0.1 google.com A
```

Expected: real Google IP address.

### TLS passthrough

```bash
openssl s_client -connect 127.0.0.1:443 -servername ns.photonengine.io -showcerts
```

If the upstream server has TLS, you will see its **real certificate** — confirming HAProxy proxies without termination.

### Rejected SNI

```bash
openssl s_client -connect 127.0.0.1:443 -servername unknown.example.com
```

Connection will be refused (no matching backend).

## Adding a new domain

### Exact domain (e.g. `newgame.io`)

**Corefile** — add to the exact domains block:

```diff
-pw.game ns.photonengine.io {
+pw.game ns.photonengine.io newgame.io {
     log
     hosts {
-        172.20.0.10 pw.game ns.photonengine.io
+        172.20.0.10 pw.game ns.photonengine.io newgame.io
         ttl 60
```

**haproxy.cfg** — add an ACL line:

```diff
 acl is_intercepted req_ssl_sni -i ns.photonengine.io
+acl is_intercepted req_ssl_sni -i newgame.io
```

### Wildcard domain (e.g. `*.newservice.com`)

**Corefile** — add a new zone block:

```
newservice.com {
    log
    template IN A newservice.com {
        match "^.+\.newservice\.com\.$"
        answer "{{ .Name }} 60 IN A 172.20.0.10"
        fallthrough
    }
    template IN AAAA newservice.com {
        match "^.+\.newservice\.com\.$"
        rcode NOERROR
        fallthrough
    }
    forward . 1.1.1.1 8.8.8.8
}
```

**haproxy.cfg** — add an ACL line:

```diff
+acl is_intercepted req_ssl_sni -m end -i .newservice.com
```

### Apply changes

```bash
docker compose restart
```

## Architecture notes

- **Fixed subnet** `172.20.0.0/24` with static IPs ensures CoreDNS can return a stable IP for HAProxy.
- **HAProxy uses public DNS** (`1.1.1.1`, `8.8.8.8`) both at the system level (`dns:` in compose) and via the `resolvers` section to avoid resolution loops through CoreDNS.
- **`do-resolve` + `set-dst`** (HAProxy 2.6+) dynamically resolves the SNI hostname and sets the upstream destination at runtime — this enables wildcard support without per-domain backends.
- **AAAA blocking** via `template IN AAAA` with `rcode NOERROR` prevents IPv6 bypass for wildcard zones.
- **`server upstream 0.0.0.0:0`** in `bk_passthrough` is a placeholder — the actual address is set by `set-dst` in the frontend.

## License

MIT
