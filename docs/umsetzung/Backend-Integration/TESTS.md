# Lokale Backend-Tests

**Epic:** Epic 2 – Automate Team Generation  
**Ticket:** #51 – Technical Task 32: Backend Integration & Validation of the Team-Building Algorithm (local)

Teil der Dokumentation zur Backend-Integration: [1. Integration und Schnittstelle](INTEGRATION.md) · [2. Lokale Backend-Tests](TESTS.md)

Alle Pfade beziehen sich auf das Repository
[teamable-backend](https://github.com/Teamprojekt-MSI-BIT-SS26/teamable-backend), Branch
`51-technical-task-32-backend-integration-validation-of-the-team-building-algorithm-local`.

## Überblick

Es gibt zwei Arten von Tests in `src/app/tests`:

| Datei | Was getestet wird | Braucht die Algorithmen-API |
| --- | --- | --- |
| `test_team_generation.py` | Die Anfrage, die das Backend an die API schickt, die Parameter, die Fehlerbehandlung, das Speichern und die Antwort der Endpunkte. Die API wird simuliert (Mock). | nein |
| `test_team_generation_integration.py` | Der komplette Ablauf mit allen vier Algorithmen: Endpunkt → Algorithmen-API → gespeicherte Teams → `team-sets`-Endpunkt | ja |

Die Tests mit simulierter API laufen in wenigen Sekunden und überall. Die Integrationstests zeigen,
dass Backend und echte Algorithmen-API zusammenarbeiten.

## Testdatensatz

Die Testdaten in `src/app/tests/data.py` werden vor jedem Test in einer eigenen Testdatenbank angelegt.
Die normale Datenbank bleibt unberührt.

| Daten | Inhalt |
| --- | --- |
| Kurs | „Teamprojekt“ mit einer lehrenden Person, die die Endpunkte aufruft |
| Studierende | **15** (Anna, Ben, Clara, … Olivia, Nachname „Test“) |
| Attribut „Programming experience“ | Zahlen-Attribut mit den Optionen 1, 2, 3. Jede Person hat einen Wert, reihum verteilt. |
| Attribut „Study program“ | Text-Attribut mit den Optionen MSI, BIT, WIN. Jeweils drei Personen hintereinander haben denselben Studiengang. |
| Freundschaften | Anna ↔ Ben, Clara → David, Emma → Felix |
| Feindschaft | Greta → Hannes |

So kommen alle Datenarten vor, die das Backend an die API schickt: Zahlen- und Text-Attribute,
Freunde und Feinde.

## Ausführen

Voraussetzung ist ein eingerichtetes Backend (siehe README des Backends). Alle Befehle laufen im
Ordner `src` des Backends.

### Nur die Tests mit simulierter API

```bash
python manage.py test app.tests.test_team_generation
```

### Alle Tests inklusive Integrationstests

1. Die Algorithmen-API auf Port 8001 starten, wie unter
   [Lokale Einrichtung](INTEGRATION.md#lokale-einrichtung) beschrieben.
2. In der `.env` des Backends `TEAM_GENERATION_API_URL` und `TEAM_GENERATION_API_KEY` eintragen.
3. Die Tests starten:

```bash
python manage.py test app.tests -v 2
```

Die Integrationstests dauern etwa 40 Sekunden, vor allem wegen des Priority-Algorithmus.

!!! note "Wann die Integrationstests übersprungen werden"
    Ist kein `TEAM_GENERATION_API_KEY` gesetzt oder die Algorithmen-API nicht erreichbar, werden die
    Integrationstests übersprungen. In der Ausgabe steht dann `OK (skipped=1)`. Lehnt die API den
    Key ab, schlagen sie mit einer Fehlermeldung fehl, weil dann die Konfiguration falsch ist.

## Was die Tests prüfen

### Integrationstests (`test_team_generation_integration.py`)

| Test | Prüfung |
| --- | --- |
| `test_test_data_has_10_to_20_students` | Der Testdatensatz hat 10 bis 20 Studierende |
| `test_default_parameters` | Ohne Parameter: Priority-Algorithmus, 3 Teams mit 3 bis 5 Personen |
| `test_every_algorithm_type` | `priority`, `weight`, `social` und `random` liefern jeweils ein gültiges Team-Set |
| `test_custom_team_sizes` | Mit `total_teams=4`, `min_team_size=3`, `max_team_size=4` hält jeder Algorithmus die Vorgaben ein |
| `test_team_sizes_that_do_not_fit_are_rejected` | 15 Studierende in 2 Teams mit höchstens 5 Personen: Das Backend antwortet `400` mit einer verständlichen Meldung |

Bei jedem erfolgreichen Lauf ruft der Test das Ergebnis über
`GET /api/v1/courses/{id}/team-sets/{team_set_id}/` ab und prüft:

- die Datenstruktur: `id`, `name`, `teams`, `unassigned_students`, und je Team `id`, `slug`, `name`, `members`,
- dass `unassigned_students` leer ist,
- dass alle 15 Studierenden genau einmal vorkommen,
- dass Teamanzahl und Teamgrößen den Parametern entsprechen.

### Tests mit simulierter API (`test_team_generation.py`)

| Bereich | Prüfung |
| --- | --- |
| Anfrage an die API | Alle 15 Studierenden mit Namen, Attributwerten (Zahlen-Attribut behält seinen Wert, Text-Attribut wird nummeriert) und Beziehungen; Standard-Teamoptionen; automatisch abgeleitete Prioritäten; Optionen für jeden Algorithmus; Auswahl einzelner Attribute; `total_teams` |
| Endpunkt | Speichert das Team-Set und liefert es über `team-sets` aus; übergibt die Parameter; das bisherige Feld `attribute` funktioniert weiter |
| Ungültige Parameter | `min_team_size` > `max_team_size`, unbekannter Algorithmus, `priorities` ohne `priority`, unbekanntes Attribut, Attribut eines anderen Kurses: `400`, die API wird nicht aufgerufen |
| Fehler der API | Ungültige Eingabe → `400`, abgelehnter Key → `502`, Serverfehler → `502`, nicht erreichbar → `502`, Zeitüberschreitung → `504`. In keinem Fall wird ein Team-Set gespeichert. |
| Ungültiges Ergebnis | Fehlt im Ergebnis eine Person, antwortet das Backend `502` und speichert nichts |
| Konfiguration | Ohne `TEAM_GENERATION_API_KEY`: `503` |

## Ergebnis des Testlaufs

Lokal unter Windows 11, Algorithmen-API mit Python 3.10, Backend mit Python 3.11:

| Lauf | Ergebnis |
| --- | --- |
| `test_team_generation` (simulierte API) | 15 Tests, alle erfolgreich, ca. 6 Sekunden |
| `test_team_generation_integration` (echte API) | 5 Tests, alle erfolgreich, ca. 36 Sekunden |
| Algorithmen-Repo: `tests.test_api`, `tests.test_local_run` | 78 + 9 Tests, alle erfolgreich |
| black und pyright (wie in der CI des Backends) | keine Beanstandungen in den geänderten Dateien |

## Probleme und Lösungen

| Problem | Ursache und Lösung |
| --- | --- |
| `OK (skipped=1)`, obwohl die Integrationstests laufen sollten | Die Algorithmen-API läuft nicht auf der eingetragenen URL, oder `TEAM_GENERATION_API_KEY` fehlt in der `.env` des Backends |
| `The algorithms API rejected TEAM_GENERATION_API_KEY` | Der Hash des Keys steht nicht in `API_AUTH_ALLOWED_KEYS` der Algorithmen-API, oder dort wurde ein anderer Key gehasht |
| Windows: „Eine Anwendungssteuerungsrichtlinie hat diese Datei blockiert“ beim Start von `.venv\Scripts\python.exe` | Die Windows-Funktion „Intelligente App-Steuerung“ blockiert die Python-Starter in virtuellen Umgebungen. Ein direkt installiertes Python funktioniert, z. B. eines, das mit `uv python install 3.10` installiert wurde. |
| `400` mit „students do not fit into … teams“ | `total_teams`, `min_team_size` und `max_team_size` passen nicht zur Anzahl der Studierenden. Das ist gewollt, die Prüfung kommt aus der Algorithmen-API. |
