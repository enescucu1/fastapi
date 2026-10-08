# Notizen zur Aufgabe 1: Appserver mit FastAPI

Gruppe 2, GitHub: enescucu1. Zeiterfassung und Projektplan stehen in `SWE_Enes_Cubukcu_Zeiterfassung_Planung.xlsx`.

## Installation (29.9. bis 30.9.2026)

- Python 3.15.0rc2 über den Python Manager (`pymanager install 3.15-dev`)
- uv 0.12.20 über pip, weil der HKA-Proxy `astral.sh` blockiert
- Rust, Cargo und Build Tools für Visual Studio 2022, damit Pakete wie
  `pydantic-core` beim `uv sync` gebaut werden können
- pnpm, Bun, Docker Desktop (per-user), Git, VS Code mit Erweiterungen
- 19 Docker-Images laut Anleitung

## Stolpersteine

- Im HKA-Netz blockiert der Proxy Downloads (`astral.sh`, `get.pnpm.io`,
  `bun.sh`). Lösung: pip bzw. anderes Netz.
- Zwei Tags in der Anleitung waren falsch, der Dozent hat sie im Forum korrigiert:
  `uv:0.12.20-python3.14-dhi` und `sonarqube:26.9.0.129388-community`.
- `C:\workspace` war schon ein Git-Repository. Deshalb hat `fastapi` ein eigenes
  Repository, im alten Repo ausgeblendet über `.git/info/exclude`.

## Projekt starten

1. `uv sync --all-groups` (ca. 4 Minuten, mehrere Pakete werden gebaut)
2. Backend-Server einrichten (PostgreSQL, Keycloak, Mailpit), siehe ReadMe-Dateien
   unter `extras/compose`
3. `uv run patient`

Ohne Backend-Server startet der Appserver, meldet aber "Keine Verbindung zu Keycloak".

## PostgreSQL einrichten (erledigt am 6.10.)

- [x] Named Volumes `pg_data`, `pg_tablespace`, `pg_init` angelegt
- [x] Volumes befüllt (SQL-Skripte, CSV, TLS-Dateien, Tablespace)
- [x] Einmal ohne TLS gestartet, Zertifikat und Schlüssel ins Datenverzeichnis kopiert
- [x] Mit TLS gestartet, Datenbank `patient`, User und Schema angelegt
- [x] Keycloak eingerichtet (siehe unten)
- [x] Mailpit (Volume `mailpit`, siehe unten)

Hinweis: Für den Start ohne TLS musste in `extras/compose/postgres/compose.yml`
temporär `command` und `user` auskommentiert werden. In der Datei fehlte die Zeile
`cap_add`, die die ReadMe erwähnt. Mit `cap_add: [CHOWN, DAC_OVERRIDE, FOWNER,
SETGID, SETUID]` hat es funktioniert. Danach `git restore` auf die Originaldatei.

## Keycloak einrichten (erledigt am 7.10.)

- [x] Datenbank, User und Schema `keycloak` in PostgreSQL angelegt
- [x] Volumes `kc_data` und `kc_tls`, Zertifikat und Schlüssel ins Volume kopiert
- [x] `docker compose up` in `extras/compose/keycloak` startet PostgreSQL und Keycloak
- [x] Im Browser (`https://localhost:8843`): Admin-Benutzer, Realm `python`, Client
  `python-client` mit Rollen `admin` und `patient`, Benutzer `admin`, Token-Zeiten
- [x] Client Secret in `src/patient/config/resources/app.toml` eingetragen. Die Datei ist
  lokal mit `git update-index --skip-worktree` ausgeblendet, weil jedes Teammitglied ein
  eigenes Secret hat und es nicht ins Repository gehört.

Beobachtung: Der Code liest das Secret nur aus `app.toml`, nicht aus einer `.env`.

## Erster Start des Appservers (7.10.)

`uv run patient` scheiterte zuerst mit `SSL error: certificate verify failed` bei der
Verbindung zu PostgreSQL. Ursache: `src/patient/config/resources/postgresql/server.crt`
war ein abgelaufenes Zertifikat (gültig bis 12.07.2026) und nicht das Zertifikat, das
der PostgreSQL-Container ausliefert (`extras/compose/postgres/init/tls/server.crt`,
gültig bis 07.02.2027). Lösung: Datei durch das aktuelle Zertifikat ersetzt.

Danach startet der Server: Tabellen werden angelegt, die CSV-Daten geladen, sechs
Benutzer in Keycloak angelegt, Uvicorn läuft auf `https://127.0.0.1:8000`.

## Bruno (7.10.)

- [x] Bruno-Desktop-App: Collection `extras/bruno/patient` geöffnet, SSL-Prüfung in den
  Preferences ausgeschaltet (selbst-signiertes Zertifikat)
- [x] Environment `patient` mit `clientId`, `username`, `clientSecret` und `password`
  (die beiden letzten als Secret, sie liegen nur lokal unter `%APPDATA%\Bruno`)
- [x] `REST`, `Token`, `Token als admin`: Status 200, Rolle ADMIN. Zuerst war
  `expires_in` 300 statt 1800, weil die Realm-Einstellung (Access Token Lifespan
  30 Minuten) nicht gespeichert war. Nach dem Nachtragen in Keycloak stimmt es.
- [x] `REST`, `Suche mit ID OAuth 2`, `Vorhandene ID 1`: zuerst 401. Das Server-Log zeigte
  `authorization_header=None`, Bruno hat also keinen Token mitgeschickt. Ursache: Der
  Auth-Modus des Requests stand nicht auf `Inherit`. Nach der Umstellung holt Bruno den
  Token über OAuth 2 (Keycloak) selbst, die App liefert Status 200 mit dem Patienten 1.

Hilfreich bei der Fehlersuche: Das Debug-Log der App (`authorization_header=...`) zeigt,
ob und was beim Server ankommt. Es gibt aber auch Tokens und Secrets aus und darf nicht
ungefiltert weitergegeben werden.

## Mailpit (8.10.)

- [x] `cd extras\compose\mailpit`, `docker compose up -d`: Container `mail` läuft
- [x] Weboberfläche auf http://localhost:8025, SMTP auf Port 1025 (in `app.toml` eingetragen)
- [x] Mail "Neuer Patient: ID=1000" von `Python Server` an `Buchhaltung` erscheint nach einem
  erfolgreichen `POST /rest`

Hinweis: Docker meldet `a network with name acme-network exists but was not created for
project "mailpit"`. Das ist harmlos, alle Compose-Projekte teilen sich das Netzwerk.
Beim `docker compose down` erscheint deshalb manchmal `Resource is still in use`, auch das
ist normal.

## Bruno, weitere Requests (8.10.)

| Request | Ergebnis |
|---|---|
| `GET /rest/1` als Admin | 200 |
| `POST /graphql` (Patient 30, alice) | 200 |
| `POST /rest` neuer Patient | 201, Mail in Mailpit |
| `POST /rest` mit vorhandener E-Mail (`alice@acme.de`) | 422 |
| `POST /rest` mit ungültigen Daten | 422 |
| `PUT /rest/30` | 204, Version von 0 auf 1 |
| `DELETE /rest/50` | 204 |

## Tests mit pytest (8.10.)

**Unit-Tests:** `uv run pytest tests/unit` ergibt 14 passed, 1 skipped.

**Integrationstests:** `uv run pytest tests/integration` ergibt **70 passed** in ca. 38 s.
Voraussetzung: PostgreSQL, Keycloak, Mailpit und der Appserver laufen. Zwei Fehler im
Code des Dozenten mussten dafür behoben werden:

1. `tests/integration/security/conftest.py` importierte `ctx` aus `common_api_test`, das dort
   nicht mehr existiert (ImportError). Die Tests wurden offenbar von einem httpx-SSL-Kontext
   auf einen Zertifikatspfad umgestellt. Fix: `certificate_path` importieren und
   `AsyncHTTPTransport(verify=certificate_path)` verwenden (Commit d60e650).
2. 67 Errors mit `ReadTimeout` bei `/dev/keycloak_populate`: Das Session-Fixture ruft diesen
   Endpunkt mit `timeout = 2` Sekunden auf, das Neuladen von Keycloak dauert bei mir aber
   ca. 3 Sekunden. Fix in `tests/integration/common_api_test.py`: die auskommentierte Zeile
   `timeout: Final = 5` aktiviert (Commit 0d9cb33).

Bekannte harmlose Meldungen:

- `CoverageWarning: No data was collected`: der Server läuft in einem eigenen Prozess, daher
  sieht pytest-cov keinen Code.
- `DeprecationWarning` von httpx (`verify=<str>` ist veraltet).

## Codeanalyse (8.10.)

- ruff: 1 Stilhinweis in `tests\integration\security\conftest.py`
- ty: 1 Meldung (`ctx` fehlt), durch den conftest-Fix behoben
- Pyrefly: 0 Diagnostics

## Versionen (verifiziert am 8.10.)

FastAPI 0.143.0, Starlette 1.7.0, uvicorn 0.54.0, Strawberry 0.332.0, SQLAlchemy 2.1.4,
psycopg 3.3.6, Pydantic 2.13.5, Python 3.15.0rc2, Keycloak 26.7.4, PostgreSQL 19beta4,
Mailpit v1.31.3, cryptography 50.0.2 (aktualisiert am 8.10., siehe Sicherheitsprüfung).

## Sicherheitsprüfung der Abhängigkeiten (8.10.)

**uv audit** meldete anfangs eine Lücke in `cryptography` 49.0.0 (GHSA-g6cj-pr64-35w5,
PKCS#7-Entschlüsselung, behoben in 50.0.0; die zweite Meldung PYSEC-2026-3552 ist dieselbe Lücke).
`cryptography` kam über `jwcrypto` und `python-keycloak` ins Projekt, `pyproject.toml` hatte aber die
Obergrenze `<50.0.0`. Vorgehen:

- Zeile in `pyproject.toml` auf `"cryptography>=50.0.0,<51.0.0"` geändert
- `uv lock --upgrade-package cryptography` und `uv sync --all-groups`: Update auf 50.0.2
- `uv audit`: keine bekannten Schwachstellen mehr
- Unit-Tests (14 passed, 1 skipped) und Integrationstests (70 passed) weiter grün (Commit 36a3389)

`uv audit --all-extras` aus der VORGEHENSWEISE gibt es in dieser uv-Version nicht (`unexpected argument`),
Extras werden von `uv audit` ohnehin geprüft (152 Pakete).

**Veraltete Pakete** (`uv tree --outdated --all-groups --depth=1`): nur `pydantic` 2.13.5 (neu: 2.14.0) und
`pydantic-core` 2.46.5 (neu: 2.50.0). Nicht aktualisiert, weil beide eng gekoppelt sind, die Versionsgrenzen
vom Dozenten stammen und `uv audit` keine Lücke dazu meldet.

**OWASP Dependency Check** (Version 13.0.0):

- NVD-API-Key von https://nvd.nist.gov/developers/request-an-api-key angefordert und in `.env` als
  `NVD_API_KEY` eingetragen (die Datei ist in `.gitignore`). In der ausgelieferten `.env` stand nur ein
  Platzhalter, damit schlug das Datenbank-Update mit `Invalid API Key` fehl.
- `uv run extras/dependency-check.py`: Bericht mit `Dependencies Scanned: 1`, also nicht aussagekräftig
  (das Skript scannt `..` und erkennt die Python-Pakete nicht). Außerdem gibt das Skript den API-Key im
  Klartext in der Konsole aus, die Ausgabe nicht weitergeben.
- Ergänzender Scan der installierten Pakete (Bericht im Temp-Ordner, nicht im Repository):

```powershell
$key = ((Get-Content .env | Select-String 'NVD_API_KEY').Line -split '=',2)[1].Trim('"')
& C:\Zimmermann\dependency-check\bin\dependency-check.bat `
  --nvdApiKey $key --project FastAPI-venv --scan .\.venv\Lib\site-packages `
  --suppression extras\suppression.xml --out "$env:TEMP\odc-venv" --data C:\Zimmermann\dependency-check-data `
  --disableAssembly --disableOssIndex --disableNodeJS --disableNodeAudit `
  --disableYarnAudit --disablePnpmAudit --disableJar --disableCentral
```

  Ergebnis: 86 Dateien gescannt. Ohne Suppression 1 Fund: `underscore.js` 1.8.3 in `wordcut.js`
  (`.venv\Lib\site-packages\material\templates\assets\javascripts\lunr\wordcut.js`, gehört zu
  mkdocs-material, läuft nur im Browser bei der Suche der Dokumentation), 2 CVEs, Schweregrad HIGH.
  Mit `extras/suppression.xml` des Dozenten: 0 Funde, 2 unterdrückt.

## Starten und Beenden

Reihenfolge: `extras\compose\keycloak` (`docker compose up`, PostgreSQL und Keycloak),
`extras\compose\mailpit` (`docker compose up -d`), dann im Projektordner `uv run patient`
und auf `Application startup complete` warten. Beenden: Appserver mit Strg+C,
`docker compose down` in beiden Ordnern. Die Daten bleiben in den Volumes erhalten.

Adressen: Swagger UI `https://127.0.0.1:8000/docs`, GraphQL `https://127.0.0.1:8000/graphql`,
Keycloak `https://localhost:8843`, Mailpit `http://localhost:8025`.

## Offen

- [ ] Bruno: Ordner Bearer Token (nicht separat dokumentiert)
- [ ] SonarQube (`sonar-scanner.py`, `sonar-project.properties`, `extras/compose/sonarqube`)
- [ ] mkdocs mit PlantUML-Diagrammen
- [ ] Docker-Image (Dockerfile mit Hardened Image) und Docker Compose
- [ ] GitHub Actions (ruff, ty bzw. Pyrefly)
- [ ] Lasttests mit Locust
- [ ] Code-Review und Abgabe
- [ ] Forum: Hinweis an den Dozenten zu `conftest.py` (ctx), Timeout 2 s, Obergrenze `cryptography<50.0.0`,
  `uv audit --all-extras` und `Dependencies Scanned: 1` beim Skript `extras/dependency-check.py`
