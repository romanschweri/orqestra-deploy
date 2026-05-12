# Orqestra — Self-Hosted Deployment

Setup-Dateien, um Orqestra auf deinem eigenen Server laufen zu lassen. Drei
Container: PostgreSQL, .NET-API, Next.js-UI. Du baust nichts selbst, du ziehst
nur vorgebaute Images aus der GitHub Container Registry (GHCR).

## Voraussetzungen

- Ein Server (Hetzner Cloud, VPS, eigene Hardware — egal) mit Linux
- Docker Engine ≥ 24 und das Compose-Plugin
  ```bash
  docker --version
  docker compose version
  ```
- Ein **GHCR-Zugang von Roman** — du musst von ihm zu den Packages
  `orqestra-api` und `orqestra-ui` als Reader eingeladen werden. Du bekommst
  eine GitHub-Mail mit der Einladung und musst annehmen.
- Empfohlen: ein Reverse Proxy (Caddy / Nginx / Traefik) für HTTPS. Die
  Auth-Cookies der API sind in Production als `Secure` markiert — Login
  funktioniert nur über HTTPS zuverlässig.

## Schritte

### 1. Repo klonen

```bash
git clone https://github.com/romanschweri/orqestra-deploy.git orqestra
cd orqestra
```

### 2. Personal Access Token für GHCR erstellen

Damit dein Server private Images pullen darf, brauchst du auf deinem eigenen
GitHub-Account einen PAT:

1. GitHub → Profilbild → **Settings**
2. Ganz unten links → **Developer settings**
3. **Personal access tokens → Tokens (classic) → Generate new token (classic)**
4. Name: z.B. `ghcr-pull-orqestra`
5. Expiration: deine Wahl (90 Tage oder "No expiration")
6. Scope: **nur** `read:packages` ankreuzen
7. Generate → Token sofort kopieren (`ghp_...`) — wird nur einmal angezeigt

### 3. Auf dem Server bei GHCR einloggen

```bash
echo "ghp_xxxxxxxxxxxx" | docker login ghcr.io -u <dein-github-username> --password-stdin
```

Erwartet: `Login Succeeded`. Docker merkt sich den Login persistent in
`~/.docker/config.json` bis der PAT abläuft.

### 4. `.env` erstellen und Werte eintragen

```bash
cp .env.example .env
nano .env
```

Mindestens diese Werte musst du setzen:

- `POSTGRES_PASSWORD` — irgendein langes random String, z.B. via
  `openssl rand -base64 32`
- `ORQESTRA_API_IMAGE` / `ORQESTRA_UI_IMAGE` — `romanschweri` als Username
  drin lassen, das ist Roman's GHCR-Namespace
- `API_PUBLIC_URL` / `UI_PUBLIC_URL` — die URLs, unter denen die App vom
  Browser aus erreichbar ist (siehe Reverse-Proxy unten)

Den ersten Admin-User legst du nicht hier an, sondern beim ersten Aufruf des
UI im Browser (siehe Schritt 6).

### 5. Starten

```bash
docker compose pull   # zieht beide Images aus GHCR
docker compose up -d  # startet alle drei Container im Hintergrund
docker compose logs -f api
```

Warten bis du `Now listening on: http://[::]:8080` siehst. `Strg+C` bricht das
Log-Tailen ab — die Container laufen weiter.

Beim ersten Start führt der `DatabaseMigrationService` die DB-Migrations aus.

### 6. Ersten Admin-User anlegen + Template Setup

Browser auf `UI_PUBLIC_URL` (z.B. `https://orqestra.deine-domain.de`).

1. Beim allerersten Aufruf zeigt das UI ein **Setup-Formular** statt dem Login —
   E-Mail + Passwort (mind. 8 Zeichen) eintragen → der erste Admin wird angelegt
   und du bist direkt eingeloggt
2. **Broker Accounts** → Alpaca Paper Account anlegen (API Key + Secret von
   [alpaca.markets](https://alpaca.markets))
3. **Settings → LLM Providers** → Anthropic API Key eintragen
4. **Settings → Template Setup → Run setup** → erstellt 5 Departments, 11
   Tools, 14 Agents und 2 Workflows in einem Rutsch
5. **Tools** → für `search_news` (Tavily) und ggf. `truth_social` API-Keys
   nachtragen + enablen. Die `alpaca_*`-Tools sind bereits mit deinem
   Broker-Account verdrahtet falls genau einer aktiv ist
6. **Agents** → Smoke-Test: einen Agent ("Walter") mit "Run now" laufen
   lassen, Output prüfen → erst dann alle Agents enablen

## Updates ziehen

Wenn Roman neue Versionen baut, einfach:

```bash
docker compose pull
docker compose up -d
```

Migrations laufen beim Boot automatisch, keine manuellen DB-Operationen
nötig. Vor grösseren Updates: Volume-Snapshot über dein Hoster-Panel machen
(`docker volume ls` zeigt `orqestra_postgres-data`).

## Reverse Proxy (empfohlen)

Die Auth-Cookies sind als `Secure` markiert — Login funktioniert
zuverlässig nur über HTTPS. Beispiel mit Caddy auf dem Host:

```caddyfile
orqestra.deine-domain.de {
    reverse_proxy /        localhost:3000
    reverse_proxy /_next/* localhost:3000
    reverse_proxy /api/*   localhost:5140
}
```

Dann in `.env`:
```
API_PUBLIC_URL=https://orqestra.deine-domain.de
UI_PUBLIC_URL=https://orqestra.deine-domain.de
```

Caddy holt automatisch ein Let's-Encrypt-Zertifikat sobald die Domain auf
deinen Server zeigt.

## Volumes / Backup

Zwei persistente Named Volumes:

- `orqestra_postgres-data` — die ganze DB (Agents, Runs, Signals, Trades, ...)
- `orqestra_api-dp-keys` — DataProtection-Keys zum Entschlüsseln von Cookies
  und gespeicherten Broker-Credentials. **Wenn du dieses Volume verlierst,
  musst du die Broker-Credentials neu eingeben** — die DB-Spalten sind dann
  nicht mehr entschlüsselbar.

Manuelles Backup:
```bash
docker compose exec postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB | gzip > backup-$(date +%F).sql.gz
docker run --rm -v orqestra_api-dp-keys:/keys -v $(pwd):/backup alpine tar czf /backup/dp-keys-$(date +%F).tar.gz /keys
```

Oder einfacher: Volume-Snapshot über das Hoster-Panel (bei Hetzner ~1.20 EUR
pro Volume und Monat).

## Troubleshooting

**`unauthorized` beim `docker compose pull`**
→ PAT abgelaufen oder Tippfehler beim Username. `docker login ghcr.io ...`
neu ausführen.

**`manifest unknown` oder `not found` beim Pull**
→ Du wurdest noch nicht als Reader zu den GHCR-Packages eingeladen, oder die
Einladung nicht angenommen. Bei Roman melden.

**API kommt nicht hoch, Logs zeigen "could not connect to postgres"**
→ Postgres-Healthcheck hat länger gedauert. `docker compose up -d` nochmal,
oder `docker compose logs postgres` checken.

**UI lädt aber API-Calls schlagen mit CORS-Fehler fehl**
→ `UI_PUBLIC_URL` und `API_PUBLIC_URL` in `.env` prüfen. Der API-Container muss
die UI-URL als Allowed Origin kennen. Nach Änderung:
`docker compose up -d --force-recreate api`.

**Cookie wird nicht gesetzt nach Login**
→ Du nutzt HTTP statt HTTPS. Cookies sind als `Secure` markiert. Reverse Proxy
mit TLS aufsetzen, oder fürs lokale Testen `ASPNETCORE_ENVIRONMENT=Development`
in der `.env` ergänzen — Cookie wird dann auch ohne HTTPS gesetzt
(**nur zum Schnuppern, nicht für Dauerbetrieb!**).
