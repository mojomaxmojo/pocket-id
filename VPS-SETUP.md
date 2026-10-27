# Pocket ID — VPS-Deployment (auth.mojobus.co)

> Setup-Doku + Configs für die **Produktions-Instanz** auf der AlmaLinux-VPS.
> Dieses Verzeichnis beschreibt das Deployment, NICHT den Quellcode im Repo.
> Master-Doku des Verbunds: `/projects/skysend/VPS-SETUP.md`

**Server:** host.mojomail.me · 194.127.179.3 · AlmaLinux 9.8 · Centminmod (Nginx `/usr/local/nginx`), CSF, acme.sh
**Instanz:** https://auth.mojobus.co · Pocket ID 2.19.1 · Podman, Port 3011 → Container 1411
**Stand:** 27.10.2026

---

## Architektur / Verbund

```
┌─ https://share.mojobus.co ─────────────────────┐
│  SkySend 3.2.0 (Podman, 127.0.0.1:3020)        │
│  Upload/Link-Erstellen: nur mit Pocket-ID-Login│
│  Download: frei per Share-Link (E2E-verschl.)  │
└──────────────┬─────────────────────────────────┘
               │ OIDC (Authorization Code, Confidential + PKCE)
┌─ https://auth.mojobus.co ──────────────────────┐
│  Pocket ID 2.19.1 (Podman, 127.0.0.1:3011)     │
│  6 Konten: 1 Admin + 5 User, Passkey-Login     │
└─────────────────────────────────────────────────┘
      beide hinter Nginx + Let's Encrypt, Cloudflare DNS-only
```

---

## 1. Container (`deploy/docker-compose.yml`)

```yaml
services:
  pocket-id:
    image: docker.io/pocketid/pocket-id:v2
    container_name: pocket-id
    restart: always
    ports:
      - "127.0.0.1:3011:1411"
    env_file: .env
    volumes:
      - /opt/pocketid/data:/app/data
    mem_limit: 256m      # Rücksicht auf Remotion-Renders auf derselben Kiste
    cpus: '0.3'
```

Deploy-Pfad auf der VPS: **`/opt/pocketid/`**

## 2. Environment (`deploy/.env.example` → `/opt/pocketid/.env`)

```bash
# === KRITISCH: Backup dieser Datei machen! ===
APP_URL=https://auth.mojobus.co
# Passkeys sind an diese Domain gebunden — APP_URL NIE ändern,
# sonst sind alle registrierten Passkeys ungültig!

ENCRYPTION_KEY=<openssl rand -base64 32>
# Verschlüsselt Token-Signing-Keys. Ohne diese Datei/Key sind Sessions weg.

TRUST_PROXY=true      # Nginx davor, Client-IP aus X-Forwarded-For
PUID=1000
PGID=1000
```

Generierung:
```bash
mkdir -p /opt/pocketid/data
tee /opt/pocketid/.env > /dev/null <<EOF
APP_URL=https://auth.mojobus.co
ENCRYPTION_KEY=$(openssl rand -base64 32)
TRUST_PROXY=true
PUID=1000
PGID=1000
EOF
```

## 3. Nginx-VHost (`deploy/nginx/auth.mojobus.co.ssl.conf`)

Ziel auf VPS: `/usr/local/nginx/conf/conf.d/auth.mojobus.co.ssl.conf`

```nginx
#x# HTTPS-DEFAULT
 server {
   listen   80;
   listen   [::]:80;
   server_name auth.mojobus.co www.auth.mojobus.co;
   return 302 https://auth.mojobus.co$request_uri;
   root /home/nginx/domains/auth.mojobus.co/public;
   include /usr/local/nginx/conf/staticfiles.conf;
   # acme-challenge durchlassen (Let's Encrypt HTTP-Validierung):
   location ^~ /.well-known/acme-challenge/ {
     default_type "text/plain";
     root /home/nginx/domains/auth.mojobus.co/public;
   }
 }

#

server {
  listen 443 ssl http2;

  server_name auth.mojobus.co www.auth.mojobus.co;

  include /usr/local/nginx/conf/ssl/auth.mojobus.co/auth.mojobus.co.crt.key.conf;
  include /usr/local/nginx/conf/ssl_include.conf;

  # mozilla recommended
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers   on;

  # HSTS: erst aktivieren, wenn alles laeuft
  #add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;";
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;

  ssl_buffer_size 1369;
  ssl_session_tickets on;

  access_log /home/nginx/domains/auth.mojobus.co/log/access.log combined buffer=256k flush=5m;
  error_log /home/nginx/domains/auth.mojobus.co/log/error.log;

  # *** KRITISCH: autoprotect AUS — blockt SvelteKit-Routen → "Internal Error" in der UI
  # include /usr/local/nginx/conf/autoprotect/auth.mojobus.co/autoprotect-auth.mojobus.co.conf;

  root /home/nginx/domains/auth.mojobus.co/public;
  #include /usr/local/nginx/conf/cloudflare.conf;   # Proxy ist OFF
  include /usr/local/nginx/conf/503include-main.conf;

  location / {
  include /usr/local/nginx/conf/503include-only.conf;

  proxy_pass http://127.0.0.1:3011;
  proxy_http_version 1.1;

  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;

  proxy_connect_timeout 30s;
  proxy_send_timeout    60s;
  proxy_read_timeout    60s;
  }

  # Pocket ID braucht kein PHP:
  # include /usr/local/nginx/conf/php.conf;

  # staticfiles aus: alle Requests an Pocket ID
  # include /usr/local/nginx/conf/pre-staticfiles-local-auth.mojobus.co.conf;
  # include /usr/local/nginx/conf/pre-staticfiles-global.conf;
  # include /usr/local/nginx/conf/staticfiles.conf;

  # *** KRITISCH: drop.conf AUS — blockt SvelteKit-Routen (GitHub pocket-id#1606/#1715)
  # include /usr/local/nginx/conf/drop.conf;

  #include /usr/local/nginx/conf/errorpage.conf;
  include /usr/local/nginx/conf/vts_server.conf;
}
```

### Aktivieren

```bash
cp /usr/local/nginx/conf/conf.d/auth.mojobus.co.ssl.conf{,.bak-$(date +%Y%m%d)} 2>/dev/null
# Datei einfügen, dann:
mkdir -p /home/nginx/domains/auth.mojobus.co/log
nginx -t && nginx -s reload
```

## 4. TLS via acme.sh (NICHT Centminmod acmetool!)

> ⚠️ acmetool erzeugte hier ein Cloudflare-Origin-Cert (100 Jahre, nur CF-trusted) →
> Browser-Warnungen + curl 000. Richtig: acme.sh mit Let's Encrypt.

```bash
# 1. DNS prüfen (muss auf 194.127.179.3 zeigen):
dig +short auth.mojobus.co

# 2. Port-80-VHost muss vor der Ausstellung aktiv sein (siehe oben)

# 3. Zertifikat ausstellen:
~/.acme.sh/acme.sh --issue -d auth.mojobus.co \
  --webroot /home/nginx/domains/auth.mojobus.co/public -f

# 4. Installieren — MIT --ecc (sonst falscher Pfad!):
mkdir -p /usr/local/nginx/conf/ssl/auth.mojobus.co
~/.acme.sh/acme.sh --install-cert -d auth.mojobus.co --ecc \
  --cert-file /usr/local/nginx/conf/ssl/auth.mojobus.co/auth.mojobus.co.crt \
  --key-file /usr/local/nginx/conf/ssl/auth.mojobus.co/auth.mojobus.co.key \
  --fullchain-file /usr/local/nginx/conf/ssl/auth.mojobus.co/auth.mojobus.co.crt \
  --reloadcmd "nginx -s reload"

# 5. Testen:
curl -s -o /dev/null -w "%{http_code}\n" https://auth.mojobus.co   # 200
```

Renewals laufen automatisch (ARI-Fenster, ~60 Tage) inkl. `nginx -s reload`.

## 5. Installation Schritt für Schritt (komplett neu)

1. **DNS:** A-Record `auth.mojobus.co → 194.127.179.3` (Cloudflare, DNS only / graue Wolke)
2. **Port-80-VHost** anlegen (Block oben) → `nginx -t && nginx -s reload`
3. **Zertifikat** via acme.sh (Abschnitt 4)
4. **HTTPS-VHost** anlegen (Abschnitt 3) → reload
5. **Container** anlegen (Abschnitt 1+2) → `cd /opt/pocketid && podman-compose up -d`
6. **Check:** `curl http://127.0.0.1:3011` → 200 · `https://auth.mojobus.co` → 200
7. **Erst-Setup:** Browser `https://auth.mojobus.co/setup` → Admin-Konto + Passkey
   *(falls Seite weg: `podman exec pocket-id /app/pocket-id one-time-access-token`)*
8. **User:** Administration → Users → 5 Konten; pro Person `⋯` → Login Code (kurz gültig, per Messenger) → Person legt eigenen Passkey an
9. **OIDC-Client `skysend`:**
   - Typ: **Vertraulich**, PKCE an
   - Callback: `https://share.mojobus.co/auth/callback` (exakt, kein trailing slash)
   - Access: **All Users**
   - Client ID + Secret sofort kopieren (Secret nur einmal sichtbar!)
10. **SkySend-Anbindung:** Variablen in `/opt/skysend/docker-compose.yml` (siehe Abschnitt 7) → `podman-compose down && up -d`
11. **Tests:** Inkognito → share.mojobus.co → Upload → Login-Redirect → Passkey → Upload ✓
    Share-Link ausgeloggt öffnen → Download ohne Login ✓

## 6. Login-Methoden (auch ohne Biometrie)

| Methode | Biometrie nötig? | Hinweis |
|---|---|---|
| Passkey (Face/Finger/PIN) | optional | Entsperrung per Geräte-PIN reicht |
| Passkey im Passwort-Manager | nein | Bitwarden/1Password/KeePassXC, Entsperrung mit Master-Passwort |
| „Sign in with another device" | nein | QR-Code, Freigabe am Handy, 5 Min gültig |
| Login-Code (Admin) | nein | einmalig, für Erst-Setup/Verlust |

Voraussetzung: **User verification: Preferred** (Application Configuration → Passkeys).

## 7. SkySend-Anbindung (Referenz)

In `/opt/skysend/docker-compose.yml` unter `environment:`:

```yaml
      - OIDC_PROVIDER=pocketid
      - OIDC_ISSUER=https://auth.mojobus.co
      - OIDC_CLIENT_ID=<Client ID aus Pocket ID>
      - OIDC_CLIENT_SECRET=<Client Secret aus Pocket ID>
      - OIDC_REDIRECT_URI=https://share.mojobus.co/auth/callback
      - OIDC_SESSION_SECRET=<openssl rand -base64 48>
      - OIDC_PROTECT_FILES=true
      - OIDC_PROTECT_NOTES=true
      - OIDC_PROTECT_REQUESTS=true
```

Effekt: Upload/Notizen/File-Requests erfordern Login; Downloads bleiben öffentlich per Link.

## 8. Betrieb & Wartung

```bash
# Start/Stop
cd /opt/pocketid && podman-compose up -d / down

# Update
cd /opt/pocketid && podman-compose pull && podman-compose up -d

# Logs
podman logs -f pocket-id
podman logs --since 5m pocket-id | grep -vE "status=200|status=204"

# Health
curl -s -o /dev/null -w "%{http_code}\n" https://auth.mojobus.co
```

### Backup-Liste

| Datei | Kritikalität |
|---|---|
| `/opt/pocketid/.env` | 🔴 ENCRYPTION_KEY — ohne ihn sind Signing-Keys weg |
| `/opt/pocketid/data/` (SQLite) | 🟡 nur Public Keys der Passkeys — Diebstahl ermöglicht KEINEN Login, aber Restore für Uptime |
| Nginx-VHost (`auth.mojobus.co.ssl.conf`) | 🟡 mit den auskommentierten Includes — ohne die crasht die UI |
| SkySend: `docker-compose.yml` | 🔴 enthält OIDC_CLIENT_SECRET |

### Fehlertabelle (erlebt & gelöst)

| Fehler | Ursache | Lösung |
|---|---|---|
| Admin-UI „Internal Error" bei Client-Erstellen/Editieren/Detail | Centminmod `drop.conf` + `autoprotect` blocken SvelteKit-Routen (`__data.json`, `/_app/...`) | Beide Includes im auth-VHost auskommentieren (Issue pocket-id#1606/#1715) |
| „You are not allowed to access this service" nach Login | OIDC-Client-Access auf Gruppe statt All Users (Admin nicht drin) | Access-Tab → All Users |
| `WRN Lease was lost for alarm - ScimSync` + SCIM-Endpoint 404 | Degenerierter Actor-State nach Erststart | `podman-compose down && up -d`; danach muss Log „SCIM sync completed" zeigen |
| SCIM-404 `/api/oidc/clients/:id/scim-service-provider` | Normal für Clients ohne SCIM | Ignorieren — kein Fehler |
| Zertifikat 100 Jahre gültig, Browser-Warnung, curl 000 | Centminmod acmetool erzeugte CF-Origin-Cert (Proxy ist aber OFF) | LE-Cert via acme.sh, `--install-cert --ecc` |
| `acme.sh --install-cert` schreibt kaputtes .crt, `nginx -t` fails | ECC-Cert ohne `--ecc`-Flag installiert → falscher Quellpfad | Install mit `--ecc` wiederholen |
| `nginx: [emerg] unknown "connection_upgrade"` | Map fehlt (betrifft SkySend-VHost) | `/usr/local/nginx/conf/conf.d/map-connection-upgrade.conf` mit der map |
| `acmetool.sh: command not found` | Centminmod-Tool nicht im PATH | Direkt `~/.acme.sh/acme.sh` nutzen |

### Ressourcen (beobachtet)

- Pocket ID: ~50–80 MB RAM idle
- SkySend: ~100–150 MB RAM idle
- Beide zusammen < 5 % der 8 GB — Remotion-Renders unbeeinträchtigt
