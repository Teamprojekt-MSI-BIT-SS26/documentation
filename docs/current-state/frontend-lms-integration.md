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
│   │   ├── profiles/              → question editor with local state only (no backend)
│   │   └── project-sets/          → not connected (see section 10)
│   ├── (marketing)/privacy/       → privacy page (names Canvas as data source)
│   ├── (providers)/
│   │   ├── query-client-provider.tsx → central fetch logic to the backend
│   │   └── auth-user-provider.tsx    → login state + redirects
│   └── page.tsx                   → start page: redirects to signup, 401 or the user's first course
├── hooks/                         → hooks for the backend actions (LMS hooks here), logout, toasts
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

- **Frontend → backend:** Only REST via `fetch` from the browser to `${BACKEND_BASE_URI}/api/v1/...`. Authentication uses a Django REST Framework token in the header `Authorization: Token <token>`.
- **Backend → Canvas:** The backend holds the teacher's Canvas OAuth2 token (model `CanvasOAuth2Token` in [canvas_oauth/models.py](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/canvas_oauth/models.py), flow in [canvas_oauth/oauth.py](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/canvas_oauth/oauth.py)) and uses it to call the Canvas API through the library `canvasapi`. Shortly before the access token expires, the backend renews it with the refresh token ([oauth.py:39-41](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/canvas_oauth/oauth.py#L39-L41)).
- **Canvas → backend → frontend:** The teacher launches Teamable from Canvas via LTI 1.3. The backend handles the launch and the OAuth flow with Canvas, then creates a Teamable token and redirects the browser to the frontend route `/authenticate` (details in 3.2).

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
| [src/app/(app)/course/[courseId]/(hooks)/useCourse.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/%28hooks%29/useCourse.tsx) | Loads the course. `course.organization.lms_type` is provided as `lmsType` (line 63) |
| [src/app/(app)/course/[courseId]/team-sets/[teamSetId]/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/team-sets/%5BteamSetId%5D/page.tsx) | Button "Export to {LMS}": the name is `title(lmsType)`, e.g. `"CANVAS"` → "Canvas" ([utils/string.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/utils/string.ts)) |
| [src/app/(app)/course/[courseId]/team-sets/[teamSetId]/columns.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/team-sets/%5BteamSetId%5D/columns.tsx) | Clicking a team member opens `member.lms_link` (profile in the LMS) |
| [src/app/(app)/course/[courseId]/setup/(hooks)/useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx) | Onboarding steps, texts mention "Canvas" hard-coded |
| [src/app/(app)/course/[courseId]/setup/(components)/GenerateOptInQuiz.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28components%29/GenerateOptInQuiz.tsx) | Button "Generate opt-in quiz" or "View opt-in quiz" |
| [src/components/GradeSourceAttributeSelector.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/components/GradeSourceAttributeSelector.tsx) | Selection of the grade attribute (the variable in the code is called `canvasAssignment`) |
| [src/_temp_types/organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts) | `LMS_TYPES = { CANVAS: "CANVAS" }`, currently the only value. The constant is not used at runtime; it only defines the type `LmsType` |
| [src/_temp_types/student.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/student.ts), [course.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/course.ts) | LMS fields `lms_id`, `sis_user_id`, `lms_link` and `lms_course_id` (see section 4) |
| [src/app/(app)/course/[courseId]/students/(table)/columns.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/students/%28table%29/columns.tsx) | Column "Student ID" shows `sis_user_id` from the LMS |
| [src/app/(marketing)/privacy/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28marketing%29/privacy/page.tsx) | Privacy text: name, student ID and grades are read from Canvas |

The error toasts of the four LMS action hooks (import students, import gradebook data, create opt-in quiz, export) and of `use-generate-teams.tsx` all say *"Please relaunch Teamable from your Canvas dashboard …"*. Relaunching helps when the backend has no Canvas token for the user, e.g. after an email login. In that case the backend answers an LMS call with a redirect to the Canvas OAuth page ([canvas_oauth/middleware.py:18-20](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/canvas_oauth/middleware.py#L18-L20)). The browser's `fetch` cannot follow it, so the call fails. An expired access token does not need a relaunch, because the backend renews it itself.

### 3.2 Login via Canvas (launch flow)

1. The teacher opens Teamable from the Canvas course. Canvas starts an **LTI 1.3** launch (library `pylti1p3`): OIDC login at `/lti/login/`, then `POST /lti/launch/` in the backend.
2. The **backend** reads the launch ([lti/views.py:45-151](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/lti/views.py#L45-L151)):
   - The custom claims `canvas_course_id` and `canvas_user_id` must be configured on the Canvas developer key; otherwise the launch fails.
   - It creates or updates the user (username `<canvas_user_id>+<issuer>`, first and last name from the launch).
   - It creates the course on the first launch: `lms_course_id` = `canvas_course_id`, name = the LTI context `label`, organization = the **first** `Organization` in the database (marked as "todo" in the code).
   - It creates the course membership. The role is derived from the LTI roles: administrator, content developer, instructor and TA become `Instructor`, everyone else becomes `Student`.
3. The backend redirects to `/oauth/initiate?redirect_path=/course/<id>` ([lti/views.py:149-151](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/lti/views.py#L149-L151)). If there is no Canvas OAuth2 token for the user yet, the OAuth flow (`authorization_code`) runs, and `access_token`, `refresh_token` and `expires` are stored in `CanvasOAuth2Token`.
4. The backend creates or fetches the Teamable API token (`rest_framework.authtoken.Token`) and redirects to
   `${FRONTEND_BASE_URL}/authenticate?token=<teamable-token>&path=/course/<id>` ([oauth.py:109-125](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/canvas_oauth/oauth.py#L109-L125)).
5. **Frontend** ([authenticate/route.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28accounts%29/authenticate/route.ts)): sets the cookie `token=<teamable-token>` (path `/`) and redirects to `path`, i.e. `/course/<id>`. The course index then redirects to `/course/<id>/setup`.
6. From now on, every frontend request sends `Authorization: Token <token>`.

Students can launch too. They get the role `Student`, but the backend only gives instructors, staff and superusers access to a course. The frontend therefore shows "Course not found" for them (see 6.3).

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

After a successful import or after creating the opt-in quiz, the frontend calls `refetch()` of the onboarding progress or the student list and thereby shows the new data. If the call fails, there is no `refetch()`. The export does not reload anything.

---

## 4. Data model (frontend types) with LMS relation

The types in [src/_temp_types/](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8/src/_temp_types) are meant to reflect the JSON responses of the backend; four fields differ (see section 3.4 of [Frontend: Code inventory](frontend-inventory.md)). The values are **always filled by the backend**. The column "Source" describes where the data comes from in terms of content.

### `Course` – [course.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/course.ts)

| Field | LMS relation | Source |
|---|---|---|
| `id` | – | Teamable-internal ID (appears in the URL `/course/{id}`) |
| `name` | yes | The LTI context `label` (usually the course code) at the first launch; it is not updated on later launches. Can also be set in the Django admin |
| `lms_course_id` | **yes** | ID of the course in the LMS (custom claim `canvas_course_id` of the launch) |
| `organization` | yes | see `Organization`. The launch always assigns the first organization in the database |
| `sections: CourseSection[]` | – | Tutorial groups (sections). They do **not** come from the LMS, but are created in Teamable (Django admin or `makefakeclass`). The student list was meant to filter by them, but the filter UI is commented out; only the URL parameter `sections` still filters. The backend sends only `id` and `name`, not `description` |

### `Organization` – [organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts)

| Field | LMS relation | Source |
|---|---|---|
| `name` | – | Institution |
| `lms_type` | **yes** | `"CANVAS"` (only defined value) or `null`. In the frontend it only controls the export labels ("Export to Canvas"). If it is `null`, `title(lmsType)` throws an error and the team set detail page crashes. In the backend it decides whether LMS calls run at all and whether `lms_link` is built |

### `Student` – [student.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/student.ts)

| Field | LMS relation | Source |
|---|---|---|
| `id` | – | Teamable-internal |
| `lms_id` | **yes** | User ID in the LMS |
| `sis_user_id` | **yes** | SIS ID (student ID number or similar) from the LMS; shown as "Student ID" |
| `lms_link` | **yes** | URL to the user profile, built by the backend as `{lms_api_url}/courses/{lms_course_id}/users/{lms_id}`. `null` if the organization is not Canvas or a value is missing ([course_member.py:56-63](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/models/course_member.py#L56-L63)) |
| `first_name`, `last_name` | yes | from the LMS import: Canvas `sortable_name` split at the comma |
| `name` | yes | built by the backend as `"{first_name} {last_name}"` |
| `sections` | – | Section membership, maintained in Teamable (not from the LMS). Missing for team members in the team set detail |
| `role` | yes | `"Student"` or `"Instructor"` ([courseMember.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/courseMember.ts)). The import sets `Student`, the launch derives the role from the LTI roles |

### `OnboardingProgress` – [onboarding.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/onboarding.ts)

| Field | Meaning |
|---|---|
| `has_created_opt_in_quiz` | Opt-in quiz was created in the LMS |
| `opt_in_quiz_link` | **Link to the quiz in the LMS**, built by the backend as `{lms_api_url}/courses/{lms_course_id}/quizzes/{quiz_id}`. `null` as long as no quiz exists, or if the organization is not Canvas or has no LMS URL (typed as `string` in the frontend) |
| `has_students` | Students have been imported |
| `has_attribute` | An attribute exists (e.g. from the gradebook) |
| `has_attribute_responses` | Values for attributes exist (e.g. imported grades) |
| `has_team_set` | At least one team set has been generated |

### `GradeAttribute` – [use-grade-attributes.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-grade-attributes.ts)

| Field | Meaning |
|---|---|
| `id`, `name` | A grade attribute. Each one belongs to one Canvas assignment; the name is `"<assignment name> - <assignment id>"` ([import_attribute.py:22-23](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/canvas/import_attribute.py#L22-L23)). The variable in `GradeSourceAttributeSelector.tsx` is called `canvasAssignment` |

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
| 1 | **Import students** | yes | Button "Import students" | `POST courses/{id}/import_students_from_lms/` | **read**: student enrollments (`StudentEnrollment`) and, if an opt-in quiz exists, its submissions |
| 1a | (optional) Opt-in quiz | yes, as long as no students have been imported | "Generate opt-in quiz" | `POST courses/{id}/create_opt_in_quiz_lms/` | **write**: create a survey "Opt-in" with the question "Do you want to opt-in?" (Yes/No). Afterwards only students who answered "Yes" are imported |
| 1b | View quiz | as long as no students have been imported, after the quiz was created | "View opt-in quiz" | – (opens `opt_in_quiz_link` in a new tab) | – |
| 2 | Define student profile | **disabled** | – | – | – |
| 3 | **Elicit student information** | yes | "Import gradebook data" | `POST courses/{id}/import_gradebook_attribute_from_lms/` | **read**: assignments and their submissions (scores) |
| 4 | (Optional) Project setup | **disabled** | – | – | – |
| 5 | **Generate teams** | yes | Dialog "Select a grade source": choose a grade attribute, then "Generate teams" | `GET courses/{id}/grade-attributes/` and then `POST courses/{id}/generate_teams/` with `{ "attribute": <id> }` | – (works on imported data) |
| 6 | View team set | – | Toast "View" or page Team sets | `GET courses/{id}/get_team_sets/`, `GET courses/{id}/team-sets/{teamSetId}/` | – |
| 7 | **Export to Canvas** | – | Button on the team set detail page | `POST courses/{id}/export_team_to_lms/` with `{ "team_set": <id> }` | **write**: create a group set named after the team set (with a number appended if the name is taken), one group per team, and the memberships. Unassigned students and members without `lms_id` are not exported ([export_team.py:22-34](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/canvas/export_team.py#L22-L34)) |

Other pages:

- **Students** (`/course/{id}/students`): paginated list `GET courses/{id}/students/?page=&per_page=&sort=&search=&sections=`. The backend searches in first name, last name and `sis_user_id`. Contains the same buttons "Import students" and "Import gradebook data".
- **Team sets** (`/course/{id}/team-sets`): list of team sets and button "Generate teams" (same dialog as step 5).
- **Profiles** (`/course/{id}/profiles`): question editor whose values stay in local form state; no backend calls.

Start page `/` ([page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/page.tsx), server-side): `GET accounts/users/me/`.
- Not logged in → `/accounts/signup`
- Without `course_memberships` → `/accounts/401`
- Otherwise → `/course/{first course_membership}/setup`

---

## 6. Login, permissions and token storage

### 6.1 Ways to log in

| Way | For whom | Flow | Endpoints |
|---|---|---|---|
| **Canvas launch** (production) | Teachers | see 3.2. The backend redirects to `/authenticate?token=…&path=…` | LTI launch and OAuth in the backend, then frontend route `/authenticate` |
| **Sign-up with invitation token** | Teachers (mainly local) | `/accounts/signup?token=<INSTRUCTOR_SIGNUP_TOKEN>`, email and password, then automatic login | `POST accounts/sign-up/` `{ email, password, token }` |
| **Login** | Teachers | `/accounts/login`, email and password | `POST accounts/log-in/` `{ email, password }` → `{ token }` |
| **Logout** | – | Delete the cookie, toast, load `accounts/users/me/` again, then `/accounts/login` | No logout endpoint: the DRF token stays valid in the backend |

The invitation token is a JWT for an existing `CourseMember` that has no user yet. It is created in the Django admin with the action "Create JWT token for selected course members" ([admin.py:28-32](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/admin.py#L28-L32)). The sign-up links the new user to this course member, so the member's role decides whether the course is accessible. The backend README creates it with the role `Instructor`.

**Students** have no UI in the current frontend. They only interact with the system in the LMS: through the opt-in quiz and through the exported groups. In the data model they appear as `Student` with `role = "Student"`. If a student launches Teamable from Canvas, they get a frontend token, but every course view shows "Course not found" (see 3.2 and 6.3).

### 6.2 Where are tokens stored?

| Token | Storage location | Who sets it |
|---|---|---|
| Teamable API token (DRF `Token`) | **Browser cookie `token`**, path `/`, not `httpOnly`, no expiry date (session cookie). The token itself does not expire in the backend | `/authenticate` (server-side via `cookies().set`) or `useLogin` (client-side via `setCookie`) |
| Canvas OAuth2 access/refresh token | **Only in the backend** (DB model `CanvasOAuth2Token`) | Backend OAuth flow |

The cookie is turned into the header `Authorization: Token <token>` via [utils/auth.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/utils/auth.ts) (client) or [utils/auth-server.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/utils/auth-server.ts) (server components).

### 6.3 Permissions and route protection

- [auth-user-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/auth-user-provider.tsx) loads `accounts/users/me/`.
  - Not logged in and not an auth page → redirect to `/accounts/login`
  - Logged in and on an auth page → redirect to `/`. This branch never runs in practice: the provider only wraps the `(app)` routes ([(app)/layout.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/layout.tsx)), and the auth pages are outside of them. A logged-in user can therefore still open `/accounts/login`.
- The frontend checks **no roles** (teacher vs. student) and no course access. Whether someone may see a course is decided by the backend alone:
  - All course endpoints use the permission `IsCourseInstructor` ([permissions.py](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/permissions.py)).
  - Access is granted to instructors of the course, staff users and superusers ([models/course.py:37-44](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/models/course.py#L37-L44)).
  - If `useCourse` does not get a course with an `id` back (e.g. on HTTP 403), the frontend shows "Course not found".
- There is no `middleware.ts`; the redirect to the login only happens in the browser after the page has loaded.

---

## 7. Data for team formation

Today exactly this LMS data is used for team formation:

1. **Student list** from the LMS course (`import_students_from_lms`), optionally filtered via the opt-in quiz. These are the people who are distributed into teams. Students of the course without a team appear as `unassigned_students`.
2. **Grades from the gradebook** (`import_gradebook_attribute_from_lms`, [import_attribute.py:55-103](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/canvas/import_attribute.py#L55-L103)):
   - Every Canvas assignment with the grading type `points` or `percent` becomes one **attribute** (`GradeAttribute`).
   - Each attribute has the two options "Above Average" and "Below Average".
   - The students are sorted by score and split at the median. A missing score counts as 0.
   - This is what the UI text means by values "indicating the relative performance of each student".
3. For "Generate teams", the teacher chooses **exactly one** grade attribute. The frontend only sends `{ attribute: <id> }` to `generate_teams`. Algorithm and configuration are entirely in the backend ("A pre-configuration we have set up for your course").
4. **Sections** do not come from the LMS, and team formation in the backend does not use them. In the frontend they could only filter the student list, and that filter UI is commented out.

What team formation does **not** use today (the steps are disabled): student profiles or surveys (step 2), project requirements (step 4) and friendship or conflict relationships (the type [relationship.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/relationship.ts) exists, but is not used).

**Minimum requirements for an LMS API from the frontend's point of view:**

- Launch with user and course context (today LTI 1.3 with the custom claims `canvas_course_id` and `canvas_user_id`, roles and course label) and user authorization for API calls (today Canvas OAuth2)
- Read the student enrollments with name, LMS ID and SIS ID. The profile link is not read; the backend builds it from the LMS URL
- Read assignments with a score per student
- Create a quiz or survey and read its submissions (for the opt-in)
- Create a group set with groups and assign members (export)
- The URL patterns for profile and quiz links (`/courses/{id}/users/{id}`, `/courses/{id}/quizzes/{id}`) are Canvas-specific

---

## 8. Local setup (also without Canvas)

### 8.1 Starting the frontend

```bash
cp sample.env .env          # BACKEND_BASE_URI=http://localhost:8000
nvm use                     # Node 20.16.0
yarn                        # install dependencies
yarn dev                    # http://localhost:3000
```

Alternatively with Docker: `docker compose up --build` → http://localhost:8003. The `.env` must exist before the build: `next.config.js` writes `BACKEND_BASE_URI` into the client bundle at build time, so changing it later needs a new build.

### 8.2 Testing without Canvas

1. Start the backend locally (see README in `teamable-backend`) and create an **instructor sign-up token** there: a `CourseMember` with the role `Instructor` and no user, then the admin action "Create JWT token for selected course members".
2. Open `http://localhost:3000/accounts/signup?token=<INSTRUCTOR_SIGNUP_TOKEN>`, enter an email and a password. The password needs at least 8 characters in the form and must pass Django's password validators in the backend. This is followed by an automatic login and the redirect to the course onboarding.
3. This bypasses the Canvas launch completely: login via email/password, token in the cookie.

**Limitations without Canvas:**

- What the buttons that need the LMS (Import students, Import gradebook data, Generate opt-in quiz, Export to Canvas) do depends on the LMS type of the organization in the backend:
    - **Organization without `lms_type`:** The backend immediately responds with `200 OK` without contacting Canvas. The buttons report success, but no data is imported or exported. In addition, the team set detail page crashes, because `title(lmsType)` is called with `null`; there is also no `lms_link` and no quiz link.
    - **Organization with `lms_type = CANVAS`:** The calls fail because the user logged in via email has no Canvas OAuth token. The backend answers with a redirect to Canvas OAuth, which `fetch` cannot follow. This shows up as the toast "Please relaunch Teamable from your Canvas dashboard …".
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

CI ([.github/workflows/check.yml](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/.github/workflows/check.yml)) runs lint and the TypeScript check (`tsc`). Tests and the build are not part of CI.

---

## 9. Complete list of backend calls

Base: `${BACKEND_BASE_URI}/api/v1/`. A trailing `/` is appended to every path automatically. GET calls go through the default `queryFn` (query key = path), POST calls through `defaultMutationFn` ([query-client-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/query-client-provider.tsx)). There are two exceptions:

- `accounts/users/me/` uses its own `queryFn` in `use-auth-user-query.ts`. Its query key is the path plus the auth header, and `app/page.tsx` also calls it on the server.
- `project-sets/page.tsx` calls `fetch` directly.

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

- Extend `LMS_TYPES` in [organization.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/_temp_types/organization.ts) with e.g. `MOODLE`. This only changes the type: the export label already uses the value from the backend (`title("MOODLE")` → "Export to Moodle").
- Replace hard-coded "Canvas" texts with `lmsType`. This affects:
  - the onboarding texts in [useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx)
  - the error toasts in the four LMS action hooks and in `use-generate-teams.tsx` under [src/hooks/](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8/src/hooks)
  - the privacy text in [privacy/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28marketing%29/privacy/page.tsx)
- Handle `lms_type = null`: today `title(lmsType)` crashes the team set detail page.
- The endpoint names are already LMS-neutral (`..._from_lms`, `..._to_lms`). The frontend can stay unchanged if the backend internally talks to Canvas or Moodle depending on `lms_type`.
- The route `/authenticate` is LMS-neutral: a Moodle launch (e.g. LTI 1.3) in the backend would only have to redirect to the same URL at the end.

**Known weaknesses in the current code:**

- `authenticate/route.ts` returns a plain object instead of a `Response` when parameters are missing. A faulty call therefore leads to a server error instead of a clean 400 response.
- The token cookie is not `httpOnly` and has no expiry. The DRF token never expires, and the logout does not invalidate it in the backend. In addition, the token briefly appears in the URL during the launch (query parameter).
- `defaultQueryFn` does not check `res.ok`. Error responses from the backend therefore end up in the cache as "data" (`useCourse`, for example, catches this with the check `"id" in course`).
- [project-sets/page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/project-sets/page.tsx) uses `NEXT_PUBLIC_BACKEND_URL` (not in `sample.env`) and sends no auth header. The endpoint `teamset-templates` is not reachable either: `TeamSetTemplateViewSet` exists in the backend, but is not registered in `app/urls.py`. The page is not linked in the navigation and currently does not work.
- The types are in `_temp_types/` and are maintained by hand. They have to be updated manually when the backend changes; four fields already differ today (see section 3.4 of [Frontend: Code inventory](frontend-inventory.md)).
