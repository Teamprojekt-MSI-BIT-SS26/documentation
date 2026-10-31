# Lokaler Lauf ohne Canvas – Doku

**English:** [Local Run without Canvas – Documentation](LocalRunWithoutCanvas.md)

## Worum es geht

Teilaufgabe aus Epic 2: Der Team-Algorithmus soll lokal in VS Code laufen, ohne Canvas.
Ergebnis: Er läuft. Testläufe mit 12 und 18 Studierenden haben mit allen vier Algorithmen ohne
Fehler Teams gebildet (Details: [Testlauf mit Beispieldaten](TestlaufMitBeispieldaten.md)).

Wichtig: Lokal ausführbar war der Algorithmus schon vorher (siehe nächster Abschnitt). Auf dem
Rechner fehlten nur die installierten Bibliotheken. Neu ist ein kleiner Ordner `local_run/`, der
genau das Datenformat von Canvas nutzt.

## Was schon da war

An diesem Code wurde nichts geändert. Diese Teile sorgen dafür, dass der Algorithmus ohne Canvas
läuft. Keiner davon braucht Canvas, einen Server oder eine Datenbank.

| Ort | Was es macht | Wieso wichtig für den lokalen Lauf |
| --- | --- | --- |
| `algorithms/ai/algorithm_runner.py` | `AlgorithmRunner`: wählt den Algorithmus aus und startet ihn | Der zentrale Einstieg. Wer ihn mit Studierenden und Einstellungen aufruft, bekommt Teams zurück |
| `algorithms/ai/` | Die Algorithmen selbst: `random`, `weight`, `social`, `priority` | Reiner Python-Code, rechnet komplett auf dem eigenen Rechner |
| `algorithms/dataclasses/` | Bausteine wie `Student`, `Team`, `TeamSet` plus Serializer (Umwandlung von/nach JSON) | Damit Daten aus einer Datei eingelesen und Teams als JSON ausgegeben werden können |
| `algorithms/api/validators/generate_teams_validator.py` | Prüft, ob die Eingabedaten richtig aufgebaut sind | Eigentlich für den Server gebaut, lässt sich aber auch ohne Server aufrufen |
| `algorithms/api/utils/generate_teams_data_loader.py` | Wandelt das geprüfte JSON in Objekte für den Algorithmus um | Ebenfalls ohne Server nutzbar |
| `runs/cli.py` | Startet den Algorithmus aus einer CSV-Datei (pro CSV-Format ein eigenes `transform.py`) | Lief schon vorher ohne Canvas |
| `manual_run/` | Älteres Skript für den Priority-Algorithmus mit CSV | Lief schon vorher ohne Canvas |
| `requirements.txt` | Liste aller Bibliotheken, die das Projekt braucht | Ohne diese Bibliotheken läuft gar nichts |

**Nicht nötig für den lokalen Lauf:** der Django-Server (`algorithms/api/views/`, `middlewares/`,
`models/`) und die Datenbank (`docker-compose-dev.yml`). Die braucht nur der Weg über Canvas.

Der Weg über Canvas:
Canvas → Server → API-Key-Prüfung → Datenbank-Log → Validator → Loader → AlgorithmRunner → Teams.

## Was neu ist

Neu ist ein Abkürzungsweg: Datei → Validator → Loader → AlgorithmRunner → Teams. Server, Datenbank,
API-Key und Canvas fallen weg. Ab dem Validator läuft alles wie im echten Server.

| Ort | Was es macht | Wieso |
| --- | --- | --- |
| `local_run/__init__.py` | Leere Datei, markiert den Ordner als Python-Paket | Ohne sie funktioniert `python -m local_run.run` nicht |
| `local_run/run.py` | Liest eine JSON-Datei, prüft sie, startet den Algorithmus, gibt die Teams aus (mit `-o` auch als Datei) | Macht dasselbe wie der Server, nur ohne Server |
| `local_run/sample_payload.json` | Testdaten: 12 erfundene Studierende mit Skill-Level und Freund-/Feind-Beziehungen, 4 Teams à 3 | Gleiches Format, das Canvas schickt, deshalb gegen echte Daten austauschbar |
| `local_run/sample_payload_18.json` | Zweiter Datensatz: 18 Studierende, 2 Attribute, 4 Teams mit 4–5 Personen | Deckt das obere Ende des Bereichs 10–20 ab |
| `tests/test_local_run/` | Automatisierter Test: beide Datensätze × alle vier Algorithmen | Macht den Testlauf jederzeit wiederholbar (siehe [Testlauf mit Beispieldaten](TestlaufMitBeispieldaten.md)) |
| `.gitignore` | Zeile `.venv/` ergänzt | Damit die virtuelle Umgebung nicht auf GitHub landet |

Die Dokus dazu liegen nicht im Repo `algorithms`, sondern hier im Repo `documentation`:

| Doku | Was drinsteht | Wieso |
| --- | --- | --- |
| [Handover](Handover_SkriptbedienungUndEinAusgabestruktur.md) | Anleitung: einrichten, starten, Eingabe und Ausgabe | Damit das Team es ohne Hilfe nutzen kann (Kriterium „Handover“) |
| Diese Doku | Was schon da war, was neu ist, offene Punkte | Hält fest, was gemacht wurde und wieso |
| [Testlauf mit Beispieldaten](TestlaufMitBeispieldaten.md) | Beispieldaten, automatisierter Test, Ergebnisse | Belegt das Kriterium „Testlauf mit 10–20 Studierenden“ |

**Nicht im Repo, nur lokal: `.venv/`.** Das ist eine eigene, abgetrennte Python-Installation für
dieses Projekt. Darin liegen die Bibliotheken aus `requirements.txt`. Ohne sie lief kein Weg
(Fehler: `No module named 'schema'`).

Die `.venv` gehört nicht auf GitHub: Sie ist mehrere hundert MB groß, funktioniert nur auf dem
eigenen Rechner und lässt sich jederzeit aus `requirements.txt` neu erzeugen. Jede Person erstellt
sie selbst (siehe [Handover](Handover_SkriptbedienungUndEinAusgabestruktur.md), Schritt 1–3).

## Testlauf

Befehl: `python -m local_run.run` mit `sample_payload.json` (Weight-Algorithmus, 12 Studierende).
Er lief ohne Fehler durch.

```
Team 1: Anna, Emma, Jonas
Team 2: Ben, Felix, Lena
Team 3: Clara, Greta, Max
Team 4: David, Hannah, Sophie
```

Zusätzlich gibt es einen zweiten Datensatz mit 18 Studierenden und einen automatisierten Test, der
beide Datensätze mit allen vier Algorithmen prüft:
`python -m unittest tests.test_local_run.test_local_run`. Was der Test genau macht, steht in
[Testlauf mit Beispieldaten](TestlaufMitBeispieldaten.md).

## Offene Punkte

- [ ] Entscheiden: `local_run` behalten (testet mit dem echten Canvas-Format inkl. Validator) oder
      nur `runs/cli.py` nutzen (CSV-Daten, ohne Validator) und `local_run` löschen.
- [x] Prüfen, warum im Testlauf alle 3 Freundespaare getrennt wurden (Anna/Ben, Emma/Felix,
      Lena/Max). **Geklärt, kein Bug:** Der Weight-Algorithmus füllt immer das kleinste Team. In
      der ersten Runde sind alle Teams leer, daher landen die ersten vier Personen automatisch in
      verschiedenen Teams. Der Social-Algorithmus hält Anna/Ben und Lena/Max zusammen. Details:
      [Testlauf mit Beispieldaten](TestlaufMitBeispieldaten.md#beobachtungen).
- [ ] Beziehungen haben unterschiedliche Formate: in der Eingabe `"friend"`, in der Ausgabe `-1`.
- [ ] In `algorithms/api/validators/generate_teams_validator.py` sind einige Methoden doppelt
      definiert. Python nutzt nur die zweite, die erste ist wirkungslos.
