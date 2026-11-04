# Aufgabe 2: Testlauf mit Beispieldaten

**Epic:** Epic 2 – Automate Team Generation  
**Akzeptanzkriterium:** Ein Testlauf mit Beispieldaten (10–20 Studierende) berechnet Teams ohne Fehler.

Teil der Dokumentation zum lokalen Algorithmus-Lauf: [1. Lokale Ausführung](LOKALE_AUSFUEHRUNG.md) · [2. Testlauf](TESTLAUF.md) · [3. Übergabe](UEBERGABE.md)  
Alle Pfade beziehen sich auf das Repository [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms) (Branch `41-bug-1-beheben`).

## Ergebnis

Der Testlauf mit **15 Studierenden** läuft fehlerfrei durch, mit allen vier Algorithmen
(`random`, `weight`, `social`, `priority`). Er ist als automatisierter Test hinterlegt und kann
jederzeit wiederholt werden.

## Beispieldaten

Datei: `local_run/sample_data/payload_priority_15_students.json`

Die Daten sind fiktiv. Sie decken die wichtigsten Merkmale ab, die der Algorithmus verarbeitet:

| Merkmal               | Inhalt in den Beispieldaten                                                          |
|-----------------------|--------------------------------------------------------------------------------------|
| Studierende           | 15 (IDs 1–15)                                                                        |
| Teams                 | 4 Projekte: Web-App, Datenanalyse, Mobile, Spiele                                    |
| Teamgröße             | min. 3, max. 4 (→ 3 Teams à 4 und 1 Team à 3)                                        |
| Attribut `1`          | Geschlecht (1 = w, 2 = m, 3 = d)                                                     |
| Attribut `2`          | Studiengang (1 = Informatik, 2 = Wirtschaftsinformatik, 3 = Medieninformatik)        |
| Attribut `3`          | Programmiererfahrung (1–5)                                                           |
| Freundschaften        | 1↔2, 5↔6↔7, 11↔12                                                                    |
| Abneigungen (enemy)   | 3→4, 9→10                                                                            |
| Projektpräferenzen    | je 3 Wünsche pro Person (`max_project_preferences = 3`)                              |
| Projektanforderung    | Team Web-App braucht mind. eine Person mit Programmiererfahrung > 3                  |

Algorithmus: `priority` mit vier Prioritäten in dieser Reihenfolge:

1. `diversity` – Geschlecht (Attribut 1) im Team mischen
2. `social_preference` – Freunde zusammen, Abneigungen getrennt
3. `project_preference` – Projektwünsche berücksichtigen
4. `project_requirement` – Projektanforderungen erfüllen

## Testlauf durchführen

```powershell
.venv\Scripts\python -m local_run --seed 1 --output local_run\output\result.json
```

Ausgabe im Terminal (stderr):

```
Generating teams from ...\local_run\sample_data\payload_priority_15_students.json
Team Web-App (4 students): Elif Yilmaz, Felix Koch, Greta Richter, Noah Lange
Team Datenanalyse (4 students): Ben Schulz, Anna Becker, Clara Wagner, Hannes Wolf
Team Mobile (4 students): Katharina Zimmer, Olivia Schmitt, Leon Hartmann, Jonas Braun
Team Spiele (3 students): Ida Neumann, Mia Krause, David Hoffmann
Teams written to ...\local_run\output\result.json
```

Laufzeit: etwa 3 Sekunden. Mit demselben `--seed` kommt immer dasselbe Ergebnis heraus.

## Auswertung des Ergebnisses (Seed 1)

| Prüfpunkt                         | Ergebnis                                                                 |
|-----------------------------------|--------------------------------------------------------------------------|
| Alle Studierenden zugeteilt       | ✅ 15 von 15, niemand doppelt                                            |
| Teamgrößen eingehalten            | ✅ 4 / 4 / 4 / 3                                                          |
| Freundschaften                    | ✅ 1+2 (Datenanalyse), 5+6+7 (Web-App), 11+12 (Mobile)                    |
| Abneigungen getrennt              | ✅ 3 (Datenanalyse) / 4 (Spiele), 9 (Spiele) / 10 (Mobile)                |
| Geschlechtermischung              | ✅ jedes Team hat Frauen und Männer                                       |
| Anforderung Web-App               | ✅ Elif Yilmaz hat Programmiererfahrung 5                                 |
| Projektwünsche                    | 8× Erstwunsch, 3× Zweitwunsch, 3× Drittwunsch, 1× kein Wunsch             |

Der eine nicht erfüllte Wunsch (Katharina Zimmer im Team Mobile) ist ein bewusster Kompromiss:
Die Prioritäten „Freunde zusammen“ und „Geschlechtermischung“ stehen vor den Projektwünschen.
Katharina wurde mit ihrem Freund Leon Hartmann zusammengelegt.

## Automatisierte Tests

Datei: `tests/test_local_run/test_run.py`

```powershell
.venv\Scripts\python -m unittest tests.test_local_run.test_run -v
```

| Test                                                    | Prüft                                                               |
|---------------------------------------------------------|---------------------------------------------------------------------|
| `test_sample_payload__has_10_to_20_students`            | Beispieldaten liegen im geforderten Bereich von 10–20 Studierenden  |
| `test_generate_teams__sample_payload_creates_valid_teams` | Alle zugeteilt, Teamanzahl und Teamgrößen stimmen                 |
| `test_generate_teams__works_for_every_algorithm_type`   | Dieselben Studierenden mit `random`, `weight`, `social`, `priority` |
| `test_generate_teams__same_seed_gives_same_teams`       | Gleicher Seed → gleiches Ergebnis                                   |
| `test_generate_teams__optional_student_fields_can_be_omitted` | Payload ohne `relationships`/`project_preferences` läuft      |
| `test_generate_teams__raises_if_students_do_not_fit_into_teams` | Unpassende Teamgrößen werden mit klarer Meldung abgelehnt   |
| `test_main__writes_result_to_output_file`               | Kompletter Lauf über die Kommandozeile mit Ausgabedatei             |
| `test_main__returns_error_code_for_missing_input`       | Fehlende Eingabedatei → Exit-Code 1 statt Absturz                   |

Ergebnis: **8 von 8 Tests bestanden** (ca. 11 Sekunden).

Die bestehenden Tests unter `tests/test_api` (Algorithmus, Validatoren, Serializer) laufen
weiterhin durch: 73 von 73. Wer `python -m unittest discover tests` mit nur
`requirements-local.txt` ausführt, sieht 7 Importfehler in `tests/test_benchmarking`. Diese Tests
brauchen `gitpython` aus der vollständigen `requirements.txt` und haben mit dem lokalen Lauf nichts
zu tun.
