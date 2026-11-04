# Aufgabe 1: Lokale Ausführung ohne Canvas

**Epic:** Epic 2 – Automate Team Generation  
**Akzeptanzkriterium:** Der Algorithmus läuft lokal (VS Code) ohne Canvas-Anbindung.

Teil der Dokumentation zum lokalen Algorithmus-Lauf: [1. Lokale Ausführung](LOKALE_AUSFUEHRUNG.md) · [2. Testlauf](TESTLAUF.md) · [3. Übergabe](UEBERGABE.md)  
Alle Pfade beziehen sich auf das Repository [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms) (Branch `41-bug-1-beheben`).

## Ergebnis

Der UBC-Algorithmus läuft lokal ohne Canvas. Er braucht auch keinen Django-Server, keine
PostgreSQL-Datenbank, keinen API-Key und kein Docker.

Dafür gibt es das neue Paket `local_run/`. Es nimmt **dieselbe JSON-Payload**
entgegen, die die API unter `POST /api/generate/teams` erwartet, und nutzt dieselbe Kette wie die API:

```
Payload (JSON-Datei)
  → GenerateTeamsValidator      (prüft das Schema, wie in der API)
  → GenerateTeamsDataLoader     (JSON → Student-, Team- und Options-Objekte)
  → AlgorithmRunner.generate()  (random | weight | social | priority)
  → TeamSetSerializer           (Ergebnis → JSON, gleiches Format wie die API-Antwort)
```

Der lokale Lauf und die API teilen sich also Validierung, Algorithmus und Ausgabeformat. Was lokal
funktioniert, funktioniert auch über die API.

## Warum das ohne Canvas geht

Der Algorithmus selbst (`algorithms/ai/`, `algorithms/dataclasses/`) hat keine Abhängigkeit zu
Canvas. Er bekommt fertige `Student`-Objekte und gibt ein `TeamSet` zurück. Canvas ist nur die
Datenquelle für diese Studierenden. Den Teil übernimmt lokal die JSON-Datei.

Auch die Django-Teile (`APILog`, Datenbank, API-Key-Middleware) liegen nur in der View
`algorithms/api/views/generate_teams.py`. Validator und DataLoader importieren kein Django und
können direkt genutzt werden.

| Komponente             | API-Betrieb               | Lokaler Lauf                 |
|------------------------|---------------------------|------------------------------|
| Datenquelle            | Canvas → HTTP-Request     | JSON-Datei                   |
| Validierung            | `GenerateTeamsValidator`  | `GenerateTeamsValidator`     |
| Algorithmus            | `AlgorithmRunner`         | `AlgorithmRunner`            |
| Logging in Datenbank   | `APILog` (PostgreSQL)     | entfällt                     |
| Authentifizierung      | API-Key (`X-Api-Key`)     | entfällt                     |
| Ausgabe                | HTTP-Antwort              | stdout oder JSON-Datei       |

## Einrichtung

Voraussetzung: **Python 3.9 oder 3.10**. Das ist die Version, mit der auch die CI testet.
Neuere Versionen wie 3.15 funktionieren mit den festen Paketversionen (z. B. `numpy==1.24.4`)
nicht.

Für den lokalen Lauf reichen drei Pakete aus `requirements-local.txt`:
`numpy`, `python-dotenv` und `schema`. Die vollständige `requirements.txt` (Django, Jupyter usw.)
ist nur für API, Benchmarking und Notebooks nötig.

**Variante A – mit `uv`** (lädt Python 3.10 bei Bedarf automatisch):

```bash
uv venv --python 3.10 .venv
uv pip install --python .venv -r requirements-local.txt
```

**Variante B – mit `venv`** (Python 3.10 muss installiert sein):

```powershell
py -3.10 -m venv .venv
.venv\Scripts\python -m pip install -r requirements-local.txt
```

In VS Code danach über `Strg+Shift+P` → *Python: Select Interpreter* die `.venv` auswählen.

## Ausführen

**Im Terminal** (aus dem Repository-Wurzelverzeichnis):

```powershell
.venv\Scripts\python -m local_run
```

Ohne Argumente läuft die mitgelieferte Beispiel-Payload mit 15 Studierenden. Eigene Dateien:

```powershell
.venv\Scripts\python -m local_run --input pfad\zur\payload.json --output local_run\output\result.json --seed 1
```

**In VS Code mit Debugger:** In `.vscode/launch.json` sind zwei
Konfigurationen hinterlegt. Man wählt sie im Bereich *Ausführen und Debuggen* (`Strg+Shift+D`):

- **Local run: Teams generieren** – startet `python -m local_run` und liest `env/local_run.env`
- **Local run: Tests** – führt die Tests aus `tests/test_local_run` aus

Damit kann man Breakpoints direkt im Algorithmus setzen, z. B. in
`algorithms/ai/priority_algorithm/priority_algorithm.py`.

## Behobene Fehler während der Umsetzung

Zusätzlich zu den Bugs 1–4 aus Spike 2 ([Offene Bugs](../../recherche/Algorithmen/BUGS.md)) ist beim lokalen Lauf ein weiterer Schema-Fehler
aufgefallen und behoben worden:

- **Optionale Felder `relationships` und `project_preferences` führten zum Absturz.** Laut Schema
  sind beide Felder optional. Fehlten sie in der Payload, brach der Validator mit einem
  `AttributeError` ab (über die API: HTTP 500). Der Serializer warf einen `KeyError` oder setzte
  `project_preferences=None`, was später im Algorithmus abstürzt. Jetzt gelten fehlende Felder als
  „keine Beziehungen“ bzw. „keine Präferenzen“.
  Geändert: `algorithms/api/validators/generate_teams_validator.py`,
  `algorithms/dataclasses/student/serializer.py`.

## Neue und geänderte Dateien

| Datei                                   | Zweck                                                    |
|-----------------------------------------|----------------------------------------------------------|
| `local_run/run.py`                      | Lokaler Runner (Validierung, Lauf, Ausgabe)              |
| `local_run/__main__.py`                 | Einstiegspunkt für `python -m local_run`                 |
| `local_run/sample_data/*.json`          | Beispiel-Payload mit 15 Studierenden                     |
| `env/local_run.env.example`             | Vorlage für die Konfiguration über Umgebungsvariablen   |
| `requirements-local.txt`                | Minimale Abhängigkeiten                                  |
| `.vscode/launch.json`                   | Start- und Debug-Konfigurationen für VS Code             |
| `tests/test_local_run/test_run.py`      | Automatisierte Tests des Testlaufs                       |
| `.gitignore`                            | `.venv/`, `env/local_run.env`, `local_run/output/`       |
