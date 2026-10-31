# Testlauf mit Beispieldaten – Was die Tests machen

**English:** [Test Run with Sample Data – What the Tests Do](TestRunWithSampleData.md)

## Worum es geht

Akzeptanzkriterium aus Epic 2: *„Ein Testlauf mit Beispieldaten (10–20 Studierende) berechnet
Teams ohne Fehler.“*

Ergebnis: Erfüllt. Mit 12 und mit 18 Studierenden bilden alle vier Algorithmen (`random`,
`weight`, `social`, `priority`) fehlerfrei gültige Teams. Ein automatisierter Test prüft das
bei jedem Aufruf neu.

Wie man den lokalen Lauf einrichtet, steht im
[Handover](Handover_SkriptbedienungUndEinAusgabestruktur.md).

## Die Beispieldaten

Beide Dateien liegen im Repo `algorithms` im Ordner `local_run/`. Alle Personen sind erfunden.
Die Dateien haben dasselbe Format, das Canvas an den Server schickt.

| Datei | Studierende | Teams | Attribute | Beziehungen |
| --- | --- | --- | --- | --- |
| `sample_payload.json` | 12 | 4 Teams à 3 | 1 Attribut (Skill-Level 1–3) | 3 Freundespaare, 1 Feindespaar |
| `sample_payload_18.json` | 18 | 4 Teams mit 4–5 Personen (5/5/4/4) | 2 Attribute (Studiengang 1–3, Erfahrung 1–3) | 4 Freundespaare, 1 Dreiergruppe, 2 Feindespaare |

**Wieso zwei Dateien:** Die 12er-Datei liegt am unteren Ende des Bereichs 10–20, die 18er-Datei
am oberen. Die 18er-Datei ist außerdem schwieriger: Die Teams sind nicht gleich groß, es müssen
zwei Attribute gleichzeitig gemischt werden, und es gibt mehr Beziehungen.

Beziehungen in der 18er-Datei:

- Freunde: Anna ↔ Ben, Emma ↔ Felix, Lena → Max, Tim ↔ Ulla, Tim → Vincent, Yusuf → Zoe
- Feinde: Greta → Hannah, Jonas ↔ Wiebke

(`↔` = beide haben sich gegenseitig angegeben, `→` = nur eine Person hat es angegeben)

## Der automatisierte Test

Datei: `tests/test_local_run/test_local_run.py` im Repo `algorithms`.

### Starten

Im Hauptordner `algorithms/`, mit aktivierter `.venv`:

```bash
python -m unittest tests.test_local_run.test_local_run
```

Erwartete Ausgabe:

```
.
----------------------------------------------------------------------
Ran 1 test in 1.5s

OK
```

Bei einem Fehler steht statt `OK` ein `FAILED`, zusammen mit der Datei und dem Algorithmus,
bei dem es schiefging (z. B. `payload='sample_payload_18.json', algorithm='social'`).

### Was der Test Schritt für Schritt macht

1. **Datei einlesen:** Er lädt nacheinander `sample_payload.json` und `sample_payload_18.json`.
2. **Größe prüfen:** Er stellt sicher, dass die Datei wirklich 10–20 Studierende enthält.
3. **Algorithmus einstellen:** Er setzt nacheinander jeden der vier Algorithmen ein. Für `random`
   werden alle anderen Optionen entfernt (der Server lehnt sie dort ab). Für `priority` wird eine
   leere Prioritätenliste ergänzt (der Server verlangt dieses Feld).
4. **Teams berechnen:** Validator → Loader → AlgorithmRunner, also genau dieselben Schritte wie
   `local_run/run.py` und der echte Server.
5. **Ergebnis prüfen:** die vier Prüfungen aus der nächsten Tabelle.

Das ergibt 2 Dateien × 4 Algorithmen = **8 Testläufe**. Jeder läuft als eigener Untertest
(`subTest`). Schlägt einer fehl, laufen die anderen trotzdem weiter.

### Die vier Prüfungen

| Prüfung | Was genau | Wieso |
| --- | --- | --- |
| Anzahl Teams | Es gibt genau so viele Teams wie in `total_teams` angegeben | Canvas erwartet genau die bestellte Anzahl |
| Alle zugeteilt | Jede Person ist in genau einem Team, niemand fehlt, niemand ist doppelt | Die wichtigste Grundregel: Sonst würde jemand ohne Team bleiben |
| Teamgröße | Jedes Team hat mindestens `min_team_size` und höchstens `max_team_size` Mitglieder | Die Lehrkraft gibt die Größe vor, sie muss eingehalten werden |
| JSON-Ausgabe | Das Ergebnis lässt sich in JSON umwandeln | Das ist das Format, das `run.py -o` schreibt und der Server an Canvas zurückschickt |

### Was der Test bewusst nicht prüft

Der Test prüft nur, ob das Ergebnis **gültig** ist, nicht ob es **gut** ist. Ob Freunde
zusammen und Feinde getrennt landen, wird nicht geprüft.

**Wieso:** Kein Algorithmus garantiert das. `random` würfelt, und `weight` trennt Freunde schon
durch seine Arbeitsweise (siehe unten). Ein Test, der das verlangt, würde ständig fehlschlagen,
obwohl der Code richtig arbeitet. Die Qualität der Teams wird stattdessen mit den Metriken in
`METRICS.md` und dem Ordner `benchmarking/` bewertet.

## Ergebnisse der Testläufe

`weight` und `social` liefern bei jedem Lauf dieselben Teams. `random` und `priority` enthalten
Zufall, dort sehen die Teams jedes Mal anders aus. Gültig sind sie aber immer.

### 12 Studierende

| Algorithmus | Team 1 | Team 2 | Team 3 | Team 4 |
| --- | --- | --- | --- | --- |
| `weight` | Anna, Emma, Jonas | Ben, Felix, Lena | Clara, Greta, Max | David, Hannah, Sophie |
| `social` | Anna, Ben, Clara | Felix, Lena, Max | David, Hannah, Jonas | Emma, Greta, Sophie |

### 18 Studierende

| Algorithmus | Team 1 | Team 2 | Team 3 | Team 4 |
| --- | --- | --- | --- | --- |
| `weight` | Anna, Hannah, Vincent, Emma, Tim | Ben, Greta, Zoe, Felix, Wiebke | Clara, Lena, Ulla, Max | David, Jonas, Yusuf, Sophie |
| `social` | Tim, Ulla, Vincent, Hannah | Anna, Ben, Clara, David, Sophie | Lena, Max, Zoe, Wiebke | Emma, Felix, Greta, Jonas, Yusuf |

### Beobachtungen

- **Feinde werden in allen Läufen getrennt** (Greta/Hannah, Jonas/Wiebke).
- **`social` hält Freunde besser zusammen:** Anna/Ben, Lena/Max und die Dreiergruppe
  Tim/Ulla/Vincent landen jeweils im selben Team.
- **`weight` trennt viele Freundespaare.** Das ist kein Bug, sondern liegt an der Arbeitsweise:
  Der Algorithmus füllt immer das gerade kleinste Team mit der Person, die am besten passt. In der
  ersten Runde sind alle Teams leer, also passen alle Personen gleich gut. Jedes Team bekommt dann
  einfach die nächste Person aus der Liste. Die ersten vier Personen (Anna, Ben, Clara, David)
  werden dadurch immer getrennt, egal welche Beziehungen sie haben.
  Damit ist der offene Punkt aus der Doku [Lokaler Lauf ohne Canvas](LokalerLaufOhneCanvas.md)
  geklärt.

## Geänderte Dateien im Repo `algorithms`

| Ort | Was | Wieso |
| --- | --- | --- |
| `local_run/sample_payload_18.json` | Neu: zweiter Datensatz mit 18 Studierenden | Deckt das obere Ende des Bereichs 10–20 ab |
| `tests/test_local_run/__init__.py` | Neu: leere Datei | Markiert den Ordner als Python-Paket, sonst findet `unittest` den Test nicht |
| `tests/test_local_run/test_local_run.py` | Neu: der automatisierte Test | Macht den Testlauf jederzeit wiederholbar |
| `local_run/run.py` | Nur der Kommentar oben: 18er-Datei und Testbefehl ergänzt | Damit man beides direkt im Code findet |
