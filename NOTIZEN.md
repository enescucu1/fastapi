# Notizen zur Aufgabe 1: Appserver mit FastAPI

Gruppe 2, GitHub: enescucu1. Die Zeiten stehen in `zeiterfassung.xlsx`.

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
- [ ] Mailpit (Volume `mailpit`)

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

## Offen

- [ ] Mailpit (Volume `mailpit`)
- [ ] Bruno: Token holen, Patient abrufen
- [ ] Tests mit pytest
