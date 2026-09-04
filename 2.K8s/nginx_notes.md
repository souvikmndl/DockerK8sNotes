# Nginx Notes

Reference notes built from an annotated `nginx.conf`. Focus is on **what each directive does, why it exists, and how to tune it** — not just syntax.

> SSL/TLS section is stubbed at the bottom and will be filled in later.

---

## 1. Mental model: how Nginx handles connections

Nginx does **not** spawn a new process or thread per connection (the classic "thread-per-connection" model used by older servers like Apache prefork). That model breaks down under high concurrency because each thread costs memory and context-switching, so a few thousand idle-but-open connections can exhaust the box (the classic **C10k problem**).

Instead, Nginx uses a small number of **worker processes**, each running a **single-threaded, non-blocking event loop**. One worker juggles thousands of connections at once by reacting to I/O readiness events (via `epoll` on Linux, `kqueue` on BSD/macOS) rather than blocking on any single connection.

The practical consequence: throughput scales with **CPU cores**, not with the number of open connections. Memory per idle connection is tiny, so keeping connections open (keep-alive, slow clients, proxied backends) is cheap.

```
                 ┌─────────────── nginx ───────────────┐
   clients  ───► │  worker 1 (event loop) ── conns...   │ ───► upstream servers
   (downstream)  │  worker 2 (event loop) ── conns...   │      (upstream)
                 │  worker N (event loop) ── conns...   │
                 └──────────────────────────────────────┘
```

---

## 2. Configuration structure: contexts

Nginx config is a tree of **contexts** (blocks). A context groups the directives that apply to one kind of traffic or scope. Directives are only valid inside the contexts that support them.

| Context   | Scope / purpose                                             |
|-----------|-------------------------------------------------------------|
| **main**  | Top level (no enclosing block). Global process settings.    |
| **events**| General connection-processing settings.                     |
| **http**  | Everything about HTTP/HTTPS traffic.                        |
| **mail**  | Mail proxy traffic (SMTP/IMAP/POP3).                        |
| **stream**| Raw TCP and UDP traffic (L4 proxying / load balancing).    |

- Directives written **outside** any block are in the **main context**.
- Contexts nest: `server` lives inside `http`, `location` lives inside `server`, etc.
- Child contexts **inherit** directives from their parent and can override them.

```
main
├── worker_processes        (main context)
├── events { }
│   └── worker_connections
└── http { }
    ├── include mime.types
    ├── upstream { }
    └── server { }
        └── location { }
```

---

## 3. Main context directives

### `worker_processes`
Controls **how many worker processes** Nginx spawns to handle client requests.

- Each worker is independent and handles its own set of connections via its own event loop.
- This is the main knob for how much parallel work Nginx can do — it should be tuned to the server's **CPU cores** and expected load.
- Set it to the number of physical CPU cores, **or** use `auto` to let Nginx detect the core count automatically.

```nginx
worker_processes auto;   # recommended default
# worker_processes 1;    # explicit single worker (fine for dev / low traffic)
```

> Rule of thumb: `auto` is almost always the right answer in production — it matches workers to cores without hardcoding a number that breaks when the box is resized.

---

## 4. `events` context

### `worker_connections`
The **maximum number of simultaneous connections a single worker can handle**.

Total theoretical connection capacity:

```
max connections = worker_processes × worker_connections
```

Example from the config: `worker_processes 1` and `worker_connections 1024` → 1024 simultaneous connections. Bump processes to 2 and you get `1024 × 2 = 2048`.

**Caveats worth remembering:**
- More connections = more memory. There's a balance point per box; don't just crank it to a huge number.
- The OS `ulimit -n` (open file descriptors) must be **at least** `worker_connections`, or you'll hit "too many open files" before reaching the configured limit. Sockets are file descriptors.
- **As a reverse proxy, each client request can consume *two* connections** — one from client→nginx and one from nginx→upstream. So effective client capacity is roughly *half* the raw number. This is easy to forget when sizing.

```nginx
events {
    worker_connections 1024;
}
```

---

## 5. `http` context

Wraps all HTTP handling. Everything below lives inside `http { }`.

### `include mime.types`
`include` pulls in another config file inline. `mime.types` is a file Nginx ships with that **maps file extensions to MIME types** (e.g. `.css` → `text/css`, `.json` → `application/json`).

Nginx uses this map to set the **`Content-Type`** response header so browsers know how to interpret each file it serves. Without it, most static files fall back to a generic type and browsers may mishandle them.

```nginx
http {
    include mime.types;
    ...
}
```

### `upstream` block — defining a pool of backend servers
An `upstream` block names a **group of backend servers** that Nginx can forward requests to. You reference the group by name in `proxy_pass`.

```nginx
upstream nodejs_cluster {
    least_conn;              # load-balancing algorithm (round robin is the default)
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
```

This is what makes Nginx a **load balancer**: it distributes incoming requests across the listed backends.

#### Load-balancing algorithms
| Directive        | Behaviour |
|------------------|-----------|
| *(none)* → **round robin** | Default. Requests handed out in rotation, one per server. |
| **`least_conn`** | Sends each request to the backend with the **fewest active connections**. Better when request durations vary a lot (some slow, some fast). |
| **`ip_hash`**    | Routes a given client IP to the **same backend** every time. Cheap "sticky sessions" without shared session storage. |
| **`hash <key>`** | Routes based on an arbitrary key (e.g. `$request_uri`), with optional `consistent` for consistent hashing. |
| **`random`**     | Picks a server at random; `random two` picks two and applies `least_conn` between them. |

Per-server tuning knobs (added after the address):
- `weight=N` — send proportionally more traffic to bigger boxes.
- `max_fails=N` / `fail_timeout=Ns` — mark a server unhealthy after N failures for a window.
- `backup` — only used when all primary servers are down.
- `down` — manually take a server out of rotation.

### `server` block — a virtual host
Defines how Nginx handles requests for a particular **domain or IP**:
- how/where to **listen** for connections,
- **which** domain/subdomain the block applies to,
- how to **route** matching requests.

```nginx
server {
    listen 8080;             # port (and optionally IP/protocol) to accept connections on
    server_name localhost;   # which host/domain this block answers for

    location / {
        proxy_pass http://nodejs_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- **`listen`** — the port (and optionally address/protocol) this virtual host binds to.
- **`server_name`** — which `Host` this block responds to. Nginx uses it to pick the right `server` block when several share a port.

### `location` block — routing within a host
Decides how **specific request paths / file types** are handled. `location /` matches **everything** unless a more specific `location` matches first.

- **`proxy_pass`** — hands the request off to another server (here, the `nodejs_cluster` upstream). This is what turns Nginx into a **reverse proxy**.
- **`proxy_set_header`** — rewrites/sets headers on the request before it goes to the backend (see §7).

#### Location matching priority (quick reference)
When multiple `location` blocks could match, Nginx picks in this order:

1. `location = /path` — **exact match** (highest priority, stops searching).
2. `location ^~ /path` — prefix match that **skips regex** if it's the longest prefix.
3. `location ~ regex` / `~* regex` — **regex** match (`~` case-sensitive, `~*` case-insensitive), first match in file order wins.
4. `location /path` — plain **prefix** match; the **longest** matching prefix is used.

So `/` is the ultimate fallback: it's the shortest possible prefix, so anything more specific beats it.

---

## 6. Reverse proxy vs. forward proxy (terminology)

- **Reverse proxy** (what this config is): sits **in front of the backend servers**. Clients think they're talking to the app; really they hit Nginx, which forwards to the backends. Used for load balancing, TLS termination, caching, hiding internal topology.
- **Upstream** = the servers Nginx forwards **to** (the app servers). Traffic flowing **from the client toward the origin/backend** is going "upstream."
- **Downstream** = back toward the **client**. Responses travel downstream.

> Mnemonic: relative to Nginx, the **client is downstream**, the **backend is upstream**. Requests flow up, responses flow down.

---

## 7. Forwarding client info with `proxy_set_header`

**The problem:** when Nginx reverse-proxies a request, the connection to the backend originates **from Nginx**, not the client. So without intervention, the backend sees **Nginx's IP** as the source and loses the original client's details (real IP, protocol, host, etc.).

**The fix:** Nginx copies the real client information into headers before forwarding. `$var_name` values are Nginx variables holding the client's original values.

### Headers used in this config
```nginx
proxy_set_header Host      $host;         # preserve the original requested hostname
proxy_set_header X-Real-IP $remote_addr;  # the client's real IP address
```

### Fuller set of commonly-forwarded headers
```nginx
location / {
    # --- client identity / network ---
    proxy_set_header Host              $host;                          # original Host header
    proxy_set_header X-Real-IP         $remote_addr;                   # client IP
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;     # appends client IP to existing chain
    proxy_set_header X-Forwarded-Proto $scheme;                        # http or https (key for TLS termination)
    proxy_set_header X-Forwarded-Port  $server_port;                   # port the client hit

    # --- browser / session ---
    proxy_set_header User-Agent        $http_user_agent;
    proxy_set_header Cookie            $http_cookie;
    proxy_set_header Accept-Language   $http_accept_language;
    proxy_set_header Referer           $http_referer;

    # --- auth ---
    proxy_set_header Authorization     $http_authorization;

    # --- custom ---
    proxy_set_header X-Custom-Header   "MyAppSpecificValue";
}
```

**Why the key ones matter:**
- `X-Forwarded-For` uses `$proxy_add_x_forwarded_for`, which **appends** the current client IP to any existing `X-Forwarded-For` value → you get the full chain of proxies, not just the last hop. Use `$remote_addr` (X-Real-IP) when you want *only* the immediate client.
- `X-Forwarded-Proto $scheme` tells the backend whether the **original** request was HTTP or HTTPS. Critical once TLS is terminated at Nginx — the backend speaks plain HTTP to Nginx but needs to know the client used HTTPS (e.g. to build correct redirect URLs / set `Secure` cookies).
- Any `$http_<name>` variable exposes an incoming request header (`$http_user_agent` = the `User-Agent` header, etc.).

> Security note: the backend must **trust** these headers only when the request genuinely came from your Nginx. `X-Forwarded-*` headers are client-spoofable if the app is reachable directly, so lock the backend to only accept them from the proxy's IP.

---

## 8. Annotated full config (as-is)

```nginx
worker_processes 1;              # main context: number of workers

events {
    worker_connections 1024;     # max simultaneous connections per worker
}

http {
    include mime.types;          # map file extensions → Content-Type

    upstream nodejs_cluster {     # backend pool
        least_conn;               # send to the least-busy backend
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
        server 127.0.0.1:3003;
    }

    server {
        listen 8080;              # accept on port 8080
        server_name localhost;    # respond for host "localhost"

        location / {              # match all paths
            proxy_pass http://nodejs_cluster;      # reverse-proxy to the pool
            proxy_set_header Host $host;            # preserve original host
            proxy_set_header X-Real-IP $remote_addr;# forward client IP
        }
    }
}
```

**Request flow for this config:**
```
client ─(GET / on :8080)─► nginx ─(least_conn pick)─► one of 127.0.0.1:3001/3002/3003
        ◄───────── response ─────────◄──────────────────────────────
```

---

## 9. Quick reference / cheat sheet

| Directive | Context | Purpose | Tune to |
|-----------|---------|---------|---------|
| `worker_processes` | main | # of worker processes | CPU cores / `auto` |
| `worker_connections` | events | conns per worker | RAM + `ulimit -n` |
| `include` | any | inline another config file | — |
| `upstream` | http | define backend pool | # of backends |
| `least_conn` / `ip_hash` / … | upstream | LB algorithm | traffic pattern |
| `listen` | server | port/address to bind | — |
| `server_name` | server | which host to answer | your domain |
| `location` | server | route by path/pattern | request shape |
| `proxy_pass` | location | forward to backend | upstream name |
| `proxy_set_header` | location | rewrite outgoing headers | client info to preserve |

---

## 10. SSL / TLS

This section is built from the HTTPS version of the config — port 443 with a self-signed cert, plus a second server block that redirects plain HTTP to HTTPS.

### 10.1 TLS termination: where encryption stops

In this config Nginx does **TLS termination**: the encrypted TLS connection ends **at Nginx**. Nginx decrypts the request, then talks to the Node backends over **plain HTTP** (`proxy_pass http://nodejs_cluster`).

```
 client ══(HTTPS / TLS on :443)══► nginx ──(plain HTTP)──► 127.0.0.1:3001/3002/3003
        encrypted, public network         decrypted, trusted loopback/internal net
```

Three ways to handle TLS at a reverse proxy — this config uses the first:

| Mode | Client↔Nginx | Nginx↔Backend | When |
|------|--------------|---------------|------|
| **TLS termination** *(this config)* | HTTPS | HTTP | Backends on a trusted/loopback network. Simplest; one place to manage certs. |
| **TLS passthrough** | HTTPS | HTTPS (Nginx doesn't decrypt) | Backend must see the raw TLS / do its own cert auth. Uses the `stream` module, not `http`. |
| **Re-encryption (end-to-end)** | HTTPS | HTTPS (Nginx decrypts then re-encrypts) | Backend hop crosses an untrusted network but you still want L7 features. |

Because Nginx terminates TLS here, the backend has **no idea the client used HTTPS** unless you tell it — that's exactly why `X-Forwarded-Proto $scheme` (§7) matters: it carries `https` down to the app so it can build correct redirect URLs and set `Secure` cookies.

### 10.2 Listening for HTTPS

```nginx
listen 443 ssl;   # 443 is the default HTTPS port; the `ssl` param enables TLS on it
```

- `443` is the well-known HTTPS port (like `80` is for HTTP) — browsers assume it for `https://` URLs, so it doesn't need to appear in the address bar.
- The **`ssl` parameter** is what actually turns TLS on for this listener. Without it, Nginx would expect plain HTTP on 443.
- Common addition (not in this config): `listen 443 ssl http2;` to enable HTTP/2, which needs TLS in practice and gives multiplexing over a single connection.

### 10.3 The certificate directives

```nginx
ssl_certificate     /Users/nana/nginx-certs/nginx-selfsigned.crt;  # public certificate
ssl_certificate_key /Users/nana/nginx-certs/nginx-selfsigned.key;  # PRIVATE key
```

| Directive | File | Contains | Secrecy |
|-----------|------|----------|---------|
| `ssl_certificate` | `.crt` (PEM) | The **public certificate** — the server's **public key** plus identity info (domain, issuer, validity dates), signed by an issuer. Sent to every client during the handshake. | Public. Safe to share. |
| `ssl_certificate_key` | `.key` (PEM) | The matching **private key**. Used to prove ownership of the cert and to complete the key exchange. | **Secret.** Never leaves the server; lock file perms to `600`/root-only. If it leaks, the cert is compromised. |

> Terminology fix vs the inline comment: the `.key` file is the **private key**, not a "private cert." There's one certificate (public); the two files are *cert* + *key*, a matched pair.

**Certificate chain gotcha (very common in prod):** for CA-signed certs, `ssl_certificate` should point at the **full chain** — your leaf cert **followed by** any intermediate CA certs, concatenated into one file (Let's Encrypt calls this `fullchain.pem`). If you serve only the leaf, many clients fail with "unable to verify" because they can't build the trust path to a root CA. Order in the file matters: leaf first, then intermediates.

### 10.4 Self-signed vs CA-signed certs

The filenames here (`nginx-selfsigned.crt/.key`) mean this is a **self-signed certificate** — signed by its own key rather than a trusted Certificate Authority.

- **Self-signed** → not in any browser/OS trust store, so clients show a security warning ("your connection is not private"). The traffic is **still encrypted**; what's missing is the *identity guarantee*. Fine for **local dev and internal tooling**.
  - Typically generated with OpenSSL, e.g.:
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout nginx-selfsigned.key -out nginx-selfsigned.crt
    ```
- **CA-signed** (e.g. Let's Encrypt via `certbot`, or a commercial CA) → chains to a trusted root, so browsers accept it silently. Required for anything **public-facing**. Let's Encrypt certs are free and auto-renewable (90-day validity).

### 10.5 HTTP → HTTPS redirect

A second `server` block catches plain-HTTP traffic and bounces it to HTTPS, so users who type `http://…` (or hit an old link) still land on the secure site.

```nginx
server {
    listen 8080;
    server_name localhost;

    location / {
        return 301 https://$host$request_uri;   # permanent redirect to the HTTPS URL
    }
}
```

- **`return 301 …`** — Nginx answers the redirect **itself**, immediately, without proxying anything. Efficient: no upstream is involved.
- **`301` (Moved Permanently)** vs `302` (Found/temporary): use **301** for a permanent HTTP→HTTPS policy — browsers and search engines cache it and stop re-requesting over HTTP. Use `302` only if the switch is temporary.
- **`https://$host$request_uri`** rebuilds the same URL on HTTPS:
  - `$host` — the hostname the client requested (preserves whatever they typed).
  - `$request_uri` — the **full original path + query string** (e.g. `/products?id=42`), so deep links survive the redirect.

> **Port note:** the inline comment says "port 80" but the directive listens on **`8080`**. Standard HTTP is port **80** — as written, this only redirects HTTP that arrives on 8080, and real browser `http://` traffic (port 80) won't be caught. For a genuine redirect-everything setup you'd want `listen 80;` here (this is likely just a dev-box port choice).

### 10.6 What this config is missing (hardening checklist)

The config gets TLS *working* but omits the directives you'd want in production. Worth adding when you productionize:

```nginx
ssl_protocols        TLSv1.2 TLSv1.3;          # disable old, broken SSL/TLS versions
ssl_ciphers          HIGH:!aNULL:!MD5;         # restrict to strong cipher suites
ssl_prefer_server_ciphers on;                  # server picks the cipher, not the client
ssl_session_cache    shared:SSL:10m;           # reuse sessions → fewer full handshakes
ssl_session_timeout  1h;
ssl_stapling on;                               # OCSP stapling: faster cert-revocation checks
ssl_stapling_verify on;

# HSTS: force browsers to always use HTTPS for this domain
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

Each buys something: `ssl_protocols`/`ssl_ciphers` cut off downgrade attacks; the session cache reduces expensive full handshakes (latency win); **HSTS** stops SSL-stripping by telling browsers never to even attempt HTTP for the domain again.

### 10.7 TLS handshake in one glance (the "why" behind the two files)

```
client                                   nginx
  │  ── ClientHello (supported ciphers) ──►
  │  ◄── ServerHello + certificate ───────   (sends ssl_certificate = public cert)
  │      verify cert, agree on keys
  │  ── key exchange ─────────────────────►  (nginx uses ssl_certificate_key to complete it)
  │  ◄══ encrypted session established ═══►
  │  ── encrypted HTTP request ───────────►  nginx decrypts, proxies plain HTTP upstream
```

The **public cert** is what the client checks and encrypts against; the **private key** is what only the server holds to complete the exchange. That asymmetry is the whole point — anyone can encrypt *to* you, only you can decrypt.

### 10.8 Annotated full HTTPS config

```nginx
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include mime.types;

    upstream nodejs_cluster {            # round robin (no algorithm specified)
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
        server 127.0.0.1:3003;
    }

    # --- HTTPS server: terminates TLS, proxies plain HTTP to the pool ---
    server {
        listen 443 ssl;                  # HTTPS, TLS enabled
        server_name localhost;

        ssl_certificate     /Users/nana/nginx-certs/nginx-selfsigned.crt;  # public cert
        ssl_certificate_key /Users/nana/nginx-certs/nginx-selfsigned.key;  # private key

        location / {
            proxy_pass http://nodejs_cluster;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            # add: proxy_set_header X-Forwarded-Proto $scheme;  → tell backend it was HTTPS
        }
    }

    # --- HTTP server: permanent redirect to HTTPS ---
    server {
        listen 8080;                     # (standard HTTP is port 80)
        server_name localhost;

        location / {
            return 301 https://$host$request_uri;   # 301 → same URL over HTTPS
        }
    }
}
```

### 10.9 SSL quick reference

| Directive / snippet | Purpose |
|---------------------|---------|
| `listen 443 ssl;` | Serve HTTPS on 443; `ssl` param enables TLS |
| `listen 443 ssl http2;` | ...and enable HTTP/2 |
| `ssl_certificate` | Public cert (leaf + intermediates as `fullchain` for CA certs) |
| `ssl_certificate_key` | Private key — keep secret, tight file perms |
| `ssl_protocols` | Allowed TLS versions (`TLSv1.2 TLSv1.3`) |
| `ssl_ciphers` / `ssl_prefer_server_ciphers` | Cipher policy |
| `return 301 https://$host$request_uri;` | Permanent HTTP→HTTPS redirect |
| `add_header Strict-Transport-Security …` | HSTS — force HTTPS on future visits |
| `X-Forwarded-Proto $scheme` | Tell the terminated backend the client used HTTPS |