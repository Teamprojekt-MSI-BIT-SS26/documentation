# Backend-Integration des Teambildungs-Algorithmus

**Epic:** Epic 2 – Automate Team Generation  
**Ticket:** #51 – Technical Task 32: Backend Integration & Validation of the Team-Building Algorithm (local)

Teil der Dokumentation zur Backend-Integration: [1. Integration und Schnittstelle](INTEGRATION.md) · [2. Lokale Backend-Tests](TESTS.md)

Der Code liegt in beiden Repositories auf dem Branch
`51-technical-task-32-backend-integration-validation-of-the-team-building-algorithm-local`:

- [algorithms](https://github.com/Teamprojekt-MSI-BIT-SS26/algorithms): Algorithmen-API
- [teamable-backend](https://github.com/Teamprojekt-MSI-BIT-SS26/teamable-backend): Anbindung im Backend

## Ergebnis

| Akzeptanzkriterium | Umsetzung |
| --- | --- |
| Algorithmus ist ans Backend angebunden und über Endpunkte aufrufbar | `POST /api/v1/courses/{id}/generate_teams/` ruft die Algorithmen-API auf und speichert die Teams als `TeamSet`. Abrufbar über `GET /api/v1/courses/{id}/team-sets/{team_set_id}/`. |
| Tests mit einem Testdatensatz (10–20 Studierende) | 15 Studierende. 15 Tests mit simulierter API, 5 Integrationstests gegen die echte API, alle erfolgreich. Siehe [Lokale Backend-Tests](TESTS.md). |
| Gültige, parameterkonforme Ergebnisse | Das Backend prüft jedes Ergebnis vor dem Speichern (siehe [Ergebnisprüfung](#ergebnisprufung)). Die Tests prüfen es zusätzlich über den `team-sets`-Endpunkt. |
| Dokumentation von Schnittstelle und lokalen Tests | Diese Seite und [Lokale Backend-Tests](TESTS.md) |

## Aufbau

Backend und Algorithmen bleiben zwei getrennte Dienste. Das Backend ruft die Algorithmen-API über HTTP auf:

```
Frontend
  │  POST /api/v1/courses/{id}/generate_teams/
  ▼
Backend (teamable-backend, Port 8000)
  │  1. sammelt Studierende, Attribut-Antworten und Beziehungen des Kurses
  │  2. POST /api/generate/teams/  (Header X-Api-Key)
  ▼
Algorithmen-API (algorithms, Port 8001)
  │  random | weight | social | priority
  ▼
Backend
  │  3. prüft das Ergebnis
  │  4. speichert TeamSet + Teams
  ▼
{"team_set_id": 12}
```

**Warum HTTP und nicht direkt als Bibliothek?** Die Algorithmen-API war bereits vorhanden, und das
Backend hatte den Aufruf schon vorbereitet. Außerdem haben die beiden Projekte unterschiedliche
Python-Versionen und Abhängigkeiten: Die Algorithmen brauchen Python 3.9 oder 3.10 und Django 3.2,
das Backend Python 3.11 und Django 5.0. Als getrennte Dienste kommen sie sich nicht in die Quere und
können später getrennt auf dem Server laufen.

## Änderungen

### Repository `algorithms`

Die API war vorher nicht startfähig:

| Problem | Behebung |
| --- | --- |
| Settings, `manage.py`, `wsgi.py`, `asgi.py` und `entrypoint.sh` verwiesen auf das nicht mehr vorhandene Modul `api.api` | Pfade auf `algorithms.api` korrigiert |
| Die API-Key-Middleware prüfte den Key erst **nach** dem Algorithmus-Lauf | Der Key wird jetzt vor dem View geprüft. Ohne eingetragene Keys wird jede Anfrage abgelehnt. |
| `utils/api_auth.py` brach ohne Datei `env/.env*` mit `StopIteration` ab | Die Settings laden jetzt gezielt `env/.env` bzw. `env/.env.prod`. Neu: `python -m utils.api_auth <key>` erzeugt den Hash für einen Key. |
| Für die lokale Entwicklung war PostgreSQL nötig | Ohne `DB_ENGINE` wird eine SQLite-Datei `db.sqlite3` verwendet. |
| Shell-Skripte bekamen unter Windows CRLF-Zeilenenden und liefen im Linux-Container nicht | `.gitattributes` mit `*.sh text eol=lf` |
| Beim Merge `f327ca3a` wurde `local_run/run.py` durch eine ältere Version überschrieben, `tests/test_local_run/test_run.py` schlug fehl | Version aus `fe7fde72` wiederhergestellt. Die Aufrufe aus `main` (`python -m local_run.run <datei>`) funktionieren weiterhin. |

Neu sind die Vorlage `env/api.env.example`, die Tests in
`tests/test_api/test_views/test_generate_teams.py` und die Anleitung `algorithms/api/README.md`.

### Repository `teamable-backend`

| Datei | Änderung |
| --- | --- |
| `src/app/services/team_generation.py` | Komplett neu: Payload bauen, API aufrufen, Ergebnis prüfen und speichern. Die vorherige, fest codierte Paarbildung ist entfernt. |
| `src/app/serializers/team_generation.py` | Neu: prüft die Parameter der Anfrage |
| `src/app/views/course.py` | `generate_teams` übergibt die Parameter und gibt Fehler als `{"error": ...}` zurück |
| `src/teamable/settings.py`, `sample.env` | `TEAM_GENERATION_API_URL`, `TEAM_GENERATION_API_KEY`, `TEAM_GENERATION_API_TIMEOUT` |
| `src/app/tests/` | Testdaten und Tests, siehe [Lokale Backend-Tests](TESTS.md) |

## Lokale Einrichtung

Voraussetzungen: Python 3.10 für `algorithms`, Python 3.11 für `teamable-backend`. Docker und
PostgreSQL sind nicht nötig.

**1. Algorithmen-API starten** (im Repository `algorithms`):

```bash
python -m pip install -r requirements.txt
cp env/api.env.example env/.env
python -m utils.api_auth mein-geheimer-key   # Ausgabe kopieren
```

Den ausgegebenen Hash in `env/.env` bei `API_AUTH_ALLOWED_KEYS=` eintragen. Dann:

```bash
python manage.py migrate
python manage.py runserver 8001
```

**2. Backend konfigurieren** (im Repository `teamable-backend`): In der `.env` (Kopie von
`sample.env`) den Key im **Klartext** eintragen:

```
TEAM_GENERATION_API_URL=http://localhost:8001/api/generate/teams/
TEAM_GENERATION_API_KEY=mein-geheimer-key
TEAM_GENERATION_API_TIMEOUT=90
```

**3. Backend starten** wie gewohnt mit `python manage.py runserver` im Ordner `src`.

| Variable (Backend) | Standard | Bedeutung |
| --- | --- | --- |
| `TEAM_GENERATION_API_URL` | `http://localhost:8001/api/generate/teams/` | Endpunkt der Algorithmen-API |
| `TEAM_GENERATION_API_KEY` | keiner | API-Key im Klartext. Ohne Key antwortet die Teamgenerierung mit `503`. |
| `TEAM_GENERATION_API_TIMEOUT` | `90` | Wartezeit in Sekunden. Der Priority-Algorithmus rechnet bis zu 30 Sekunden. |

!!! info "Warum zwei Versionen des Keys?"
    Die Algorithmen-API speichert nur einen bcrypt-Hash des Keys, das Backend schickt den Klartext
    im Header `X-Api-Key` mit. Wer die `.env` der Algorithmen-API liest, kennt den Key dadurch nicht.

## Schnittstelle

`POST /api/v1/courses/{course_id}/generate_teams/`

Nur Lehrende des Kurses dürfen den Endpunkt aufrufen. Alle Felder sind optional:

| Feld | Typ | Standard | Bedeutung |
| --- | --- | --- | --- |
| `algorithm_type` | `"priority"`, `"weight"`, `"social"`, `"random"` | `"priority"` | Algorithmus |
| `min_team_size` | Zahl ≥ 1 | `3` | Mindestgröße eines Teams |
| `max_team_size` | Zahl ≥ 1 | `5` | Höchstgröße eines Teams |
| `total_teams` | Zahl ≥ 1 | so wenige wie möglich: `ceil(Studierende / max_team_size)` | Anzahl der Teams |
| `attributes` | Liste von Attribut-IDs | alle Attribute des Kurses mit Antwortoptionen | Attribute, nach denen gemischt wird |
| `attribute` | Attribut-ID | | Einzelnes Attribut, wie es das Frontend bisher schickt. Entspricht `attributes: [id]`. |
| `priorities` | Liste von Objekten | automatisch, siehe unten | Nur für `priority`: Prioritäten im Format der Algorithmen-API, wichtigste zuerst |

**Priority als Standard:** Der Priority-Algorithmus ist der Beitrag der UBC und liefert laut Paper die
besten Teams. Die anderen Algorithmen bleiben über `algorithm_type` wählbar.

Beispiel:

```json
{
  "algorithm_type": "priority",
  "min_team_size": 3,
  "max_team_size": 4,
  "total_teams": 4
}
```

Antwort `200`:

```json
{"team_set_id": 12}
```

Die Teams liefert danach `GET /api/v1/courses/{course_id}/team-sets/{team_set_id}/`:

```json
{
  "id": 12,
  "name": "Team Set 1 (priority)",
  "teams": [
    {
      "id": 40,
      "slug": "3f0c…",
      "name": "Team 1",
      "members": [{"id": 3, "name": "Anna Test", "role": "Student", "…": "…"}]
    }
  ],
  "unassigned_students": []
}
```

### Fehler

Fehler kommen als `{"error": "<Meldung>"}` zurück, ungültige Felder als die üblichen Serializer-Fehler.
Bei einem Fehler wird **kein** Team-Set gespeichert.

| Status | Ursache |
| --- | --- |
| `400` | Ungültige Felder, z. B. `min_team_size` > `max_team_size`, ein Attribut eines anderen Kurses, `priorities` ohne `priority` |
| `400` | Die Algorithmen-API lehnt die Eingabe ab, z. B. weil die Studierenden nicht in `total_teams` Teams der angegebenen Größen passen |
| `400` | Der Kurs hat keine Studierenden |
| `502` | Die Algorithmen-API ist nicht erreichbar, lehnt den Key ab, schlägt fehl oder liefert ein ungültiges Team-Set |
| `503` | `TEAM_GENERATION_API_KEY` ist nicht gesetzt |
| `504` | Die Algorithmen-API antwortet nicht innerhalb von `TEAM_GENERATION_API_TIMEOUT` Sekunden |

## Datenumwandlung

Backend und Algorithmen speichern Daten unterschiedlich. Das Backend wandelt sie so um:

| Backend | Algorithmen-API |
| --- | --- |
| `CourseMember` mit Rolle `Student` | `students[].id`, `students[].name` |
| `AttributeResponse` | `students[].attributes`: `{"<Attribut-ID>": [Werte]}`. Zahlen-Attribute behalten ihren Wert, alle anderen werden je Antwortoption durchnummeriert (0, 1, 2, …). |
| `Relationship` vom Typ `Friend` / `Enemy` | `students[].relationships`: `{"<Studierenden-ID>": "friend" \| "enemy"}`. Beziehungen zu Personen außerhalb des Kurses werden ignoriert. |
| Felder der Anfrage | `team_generation_options` und `algorithm_options` |

Für `weight`, `social` und `priority` gehen die gewählten Attribute als `attributes_to_diversify`
an die API. Der Priority-Algorithmus nutzt diese Optionen für sein Start-Team-Set.

Ohne `priorities` bekommt der Priority-Algorithmus diese Prioritäten, wichtigste zuerst:

1. `social_preference`: Freunde zusammen, Feinde getrennt. Nur, wenn es Beziehungen gibt.
2. `diversity` mit Strategie `diversify`, eine pro Attribut.

## Ergebnisprüfung

Bevor das Backend speichert, prüft es das Ergebnis der API. Schlägt eine Prüfung fehl, antwortet es
mit `502` und speichert nichts:

- Die Anzahl der Teams ist `total_teams`.
- Jede Person des Kurses ist in genau einem Team, und es kommt keine unbekannte Person zurück.
- Jedes Team hat zwischen `min_team_size` und `max_team_size` Mitglieder.

## Noch nicht umgesetzt

- **Team-Anforderungen und Projekt-Präferenzen:** `TeamRequirement`, Team-Vorlagen und
  Projekt-Präferenzen werden noch nicht übertragen. `initial_teams` ist leer,
  `max_project_preferences` ist 0. Im Backend hängen Anforderungen an Team-Vorlagen, die noch keinem
  Kurs zugeordnet werden, und die Operatoren lassen sich nur teilweise übersetzen: Das Backend kennt
  `GT`, `IN`, `CONTAINS` usw., die Algorithmen nur `exactly`, `less than` und `more than`.
- **Frontend:** Das Frontend schickt nur `attribute` und nutzt damit immer die Standardwerte für
  Algorithmus und Teamgrößen.
- **Lange Anfragen auf dem Server:** Die Anfrage läuft synchron. Auf dem Server müssen die Timeouts
  von gunicorn und nginx beider Dienste länger sein als die Laufzeit des Priority-Algorithmus.
  `entrypoint.sh` der Algorithmen leert außerdem bei jedem Start die Datenbank (`flush`).
