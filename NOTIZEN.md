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
