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

**Integrationstests:** `uv run pytest tests/integration` ergibt **70 passed** in ca. 35 bis
38 s. Voraussetzung: PostgreSQL, Keycloak, Mailpit und der Appserver laufen. Zwei Fehler im
Code des Dozenten mussten dafür behoben werden:

1. `tests/integration/security/conftest.py` importierte `ctx` aus `common_api_test`, das dort
   nicht mehr existiert (ImportError). Ich hatte es mit `certificate_path` repariert
   (Commit d60e650). Laut Dozent (Forum) wird die Datei nicht mehr benötigt, sie ist
   gelöscht. Danach weiterhin 70 passed (12.10.).
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
  (`pytest-incorrect-pytest-import`), durch das Löschen der Datei erledigt
- ty: 1 Meldung (`ctx` fehlt), durch den conftest-Fix behoben
- Pyrefly: 0 Diagnostics

## uv audit (8.10.)

- Fund: `cryptography` 49.0.0, GHSA-g6cj-pr64-35w5 (Bleichenbacher-Orakel in PKCS#7),
  behoben in 50.0.0.
- Lösung: in `pyproject.toml` `"cryptography>=50.0.0,<51.0.0"`, `uv lock --upgrade-package
  cryptography`, Update auf 50.0.2. Danach `uv audit` ohne Fund (Commit 36a3389).
- Veraltete Pakete: pydantic 2.13.5 (neu: 2.14.0) und pydantic-core 2.46.5 (neu: 2.50.0).
  Nicht aktualisiert: keine Sicherheitslücke, enge Kopplung der beiden Pakete, Pins vom
  Dozenten. (Pydantic wurde am 21.10. mit dem `uv.lock`-Update auf 2.14.1 gehoben, siehe
  unten.)
- `uv audit --all-extras` gibt es in dieser uv-Version nicht, es werden ohnehin alle
  152 Pakete geprüft.

## OWASP Dependency Check 13.0.0 (8.10.)

- NVD-API-Key bei nvd.nist.gov angefordert, in `.env` als `NVD_API_KEY` (in `.gitignore`).
  Der Platzhalter-Key aus der Vorlage schlägt fehl, es braucht einen echten Key.
- `uv run extras/dependency-check.py` scannt nur 1 Datei (das Skript scannt `..` und
  erkennt keine Python-Pakete), das Ergebnis sagt deshalb nichts aus.
- Ergänzender Scan von `.venv\Lib\site-packages` mit `--suppression extras\suppression.xml`:
  86 Dateien, 0 Vulnerable, 2 Suppressed. Einziger Fund: `underscore.js` 1.8.3 in
  `wordcut.js` (gehört zu mkdocs-material, läuft nur im Browser), CVE-2021-23358 und
  CVE-2026-27601, laut `suppression.xml` des Dozenten unterdrückt.

## SonarQube (11.10.)

- [x] `extras\compose\sonarqube`: `docker compose up`, Weboberfläche auf http://localhost:9000
- [x] Passwort geändert, Global Analysis Token `HKA` erzeugt und in
  `sonar-project.properties` eingetragen (Datei ist in `.gitignore`)
- [x] `uv run sonar-scanner.py`: `ANALYSIS SUCCESSFUL`

Stolperstein beim Start: Der Container startete in einer Endlosschleife neu mit
`Found two versions of the plugin ... [javascript]`. Im Bind-Mount
`C:\Zimmermann\volumes\sonarqube\extensions` lag noch `sonar-javascript-plugin-11.4.1`,
das Image bringt 14.0.0 mit. Lösung: `docker compose down`, das alte Jar gelöscht,
neu gestartet. Die Meldung zu `vm.max_map_count` kam nicht.

Ergebnis (Overall Code): Quality Gate Passed, je 0 Issues bei Security, Reliability und
Maintainability (Rating A), 0 Security Hotspots, Duplications 0,0 %, Coverage 0,0 %
(keine Coverage-Reports eingebunden, die Tests laufen separat mit pytest).

**Einschränkungen:**

- Das Python-Plugin 5.32.0 kann die Python-3.15-Syntax `lazy from ... import` nicht
  parsen. Ca. 31 von 70 Dateien wurden nicht analysiert (`Parse error`), in den Metriken
  stehen 868 statt ca. 1955 Zeilen. Das Ergebnis gilt nur für die geparsten Dateien.
- Die Community Build prüft keine Injection-Schwachstellen (SQL-Injection, XSS).
- Ergänzend decken ruff, ty, Pyrefly, `uv audit` und OWASP Dependency Check den gesamten
  Code bzw. alle Abhängigkeiten ab.

## GitHub Actions (11.10.)

- [x] `.github/workflows/ci.yml`: bei Push auf `main` und bei Pull Requests laufen
  `ruff check --preview`, `ruff format --preview --check`, `ty check` und
  `pyrefly check` (Python 3.15 über `uv python install`, Abhängigkeiten mit
  `uv sync --all-groups`)
- [x] Vorher lokal geprüft: ruff format (91 Dateien ok), ty und Pyrefly ohne Fund
- [x] ruff meldete in `tests\integration\security\conftest.py` einen Hinweis
  (`pytest-incorrect-pytest-import`, `from pytest import fixture`), den ich zunächst in
  dieser Zeile unterdrückt hatte. Die Datei wurde später auf Anweisung des Dozenten
  gelöscht.
- Ergebnis des ersten Laufs im Tab Actions: grün
- Lauf zum Commit `f64ee53` (26.10.): grün

## Docker (11.10. bis 12.10.)

Im Repo liegen `Dockerfile` (Hardened), `Dockerfile.trixie`, `Dockerfile.alpine`,
`docker-bake.hcl` und `extras/compose/patient/compose.yml`.

**Hardened-Image funktioniert mit Python 3.15 nicht:**

- hadolint für `Dockerfile`: keine Beanstandung.
- `docker build` bricht im Builder bei `uv sync --frozen` ab:
  `error: No interpreter found for Python >=3.15 in search path`.
- Ursache: Builder (`uv:0.12.20-python3.14-dhi`) und Final-Image
  (`dhi.io/python:3.14.7-debian13`) haben nur Python 3.14, das Projekt verlangt
  `requires-python = ">=3.15"`. Das Dockerfile setzt `UV_NO_MANAGED_PYTHON=true` und
  `UV_PYTHON_DOWNLOADS=0`, uv darf also kein Python nachladen.
- `docker manifest inspect dhi.io/python:3.15-debian13` liefert `unknown:`, während
  `3.14.7-debian13` ein gültiges Manifest liefert (Login funktioniert). Den Tag für 3.15
  gibt es bei dhi.io also derzeit nicht.

**Trixie-Variante funktioniert:**

- hadolint für `Dockerfile.trixie`: 1 Warnung DL4006 (Zeile 63, `pipefail` vor `RUN` mit
  Pipe), nicht behoben.
- `docker buildx bake trixie`: Build erfolgreich in ca. 165 s, Image
  `juergenzimmermann/patient:2026.10.1-trixie` (365 MB, Python 3.15.0rc2).
- In `extras/compose/patient/compose.yml` ist dafür die Zeile `image:` auf den Tag
  `2026.10.1-trixie` umgestellt (Hardened-Zeile auskommentiert).
- `extras/compose/patient/app.toml` enthält das Client Secret und ist lokal mit
  `git update-index --skip-worktree` ausgeblendet.
- `docker compose up` in `extras/compose/patient` startet PostgreSQL, Mailpit, Keycloak
  und den Appserver: DB und Keycloak werden neu geladen, Banner zeigt Python 3.15.0rc2,
  `Application startup complete`, Uvicorn läuft auf `https://0.0.0.0:8000`.

Vor dem Start mit Compose müssen der lokale Appserver sowie die Compose-Projekte
`keycloak` und `mailpit` beendet sein (Port 8000, Backend kommt per `include`).

**Test gegen den Container (12.10.):** Bruno-Requests gegen `https://127.0.0.1:8000`
(Compose-Stack mit Trixie-Image): Token 200, `GET /rest/1` 200, `POST /rest` 201 (Mail
in Mailpit) und 422 (vorhandene E-Mail, ungültige Daten), `PUT /rest/30` 204 (Version
0 auf 1), `DELETE /rest/50` 204, `POST /graphql` 200. Der Container startet mit
`User app` und `cap_drop: [ALL]`. Beim Start werden DB und Keycloak neu geladen.

## Lasttests mit Locust (12.10.)

Vorbereitung laut `tests/lasttest/README.md`: Im Container `LOGURU_LEVEL=INFO` in
`extras/compose/patient/.env` aktiviert, in `app.toml` `log-statements` und
`graphiql-enabled` auskommentiert, danach `docker compose restart patient`.
Aufruf: `cd tests\lasttest`, `uvx locust`, im Browser `http://localhost:8089` mit
50 Users, Ramp up 5, Host `https://localhost:8000`.

**Fehler im Skript des Dozenten:** Beim ersten Lauf brach `on_start` bei allen 50 Users mit
`KeyError: 'token'` ab. Ursache: Das Skript schickt `POST /auth/token`, der Endpunkt
erwartet aber die HTTP-Methode `QUERY` (wie bei Bruno). Fix in `locustfile.py`:
`self.client.request("QUERY", url="/auth/token", json=...)`. Der Handtest mit
`curl -X QUERY` lieferte 200 mit dem Feld `token`, die Zugangsdaten `admin` / `p` stimmen.

**Ergebnis** (50 Users, ca. 3,6 Minuten Laufzeit):

| Kennzahl | Wert |
|---|---|
| Requests gesamt | 5992 |
| Fehler | 0 |
| Median | 110 ms |
| 95. Perzentil | 260 ms |
| 99. Perzentil | 310 ms |
| Durchsatz | 27,6 Requests/s |
| Langsamster Request | 895 ms (Login) |

Der Durchsatz entspricht dem, was das Skript vorgibt (`constant_throughput(0.1)` bei
50 Users, also 5 Durchläufe pro Sekunde mit je 5 bis 6 Requests). Der Server war also
nicht am Limit. Der Login über Keycloak ist mit 470 ms Median am langsamsten, er läuft
nur einmal pro User. `/rest?nachname=a` braucht 250 ms, der Buchstabe trifft die meisten
Patienten.

**Einordnung:** Alles läuft auf einem Notebook, virtualisiert in Containern, mit HTTPS.
Appserver, PostgreSQL, Keycloak, Mailpit und Locust teilen sich dieselbe Hardware. Das ist
ein Last-, kein Stresstest, die Belastungsgrenze wurde nicht gesucht.

Nach dem Test `.env` und `app.toml` wieder auf den Entwicklungszustand zurückgesetzt.

## mkdocs und PlantUML (12.10.)

- [x] `uv run mkdocs build` und `uv run mkdocs serve` laufen durch
- Das Plugin `build_plantuml` rendert die PlantUML-Diagramme über plantuml.com
- `strict: true`, d. h. jede Warnung bricht den Build ab
- `mkdocstrings` erzeugt die API-Doku aus den Docstrings im Sphinx-Stil

## Library Skills (12.10.)

- [x] `uvx library-skills --claude --copy`: FastAPI-Skill installiert
- `--copy` statt Symlinks, weil Windows die Symlinks blockiert
- Dateien: `.agents\skills\fastapi\SKILL.md` und `.claude\skills\fastapi\SKILL.md`
- Committet und gepusht

## Code-Review

- Tool: <z. B. Claude Code / manuelles Review / AgentVSCode>
- Datum und Dauer: <z. B. 22.10., 1 h - auch über Tage hinweg>
- Ergebnis: ist ok

## Versionen (verifiziert am 26.10. aus dem Server-Banner)

| Komponente | Version |
|---|---|
| Python | 3.15.0rc2 |
| uv | 0.13.4 |
| FastAPI | 0.162.0 |
| Starlette | 1.9.0 |
| uvicorn | 0.54.0 |
| Pydantic | 2.14.1 |
| Strawberry | 0.336.2 |
| SQLAlchemy | 2.1.5 |
| psycopg | 3.3.6 |
| cryptography | 50.0.2 |
| python-keycloak | 7.1.1 |
| pytest | 9.1.1 |
| Ruff / ty / Pyrefly | 0.17.1 / 0.0.90 / 1.4.0 |
| locust | 2.46.7 |
| Keycloak (Image) | 26.7.4 |
| PostgreSQL (Image) | 19beta4 |
| Mailpit (Image) | v1.31.3 |

## Update (21.10. und 26.10.)

**Durchgeführt:**

- 21.10.: `uv.lock` aktualisiert (FastAPI 0.156.5, Starlette 1.9.0, Pydantic 2.14.1,
  SQLAlchemy 2.1.5, Strawberry 0.336.2).
- 26.10.: `uv lock --upgrade-package fastapi` auf 0.162.0. `uv lock --upgrade-package
  brotli` auf 1.2.1, weil `uv audit` PYSEC-2026-4322 (Out-of-bounds-Zugriff) meldete.
  Brotli kommt über Locust und geventhttpclient ins Projekt. Danach meldet `uv audit`
  keine bekannten Lücken. Commit `f64ee53`, gepusht.
- Prüfung nach dem Update: pytest 84 passed, 1 skipped (19,5 s), ruff und ty ohne Fund,
  Pyrefly 0 Diagnostics, `mkdocs build` in 4,1 s ohne Warnung.

**Bewusst nicht aktualisiert:**

- Python bleibt bei 3.15.0rc2. Die rc2 erfüllt `>=3.15`, bis zur Final gibt es Fixes bei
  Lazy Imports, aber kurz vor der Abgabe will ich kein Risiko eingehen.
- Docker-Images bleiben auf den Tags des Dozenten. PostgreSQL 19beta4: Die Final ist für
  den 19.11.2026 geplant, ein Wechsel birgt ein Risiko für die Volumes. Keycloak 26.7.4:
  neuere Versionen schließen Sicherheitslücken, der Server läuft aber nur lokal.
- Weitere Pakete (SQLAlchemy 2.1.6, Starlette 1.10.0, Strawberry 0.338.0, Ruff 0.17.2,
  ty 0.0.92, mkdocs-material 9.8.0) nicht eingespielt. Der Stand ist getestet, neue
  Lint-Regeln wären ein Risiko.

## Starten und Beenden

**Lokal:**

1. Docker Desktop starten
2. `cd extras\compose\keycloak`, `docker compose up` (PostgreSQL und Keycloak), warten
   bis Keycloak läuft
3. `cd extras\compose\mailpit`, `docker compose up -d`
4. Im Projektordner `$env:LOGURU_LEVEL = 'INFO'`, dann `uv run patient`, warten auf
   `Application startup complete`
5. Doku: `uv run mkdocs serve -a 127.0.0.1:8001` (Port 8001, weil 8000 der Appserver
   belegt)

Beenden: Appserver mit Strg+C, `docker compose down` in mailpit und keycloak. Die Daten
bleiben in den Volumes erhalten.

**Mit Docker:** `cd extras\compose\patient`, `docker compose up`, beenden mit Strg+C und
`docker compose down`.

**Adressen:** Swagger UI `https://127.0.0.1:8000/docs`, GraphQL
`https://127.0.0.1:8000/graphql`, Health `/health/liveness` und `/health/readiness`,
Metriken `/metrics`, Keycloak `https://localhost:8843`, Mailpit `http://localhost:8025`,
Dokumentation `http://127.0.0.1:8001`.

## Offen

- [ ] Bruno: Ordner Bearer Token (nicht separat dokumentiert)
- [ ] GitHub Actions zum Commit `f64ee53` prüfen
- [ ] Excel-Zeiterfassung vervollständigen
- [ ] Abgabe
