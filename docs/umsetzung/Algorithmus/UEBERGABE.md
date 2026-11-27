# Aufgabe 3: Übergabe – Ausführung, Schnittstelle und Konfiguration

**Epic:** Epic 2 – Automate Team Generation  
**Akzeptanzkriterium:** Kurze Dokumentation, wie das Skript ausgeführt wird und wie Ein- und Ausgabe aufgebaut sind.

Teil der Dokumentation zum lokalen Algorithmus-Lauf: [1. Lokale Ausführung](LOKALE_AUSFUEHRUNG.md) · [2. Testlauf](TESTLAUF.md) · [3. Übergabe](UEBERGABE.md)  
Alle Pfade beziehen sich auf das Repository [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms) (Branch `41-bug-1-beheben`).

## 1. Kurzanleitung

```powershell
# einmalig: Umgebung einrichten (Python 3.10)
uv venv --python 3.10 .venv
uv pip install --python .venv -r requirements-local.txt

# optional: Konfiguration anlegen
copy env\local_run.env.example env\local_run.env

# Teams generieren
.venv\Scripts\python -m local_run --input <payload.json> --output <result.json> --seed 1

# Tests
.venv\Scripts\python -m unittest tests.test_local_run.test_run -v
```

| Argument           | Kurzform | Umgebungsvariable | Standard                                       |
|--------------------|----------|-------------------|------------------------------------------------|
| `--input`          | `-i`     | `TEAMGEN_INPUT`   | `local_run/sample_data/payload_priority_15_students.json` |
| `--output`         | `-o`     | `TEAMGEN_OUTPUT`  | Ausgabe auf stdout                             |
| `--seed`           | `-s`     | `TEAMGEN_SEED`    | kein Seed (jeder Lauf zufällig)                |

Relative Pfade gelten ab dem Repository-Wurzelverzeichnis. Die Teamübersicht und Meldungen gehen
auf stderr, das JSON-Ergebnis auf stdout oder in die Datei. Man kann die Ausgabe also auch direkt
weiterleiten, z. B. `python -m local_run > result.json`.

**Exit-Codes:** `0` = Teams erzeugt. `1` = Fehler, z. B. Datei nicht gefunden, ungültige Payload,
Teamgrößen passen nicht zur Anzahl der Studierenden oder jemand wurde keinem Team zugeteilt. Die
Fehlermeldung steht auf stderr.

## 2. Eingabe: Payload

Die Payload ist **identisch mit dem Request-Body von `POST /api/generate/teams`**. Ein Backend, das
später die Daten aus Canvas holt, muss genau dieses Format erzeugen.

```json
{
  "algorithm_options": { "algorithm_type": "priority", "...": "..." },
  "team_generation_options": {
    "max_team_size": 4,
    "min_team_size": 3,
    "total_teams": 4,
    "initial_teams": [ { "id": 1, "name": "Team Web-App", "project_id": 1, "requirements": [] } ]
  },
  "students": [
    {
      "id": 1,
      "name": "Anna Becker",
      "attributes": { "1": [1], "2": [1], "3": [2] },
      "relationships": { "2": "friend" },
      "project_preferences": [1, 4, 2]
    }
  ]
}
```

### `students[]`

| Feld                  | Typ                       | Pflicht | Bedeutung                                                         |
|-----------------------|---------------------------|---------|-------------------------------------------------------------------|
| `id`                  | int                       | ja      | Eindeutige ID, z. B. Canvas-User-ID                               |
| `attributes`          | `{ "<attr-id>": [int] }`  | ja      | Attributwerte als Liste, `{}` erlaubt. Schlüssel als String-Zahl  |
| `name`                | string                    | nein    | Anzeigename                                                       |
| `relationships`       | `{ "<student-id>": str }` | nein    | `"friend"`, `"enemy"` oder `"default"`. IDs müssen existieren     |
| `project_preferences` | `[int]`                   | nein    | `project_id`s in Wunschreihenfolge, höchstens `max_project_preferences` |

Attribute sind bewusst nur Zahlen. Was z. B. Attribut `3` bedeutet und was der Wert `5` heißt, legt
der Aufrufer fest. Das Mapping (Umfrage → Zahlen) muss das Backend dokumentieren.

### `team_generation_options`

| Feld             | Typ    | Bedeutung                                                                     |
|------------------|--------|-------------------------------------------------------------------------------|
| `total_teams`    | int    | Anzahl der Teams                                                              |
| `min_team_size`  | int    | Mindestgröße pro Team                                                         |
| `max_team_size`  | int    | Höchstgröße pro Team                                                          |
| `initial_teams`  | list   | Vordefinierte Teams/Projekte, `[]` erlaubt. Pro Team: `id` (Pflicht), `name`, `project_id`, `requirements`, `is_locked` |

Die Teamgrößen müssen zur Anzahl der Studierenden passen:
`total_teams × min_team_size ≤ Anzahl ≤ total_teams × max_team_size`. Gesperrte Teams
(`is_locked`) zählen dabei nicht mit. Andernfalls wird die Payload abgelehnt.

`requirements[]` pro Team: `attribute` (int), `operator` (`"exactly"`, `"less than"` oder
`"more than"`), `value` (int) und optional `criteria`:

- `"someone_meets_requirement"` (Standard)
- `"everyone_meets_requirement"`
- `"some_number_of_members_meet_requirement"`, dann zusätzlich `num_members_required`

### `algorithm_options`

| `algorithm_type` | Weitere Felder                                                                              |
|------------------|---------------------------------------------------------------------------------------------|
| `random`         | keine                                                                                       |
| `weight`, `social` | `max_project_preferences` (Pflicht); optional `social_weight`, `diversity_weight`, `preference_weight`, `requirement_weight`, `friend_behaviour`/`enemy_behaviour` (`enforce`, `invert`, `ignore`), `attributes_to_diversify`, `attributes_to_concentrate` |
| `priority`       | wie `weight`, zusätzlich `priorities` (Pflicht, Reihenfolge = Wichtigkeit)                  |

Mögliche Einträge in `priorities[]`:

| `priority_type`        | Felder                                                                                   |
|------------------------|------------------------------------------------------------------------------------------|
| `diversity`            | `attribute_id`, `strategy` (`diversify` oder `concentrate`), `max_num_choices` (Pflicht bei `concentrate`) |
| `tokenization`         | `attribute_id`, `strategy`, `direction` (`min_of` oder `max_of`), `threshold`, `value`    |
| `social_preference`    | `max_num_friends`, `max_num_enemies`                                                     |
| `project_preference`   | `direction` (`include` oder `exclude`), `max_project_preferences`                        |
| `project_requirement`  | keine                                                                                    |

Ein vollständiges Beispiel steht in
`local_run/sample_data/payload_priority_15_students.json`.

## 3. Ausgabe

Das Format ist **identisch mit der API-Antwort**:

```json
{
  "teams": {
    "id": null,
    "name": null,
    "teams": [
      {
        "id": 1,
        "name": "Team Web-App",
        "project_id": 1,
        "requirements": [
          { "attribute": 3, "operator": "more than", "value": 3,
            "criteria": "someone_meets_requirement", "num_members_required": null }
        ],
        "students": [
          {
            "id": 5,
            "name": "Elif Yilmaz",
            "attributes": { "1": [1], "2": [3], "3": [5] },
            "relationships": { "6": -1 },
            "project_preferences": [1, 3, 4]
          }
        ]
      }
    ]
  },
  "metadata": { "version": "0.1.0", "timestamp": 1793791395.05 },
  "error": null
}
```

- Für die Weiterverarbeitung reicht in der Regel `teams.teams[].id` bzw. `project_id` und
  `teams.teams[].students[].id`.
- Hinweis: In der Ausgabe stehen Beziehungen als **Zahlen** (`-1` = friend, `1.1` = enemy,
  `0` = default), in der Eingabe als Text. Das ist das bestehende Verhalten der API
  (`StudentSerializer`). Der Serializer liest beide Formen wieder ein.
- Lokal bricht der Lauf mit Exit-Code 1 ab, wenn jemand keinem Team zugeteilt wurde. Die API
  liefert in diesem Fall kein Feld dafür.

## 4. Konfiguration über Umgebungsvariablen

**Grundsatz:** Die **Payload** enthält die fachlichen Daten: Studierende, Teams und die Optionen,
die Lehrende wählen. **Umgebungsvariablen** enthalten technische Laufzeit-Einstellungen und
Zugangsdaten. Diese gehören nicht in den Code und nicht ins Repository.

### Lokaler Lauf (`env/local_run.env`)

Die Vorlage liegt in `env/local_run.env.example` und wird nach
`env/local_run.env` kopiert. `env/local_run.env` ist in `.gitignore` eingetragen.

| Variable                       | Bedeutung                                                    |
|--------------------------------|--------------------------------------------------------------|
| `TEAMGEN_INPUT`                | Pfad zur Payload                                             |
| `TEAMGEN_OUTPUT`               | Pfad zur Ausgabedatei (leer = stdout)                        |
| `TEAMGEN_SEED`                 | Zufalls-Seed für reproduzierbare Ergebnisse                  |
| `TEAMGEN_PRIORITY_MAX_TIME`    | Priority-Algorithmus: maximale Laufzeit in Sekunden (Standard 30) |
| `TEAMGEN_PRIORITY_MAX_ITERATE` | Priority-Algorithmus: maximale Iterationen (Standard 1500)   |
| `TEAMGEN_PRIORITY_MAX_KEEP`    | Priority-Algorithmus: behaltene Teamsets pro Runde (Standard 3) |
| `TEAMGEN_PRIORITY_MAX_SPREAD`  | Priority-Algorithmus: Mutationen pro Teamset (Standard 3)    |

**Reihenfolge, wenn ein Wert mehrfach gesetzt ist:** Kommandozeilen-Argument vor Variable in der
Shell vor `env/local_run.env` vor Standardwert. Ungültige Zahlen führen zu einer klaren
Fehlermeldung.

### API-Betrieb (`env/.env.prod`, siehe `env/sample.env`)

| Variable                                   | Bedeutung                                                      |
|--------------------------------------------|----------------------------------------------------------------|
| `DJANGO_SECRET_KEY`                        | Django-Secret (nur in `settings/prod.py` genutzt)              |
| `DJANGO_ALLOWED_HOSTS`                     | Erlaubte Hosts, durch Leerzeichen getrennt                     |
| `DB_ENGINE`, `DB_DATABASE`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT` | Datenbankverbindung (Alternativ: `RDS_*`) |
| `API_AUTH_ALLOWED_KEYS`                    | Erlaubte API-Keys als bcrypt-Hashes (hex), durch Leerzeichen getrennt |

Canvas-Zugangsdaten (z. B. ein Canvas-API-Token) braucht der Algorithmus nicht. Sie gehören in
das Backend, das die Daten aus Canvas holt, und dort ebenfalls in Umgebungsvariablen.

### Sicherheitshinweise aus der Analyse

Diese Punkte betreffen den API-Betrieb, nicht den lokalen Lauf. Punkt 1 und 3 sind mit der
[Backend-Integration](../Backend-Integration/INTEGRATION.md) behoben:

1. **Die API-Key-Middleware prüft den Key erst nach der Verarbeitung.** **Behoben:** Der Key
   wird jetzt vor dem View geprüft. Ohne eingetragene Keys wird jede Anfrage abgelehnt.
2. **Die Dev-Settings enthalten fest eingetragene Werte (noch offen).**
   `algorithms/api/settings/__init__.py` setzt `DEBUG = True` und `ALLOWED_HOSTS = ["*"]`.
   `SECRET_KEY` kommt inzwischen aus `DJANGO_SECRET_KEY` in `env/.env`, mit einem festen Wert als
   Rückfall. Für den produktiven Betrieb muss daher `settings/prod.py` verwendet werden.
3. **`utils/api_auth.py` sucht beim Import eine Datei `env/.env*`.** **Behoben:** Die Settings
   laden `env/.env` bzw. `env/.env.prod` gezielt, `utils/api_auth.py` lädt keine Datei mehr.
