# Handover: Lokaler Lauf mit `local_run/`

**English:** [Handover: Local Run with `local_run/`](Handover_ScriptUsageAndInputOutputStructure.md)

Startet den Team-Algorithmus lokal auf deinem Rechner – ohne Canvas, ohne Server, ohne Datenbank.

Als Eingabe dient dieselbe JSON-Datei, die Canvas sonst an den Server schickt. Das Skript prüft
die Daten genauso wie der echte Server und startet dann den Algorithmus.

## Einrichten (nur beim ersten Mal)

Alle Befehle im Hauptordner des Projekts (`algorithms/`) im Terminal ausführen.

### Schritt 1: Virtuelle Umgebung anlegen

```bash
python3 -m venv .venv
```

**Wieso:** Das legt den Ordner `.venv/` an, eine eigene Python-Installation nur für dieses Projekt.
So kommen sich die Bibliotheken nicht mit anderen Projekten auf deinem Rechner in die Quere.

### Schritt 2: Virtuelle Umgebung aktivieren

```bash
source .venv/bin/activate
```

**Wieso:** Damit benutzt das Terminal das Python aus `.venv/` statt das System-Python.
Vorne im Terminal steht dann `(.venv)`. Diesen Schritt musst du in jedem neuen Terminal wiederholen.

### Schritt 3: Bibliotheken installieren

```bash
pip install -r requirements.txt
```

**Wieso:** Der Algorithmus braucht fremde Bibliotheken (z. B. `numpy`, `schema`).
Ohne sie bricht das Programm mit `No module named ...` ab. `requirements.txt` listet alle auf.

### Schritt 4: VS Code auf die Umgebung einstellen

`Cmd+Shift+P` → „Python: Select Interpreter“ → `.venv` auswählen.

**Wieso:** Sonst kennt VS Code die installierten Bibliotheken nicht, markiert Imports als Fehler
und der „Run“-Knopf benutzt das falsche Python.

## Ausführen

### Schritt 5: Mit den Beispieldaten starten

```bash
python -m local_run.run
```

**Wieso:** So siehst du schnell, ob alles funktioniert. Das Skript nimmt automatisch
`sample_payload.json` (12 Studierende) und zeigt die Teams im Terminal an.
Ein zweites Beispiel mit 18 Studierenden startest du mit
`python -m local_run.run local_run/sample_payload_18.json`.

### Schritt 6 (optional): Eigene Daten verwenden und Ergebnis speichern

```bash
python -m local_run.run meine_daten.json -o teams.json
```

**Wieso:** Mit eigenen Daten testest du echte Fälle. `-o` speichert das komplette Ergebnis
als JSON-Datei, damit man es weiterverarbeiten kann.

### Schritt 7 (optional): Automatisierten Test laufen lassen

```bash
python -m unittest tests.test_local_run.test_local_run
```

**Wieso:** Der Test schickt beide Beispieldateien durch alle vier Algorithmen und prüft, ob die
Teams gültig sind. Am Ende muss `OK` stehen. Was genau geprüft wird, steht in
[Testlauf mit Beispieldaten](TestlaufMitBeispieldaten.md).

## Aufbau der Eingabe

```json
{
  "students": [
    {"id": 1, "name": "Anna", "attributes": {"1": [1]}, "relationships": {"2": "friend"}}
  ],
  "algorithm_options": {"algorithm_type": "weight", "max_project_preferences": 0},
  "team_generation_options": {
    "total_teams": 4, "max_team_size": 3, "min_team_size": 3,
    "initial_teams": [{"id": 1}, {"id": 2}, {"id": 3}, {"id": 4}]
  }
}
```

- `students`: alle Studierenden. `id` muss eindeutig sein.
- `attributes`: Eigenschaften einer Person, z. B. Skill-Level. Schlüssel = Attribut-Nummer als Text, Wert = Liste von Zahlen.
- `relationships`: Beziehung zu anderen Studierenden (über deren `id`): `friend`, `enemy` oder `default`.
- `algorithm_type`: welcher Algorithmus – `random`, `weight`, `social` oder `priority`.
- `team_generation_options`: Anzahl Teams und erlaubte Teamgröße. `initial_teams` braucht pro Team eine `id`.

Vollständige Beispiele stehen in `sample_payload.json` (12 Studierende) und
`sample_payload_18.json` (18 Studierende).

## Aufbau der Ausgabe

Im Terminal steht pro Team eine Zeile mit den Namen:

```
Team 1: Anna, Emma, Jonas
Team 2: Ben, Felix, Lena
```

Mit `-o` wird zusätzlich eine JSON-Datei geschrieben. Darin steht pro Team die Liste der
Studierenden mit allen Daten, im selben Format wie die Antwort des Servers an Canvas.
