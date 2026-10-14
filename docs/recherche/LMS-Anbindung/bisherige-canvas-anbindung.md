# Bisherige Canvas-Anbindung

## Kurzfassung

| Teilfrage | Antwort |
| --- | --- |
| **Schnittstelle** | Canvas REST API (`/api/v1/…`) für die Daten, LTI 1.3 für den Einstieg ins Tool |
| **Authentifizierung** | LTI 1.3 (wer ist die Person?) und danach OAuth2 (darf Teamable Daten holen?) |
| **Abgerufene Felder** | Canvas-ID, Name und SIS-ID der Studierenden, Antworten aus einem Opt-in-Quiz, Punktzahlen aus Aufgaben |

## Ausgangslage

Teamable und Canvas sind zwei getrennte Webanwendungen. In Canvas liegen Kurse, Teilnehmerlisten und Noten. Teamable braucht diese Daten, um Teams zu bilden. Dafür müssen zwei Fragen geklärt werden:

1. **Wer ist die Person**, die Teamable gerade öffnet, und aus welchem Kurs kommt sie? → **LTI**
2. **Darf Teamable** Daten aus diesem Kurs abrufen? → **OAuth2**, danach Abruf über die **REST API**

## Was ist LTI?

**LTI (Learning Tools Interoperability)** ist ein offener Standard, mit dem externe Tools in ein LMS eingebunden werden. Er ist nicht Canvas-spezifisch: Auch Moodle, Blackboard oder ILIAS unterstützen ihn.

**Im Alltag:** Im Canvas-Kurs steht im Menü ein Eintrag „Teamable“. Ein Klick darauf öffnet Teamable, ohne neues Passwort und ohne zweiten Login.

**Technisch:** Beim Klick schickt Canvas einen **digital signierten Datensatz** (JWT) an Teamable, vergleichbar mit einem Ausweis. Er enthält:

- wer die Person ist (Canvas-ID, Vor- und Nachname),
- welche Rolle sie hat (z. B. Lehrende oder Studierende),
- aus welchem Kurs sie kommt (Canvas-Kurs-ID, Kurskürzel).

Teamable prüft die Signatur mit einem Schlüssel, der vorher zwischen Canvas und Teamable ausgetauscht wurde. Ist sie gültig, legt Teamable Person und Kurs bei sich an und meldet die Person an.


## OAuth2 und REST API

**REST API:** Canvas bietet eine Programmierschnittstelle. Programme rufen URLs auf und erhalten Daten im JSON-Format zurück, z. B.:

```text
GET https://canvas.ubc.ca/api/v1/courses/123/enrollments
→ Liste aller Studierenden im Kurs 123
```

Teamable spricht diese Schnittstelle über die Python-Bibliothek `canvasapi` an.

**OAuth2** ist der Erlaubnisschein für diese Abfragen. Das Prinzip ist von „App XY möchte auf dein Google-Konto zugreifen – Erlauben?“ bekannt:

1. Teamable leitet die Lehrperson kurz zu Canvas weiter.
2. Canvas fragt, ob Teamable in ihrem Namen auf Kursdaten zugreifen darf.
3. Die Lehrperson klickt „Autorisieren“.
4. Canvas gibt Teamable einen **Token** (Zugangsschlüssel), den Teamable pro Person speichert und bei Ablauf automatisch erneuert.
5. Jede API-Anfrage enthält diesen Token. Teamable sieht damit genau das, was die Lehrperson in Canvas sehen darf.

## Ablauf

1. **Lehrperson klickt in Canvas auf „Teamable“:** Per LTI erfährt Teamable, wer sie ist und aus welchem Kurs sie kommt.
2. **Erlaubnis einholen:** Per OAuth2 klickt sie einmalig „Autorisieren“, Teamable erhält den Token.
3. **Weiterleitung** in die Teamable-Oberfläche.
4. **„Studierende importieren“:** Teamable ruft mit dem Token die Teilnehmerliste über die REST API ab und speichert sie.
5. **Optional:** Antworten aus einem Opt-in-Quiz („Möchtest du mitmachen? Ja/Nein“) und Noten aus Aufgaben abrufen.
6. **Teams bilden** und optional als Gruppen zurück nach Canvas schreiben.

Alle Abrufe werden manuell von Lehrenden per Klick ausgelöst. Es gibt keine automatische Synchronisation im Hintergrund.

## Abgerufene Felder

| Datenquelle in Canvas | Feld | Verwendung in Teamable |
| --- | --- | --- |
| LTI-Launch | Canvas-User-ID, Vor- und Nachname, Rolle | Benutzerkonto und Kursmitgliedschaft der Person, die das Tool öffnet |
| LTI-Launch | Canvas-Kurs-ID, Kurskürzel | Kurs in Teamable anlegen bzw. wiederfinden |
| Teilnehmerliste (Enrollments) | Canvas-User-ID | Eindeutige Zuordnung der Studierenden |
| Teilnehmerliste (Enrollments) | Sortierbarer Name („Nachname, Vorname“) | Vor- und Nachname |
| Teilnehmerliste (Enrollments) | SIS-ID (Studierendennummer, entspricht der Matrikelnummer) | Optional, nur wenn die Lehrperson sie in Canvas sehen darf |
| Opt-in-Quiz | Antwort Ja/Nein pro Person | Nur Studierende mit „Ja“ werden importiert |
| Aufgaben und Abgaben | Punktzahl pro Person und Aufgabe | Einordnung über oder unter dem Median; die Punktzahl selbst wird nicht gespeichert |

**Nicht abgerufen** werden u. a. E-Mail-Adressen, Login-Namen und Übungsgruppen (Sections).

**Nach Canvas geschrieben** wird ebenfalls: Teamable legt das Opt-in-Quiz in Canvas an und exportiert fertige Teams als Canvas-Gruppen.

**Details: Alle genutzten Canvas-API-Endpunkte**    

| Methode | Pfad relativ zu `/api/v1` | Zweck |
|---------|----------------------------|-------|
| `GET` | `/courses/:id` | Kurs laden |
| `GET` | `/courses/:course_id/enrollments?type[]=StudentEnrollment` | Teilnehmerliste laden |
| `GET` | `/courses/:course_id/quizzes/:id` | Opt-in-Quiz laden |
| `GET` | `/courses/:course_id/quizzes/:quiz_id/questions` | Richtige Antwort („Yes“) ermitteln |
| `GET` | `/courses/:course_id/quizzes/:quiz_id/submissions` | Quiz-Abgaben laden |
| `GET` | `/quiz_submissions/:quiz_submission_id/questions` | Antwort je Abgabe laden |
| `GET` | `/courses/:course_id/assignments` | Aufgaben laden |
| `GET` | `/courses/:course_id/assignments/:assignment_id/submissions` | Punktzahlen laden |
| `POST` | `/courses/:course_id/quizzes` | Opt-in-Quiz anlegen |
| `POST` | `/courses/:course_id/quizzes/:quiz_id/questions` | Ja/Nein-Frage anlegen |
| `POST` | `/courses/:course_id/group_categories` | Gruppenkategorie für Team-Export anlegen |
| `POST` | `/group_categories/:group_category_id/groups` | Gruppe pro Team anlegen |
| `POST` | `/groups/:group_id/memberships` | Mitglieder zu Gruppen hinzufügen |

    Diese 13 Endpunkte entsprechen genau den OAuth2-Scopes, die Teamable bei Canvas anfordert.

## Bedeutung für eine Moodle-Anbindung

- **LTI ist ein Standard.** Der Einstieg per LTI wäre mit Moodle grundsätzlich ähnlich umsetzbar. Teamable erwartet aber zwei Canvas-spezifische Zusatzfelder (`canvas_course_id`, `canvas_user_id`) statt der LTI-Standardfelder.
- **REST API und OAuth2-Ablauf sind Canvas-spezifisch.** Moodle hat eine eigene Schnittstelle (Web Services). Dieser Teil müsste neu gebaut oder hinter einer gemeinsamen Schnittstelle versteckt werden.
- **Canvas ist fest verdrahtet.** Es gibt keine Abstraktionsschicht. Die Canvas-Logik steckt in `src/app/canvas/`, aber auch im LTI-Launch, in den Datenmodellen (Links im Canvas-URL-Schema) und in der Teambildung.
- **Opt-in hängt an Canvas-Quizzes.** Diese Funktion müsste für Moodle anders gelöst werden.

