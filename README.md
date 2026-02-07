# pwproxy

DNS + TLS passthrough proxy in Docker. Intercepts configured domains at the DNS level and transparently proxies their TLS traffic through HAProxy without termination. Supports plain DNS, DNS-over-TLS (DoT) and DNS-over-HTTPS (DoH).

## How it works

```
Client ─── DNS (53) / DoT (853) / DoH (8443) ──► CoreDNS
                                                    ├─ intercepted domain → returns host IP
                                                    └─ everything else    → forwards to 1.1.1.1 / 8.8.8.8

Client ─── TLS (443) ──► HAProxy ─── reads SNI ──► forwards TCP stream to real upstream
                          (no TLS termination, L4 proxy)
```

1. CoreDNS resolves intercepted domains to the **host's public IP**.
2. The client connects to the host on port 443, thinking it is the real server.
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
├── certs/
│   ├── cert.pem          # TLS certificate (+ chain)
│   └── key.pem           # Private key
├── coredns/
│   └── Corefile
└── haproxy/
    └── haproxy.cfg
```

## Ports

| Port | Protocol | Service |
|---|---|---|
| 53/udp, 53/tcp | Plain DNS | CoreDNS |
| 853/tcp | DNS-over-TLS | CoreDNS |
| 8443/tcp | DNS-over-HTTPS | CoreDNS |
| 443/tcp | TLS passthrough | HAProxy |

DoH uses port 8443 instead of 443 because HAProxy occupies 443 for TLS passthrough.

## Requirements

- Docker Engine 20.10+
- Docker Compose v2+
- TLS certificate and private key in `certs/`
- Ports 53, 443, 853, 8443 available on the host

## Configuration

### Host IP

Set your server's **public IP** in `coredns/Corefile`. Replace the IP address in the `(intercept)` snippet — it appears in the `hosts` block and in every `template` answer:

```
hosts {
    203.0.113.50 pw.game ns.photonengine.io
    ...
}
```

```
answer "{{ .Name }} 60 IN A 203.0.113.50"
```

### TLS certificates

Place your certificate and key in the `certs/` directory:

```
certs/cert.pem    # Server certificate (include intermediate chain if needed)
certs/key.pem     # Private key
```

These are used by CoreDNS for DoT and DoH. HAProxy does **not** use them — it does not terminate TLS.

You can use a self-signed certificate for testing:

```bash
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
  -days 3650 -nodes -subj "/CN=dns" \
  -keyout certs/key.pem -out certs/cert.pem
```

Or use a real certificate from Let's Encrypt / any CA for production.

## Quick start

```bash
# 1. Place your certs
cp /path/to/cert.pem certs/cert.pem
cp /path/to/key.pem  certs/key.pem

# 2. Start
docker compose up -d

# 3. Verify
docker compose ps
```

## Client setup

On the client machine, configure DNS to point to your server.

**Plain DNS** — set DNS server to your host IP:

```
# Linux: /etc/resolv.conf
nameserver 217.30.10.72
```

**DoT** — configure your client to use:

```
Server: 217.30.10.72
Port:   853
```

**DoH** — configure your client to use:

```
URL: https://217.30.10.72:8443/dns-query
```

## Testing

### DNS — intercepted domain

```bash
dig @<HOST_IP> pw.game A
```

Expected: `pw.game. 60 IN A <HOST_IP>`

### DNS — wildcard domain

```bash
dig @<HOST_IP> ns1.exitgames.com A
```

Expected: `ns1.exitgames.com. 60 IN A <HOST_IP>`

### DNS — non-intercepted domain

```bash
dig @<HOST_IP> google.com A
```

Expected: real Google IP.

### DoT

```bash
kdig @<HOST_IP> +tls pw.game A
```

Or with openssl:

```bash
openssl s_client -connect <HOST_IP>:853 -quiet
```

### DoH

```bash
curl -s "https://<HOST_IP>:8443/dns-query?name=pw.game&type=A" \
  -H "Accept: application/dns-json" --insecure
```

### TLS passthrough

```bash
openssl s_client -connect <HOST_IP>:443 -servername ns.photonengine.io -showcerts
```

You will see the **real upstream certificate** — confirming no TLS termination.

## Adding a new domain

### Exact domain (e.g. `newgame.io`)

**Corefile** — add to the `hosts` block inside `(intercept)`:

```diff
 hosts {
-    217.30.10.72 pw.game ns.photonengine.io
+    217.30.10.72 pw.game ns.photonengine.io newgame.io
     ttl 60
```

**haproxy.cfg** — add an ACL line:

```diff
+acl is_intercepted req_ssl_sni -i newgame.io
```

### Wildcard domain (e.g. `*.newservice.com`)

**Corefile** — add templates inside `(intercept)`, before `forward`:

```
template IN A newservice.com {
    match "^.+\.newservice\.com\.$"
    answer "{{ .Name }} 60 IN A 217.30.10.72"
    fallthrough
}
template IN AAAA newservice.com {
    match "^.+\.newservice\.com\.$"
    rcode NOERROR
    fallthrough
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

- **HAProxy uses public DNS** (`1.1.1.1`, `8.8.8.8`) to avoid resolution loops through CoreDNS.
- **`do-resolve` + `set-dst`** (HAProxy 2.6+) resolves the SNI hostname at runtime — enables wildcard support with a single backend.
- **AAAA blocking** via `template IN AAAA` with `rcode NOERROR` prevents IPv6 bypass.
- **CoreDNS `import` snippets** keep the Corefile DRY — domain logic is defined once and shared across all three protocols.

## License

MIT
