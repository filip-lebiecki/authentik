# Zero-Trust Self-Hosted Access Stack

Expose self-hosted apps to the internet with **zero open ports**, centralized login, and MFA — using **Cloudflare Tunnel**, **Authentik**, and **Cloudflare Access**.

---

## Architecture Overview

```
Browser → Cloudflare Edge (TLS termination + Access policy enforcement)
              ↓  (encrypted tunnel — no open ports)
          cloudflared container (dmz_net)
              ↓
          Authentik server (dmz_net + auth_net)
              ↓
          Your self-hosted apps (Proxmox, Home Assistant, etc.)
```

**Docker networks:**
- `auth_net` — internal network for Authentik components (server ↔ DB ↔ worker)
- `dmz_net` — external-facing network shared by `cloudflared` and the Authentik server

---

## Prerequisites

- Ubuntu host with Docker installed
- A domain managed on Cloudflare (can be registered on OVH/GoDaddy and transferred)
- No public IP required

```bash
lsb_release -a
docker version
```

---

## 1. Cloudflare Tunnel

`cloudflared` opens an outbound encrypted tunnel to Cloudflare — no inbound ports needed.

```bash
cd cloudflare/
cat docker-compose.yml
cat .env
```

Get the tunnel token from **Cloudflare Zero Trust → Networks → Connectors → Add a tunnel → cloudflared**. Copy the token from the generated Docker run command and paste it into `.env`:

```bash
vi .env
```

Bring up the container:

```bash
docker compose up -d
docker ps
```

Verify the tunnel shows **Connected** in the Cloudflare console under **Connectors**.

> **Tip — secrets hygiene:** Store the tunnel token in `.env`, not in `docker-compose.yml`. Add `.env` to `.gitignore` so it's never committed.
>
> ```bash
> cat .env
> cat .gitignore
> ```

### Adding tunnel routes

In **Published application routes**, map a hostname to a backend service:

| Field | Example |
|---|---|
| Domain | `app1.yourdomain.com` |
| Service | `http://container1:8080` |

One tunnel can serve many apps — just add more routes. Cloudflare automatically creates a CNAME and handles TLS certificates for each route.

**Path-based routing** is also supported — add a route with a subdomain and a path prefix.

**Redundancy:** Run multiple `cloudflared` instances; Cloudflare load-balances across them automatically.

---

## 2. Authentik (Identity Provider)

```bash
cd authentik/
cat docker-compose.yml | more
```

### Network layout

| Container | Networks |
|---|---|
| PostgreSQL (db) | `auth_net` only |
| Authentik server | `auth_net` + `dmz_net` (alias: `authentik`) |
| Authentik worker | `auth_net` only |

`dmz_net` is declared `external` so Compose reuses the network created by the Cloudflare stack.

```bash
docker compose up -d
docker ps
```

### Initial setup

Grab the generated admin password:

```bash
cat .env
```

Navigate to `http://<host>:8080`, log in as `akadmin`, then switch to the **Admin interface**.

**Create a proper admin account:**

1. **Directory → Users → New User** — fill in username, display name, and **email address** (required by many OIDC clients)
2. Set a password for the new user
3. **Groups → Add to existing group** → select `authentik Admins`
4. Log out, log back in as the new user
5. Delete the default `akadmin` user

### Expose Authentik via Cloudflare Tunnel

Add a route in the tunnel's **Published application routes**:

| Field | Value |
|---|---|
| Domain | `auth.yourdomain.com` |
| Service | `http://authentik:9000` |

(`authentik` resolves via the Docker network alias on `dmz_net`.)

Navigate to `https://auth.yourdomain.com` and confirm the login page loads over HTTPS.

### Enable MFA

**Settings → MFA Devices → Enroll** — register a TOTP app (Google Authenticator, Bitwarden) or a passkey (Touch ID, Windows Hello, hardware key, browser-stored passkey).

> MFA must be enabled on Authentik because it sits in front of all protected apps.
> Passkey enrollment requires HTTPS.

---

## 3. Cloudflare WAF Rules

Cloudflare's free tier provides **5 custom rules** and **1 rate-limiting rule**. Recommended configuration for `auth.yourdomain.com`:

> **Rule order matters** — Cloudflare evaluates top-to-bottom. Place the allowlist rule first.

### Rule 1 — Allow home IPs (top of list, action: Skip)

```
http.host eq "auth.yourdomain.com" and (ip.src in {<home-ip-1> <home-ip-2>})
```

### Rule 2 — Country allowlist (action: Block)

```
http.host eq "auth.yourdomain.com" and ip.src.country ne "PL"
```

Replace `"PL"` with your own country code.

### Rule 3 — Block bots (action: Block)

```
http.host eq "auth.yourdomain.com" and cf.client.bot
```

### Rule 4 — Rate limit (action: Block, 100 req / 10 s)

Go to **Security → Security Rules → Create rule → Rate limiting rule**:

```
(http.host eq "auth.yourdomain.com" and
 (http.request.uri.path contains "/if/flow/" or
  http.request.uri.path contains "/api/" or
  http.request.uri.path contains "/application/o/" or
  http.request.uri.path contains "/if/admin/"))
```

Threshold: **100 requests per 10 seconds**, block for **10 seconds**.

---

## 4. Cloudflare Access + Authentik OIDC Integration

### Register Authentik as an Identity Provider in Cloudflare

**Cloudflare Zero Trust → Integrations → Identity Providers → Add → OpenID Connect**

In **Authentik → Applications → Create with provider**:
- Name: `Cloudflare`
- Provider type: OpenID Connect
- Authorization flow: Implicit consent
- Redirect URI: `https://<your-team-name>.cloudflareaccess.com/cdn-cgi/access/callback`

Copy **Client ID** and **Client Secret** from Authentik into Cloudflare.

From the Authentik provider detail page, copy and paste into Cloudflare:
- Authorize URL
- Token URL
- JWKS URL

Click **Save**, then **Test** to verify the full OIDC flow.

---

## 5. Protecting an App — Three-Step Pattern

Every protected application follows the same three steps.

### Step 1 — Add a tunnel route

**Networks → Connectors → [your tunnel] → Published application routes → Add**

### Step 2 — Create an access policy

**Access Controls → Policies → Add a policy**

Policies are reusable — define once, apply to multiple apps.

Example: match by email address.

### Step 3 — Create a protected application

**Applications → Add an application → Self-hosted**

Attach the tunnel hostname and the policy. This is the step that actually enforces authentication.

---

## Example: Browser-Based SSH

| Field | Value |
|---|---|
| Tunnel route domain | `ssh.yourdomain.com` |
| Service type | SSH |
| Target | `192.168.x.x:22` |
| Browser rendering | SSH |
| Policy | email match |
| Identity provider | Authentik |
| Instant auth | ✅ (single IdP) |

After saving, visiting `ssh.yourdomain.com` redirects to Authentik → on success, drops the user into a full in-browser SSH terminal.

> **Security note:** The browser SSH session is visible to malicious extensions or keyloggers on the client machine. Use a trusted, clean device.

---

## Example: HTTPS Web App (Proxmox)

| Field | Value |
|---|---|
| Tunnel route domain | `proxmox.yourdomain.com` |
| Service type | HTTPS |
| Target | `192.168.x.x:8006` |
| No TLS Verify | ✅ (self-signed cert) |
| Policy | reuse existing email policy |

The Proxmox host does not need to run on the same machine as `cloudflared` — it just needs to be reachable over the local network.

---

## Example: Proxmox SSO via OIDC

Configure Proxmox to authenticate users through Authentik directly.

**In Authentik — create a new application + provider:**
- Name: `Proxmox`
- Provider type: OpenID Connect
- Authorization flow: Implicit consent
- Redirect URI: `https://proxmox.yourdomain.com`
- Subject mode: Email

**In Proxmox → Realms → Add → OpenID Connect:**

| Field | Value |
|---|---|
| Realm name | `Authentik` (any name) |
| Client ID | (from Authentik) |
| Client Secret | (from Authentik) |
| Issuer URL | (from Authentik provider detail) |
| Username claim | `username` |
| Default | ✅ |

Create a matching local user in Proxmox (**Users → Add**) with the same username as in Authentik.

After configuration, the Proxmox login screen shows **Login with OpenID Connect**. Users authenticate via Authentik (including MFA) and land directly on the dashboard — no separate Proxmox credentials required.

> Follow the principle of least privilege when assigning Proxmox roles to SSO users.

---

## Security Considerations

| Risk | Mitigation |
|---|---|
| Compromised Authentik password | Enforce MFA — non-negotiable |
| Weak SSH/app credentials | Use strong passwords; Authentik is a first layer only |
| Browser-based session exposure | Use trusted devices; watch for malicious extensions |
| Cloudflare as traffic intermediary | Cloudflare terminates TLS at the edge — you are trusting their infrastructure |

---

## Repository Structure

```
.
├── cloudflare/
│   ├── docker-compose.yml   # cloudflared tunnel container
│   └── .env                 # TUNNEL_TOKEN (never commit this)
├── authentik/
│   ├── docker-compose.yml   # Authentik server, worker, PostgreSQL
│   └── .env                 # Authentik secrets (never commit this)
└── README.md
```

Add `.env` to `.gitignore` in each directory before committing.
