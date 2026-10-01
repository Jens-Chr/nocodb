# NocoDB für „Sali hüpft“

Eigenständiger Self-Hosting-Stack im Repository `Jens-Chr/nocodb` mit eigenem
Docker Compose, Datenvolumes sowie Backup- und Upgrade-Lifecycle. NocoDB gehört
weder zu `twitch-appstack-deployment` noch zum bestehenden n8n-Compose.
Der Pilot ist auf dem Docker-Host unter <http://localhost:8081> erreichbar.

**CE-Einschränkung:** Der gewünschte separate Worker-Container ist vorbereitet.
Beim Laufzeittest des offiziellen Images `2026.09.0` startet er im CE-Modus
auch mit gesetztem Worker-Schalter einen HTTP-Server. Ein ausschließlich Jobs
verarbeitender Worker ist mit dieser Konfiguration daher nicht zugesichert;
siehe die Versionshinweise und Testergebnisse unten.

## Architektur

| Compose-Service | Image | Aufgabe | Netzwerke |
| --- | --- | --- | --- |
| `nocodb` | `nocodb/nocodb:2026.09.0` | Community Edition, Web/API | `nocodb-internal`, externes `n8n-network` |
| `nocodb-worker` | `nocodb/nocodb:2026.09.0` | Hintergrundverarbeitung nach offiziellem Worker-Aufbau | `nocodb-internal` |
| `nocodb-postgres` | `postgres:17.10` | Primäre Datenbank für Metadaten und Nutzdaten | `nocodb-internal` |
| `nocodb-redis` | `redis:7.4.8-alpine` | Cache und Job-Infrastruktur | `nocodb-internal` |

Der Compose-Projektname ist `sali-nocodb`. Compose generiert eindeutige Namen wie
`sali-nocodb-nocodb-1`; feste globale `container_name`-Namen sind nicht nötig.
Netzwerk und Volumes erhalten ebenfalls den Projektpräfix. Den Projektnamen im
Betrieb beibehalten, sonst verwendet Compose andere Volumes.

`nocodb-internal` wird als `sali-nocodb_nocodb-internal` mit `internal: true`
angelegt. Alle vier Dienste hängen darin. Nur Web/API hängt zusätzlich am
bereits bestehenden `n8n-network`, das vom separaten n8n-/Traefik-Stack verwaltet
wird. Dieses Compose legt das externe Netz niemals an und löscht es auch nicht.
PostgreSQL, Redis und Worker haben weder Host-Portfreigaben noch Zugang zum
gemeinsamen Netzwerk. Web/API veröffentlicht ausschließlich
`127.0.0.1:8081:8080`. Teilnehmer des gemeinsamen Netzes können Web/API trotzdem
direkt erreichen; localhost begrenzt die **Host-Portfreigabe**.

Für spätere n8n-Aufrufe ist der eindeutige Netzwerkalias
`http://sali-nocodb:8080` vorbereitet. API-Authentifizierung ist dabei weiterhin
erforderlich. Das interne Netz bietet dem Worker keinen Internetzugang;
URL-Imports, Airtable-Import oder ausgehende Worker-Webhooks benötigen später
eine bewusst geplante Egress-Lösung, ohne Worker/DB/Redis ins `n8n-network`
aufzunehmen.

## Konfiguration und offizielle Grundlagen

Vor der Implementierung geprüft am 1. Oktober 2026:

- [Offizieller Quickstart](https://nocodb.com/docs/self-hosting/installation/quickstart)
- [Compose-Beispiel im Tag 2026.09.0](https://github.com/nocodb/nocodb/blob/2026.09.0/docker-compose/examples/quickstart-demo/docker-compose.yml)
- [Aktuelle Environment-Variablen](https://nocodb.com/docs/self-hosting/environment-variables)
- [Redis-Variablen im Release-Quellcode](https://github.com/nocodb/nocodb/blob/2026.09.0/packages/nocodb/src/helpers/redisHelpers.ts)

Web/API und Worker verwenden denselben Environment-Block:

| Variable | Verwendung |
| --- | --- |
| `NC_DB` | `pg://nocodb-postgres:5432?u=nocodb&p=<Passwort aus .env>&d=nocodb`; PostgreSQL, kein SQLite-Fallback |
| `NC_CACHE_REDIS_URL` | Authentifizierter Cache auf `nocodb-redis:6379`, logische DB `0` |
| `NC_JOBS_REDIS_URL` | Dieselbe authentifizierte Redis-Instanz, logische DB `1` für Jobs |
| `NC_AUTH_JWT_SECRET` | Gemeinsames, dauerhaftes Secret für Authentifizierung |
| `NC_APP_DATA_DIR` | `/usr/app/data`, auf beiden Containern dasselbe Volume |
| `NC_SITE_URL` | Im Pilot `http://localhost:8081`, später die HTTPS-Adresse |

Web/API setzt wie das Release-Beispiel `NC_DISABLE_MUX=true`; der Worker setzt
den aktuell dokumentierten Schalter `NC_WORKER_MODE_ENABLED=true` und zusätzlich
den Alias `NC_WORKER_CONTAINER=true`: Die [Cache-Initialisierung in 2026.09.0](https://github.com/nocodb/nocodb/blob/2026.09.0/packages/nocodb/src/cache/RedisCacheMgr.ts)
prüft noch ausdrücklich den Alias, um Cache-Resets im Worker zu unterdrücken.
Auch das Release-Beispiel verwendet diesen Alias.
Die Dokumentation kennzeichnet den separaten Worker-/Redis-Job-Modus als
On-Premise-Funktion; die [Community-Quellen im Release](https://github.com/nocodb/nocodb/blob/2026.09.0/packages/nocodb/src/utils/envs.ts)
setzen `isWorker` und `isMuxEnabled` fest auf `false` und enthalten auch eine
lokale Fallback-Queue. Im getesteten offiziellen Image bedienen sowohl der alte
als auch der neue Worker-Schalter im CE-Modus weiterhin HTTP. Der vierteilige
Aufbau folgt dem offiziellen Image-Beispiel und aktiviert keine kostenpflichtige
Lizenz. Vor produktiver Nutzung die tatsächliche Job-Verteilung prüfen und
die gewünschte Worker-Trennung mit den CE-Funktionen abgleichen.
Es werden keine API-Rate-Limit-Variablen gesetzt; die Standardwerte bleiben erhalten.

Redis verlangt ein eigenes Passwort und schreibt AOF-Dateien mit
`appendfsync everysec` ins Redis-Volume. Cache und Jobs verwenden getrennte
logische Redis-Datenbanken, damit ein Cache-Reset die Job-Queue nicht leert.
Web/API und Worker verwerfen alle
Linux-Capabilities. Alle Dienste setzen `no-new-privileges:true` und
`restart: unless-stopped`. PostgreSQL und Redis erhalten ausschließlich `CHOWN`,
`DAC_OVERRIDE`, `FOWNER`, `SETGID` und `SETUID` für ihre offiziellen Entrypoints
(Volume-Initialisierung und Benutzerwechsel); alle übrigen Capabilities entfallen.

## Voraussetzungen

- Docker Engine und Docker Compose v2 oder neuer (`docker compose version`).
- Berechtigung für den Docker-Daemon.
- Bereits vorhandenes externes Docker-Netzwerk `n8n-network`:

  ```sh
  docker network inspect n8n-network
  ```

Fehlt das Netz auf einem lokalen Rechner, muss es vor dem Start durch die lokale
Infrastruktur bereitgestellt werden. Auf dem Server wird es vom bestehenden
n8n-Stack bereitgestellt. Dieses Repository erzeugt keinen Ersatz für das Netz
und startet keinen eigenen Traefik.

## Ersteinrichtung

Alle Befehle im Verzeichnis dieses Repositories ausführen:

```sh
cp .env.example .env
chmod 600 .env
openssl rand -hex 32
```

Den Generator **dreimal** ausführen und die drei unterschiedlichen Werte in
`POSTGRES_PASSWORD`, `REDIS_PASSWORD` und `NC_AUTH_JWT_SECRET` in `.env` eintragen.
Es gibt keine vorgegebenen Passwörter. Leere Secrets verhindern bereits die
Compose-Validierung. Für die Passwörter Hex-Werte verwenden: Sie sind direkt in
den DB-/Redis-Verbindungs-URLs verwendbar. Andere Sonderzeichen müssten korrekt
URL-kodiert werden.

`.env` ist ignoriert und darf nicht committed werden. Geheimnisse werden über
Container-Environment bereitgestellt; Docker-Administratoren können diese
einsehen. `docker compose config` gibt die aufgelösten Secrets aus, deshalb
Ausgaben nicht öffentlich teilen. Für eine Prüfung ohne Ausgabe `-q` verwenden.
Eine Änderung von `POSTGRES_PASSWORD` in `.env` ändert bei einem bestehenden
Volume **nicht** automatisch das Datenbankpasswort; die Rotation muss in
PostgreSQL und der Anwendung abgestimmt erfolgen.

## Betrieb

```sh
# Konfiguration prüfen
docker compose config -q

# Start
docker compose up -d

# Status und Healthzustände
docker compose ps

# Logs aller Dienste
docker compose logs -f

# Health-Endpunkt und Oberfläche
curl -fsS http://localhost:8081/api/v1/health
curl -fsS -o /dev/null http://localhost:8081/

# Stoppen, Daten behalten
docker compose down
```

Im Browser <http://localhost:8081> öffnen und den ersten Benutzer mit einem
sicheren Passwort registrieren; dieser wird Super-Admin. Bei einem entfernten
Server ist die localhost-Freigabe über einen SSH-Tunnel erreichbar, zum Beispiel
`ssh -L 8081:127.0.0.1:8081 user@server`.

PostgreSQL wird mit `pg_isready`, Redis mit authentifiziertem `redis-cli ping`
und Web/API mit dem offiziellen Endpunkt `/api/v1/health` geprüft. Web/API wartet
auf gesunde DB und Redis; der Worker wartet zusätzlich auf gesundes Web/API.
Für den Worker ist kein HTTP-Healthcheck konfiguriert: Ein HTTP-Erfolg weist
keine Job-Verarbeitung nach und wäre bei einem reinen Worker ohne HTTP-Server
ungeeignet. Seine Logs und erfolgreich abgeschlossene Hintergrundjobs sind
zusätzlich zum Containerstatus zu prüfen.

## Persistenz

| Docker-Volume bei Standard-Projektname | Mount | Inhalt |
| --- | --- | --- |
| `sali-nocodb_postgres-data` | `/var/lib/postgresql/data` | PostgreSQL-Metadaten und Nutzdaten |
| `sali-nocodb_redis-data` | `/data` | Redis-AOF und Snapshots |
| `sali-nocodb_nocodb-data` | `/usr/app/data` | Gemeinsame NocoDB-Dateien, Uploads und Attachments |

Es werden ausschließlich eigene Named Volumes verwendet, keine Verzeichnisse
anderer Repositories. `docker compose down` entfernt Container und das eigene
interne Netz; Volumes und externes `n8n-network` bleiben erhalten.
**`docker compose down -v` löscht die eigenen Datenvolumes** und ist kein normaler
Stop-Befehl.

## Backup

Für einen zusammengehörigen Stand Schreibzugriffe unterbrechen und Web/API und
Worker stoppen. PostgreSQL bleibt für `pg_dump` aktiv. Redis wird für das
Dateibackup sauber gestoppt. Das folgende Beispiel benötigt eine POSIX-Shell:

```sh
umask 077
backup_dir="backups/$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$backup_dir"
docker compose stop nocodb nocodb-worker nocodb-redis

docker compose exec -T nocodb-postgres \
  pg_dump -U nocodb -d nocodb --format=custom > "$backup_dir/postgres.dump"

docker compose run --rm --no-deps -T --entrypoint tar nocodb \
  -czf - -C /usr/app/data . > "$backup_dir/nocodb-data.tar.gz"

docker compose run --rm --no-deps -T --entrypoint tar nocodb-redis \
  -czf - -C /data . > "$backup_dir/redis-data.tar.gz"

docker compose up -d
```

Jeden Befehl auf erfolgreichen Abschluss prüfen; unvollständige Dateien sind
kein gültiges Backup. Zusätzlich Compose-Datei, verwendete Image-Versionen und
`.env` getrennt und geschützt sichern. Backups extern und verschlüsselt ablegen;
`backups/` ist in Git ignoriert. Regelmäßig eine Wiederherstellung in einer
separaten Umgebung erproben. PostgreSQL und Attachments sind gemeinsam nötig;
das Redis-Backup erhält zusätzlich den Cache-/Queue-Stand, soweit verwendet.

## Restore-Grundprinzip

In eine **leere, separate Wiederherstellungsumgebung** mit derselben
NocoDB-Version und PostgreSQL-Major-Version restaurieren. Geschützte `.env`
wiederherstellen, externes Netz bereitstellen und nur PostgreSQL starten:

```sh
docker compose up -d --wait nocodb-postgres
docker compose exec -T nocodb-postgres \
  pg_restore -U nocodb -d nocodb --exit-on-error < /path/to/backup/postgres.dump

docker compose run --rm --no-deps -T --entrypoint tar nocodb \
  -xzf - -C /usr/app/data < /path/to/backup/nocodb-data.tar.gz

docker compose run --rm --no-deps -T --entrypoint tar nocodb-redis \
  -xzf - -C /data < /path/to/backup/redis-data.tar.gz

docker compose up -d
docker compose ps
curl -fsS http://localhost:8081/api/v1/health
```

Web/API, Worker und Redis dürfen während des Restores nicht laufen. Die
Dateiarchive nur in leere Volumes entpacken, um keine alten Dateien zu vermischen.
Danach Anmeldung, Datensätze, Attachments und Hintergrundjobs überprüfen.
Ein Restore in bestehende Daten benötigt einen gesonderten Plan zum Ersetzen
der Daten und wird durch diese Befehle nicht abgedeckt.

## Upgrade

Vor jedem Upgrade ein geprüftes Backup erstellen und die offiziellen
Release-/Migrationshinweise lesen. Den NocoDB-Tag bewusst an der gemeinsamen
Image-Definition `x-nocodb-common` in `docker-compose.yml` ändern, damit Web/API
und Worker immer dieselbe Version verwenden. Dann:

```sh
docker compose config -q
docker compose pull && docker compose up -d
docker compose ps
curl -fsS http://localhost:8081/api/v1/health
```

Logs und Anwendung inklusive Attachments/Jobs prüfen. PostgreSQL und Redis
haben ihren eigenen Upgrade-Lifecycle. Ein PostgreSQL-Major-Upgrade benötigt
`pg_upgrade` oder Dump/Restore und darf nicht durch bloßes Wechseln des Tags
mit dem alten Datenvolume erfolgen. Ein NocoDB-Downgrade nach DB-Migrationen
benötigt gegebenenfalls den Restore des Backups mit dem vorherigen Image.

## Validierung dieses Stacks

Am 1. Oktober 2026 mit Docker Engine `29.8.1` und Compose `v5.5.1` geprüft:

- `docker compose config` mit einer temporären Secret-Datei; zusätzlich
  Netzwerkzuordnung, localhost-Portbindung, gemeinsame Konfiguration,
  Image-Pins, Volumes, Capabilities und Ablehnung leerer Secrets geprüft.
- Alle gepinnten Images erfolgreich geladen und den Stack gestartet.
  `docker compose ps`: Web/API, PostgreSQL und Redis `healthy`, Worker `running`.
- Oberfläche und `/api/v1/health` vor und nach Neustart mit HTTP `200` erreichbar.
- Redis lehnt unauthentifizierten Zugriff ab; AOF ist aktiv. Jobs verwenden
  tatsächlich Redis-DB `1`; Testdatei und Redis-Schlüssel bleiben nach Neustart
  erhalten.
- PostgreSQL-Erstinitialisierung mit reduzierten Capabilities auf einem leeren
  temporären Dateisystem erfolgreich.
- PostgreSQL-Dump sowie Daten-/Redis-Archive erstellt und gelesen.
  PostgreSQL-Dump erfolgreich in eine separate Testdatenbank restauriert.
  Ein vollständiger Anwendungsrestore inklusive Anmeldung/Attachments wurde
  damit noch nicht erprobt.
- Stack mit `docker compose down` gestoppt; alle drei Testvolumes bleiben
  als `sali-nocodb-validation_*` erhalten.

In der Testumgebung war `n8n-network` nicht vorhanden. Deshalb wurde ausschließlich
im separaten Projekt `sali-nocodb-validation` ein temporäres externes Testnetz
per Override verwendet. Compose ließ es beim Stoppen erhalten; anschließend
wurde nur dieses Testnetz entfernt. Das echte `n8n-network` wurde weder angelegt
noch verändert. Die CE-Worker-Einschränkung oben bleibt ein offener Punkt;
der erfolgreiche Containerstart belegt keine exklusive Worker-Verarbeitung.

## Spätere Schritte

- Produktives HTTPS über den vorhandenen Traefik: dessen Docker-Provider und
  automatische Exponierung prüfen, explizite Route/Netzwerkauswahl ergänzen und
  `NC_SITE_URL` auf die HTTPS-Domain setzen. Im Pilot gibt es keine
  Traefik-Labels, Domain-Routen oder Authentik-Integration. Der bestehende Proxy
  muss unlabeled Container ignorieren, damit der Pilot nicht automatisch über
  eine Default-Route veröffentlicht wird.
- MCP: erreichbaren HTTPS-Endpunkt und die für Community Edition verfügbaren
  Werkzeuge/Berechtigungen gesondert konfigurieren und testen.
- Airtable-Import: Datenmodell, Umfang und Anhänge planen; Egress und tatsächliche
  Job-Verarbeitung vor dem Import testen.
- n8n: API-Token mit passenden Rechten hinterlegen, Aufrufe über
  `http://sali-nocodb:8080` prüfen und Standard-Rate-Limits berücksichtigen.
