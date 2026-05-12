# Deployment

Orqestra läuft als drei Container: PostgreSQL, .NET-API, Next.js-UI. Empfohlene
Variante ist Docker Compose mit den vorgebauten Images aus GitHub Container
Registry (GHCR). Build-Zeit bei dir = 0.

## Voraussetzungen auf dem Host

- Docker Engine ≥ 24 und das Compose-Plugin (`docker compose version`)
- Ein Domain-Name oder eine öffentliche IP, wenn du die App von außen erreichbar
  machen willst
- Optional: ein Reverse Proxy (Caddy / Nginx / Traefik) der TLS terminiert. Stark
  empfohlen sobald du LLM-Provider-API-Keys oder Broker-Credentials in der App
  ablegst.

## Schritte für dich (Maintainer)

### 1. GitHub Actions auf "build & push" einstellen

Beim Push auf `main` oder beim Setzen eines `v*`-Tags baut der Workflow in
`.github/workflows/build-images.yml` automatisch beide Images und published sie
nach GHCR unter:

- `ghcr.io/<dein-github-user>/orqestra-api:latest`
- `ghcr.io/<dein-github-user>/orqestra-ui:latest`

Plus `:<branch>` und `:<semver>` Tags wenn du mit Git-Tags arbeitest.

**Einmalig**:
- In den Repo-Settings → Actions → Workflow permissions → "Read and write" für
  packages aktivieren.
- Nach dem ersten Push: GHCR-Package-Sichtbarkeit prüfen (default Private). Auf
  Private lassen wenn nur du / deine Friends pullen sollen → dann auf den
  Hosts ein `docker login ghcr.io` mit einem PAT mit `read:packages`-Scope nötig.

### 2. `.env` erzeugen und Secrets eintragen

```bash
cp .env.example .env
# editiere .env — vor allem POSTGRES_PASSWORD und ORQESTRA_BOOTSTRAP_PASSWORD
```

Beachte:
- `ORQESTRA_API_IMAGE` / `ORQESTRA_UI_IMAGE` müssen auf deine GHCR-Images zeigen.
- `API_PUBLIC_URL` ist die URL, die der Browser benutzt um die API zu erreichen
  — i.d.R. `https://orqestra.example.com/api` wenn du einen Proxy davorhängst,
  oder `http://your-vps-ip:5140` im einfachen Fall.
- `UI_PUBLIC_URL` ist die UI-URL — wird für CORS auf der API benötigt.

### 3. Erster Start

```bash
docker compose pull   # zieht die Images aus GHCR
docker compose up -d  # startet alle drei Container
docker compose logs -f api  # schau auf Migration + Bootstrap-User
```

Der `BootstrapAdminService` läuft beim API-Start, führt EF-Migrations aus und
erstellt einen Admin-User mit den `ORQESTRA_BOOTSTRAP_*`-Credentials.

### 4. Template Setup im UI

Login mit dem Bootstrap-Admin, dann:

1. **Broker Accounts** → Alpaca Paper Account anlegen (API Key + Secret von
   alpaca.markets)
2. **Settings → LLM Providers** → Anthropic API Key eintragen
3. **Settings → Template Setup → Run setup** → erstellt 5 Departments, 11 Tools
   (manche disabled), 14 Agents und 2 Workflows in einem Rutsch
4. **Tools** → für `search_news` (Tavily) und ggf. `truth_social` API-Keys
   nachtragen + enablen. Die `alpaca_*`-Tools sind bereits mit deinem Broker-
   Account verdrahtet falls genau einer aktiv ist.
5. **Agents** → smoke-test mit Walter "Run now" → Output prüfen → erst dann
   alle Agents enablen.

### 5. Updates

```bash
docker compose pull
docker compose up -d
```

Migrations laufen beim Boot automatisch (`BootstrapAdminService.MigrateAsync`),
keine manuellen DB-Operationen nötig. Vor größeren Updates: Hetzner-Snapshot
des Volumes machen — `docker volume ls` zeigt `orqestra_postgres-data`.

## Schritte für Friends (User)

Wenn deine Friends die App auf ihrem eigenen Host laufen lassen wollen:

```bash
# 1. Compose-File + Beispiel-env holen
curl -O https://raw.githubusercontent.com/<dein-github-user>/orqestra/main/docker-compose.yml
curl -o .env https://raw.githubusercontent.com/<dein-github-user>/orqestra/main/.env.example

# 2. .env editieren (Passwörter, Public URLs)
nano .env

# 3. (Falls Package private:) GHCR-Login
echo $GITHUB_PAT | docker login ghcr.io -u <user> --password-stdin

# 4. Starten
docker compose pull
docker compose up -d

# 5. Logs anschauen bis "Now listening on: http://[::]:8080"
docker compose logs -f api
```

Dann unter `http://localhost:3000` (oder ihrer Domain) einloggen und Template
Setup laufen lassen wie oben.

## Reverse Proxy (empfohlen für Production)

Beispiel Caddy-Konfig auf dem Host:

```caddyfile
orqestra.example.com {
    # UI
    reverse_proxy /        localhost:3000
    reverse_proxy /_next/* localhost:3000

    # API
    reverse_proxy /api/*   localhost:5140
}
```

Dann in `.env`:
```
API_PUBLIC_URL=https://orqestra.example.com
UI_PUBLIC_URL=https://orqestra.example.com
```

Caddy holt automatisch ein Let's-Encrypt-Zertifikat. Die Cookies der API sind
in Production-Mode als `Secure` markiert — funktioniert nur über HTTPS, daher
Proxy quasi Pflicht außerhalb von Localhost-Tests.

## Volumes / Backup

Zwei Named Volumes mit persistentem State:

- `orqestra_postgres-data` — die ganze DB (Agents, Runs, Signals, Trades, ...)
- `orqestra_api-dp-keys` — DataProtection-Keys zum Entschlüsseln von Cookies
  und gespeicherten Broker-Credentials. **Wenn du diesen Volume verlierst,
  musst du die Broker-Credentials neu eingeben** — die DB-Spalten sind dann
  nicht mehr entschlüsselbar.

Backup-Strategie für Hetzner: Volume-Snapshot über die Hetzner-Console kaufen
(~1.20 EUR / Monat). Für Manual-Backups:

```bash
docker compose exec postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB | gzip > backup-$(date +%F).sql.gz
docker run --rm -v orqestra_api-dp-keys:/keys -v $(pwd):/backup alpine tar czf /backup/dp-keys-$(date +%F).tar.gz /keys
```

## Lokale Entwicklung (build statt pull)

Wenn du am Code arbeitest und gegen lokal gebaute Images testen willst:

```bash
docker compose build api ui   # baut beide images vor Ort
docker compose up -d
```

Oder einfach `dotnet run` (API) + `npm run dev` (UI) gegen ein separat laufendes
Postgres — siehe `appsettings.Development.json` für die lokale Connection-String.

## Troubleshooting

**API kommt nicht hoch, Logs zeigen "could not connect to postgres"**
→ Postgres-Healthcheck hat länger gedauert. `docker compose up -d` nochmal,
oder `docker compose logs postgres` checken. Bei dauerhaftem Fehler: Volume
löschen (`docker volume rm orqestra_postgres-data`) und Start wiederholen —
**Achtung**: löscht alle DB-Daten.

**UI lädt aber API-Calls schlagen mit CORS-Fehler fehl**
→ `UI_PUBLIC_URL` und `API_PUBLIC_URL` in `.env` prüfen. Der API-Container muss
die UI-URL als Allowed Origin kennen. Nach Änderung: `docker compose up -d --force-recreate api`.

**Cookie wird nicht gesetzt nach Login**
→ Du nutzt HTTP statt HTTPS. Cookies sind als Secure markiert in Production.
Reverse Proxy mit TLS aufsetzen oder fürs lokale Testen
`ASPNETCORE_ENVIRONMENT=Development` im Compose setzen (Cookie wird dann
SameAsRequest, also auch ohne HTTPS gesetzt — **nicht für Production**).

**Embedded Templates werden beim Setup nicht gefunden**
→ Das Build-Stage muss den `docs/template-setup`-Ordner enthalten. Im Dockerfile
ist das mit `COPY docs/template-setup/ docs/template-setup/` gewährleistet —
wenn du selbst baust, sicherstellen dass der Build-Context der Repo-Root ist
(`docker build -f Orqestra.API/Dockerfile .`).
