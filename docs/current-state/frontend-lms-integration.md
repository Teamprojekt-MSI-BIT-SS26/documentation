# Teamable Frontend – Dokumentation zur LMS-Anbindung

> **Zweck:** Diese Doku beschreibt den aktuellen Stand von `teamable-frontend` mit Blick auf die Anbindung ans LMS (derzeit nur Canvas). Sie zeigt, **welche LMS-Funktionen das System heute braucht**. Eine vollständige Code-Doku ist sie nicht.
>
> **Wichtig vorab:** Das Frontend spricht **nie direkt** mit Canvas. Alle LMS-Zugriffe laufen über das Backend (`teamable-backend`). Das Frontend löst nur Aktionen aus (z. B. „Studierende importieren“) und zeigt Daten an, die das Backend vorher aus dem LMS geholt hat. Was das Backend dabei konkret in Canvas aufruft, steht in der Backend-Doku. Hier sind die Backend-Endpunkte nur als Schnittstelle aufgeführt.

---

## 1. Technischer Überblick

| Thema | Stand |
|---|---|
| Framework | Next.js 14 (App Router), React 18, TypeScript |
| Datenabruf | TanStack React Query v5 mit einer globalen Standard-`queryFn` |
| UI | Tailwind CSS, shadcn/ui (Radix), Formik + Yup für Formulare |
| Auth-Cookie | `cookies-next` (Client) bzw. `next/headers` (Server) |
| Node | v20.16.0 (`.nvmrc`) |
| Konfiguration | `.env` mit `BACKEND_BASE_URI` (siehe [sample.env](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/sample.env)) |
| Deployment | [Dockerfile](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/Dockerfile) + [docker-compose.yml](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/docker-compose.yml) (Port 8003 → 3000) |

### Ordnerstruktur (relevante Teile)

```
src/
├── app/
│   ├── (accounts)/
│   │   ├── accounts/login/        → Login mit E-Mail/Passwort
│   │   ├── accounts/signup/       → Registrierung mit Einladungs-Token
│   │   ├── accounts/401/          → Fehlerseite „Authentication error“
│   │   └── authenticate/route.ts  → Einstieg nach Canvas-Login (setzt Token-Cookie)
│   ├── (app)/course/[courseId]/
│   │   ├── (hooks)/useCourse.tsx  → lädt den Kurs inkl. LMS-Typ
│   │   ├── setup/                 → Onboarding-Assistent (Hauptablauf)
│   │   ├── students/              → Studierendenliste + Import-Buttons
│   │   ├── team-sets/             → Team-Sets, Teams generieren, Export ins LMS
│   │   ├── profiles/              → statische Platzhalter-UI (kein Backend)
│   │   └── project-sets/          → nicht angebunden (siehe Abschnitt 10)
│   ├── (marketing)/privacy/       → Datenschutzseite (nennt Canvas als Datenquelle)
│   ├── (providers)/
│   │   ├── query-client-provider.tsx → zentrale fetch-Logik zum Backend
│   │   └── auth-user-provider.tsx    → Login-Status + Weiterleitungen
│   └── page.tsx                   → Startseite: leitet auf ersten Kurs weiter
├── hooks/                         → ein Hook pro Backend-Aktion (LMS-Hooks hier)
├── _temp_types/                   → TypeScript-Typen der Backend-Antworten
└── routes.ts                      → Auth-Routen
utils/
├── auth.ts / auth-server.ts       → baut den Authorization-Header aus dem Cookie
└── string.ts
```

---

## 2. Architektur: Wer spricht mit dem LMS?

```
┌────────────┐  LTI/OAuth-Start   ┌──────────────────┐   REST + OAuth2-Token   ┌──────────┐
│  Canvas    │ ─────────────────▶ │ teamable-backend │ ◀─────────────────────▶ │ Canvas   │
│ (Kurslink) │                    │  (Django, :8000) │                         │ REST-API │
└────────────┘                    └──────────────────┘                         └──────────┘
                                     ▲          │
                   REST /api/v1/...  │          │ Redirect auf
                   Header:           │          │ /authenticate?token=…&path=…
                   "Token <token>"   │          ▼
                                  ┌──────────────────┐
                                  │ teamable-frontend│
                                  │ (Next.js, :3000) │
                                  └──────────────────┘
```

- **Frontend → Backend:** Ausschließlich REST über `fetch` auf `${BACKEND_BASE_URI}/api/v1/...`. Authentifiziert wird mit einem Django-REST-Framework-Token im Header `Authorization: Token <token>`.
- **Backend → Canvas:** Das Backend hält das Canvas-OAuth2-Token der Lehrperson (Modell `CanvasOAuth2Token`, siehe `teamable-backend/src/canvas_oauth/oauth.py`) und ruft damit die Canvas-API auf.
- **Canvas → Backend → Frontend:** Die Lehrperson startet Teamable aus Canvas. Das Backend erledigt den OAuth-Flow mit Canvas, erzeugt danach ein Teamable-Token und leitet den Browser auf die Frontend-Route `/authenticate` weiter.

**Für Moodle heißt das:** Wird Moodle angebunden, muss im Frontend fast nichts geändert werden. Es braucht im Wesentlichen nur andere Texte (siehe Abschnitt 10). Die eigentliche Arbeit liegt im Backend: Launch, OAuth bzw. Token und API-Aufrufe.

---

## 3. Canvas-Anbindung im Frontend-Code

### 3.1 Alle Stellen mit LMS-/Canvas-Bezug

| Datei | Bezug |
|---|---|
| [src/app/(accounts)/authenticate/route.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28accounts%29/authenticate/route.ts) | Ziel des Redirects nach Canvas-Login: übernimmt `token` und `path` aus der URL, setzt das Cookie `token`, leitet auf `path` weiter |
| [src/hooks/use-import-students-from-lms.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-import-students-from-lms.ts) | `POST courses/{id}/import_students_from_lms/` |
| [src/hooks/use-import-student-gradebook-data.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-import-student-gradebook-data.ts) | `POST courses/{id}/import_gradebook_attribute_from_lms/` |
| [src/hooks/use-create-opt-in-quiz.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-create-opt-in-quiz.ts) | `POST courses/{id}/create_opt_in_quiz_lms/` (legt ein Quiz in Canvas an) |
| [src/hooks/use-export-team-set-lms.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-export-team-set-lms.ts) | `POST courses/{id}/export_team_to_lms/` mit `{ team_set }` (schreibt Gruppen ins LMS) |
| [src/hooks/use-grade-attributes.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-grade-attributes.ts) | `GET courses/{id}/grade-attributes/`: Noten-Attribute, die aus dem LMS-Gradebook stammen |
| [src/hooks/use-onboarding-progress.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-onboarding-progress.ts) | `GET courses/{id}/get_onboarding_progress/`, enthält u. a. `opt_in_quiz_link` (Link ins LMS) |
| [src/app/(app)/course/[courseId]/(hooks)/useCourse.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/%28hooks%29/useCourse.tsx) | Lädt den Kurs. `course.organization.lms_type` wird als `lmsType` bereitgestellt |
| [src/app/(app)/course/[courseId]/team-sets/[teamSetId]/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/team-sets/%5BteamSetId%5D/page.tsx) | Button „Export to {LMS}“ (Name kommt aus `lmsType`) |
| [src/app/(app)/course/[courseId]/team-sets/[teamSetId]/columns.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/team-sets/%5BteamSetId%5D/columns.tsx) | Klick auf ein Teammitglied öffnet `member.lms_link` (Profil im LMS) |
| [src/app/(app)/course/[courseId]/setup/(hooks)/useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx) | Onboarding-Schritte, Texte nennen „Canvas“ fest |
| [src/app/(app)/course/[courseId]/setup/(components)/GenerateOptInQuiz.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28components%29/GenerateOptInQuiz.tsx) | Button „Generate opt-in quiz“ bzw. „View opt-in quiz“ |
| [src/components/GradeSourceAttributeSelector.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/components/GradeSourceAttributeSelector.tsx) | Auswahl des Noten-Attributs (im Code heißt die Variable `canvasAssignment`) |
| [src/_temp_types/organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts) | `LMS_TYPES = { CANVAS: "CANVAS" }`, derzeit einziger Wert |
| [src/app/(marketing)/privacy/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28marketing%29/privacy/page.tsx) | Datenschutztext: Name, Student-ID und Noten werden aus Canvas gelesen |

Alle Fehlermeldungen der LMS-Hooks lauten gleich: *„Please relaunch Teamable from your Canvas dashboard …“*. Bei Fehlern (z. B. abgelaufenes Canvas-Token im Backend) soll die Lehrperson Teamable also neu aus Canvas starten. Dabei wird der OAuth-Flow erneut durchlaufen.

### 3.2 Login über Canvas (Launch-Flow)

1. Die Lehrperson öffnet Teamable aus dem Canvas-Kurs.
2. Das **Backend** prüft, ob für den Nutzer schon ein gültiges Canvas-OAuth2-Token existiert. Falls nicht, startet es den OAuth-Flow (`authorization_code`) und speichert `access_token`, `refresh_token` und `expires` in `CanvasOAuth2Token`.
3. Das Backend erzeugt bzw. holt das Teamable-API-Token (`rest_framework.authtoken.Token`) und leitet weiter auf
   `${FRONTEND_BASE_URL}/authenticate?token=<teamable-token>&path=<redirect_path>`
4. **Frontend** ([authenticate/route.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28accounts%29/authenticate/route.ts)): setzt das Cookie `token=<teamable-token>` (Pfad `/`) und leitet auf `path` weiter, z. B. `/course/12/setup`.
5. Ab jetzt schickt jeder Frontend-Request `Authorization: Token <token>`.

→ **Das Canvas-Token selbst sieht das Frontend nie.** Es liegt nur im Backend.

### 3.3 Wann werden Daten geholt oder zurückgeschrieben?

Alle LMS-Aktionen werden **manuell per Button** ausgelöst. Es gibt keine automatische Synchronisation und kein Polling.

| Aktion (Button) | Richtung | Backend-Endpunkt | Wo in der UI |
|---|---|---|---|
| Import students | LMS → Teamable | `POST courses/{id}/import_students_from_lms/` | Setup-Schritt 1, Studierenden-Seite |
| Generate opt-in quiz | Teamable → LMS (Quiz anlegen) | `POST courses/{id}/create_opt_in_quiz_lms/` | Setup-Schritt 1 |
| View opt-in quiz | Link ins LMS | `opt_in_quiz_link` aus `get_onboarding_progress` | Setup-Schritt 1 |
| Import gradebook data | LMS → Teamable | `POST courses/{id}/import_gradebook_attribute_from_lms/` | Setup-Schritt 3, Studierenden-Seite |
| Export to Canvas | Teamable → LMS (Gruppen anlegen) | `POST courses/{id}/export_team_to_lms/` Body `{ "team_set": <id> }` | Team-Set-Detailseite |
| Klick auf Teammitglied | Link ins LMS | `lms_link` des Studierenden | Team-Set-Detailseite |

Nach jedem Import ruft das Frontend `refetch()` auf, z. B. für den Onboarding-Fortschritt oder die Studierendenliste, und zeigt so die neuen Daten an.

---

## 4. Datenmodell (Frontend-Typen) mit LMS-Bezug

Die Typen in [src/_temp_types/](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8/src/_temp_types) bilden die JSON-Antworten des Backends ab. Die Werte füllt **immer das Backend**. Die Spalte „Herkunft“ beschreibt, woher die Daten fachlich stammen; die Details zur Befüllung stehen im Backend.

### `Course` – [course.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/course.ts)

| Feld | LMS-Bezug | Herkunft |
|---|---|---|
| `id` | – | Teamable-interne ID (steht in der URL `/course/{id}`) |
| `name` | ja | Kursname (aus dem LMS-Kurs) |
| `lms_course_id` | **ja** | ID des Kurses im LMS |
| `organization` | ja | siehe `Organization` |
| `sections: CourseSection[]` | – | Übungsgruppen (Sections), im Frontend als Filter in der Studierendenliste genutzt. Sie kommen **nicht** aus dem LMS, sondern werden in Teamable angelegt (Django-Admin oder `makefakeclass`). |

### `Organization` – [organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts)

| Feld | LMS-Bezug | Herkunft |
|---|---|---|
| `name` | – | Institution |
| `lms_type` | **ja** | `"CANVAS"` (einziger definierter Wert). Steuert nur Beschriftungen wie „Export to Canvas“ |

### `Student` – [student.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/student.ts)

| Feld | LMS-Bezug | Herkunft |
|---|---|---|
| `id` | – | Teamable-intern |
| `lms_id` | **ja** | User-ID im LMS |
| `sis_user_id` | **ja** | SIS-ID (Matrikelnummer o. Ä.) aus dem LMS |
| `lms_link` | **ja** | URL zum Nutzerprofil im LMS (`null` möglich) |
| `name`, `first_name`, `last_name` | ja | aus dem LMS-Import |
| `sections` | – | Zugehörigkeit zu Sections, in Teamable gepflegt (nicht aus dem LMS) |
| `role` | ja | `"Student"` oder `"Instructor"` ([courseMember.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/courseMember.ts)) |

### `OnboardingProgress` – [onboarding.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/onboarding.ts)

| Feld | Bedeutung |
|---|---|
| `has_created_opt_in_quiz` | Opt-in-Quiz im LMS wurde angelegt |
| `opt_in_quiz_link` | **Link zum Quiz im LMS** |
| `has_students` | Studierende wurden importiert |
| `has_attribute` | Ein Attribut existiert (z. B. aus dem Gradebook) |
| `has_attribute_responses` | Werte zu Attributen liegen vor (z. B. importierte Noten) |
| `has_team_set` | Mindestens ein Team-Set wurde generiert |

### `GradeAttribute` – [use-grade-attributes.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-grade-attributes.ts)

| Feld | Bedeutung |
|---|---|
| `id`, `name` | Ein Noten-Attribut. Der Name entspricht in der Regel einer LMS-Aufgabe bzw. einer Gradebook-Spalte (im Code `canvasAssignment` genannt) |

### `TeamSet` / `Team` – [teamSet.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/teamSet.ts), [team.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/team.ts)

Teams und Team-Sets entstehen in Teamable. Einen LMS-Bezug haben sie nur über den **Export** (`export_team_to_lms`) und über ihre Mitglieder (`Student`).

### `AuthUser` – [user.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/user.ts)

`id`, `username`, `first_name`, `last_name`, `email`, `is_staff`, `course_memberships: [{ course }]`. Über `course_memberships` wird festgelegt, auf welchen Kurs die Startseite weiterleitet.

---

## 5. Ablauf aus Sicht der Lehrenden

Der Hauptablauf ist der Onboarding-Assistent unter `/course/{courseId}/setup` ([useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx)). Welcher Schritt gerade aktiv ist, ergibt sich aus `get_onboarding_progress`.

| # | UI-Schritt | Aktiv? | Aktion | Backend-Endpunkt | LMS-Zugriff (im Backend) |
|---|---|---|---|---|---|
| 0 | Teamable aus Canvas starten | – | Redirect | Backend-OAuth → `GET /authenticate?token&path` (Frontend) | Canvas OAuth2 |
| 0b | Seite laden | – | Kurs + Nutzer laden | `GET courses/{id}/`, `GET accounts/users/me/` | – |
| 1 | **Import students** | ja | Button „Import students“ | `POST courses/{id}/import_students_from_lms/` | **lesen**: Teilnehmende (ggf. Antworten im Opt-in-Quiz) |
| 1a | (optional) Opt-in-Quiz | ja, solange keine Studierenden importiert sind | „Generate opt-in quiz“ | `POST courses/{id}/create_opt_in_quiz_lms/` | **schreiben**: Quiz anlegen. Danach werden nur Studierende importiert, die zugestimmt haben |
| 1b | Quiz ansehen | ja | „View opt-in quiz“ | – (öffnet `opt_in_quiz_link`) | – |
| 2 | Define student profile | **deaktiviert** | – | – | – |
| 3 | **Elicit student information** | ja | „Import gradebook data“ | `POST courses/{id}/import_gradebook_attribute_from_lms/` | **lesen**: Gradebook, Noten |
| 4 | (Optional) Project setup | **deaktiviert** | – | – | – |
| 5 | **Generate teams** | ja | Dialog „Select a grade source“: Noten-Attribut wählen, dann „Generate teams“ | `GET courses/{id}/grade-attributes/` und dann `POST courses/{id}/generate_teams/` mit `{ "attribute": <id> }` | – (arbeitet auf importierten Daten) |
| 6 | Team-Set ansehen | – | Toast „View“ bzw. Seite Team sets | `GET courses/{id}/get_team_sets/`, `GET courses/{id}/team-sets/{teamSetId}/` | – |
| 7 | **Export to Canvas** | – | Button auf der Team-Set-Detailseite | `POST courses/{id}/export_team_to_lms/` mit `{ "team_set": <id> }` | **schreiben**: Gruppen anlegen |

Weitere Seiten:

- **Students** (`/course/{id}/students`): paginierte Liste `GET courses/{id}/students/?page=&per_page=&sort=&search=&sections=`. Enthält dieselben Buttons „Import students“ und „Import gradebook data“.
- **Team sets** (`/course/{id}/team-sets`): Liste der Team-Sets und Button „Generate teams“ (gleicher Dialog wie Schritt 5).
- **Profiles** (`/course/{id}/profiles`): nur statische Demo-Komponenten ohne Backend-Aufruf.

Startseite `/` ([page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/page.tsx), serverseitig): `GET accounts/users/me/`.
- Nicht eingeloggt → `/accounts/signup`
- Ohne `course_memberships` → `/accounts/401`
- Sonst → `/course/{erste course_membership}/setup`

---

## 6. Anmeldung, Rechte und Token-Speicherung

### 6.1 Anmeldewege

| Weg | Für wen | Ablauf | Endpunkte |
|---|---|---|---|
| **Canvas-Launch** (Produktiv) | Lehrende | siehe 3.2. Das Backend leitet auf `/authenticate?token=…&path=…` weiter | Backend-OAuth, danach Frontend-Route `/authenticate` |
| **Sign-up mit Einladungs-Token** | Lehrende (v. a. lokal) | `/accounts/signup?token=<INSTRUCTOR_SIGNUP_TOKEN>`, E-Mail und Passwort, danach automatischer Login | `POST accounts/sign-up/` `{ email, password, token }` |
| **Login** | Lehrende | `/accounts/login`, E-Mail und Passwort | `POST accounts/log-in/` `{ email, password }` → `{ token }` |
| **Logout** | – | Cookie löschen, danach `/accounts/login` (kein Backend-Aufruf) | – |

**Studierende** melden sich im aktuellen Frontend **nicht** an. Es gibt keine Studierenden-Oberfläche. Sie haben nur im LMS mit dem System zu tun: über das Opt-in-Quiz und über die exportierten Gruppen. Im Datenmodell tauchen sie nur als `Student` mit `role = "Student"` auf.

### 6.2 Wo liegen Tokens?

| Token | Speicherort | Wer setzt es |
|---|---|---|
| Teamable-API-Token (DRF `Token`) | **Browser-Cookie `token`**, Pfad `/`, nicht `httpOnly` | `/authenticate` (serverseitig via `cookies().set`) oder `useLogin` (clientseitig via `setCookie`) |
| Canvas-OAuth2-Access/Refresh-Token | **nur im Backend** (DB-Modell `CanvasOAuth2Token`) | Backend-OAuth-Flow |

Das Cookie wird über [utils/auth.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/utils/auth.ts) (Client) bzw. [utils/auth-server.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/utils/auth-server.ts) (Server-Komponenten) in den Header `Authorization: Token <token>` übersetzt.

### 6.3 Rechte und Routenschutz

- [auth-user-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/auth-user-provider.tsx) lädt `accounts/users/me/`.
  - Nicht eingeloggt und keine Auth-Seite → Weiterleitung zu `/accounts/login`
  - Eingeloggt und Auth-Seite → Weiterleitung zu `/`
- Das Frontend prüft **keine Rollen** (Lehrende vs. Studierende) und keinen Kurszugriff. Ob jemand einen Kurs sehen darf, entscheidet allein das Backend. Bekommt `useCourse` keine Kurs-ID zurück, zeigt das Frontend „Course not found“.

---

## 7. Daten für die Team-Bildung

Heute fließen genau diese LMS-Daten in die Team-Bildung ein:

1. **Studierendenliste** aus dem LMS-Kurs (`import_students_from_lms`), optional gefiltert über das Opt-in-Quiz. Das sind die Personen, die auf Teams verteilt werden. Studierende ohne Team erscheinen als `unassigned_students`.
2. **Noten aus dem Gradebook** (`import_gradebook_attribute_from_lms`). Laut UI-Text werden sie *„zu Werten verarbeitet, die die relative Leistung jedes Studierenden angeben“*. Jede importierte Notenquelle wird zu einem **Attribut** (`GradeAttribute`).
3. Bei „Generate teams“ wählt die Lehrperson **genau ein** Noten-Attribut. Das Frontend schickt nur `{ attribute: <id> }` an `generate_teams`. Algorithmus und Konfiguration liegen vollständig im Backend („A pre-configuration we have set up for your course“).
4. **Sections** werden im Frontend nur zum Filtern der Studierendenliste genutzt. Sie kommen nicht aus dem LMS, und die Team-Bildung im Backend nutzt sie nicht.

Was die Team-Bildung **heute nicht** nutzt (die Schritte sind deaktiviert): Studierendenprofile oder Umfragen (Schritt 2), Projektvorgaben (Schritt 4) und Freundschafts- bzw. Konfliktbeziehungen (der Typ [relationship.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/relationship.ts) existiert, wird aber nicht verwendet).

**Mindestanforderungen an eine LMS-API aus Sicht des Frontends:**

- Kursinformationen lesen (Name, Kurs-ID)
- Teilnehmende mit Rolle, Namen, LMS-ID, SIS-ID und Profil-Link lesen
- Gradebook bzw. Aufgaben mit Noten pro Studierendem lesen
- Ein Quiz bzw. eine Umfrage anlegen und deren Ergebnisse lesen (für das Opt-in)
- Gruppen bzw. Gruppensets anlegen und Mitglieder zuordnen (Export)
- Launch mit Nutzer- und Kurskontext sowie Nutzer-Autorisierung (OAuth2 oder vergleichbar)

---

## 8. Lokales Setup (auch ohne Canvas)

### 8.1 Frontend starten

```bash
cp sample.env .env          # BACKEND_BASE_URI=http://localhost:8000
nvm use                     # Node 20.16.0
yarn                        # Abhängigkeiten installieren
yarn dev                    # http://localhost:3000
```

Alternativ mit Docker: `docker compose up --build` → http://localhost:8003. Die `.env` muss vorhanden sein.

### 8.2 Ohne Canvas testen

1. Backend lokal starten (siehe README in `teamable-backend`) und dort ein **Instructor-Signup-Token** erzeugen.
2. `http://localhost:3000/accounts/signup?token=<INSTRUCTOR_SIGNUP_TOKEN>` öffnen, E-Mail und ein starkes Passwort eingeben. Danach folgt der automatische Login und die Weiterleitung zum Kurs-Onboarding.
3. Der Canvas-Launch wird so komplett umgangen: Login per E-Mail/Passwort, Token im Cookie.

**Einschränkungen ohne Canvas:**

- Was die Buttons, die das LMS brauchen (Import students, Import gradebook data, Generate opt-in quiz, Export to Canvas), tun, hängt vom LMS-Typ der Organisation im Backend ab:
    - **Organisation ohne `lms_type`:** Das Backend antwortet sofort mit `200 OK`, ohne Canvas anzusprechen. Die Buttons melden Erfolg, es werden aber keine Daten importiert oder exportiert.
    - **Organisation mit `lms_type = CANVAS`:** Die Aufrufe schlagen fehl, weil der per E-Mail angemeldete Nutzer kein Canvas-OAuth-Token hat. Sichtbar wird das als Toast „Please relaunch Teamable from your Canvas dashboard …“.
- Um Team-Bildung und Team-Ansichten zu testen, müssen Studierende, Noten-Attribute und Antworten **im Backend** angelegt werden, z. B. per Django-Admin oder Seed-Daten. Das Frontend hat keine Mock-Daten und keinen Mock-Modus.
- Den Canvas-Redirect kann man manuell simulieren, wenn ein gültiges Teamable-Token vorliegt:
  `http://localhost:3000/authenticate?token=<DRF-Token>&path=/course/<id>/setup`

### 8.3 Weitere Befehle

| Befehl | Zweck |
|---|---|
| `yarn build` / `yarn start` | Produktions-Build bzw. -Server |
| `yarn lint` / `yarn format` | ESLint |
| `yarn test` | Jest (nur Beispieltest) |
| `yarn cypress:open` | Cypress (nur Beispielspec) |

CI ([.github/workflows/check.yml](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/.github/workflows/check.yml)) führt Lint und Build aus.

---

## 9. Vollständige Liste der Backend-Aufrufe

Basis: `${BACKEND_BASE_URI}/api/v1/`. An jeden Pfad wird automatisch ein abschließender `/` angehängt. GET-Aufrufe laufen über die Standard-`queryFn` (Query-Key = Pfad), POST-Aufrufe über `defaultMutationFn` ([query-client-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/query-client-provider.tsx)).

| Methode | Pfad | Body | Hook / Datei | LMS |
|---|---|---|---|---|
| GET | `accounts/users/me/` | – | `use-auth-user-query.ts`, `app/page.tsx` | – |
| POST | `accounts/log-in/` | `{ email, password }` | `use-login.ts` | – |
| POST | `accounts/sign-up/` | `{ email, password, token }` | `use-sign-up.ts` | – |
| GET | `courses/{id}/` | – | `useCourse.tsx` | liefert `lms_type`, `lms_course_id` |
| GET | `courses/{id}/get_onboarding_progress/` | – | `use-onboarding-progress.ts` | liefert `opt_in_quiz_link` |
| GET | `courses/{id}/students/?page&per_page&sort&search&sections` | – | `useStudents.tsx` | liefert `lms_id`, `lms_link`, `sis_user_id` |
| POST | `courses/{id}/import_students_from_lms/` | – | `use-import-students-from-lms.ts` | **LMS lesen** |
| POST | `courses/{id}/import_gradebook_attribute_from_lms/` | – | `use-import-student-gradebook-data.ts` | **LMS lesen** |
| POST | `courses/{id}/create_opt_in_quiz_lms/` | – | `use-create-opt-in-quiz.ts` | **LMS schreiben** |
| GET | `courses/{id}/grade-attributes/` | – | `use-grade-attributes.ts` | indirekt (Gradebook) |
| POST | `courses/{id}/generate_teams/` | `{ attribute }` | `use-generate-teams.tsx` | – |
| GET | `courses/{id}/get_team_sets/` | – | `use-list-team-sets.ts` | – |
| GET | `courses/{id}/team-sets/{teamSetId}/` | – | `use-team-set.ts` | – |
| POST | `courses/{id}/export_team_to_lms/` | `{ team_set }` | `use-export-team-set-lms.ts` | **LMS schreiben** |
| GET | `teamset-templates` (über `NEXT_PUBLIC_BACKEND_URL`) | – | `project-sets/page.tsx` | – (nicht funktionsfähig, s. u.) |

---

## 10. Auffälligkeiten und Hinweise für eine Moodle-Anbindung

**Was im Frontend für Moodle anzupassen wäre (wenig):**

- `LMS_TYPES` in [organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts) um z. B. `MOODLE` erweitern. Die Export-Beschriftung nutzt `lmsType` bereits dynamisch (Ausgabe „Export to Moodle“).
- Fest eingebaute „Canvas“-Texte durch `lmsType` ersetzen. Betroffen sind:
  - die Onboarding-Texte in [useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx)
  - die Fehler-Toasts in allen LMS-Hooks unter [src/hooks/](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8/src/hooks)
  - der Datenschutztext in [privacy/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28marketing%29/privacy/page.tsx)
- Die Endpunktnamen sind schon LMS-neutral (`..._from_lms`, `..._to_lms`). Das Frontend kann unverändert bleiben, wenn das Backend je nach `lms_type` intern Canvas oder Moodle anspricht.
- Die Route `/authenticate` ist LMS-neutral: Ein Moodle-Launch (z. B. LTI 1.3) im Backend müsste am Ende nur auf dieselbe URL umleiten.

**Bekannte Schwachstellen im aktuellen Code:**

- `authenticate/route.ts` gibt bei fehlenden Parametern ein einfaches Objekt statt einer `Response` zurück. Ein fehlerhafter Aufruf führt deshalb zu einem Serverfehler statt zu einer sauberen 400-Antwort.
- Das Token-Cookie ist nicht `httpOnly` und hat keine Ablaufzeit. Zusätzlich steht das Token beim Launch kurz in der URL (Query-Parameter).
- `defaultQueryFn` prüft `res.ok` nicht. Fehlerantworten des Backends landen deshalb als „Daten“ im Cache (das fängt `useCourse` z. B. über die Prüfung `"id" in course` ab).
- [project-sets/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/project-sets/page.tsx) nutzt `NEXT_PUBLIC_BACKEND_URL` (nicht in `sample.env`) und schickt keinen Auth-Header. Die Seite ist nicht in der Navigation verlinkt und funktioniert derzeit nicht.
- Die Typen liegen in `_temp_types/` und werden von Hand gepflegt. Sie müssen bei Backend-Änderungen manuell nachgezogen werden.
