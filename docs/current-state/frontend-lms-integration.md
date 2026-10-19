# Frontend: LMS integration

> **Purpose:** This page describes the current state of `teamable-frontend` with a focus on the LMS integration (currently Canvas only). It shows **which LMS functions the system needs today**. It is not complete code documentation.
>
> **Important upfront:** The frontend **never** talks to Canvas directly. All LMS access goes through the backend (`teamable-backend`). The frontend only triggers actions (e.g. "import students") and displays data that the backend has fetched from the LMS before. What the backend actually calls in Canvas is described in the backend documentation. Here, the backend endpoints are only listed as the interface.

---

## 1. Technical overview

| Topic | Status |
|---|---|
| Framework | Next.js 14 (App Router), React 18, TypeScript |
| Data fetching | TanStack React Query v5 with a global default `queryFn` |
| UI | Tailwind CSS, shadcn/ui (Radix), Formik + Yup for forms |
| Auth cookie | `cookies-next` (client) or `next/headers` (server) |
| Node | v20.16.0 (`.nvmrc`) |
| Configuration | `.env` with `BACKEND_BASE_URI` (see [sample.env](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/sample.env)) |
| Deployment | [Dockerfile](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/Dockerfile) + [docker-compose.yml](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/docker-compose.yml) (port 8003 → 3000) |

### Folder structure (relevant parts)

```
src/
├── app/
│   ├── (accounts)/
│   │   ├── accounts/login/        → login with email/password
│   │   ├── accounts/signup/       → registration with invitation token
│   │   ├── accounts/401/          → error page "Authentication error"
│   │   └── authenticate/route.ts  → entry point after Canvas login (sets token cookie)
│   ├── (app)/course/[courseId]/
│   │   ├── (hooks)/useCourse.tsx  → loads the course incl. LMS type
│   │   ├── setup/                 → onboarding wizard (main flow)
│   │   ├── students/              → student list + import buttons
│   │   ├── team-sets/             → team sets, generate teams, export to LMS
│   │   ├── profiles/              → static placeholder UI (no backend)
│   │   └── project-sets/          → not connected (see section 10)
│   ├── (marketing)/privacy/       → privacy page (names Canvas as data source)
│   ├── (providers)/
│   │   ├── query-client-provider.tsx → central fetch logic to the backend
│   │   └── auth-user-provider.tsx    → login state + redirects
│   └── page.tsx                   → start page: redirects to the first course
├── hooks/                         → one hook per backend action (LMS hooks here)
├── _temp_types/                   → TypeScript types of the backend responses
└── routes.ts                      → auth routes
utils/
├── auth.ts / auth-server.ts       → builds the Authorization header from the cookie
└── string.ts
```

---

## 2. Architecture: Who talks to the LMS?

```
┌────────────┐  LTI/OAuth start   ┌──────────────────┐   REST + OAuth2 token   ┌──────────┐
│  Canvas    │ ─────────────────▶ │ teamable-backend │ ◀─────────────────────▶ │ Canvas   │
│(course link│                    │  (Django, :8000) │                         │ REST API │
└────────────┘                    └──────────────────┘                         └──────────┘
                                     ▲          │
                   REST /api/v1/...  │          │ Redirect to
                   Header:           │          │ /authenticate?token=…&path=…
                   "Token <token>"   │          ▼
                                  ┌──────────────────┐
                                  │ teamable-frontend│
                                  │ (Next.js, :3000) │
                                  └──────────────────┘
```

- **Frontend → backend:** Only REST via `fetch` to `${BACKEND_BASE_URI}/api/v1/...`. Authentication uses a Django REST Framework token in the header `Authorization: Token <token>`.
- **Backend → Canvas:** The backend holds the teacher's Canvas OAuth2 token (model `CanvasOAuth2Token`, see `teamable-backend/src/canvas_oauth/oauth.py`) and uses it to call the Canvas API.
- **Canvas → backend → frontend:** The teacher launches Teamable from Canvas. The backend handles the OAuth flow with Canvas, then creates a Teamable token and redirects the browser to the frontend route `/authenticate`.

**What this means for Moodle:** If Moodle is integrated, almost nothing has to change in the frontend. Essentially only some texts are needed (see section 10). The actual work is in the backend: launch, OAuth or token, and API calls.

---

## 3. Canvas integration in the frontend code

### 3.1 All places related to the LMS / Canvas

| File | Relation |
|---|---|
| [src/app/(accounts)/authenticate/route.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28accounts%29/authenticate/route.ts) | Target of the redirect after the Canvas login: takes `token` and `path` from the URL, sets the cookie `token`, redirects to `path` |
| [src/hooks/use-import-students-from-lms.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-import-students-from-lms.ts) | `POST courses/{id}/import_students_from_lms/` |
| [src/hooks/use-import-student-gradebook-data.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-import-student-gradebook-data.ts) | `POST courses/{id}/import_gradebook_attribute_from_lms/` |
| [src/hooks/use-create-opt-in-quiz.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-create-opt-in-quiz.ts) | `POST courses/{id}/create_opt_in_quiz_lms/` (creates a quiz in Canvas) |
| [src/hooks/use-export-team-set-lms.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-export-team-set-lms.ts) | `POST courses/{id}/export_team_to_lms/` with `{ team_set }` (writes groups to the LMS) |
| [src/hooks/use-grade-attributes.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-grade-attributes.ts) | `GET courses/{id}/grade-attributes/`: grade attributes that come from the LMS gradebook |
| [src/hooks/use-onboarding-progress.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-onboarding-progress.ts) | `GET courses/{id}/get_onboarding_progress/`, contains `opt_in_quiz_link` (link into the LMS) among others |
| [src/app/(app)/course/[courseId]/(hooks)/useCourse.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/%28hooks%29/useCourse.tsx) | Loads the course. `course.organization.lms_type` is provided as `lmsType` |
| [src/app/(app)/course/[courseId]/team-sets/[teamSetId]/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/team-sets/%5BteamSetId%5D/page.tsx) | Button "Export to {LMS}" (name comes from `lmsType`) |
| [src/app/(app)/course/[courseId]/team-sets/[teamSetId]/columns.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/team-sets/%5BteamSetId%5D/columns.tsx) | Clicking a team member opens `member.lms_link` (profile in the LMS) |
| [src/app/(app)/course/[courseId]/setup/(hooks)/useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx) | Onboarding steps, texts mention "Canvas" hard-coded |
| [src/app/(app)/course/[courseId]/setup/(components)/GenerateOptInQuiz.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28components%29/GenerateOptInQuiz.tsx) | Button "Generate opt-in quiz" or "View opt-in quiz" |
| [src/components/GradeSourceAttributeSelector.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/components/GradeSourceAttributeSelector.tsx) | Selection of the grade attribute (the variable in the code is called `canvasAssignment`) |
| [src/_temp_types/organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts) | `LMS_TYPES = { CANVAS: "CANVAS" }`, currently the only value |
| [src/app/(marketing)/privacy/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28marketing%29/privacy/page.tsx) | Privacy text: name, student ID and grades are read from Canvas |

All error messages of the LMS hooks are the same: *"Please relaunch Teamable from your Canvas dashboard …"*. On errors (e.g. an expired Canvas token in the backend), the teacher is expected to relaunch Teamable from Canvas. This runs through the OAuth flow again.

### 3.2 Login via Canvas (launch flow)

1. The teacher opens Teamable from the Canvas course.
2. The **backend** checks whether a valid Canvas OAuth2 token already exists for the user. If not, it starts the OAuth flow (`authorization_code`) and stores `access_token`, `refresh_token` and `expires` in `CanvasOAuth2Token`.
3. The backend creates or fetches the Teamable API token (`rest_framework.authtoken.Token`) and redirects to
   `${FRONTEND_BASE_URL}/authenticate?token=<teamable-token>&path=<redirect_path>`
4. **Frontend** ([authenticate/route.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28accounts%29/authenticate/route.ts)): sets the cookie `token=<teamable-token>` (path `/`) and redirects to `path`, e.g. `/course/12/setup`.
5. From now on, every frontend request sends `Authorization: Token <token>`.

→ **The frontend never sees the Canvas token itself.** It is only kept in the backend.

### 3.3 When is data fetched or written back?

All LMS actions are triggered **manually by a button**. There is no automatic synchronization and no polling.

| Action (button) | Direction | Backend endpoint | Where in the UI |
|---|---|---|---|
| Import students | LMS → Teamable | `POST courses/{id}/import_students_from_lms/` | Setup step 1, students page |
| Generate opt-in quiz | Teamable → LMS (create quiz) | `POST courses/{id}/create_opt_in_quiz_lms/` | Setup step 1 |
| View opt-in quiz | Link into the LMS | `opt_in_quiz_link` from `get_onboarding_progress` | Setup step 1 |
| Import gradebook data | LMS → Teamable | `POST courses/{id}/import_gradebook_attribute_from_lms/` | Setup step 3, students page |
| Export to Canvas | Teamable → LMS (create groups) | `POST courses/{id}/export_team_to_lms/` body `{ "team_set": <id> }` | Team set detail page |
| Click on a team member | Link into the LMS | `lms_link` of the student | Team set detail page |

After every import, the frontend calls `refetch()`, e.g. for the onboarding progress or the student list, and thereby shows the new data.

---

## 4. Data model (frontend types) with LMS relation

The types in [src/_temp_types/](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8/src/_temp_types) reflect the JSON responses of the backend. The values are **always filled by the backend**. The column "Source" describes where the data comes from in terms of content; how it is filled is described in the backend.

### `Course` – [course.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/course.ts)

| Field | LMS relation | Source |
|---|---|---|
| `id` | – | Teamable-internal ID (appears in the URL `/course/{id}`) |
| `name` | yes | Course name (from the LMS course) |
| `lms_course_id` | **yes** | ID of the course in the LMS |
| `organization` | yes | see `Organization` |
| `sections: CourseSection[]` | – | Tutorial groups (sections), used in the frontend as a filter in the student list. They do **not** come from the LMS, but are created in Teamable (Django admin or `makefakeclass`). |

### `Organization` – [organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts)

| Field | LMS relation | Source |
|---|---|---|
| `name` | – | Institution |
| `lms_type` | **yes** | `"CANVAS"` (only defined value). Only controls labels such as "Export to Canvas" |

### `Student` – [student.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/student.ts)

| Field | LMS relation | Source |
|---|---|---|
| `id` | – | Teamable-internal |
| `lms_id` | **yes** | User ID in the LMS |
| `sis_user_id` | **yes** | SIS ID (student ID number or similar) from the LMS |
| `lms_link` | **yes** | URL to the user profile in the LMS (can be `null`) |
| `name`, `first_name`, `last_name` | yes | from the LMS import |
| `sections` | – | Section membership, maintained in Teamable (not from the LMS) |
| `role` | yes | `"Student"` or `"Instructor"` ([courseMember.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/courseMember.ts)) |

### `OnboardingProgress` – [onboarding.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/onboarding.ts)

| Field | Meaning |
|---|---|
| `has_created_opt_in_quiz` | Opt-in quiz was created in the LMS |
| `opt_in_quiz_link` | **Link to the quiz in the LMS** |
| `has_students` | Students have been imported |
| `has_attribute` | An attribute exists (e.g. from the gradebook) |
| `has_attribute_responses` | Values for attributes exist (e.g. imported grades) |
| `has_team_set` | At least one team set has been generated |

### `GradeAttribute` – [use-grade-attributes.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-grade-attributes.ts)

| Field | Meaning |
|---|---|
| `id`, `name` | A grade attribute. The name usually corresponds to an LMS assignment or a gradebook column (called `canvasAssignment` in the code) |

### `TeamSet` / `Team` – [teamSet.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/teamSet.ts), [team.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/team.ts)

Teams and team sets are created in Teamable. They only relate to the LMS through the **export** (`export_team_to_lms`) and through their members (`Student`).

### `AuthUser` – [user.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/user.ts)

`id`, `username`, `first_name`, `last_name`, `email`, `is_staff`, `course_memberships: [{ course }]`. `course_memberships` determines which course the start page redirects to.

---

## 5. Flow from the teacher's perspective

The main flow is the onboarding wizard under `/course/{courseId}/setup` ([useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx)). Which step is currently active is derived from `get_onboarding_progress`.

| # | UI step | Active? | Action | Backend endpoint | LMS access (in the backend) |
|---|---|---|---|---|---|
| 0 | Launch Teamable from Canvas | – | Redirect | Backend OAuth → `GET /authenticate?token&path` (frontend) | Canvas OAuth2 |
| 0b | Load page | – | Load course + user | `GET courses/{id}/`, `GET accounts/users/me/` | – |
| 1 | **Import students** | yes | Button "Import students" | `POST courses/{id}/import_students_from_lms/` | **read**: participants (and responses in the opt-in quiz, if any) |
| 1a | (optional) Opt-in quiz | yes, as long as no students have been imported | "Generate opt-in quiz" | `POST courses/{id}/create_opt_in_quiz_lms/` | **write**: create quiz. Afterwards only students who agreed are imported |
| 1b | View quiz | yes | "View opt-in quiz" | – (opens `opt_in_quiz_link`) | – |
| 2 | Define student profile | **disabled** | – | – | – |
| 3 | **Elicit student information** | yes | "Import gradebook data" | `POST courses/{id}/import_gradebook_attribute_from_lms/` | **read**: gradebook, grades |
| 4 | (Optional) Project setup | **disabled** | – | – | – |
| 5 | **Generate teams** | yes | Dialog "Select a grade source": choose a grade attribute, then "Generate teams" | `GET courses/{id}/grade-attributes/` and then `POST courses/{id}/generate_teams/` with `{ "attribute": <id> }` | – (works on imported data) |
| 6 | View team set | – | Toast "View" or page Team sets | `GET courses/{id}/get_team_sets/`, `GET courses/{id}/team-sets/{teamSetId}/` | – |
| 7 | **Export to Canvas** | – | Button on the team set detail page | `POST courses/{id}/export_team_to_lms/` with `{ "team_set": <id> }` | **write**: create groups |

Other pages:

- **Students** (`/course/{id}/students`): paginated list `GET courses/{id}/students/?page=&per_page=&sort=&search=&sections=`. Contains the same buttons "Import students" and "Import gradebook data".
- **Team sets** (`/course/{id}/team-sets`): list of team sets and button "Generate teams" (same dialog as step 5).
- **Profiles** (`/course/{id}/profiles`): only static demo components without backend calls.

Start page `/` ([page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/page.tsx), server-side): `GET accounts/users/me/`.
- Not logged in → `/accounts/signup`
- Without `course_memberships` → `/accounts/401`
- Otherwise → `/course/{first course_membership}/setup`

---

## 6. Login, permissions and token storage

### 6.1 Ways to log in

| Way | For whom | Flow | Endpoints |
|---|---|---|---|
| **Canvas launch** (production) | Teachers | see 3.2. The backend redirects to `/authenticate?token=…&path=…` | Backend OAuth, then frontend route `/authenticate` |
| **Sign-up with invitation token** | Teachers (mainly local) | `/accounts/signup?token=<INSTRUCTOR_SIGNUP_TOKEN>`, email and password, then automatic login | `POST accounts/sign-up/` `{ email, password, token }` |
| **Login** | Teachers | `/accounts/login`, email and password | `POST accounts/log-in/` `{ email, password }` → `{ token }` |
| **Logout** | – | Delete the cookie, then `/accounts/login` (no backend call) | – |

**Students** do **not** log in to the current frontend. There is no student UI. They only interact with the system in the LMS: through the opt-in quiz and through the exported groups. In the data model they only appear as `Student` with `role = "Student"`.

### 6.2 Where are tokens stored?

| Token | Storage location | Who sets it |
|---|---|---|
| Teamable API token (DRF `Token`) | **Browser cookie `token`**, path `/`, not `httpOnly` | `/authenticate` (server-side via `cookies().set`) or `useLogin` (client-side via `setCookie`) |
| Canvas OAuth2 access/refresh token | **Only in the backend** (DB model `CanvasOAuth2Token`) | Backend OAuth flow |

The cookie is turned into the header `Authorization: Token <token>` via [utils/auth.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/utils/auth.ts) (client) or [utils/auth-server.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/utils/auth-server.ts) (server components).

### 6.3 Permissions and route protection

- [auth-user-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/auth-user-provider.tsx) loads `accounts/users/me/`.
  - Not logged in and not an auth page → redirect to `/accounts/login`
  - Logged in and on an auth page → redirect to `/`
- The frontend checks **no roles** (teacher vs. student) and no course access. Whether someone may see a course is decided by the backend alone. If `useCourse` does not get a course ID back, the frontend shows "Course not found".

---

## 7. Data for team formation

Today exactly this LMS data is used for team formation:

1. **Student list** from the LMS course (`import_students_from_lms`), optionally filtered via the opt-in quiz. These are the people who are distributed into teams. Students without a team appear as `unassigned_students`.
2. **Grades from the gradebook** (`import_gradebook_attribute_from_lms`). According to the UI text, they are processed into values that indicate each student's relative performance. Every imported grade source becomes an **attribute** (`GradeAttribute`).
3. For "Generate teams", the teacher chooses **exactly one** grade attribute. The frontend only sends `{ attribute: <id> }` to `generate_teams`. Algorithm and configuration are entirely in the backend ("A pre-configuration we have set up for your course").
4. **Sections** are only used in the frontend to filter the student list. They do not come from the LMS, and team formation in the backend does not use them.

What team formation does **not** use today (the steps are disabled): student profiles or surveys (step 2), project requirements (step 4) and friendship or conflict relationships (the type [relationship.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/relationship.ts) exists, but is not used).

**Minimum requirements for an LMS API from the frontend's point of view:**

- Read course information (name, course ID)
- Read participants with role, name, LMS ID, SIS ID and profile link
- Read the gradebook or assignments with grades per student
- Create a quiz or survey and read its results (for the opt-in)
- Create groups or group sets and assign members (export)
- Launch with user and course context as well as user authorization (OAuth2 or similar)

---

## 8. Local setup (also without Canvas)

### 8.1 Starting the frontend

```bash
cp sample.env .env          # BACKEND_BASE_URI=http://localhost:8000
nvm use                     # Node 20.16.0
yarn                        # install dependencies
yarn dev                    # http://localhost:3000
```

Alternatively with Docker: `docker compose up --build` → http://localhost:8003. The `.env` must exist.

### 8.2 Testing without Canvas

1. Start the backend locally (see README in `teamable-backend`) and create an **instructor sign-up token** there.
2. Open `http://localhost:3000/accounts/signup?token=<INSTRUCTOR_SIGNUP_TOKEN>`, enter an email and a strong password. This is followed by an automatic login and the redirect to the course onboarding.
3. This bypasses the Canvas launch completely: login via email/password, token in the cookie.

**Limitations without Canvas:**

- What the buttons that need the LMS (Import students, Import gradebook data, Generate opt-in quiz, Export to Canvas) do depends on the LMS type of the organization in the backend:
    - **Organization without `lms_type`:** The backend immediately responds with `200 OK` without contacting Canvas. The buttons report success, but no data is imported or exported.
    - **Organization with `lms_type = CANVAS`:** The calls fail because the user logged in via email has no Canvas OAuth token. This shows up as the toast "Please relaunch Teamable from your Canvas dashboard …".
- To test team formation and the team views, students, grade attributes and responses have to be created **in the backend**, e.g. via the Django admin or seed data. The frontend has no mock data and no mock mode.
- The Canvas redirect can be simulated manually if a valid Teamable token is available:
  `http://localhost:3000/authenticate?token=<DRF-Token>&path=/course/<id>/setup`

### 8.3 Other commands

| Command | Purpose |
|---|---|
| `yarn build` / `yarn start` | Production build or server |
| `yarn lint` / `yarn format` | ESLint |
| `yarn test` | Jest (example test only) |
| `yarn cypress:open` | Cypress (example spec only) |

CI ([.github/workflows/check.yml](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/.github/workflows/check.yml)) runs lint and build.

---

## 9. Complete list of backend calls

Base: `${BACKEND_BASE_URI}/api/v1/`. A trailing `/` is appended to every path automatically. GET calls go through the default `queryFn` (query key = path), POST calls through `defaultMutationFn` ([query-client-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/query-client-provider.tsx)).

| Method | Path | Body | Hook / file | LMS |
|---|---|---|---|---|
| GET | `accounts/users/me/` | – | `use-auth-user-query.ts`, `app/page.tsx` | – |
| POST | `accounts/log-in/` | `{ email, password }` | `use-login.ts` | – |
| POST | `accounts/sign-up/` | `{ email, password, token }` | `use-sign-up.ts` | – |
| GET | `courses/{id}/` | – | `useCourse.tsx` | returns `lms_type`, `lms_course_id` |
| GET | `courses/{id}/get_onboarding_progress/` | – | `use-onboarding-progress.ts` | returns `opt_in_quiz_link` |
| GET | `courses/{id}/students/?page&per_page&sort&search&sections` | – | `useStudents.tsx` | returns `lms_id`, `lms_link`, `sis_user_id` |
| POST | `courses/{id}/import_students_from_lms/` | – | `use-import-students-from-lms.ts` | **reads LMS** |
| POST | `courses/{id}/import_gradebook_attribute_from_lms/` | – | `use-import-student-gradebook-data.ts` | **reads LMS** |
| POST | `courses/{id}/create_opt_in_quiz_lms/` | – | `use-create-opt-in-quiz.ts` | **writes LMS** |
| GET | `courses/{id}/grade-attributes/` | – | `use-grade-attributes.ts` | indirect (gradebook) |
| POST | `courses/{id}/generate_teams/` | `{ attribute }` | `use-generate-teams.tsx` | – |
| GET | `courses/{id}/get_team_sets/` | – | `use-list-team-sets.ts` | – |
| GET | `courses/{id}/team-sets/{teamSetId}/` | – | `use-team-set.ts` | – |
| POST | `courses/{id}/export_team_to_lms/` | `{ team_set }` | `use-export-team-set-lms.ts` | **writes LMS** |
| GET | `teamset-templates` (via `NEXT_PUBLIC_BACKEND_URL`) | – | `project-sets/page.tsx` | – (not working, see below) |

---

## 10. Issues and notes for a Moodle integration

**What would have to be adapted in the frontend for Moodle (little):**

- Extend `LMS_TYPES` in [organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts) with e.g. `MOODLE`. The export label already uses `lmsType` dynamically (output "Export to Moodle").
- Replace hard-coded "Canvas" texts with `lmsType`. This affects:
  - the onboarding texts in [useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx)
  - the error toasts in all LMS hooks under [src/hooks/](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8/src/hooks)
  - the privacy text in [privacy/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28marketing%29/privacy/page.tsx)
- The endpoint names are already LMS-neutral (`..._from_lms`, `..._to_lms`). The frontend can stay unchanged if the backend internally talks to Canvas or Moodle depending on `lms_type`.
- The route `/authenticate` is LMS-neutral: a Moodle launch (e.g. LTI 1.3) in the backend would only have to redirect to the same URL at the end.

**Known weaknesses in the current code:**

- `authenticate/route.ts` returns a plain object instead of a `Response` when parameters are missing. A faulty call therefore leads to a server error instead of a clean 400 response.
- The token cookie is not `httpOnly` and has no expiry. In addition, the token briefly appears in the URL during the launch (query parameter).
- `defaultQueryFn` does not check `res.ok`. Error responses from the backend therefore end up in the cache as "data" (`useCourse`, for example, catches this with the check `"id" in course`).
- [project-sets/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/project-sets/page.tsx) uses `NEXT_PUBLIC_BACKEND_URL` (not in `sample.env`) and sends no auth header. The page is not linked in the navigation and currently does not work.
- The types are in `_temp_types/` and are maintained by hand. They have to be updated manually when the backend changes.
