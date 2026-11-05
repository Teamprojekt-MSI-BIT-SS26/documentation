# Frontend: Code inventory details

> **Purpose:** This page adds detail to [Frontend: Code inventory](frontend-inventory.md): the full folder tree with the configuration files and conventions, the key files with line references, and the complete list of known gaps. Read the main page first.
>
> **Version:** `teamable-frontend` @ [`98ce1c8`](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8), `teamable-backend` @ [`183a36c`](https://github.com/Teamable-Analytics/teamable-backend/tree/183a36c) (as of 2026-10-29). Paths are relative to `teamable-frontend/` unless stated otherwise.

---

## 1. Project structure


Every folder in `src/app/` is a URL segment, and a `page.tsx` (or `route.ts`) makes it reachable. Folders in `( … )` add no segment.

```
teamable-frontend/
├── src/
│   ├── app/
│   │   ├── layout.tsx                         → root layout of all pages: font, QueryClientProvider, Toaster
│   │   ├── page.tsx                           → "/": redirects to signup, 401 or the user's first course
│   │   ├── not-found.tsx                      → 404 page
│   │   ├── globals.css                        → Tailwind base styles and theme colours (CSS variables)
│   │   ├── favicon.ico
│   │   ├── (providers)/                       → no pages, shared logic only
│   │   │   ├── query-client-provider.tsx      → all backend calls: defaultQueryFn (GET), defaultMutationFn (POST)
│   │   │   └── auth-user-provider.tsx         → AuthUserContext: who is logged in, redirect to the login
│   │   ├── (accounts)/
│   │   │   ├── accounts/
│   │   │   │   ├── layout.tsx                 → bare layout (no navbar, no footer)
│   │   │   │   ├── login/                     → /accounts/login: page.tsx + (components)/LogInForm.tsx
│   │   │   │   ├── signup/                    → /accounts/signup: page.tsx + (components)/SignupForm.tsx
│   │   │   │   └── 401/page.tsx               → /accounts/401: static error page
│   │   │   └── authenticate/route.ts          → /authenticate: sets the token cookie after the LMS launch
│   │   ├── (marketing)/
│   │   │   ├── layout.tsx                     → plain navbar + footer
│   │   │   └── privacy/page.tsx               → /privacy
│   │   └── (app)/
│   │       ├── layout.tsx                     → AuthUserContextProvider (client-side redirect to the login) + footer
│   │       └── course/[courseId]/             → /course/{courseId}/…
│   │           ├── layout.tsx                 → CourseProvider (loads the course) + navbar for all course views
│   │           ├── page.tsx                   → redirects to setup
│   │           ├── (hooks)/useCourse.tsx      → CourseContext: GET courses/{id}/
│   │           ├── components/
│   │           │   └── GenerateTeamsAttributeSelector.tsx → dialog "select a grade attribute"
│   │           ├── setup/                     → /…/setup: onboarding wizard
│   │           │   ├── page.tsx
│   │           │   ├── (hooks)/useSetupSteps.tsx → the 5 steps, their state and actions
│   │           │   └── (components)/          → SetupStepDetailCard.tsx (step list), GenerateOptInQuiz.tsx
│   │           ├── students/                  → /…/students: student list
│   │           │   ├── page.tsx               → page with the two import buttons
│   │           │   ├── (hooks)/useStudents.tsx → StudentsContext: search, sorting, pages, synced with the URL
│   │           │   └── (table)/               → student-table, student-table-toolbar, columns,
│   │           │                                student-table-sections-filter (not used, commented out)
│   │           ├── team-sets/                 → /…/team-sets: team set list + "Generate teams"
│   │           │   ├── page.tsx, columns.tsx
│   │           │   └── [teamSetId]/           → /…/team-sets/{id}: teams and members, export to the LMS
│   │           │       └── page.tsx, columns.tsx
│   │           ├── profiles/                  → /…/profiles: question editor (placeholder, no backend)
│   │           │   ├── page.tsx
│   │           │   └── (components)/          → CategoricalSingle.tsx, CategoricalMulti.tsx
│   │           └── project-sets/              → /…/project-sets: project set list (not working)
│   │               └── page.tsx, columns.tsx
│   ├── components/
│   │   ├── Navbar.tsx                         → navbar of the course views: menu, course name, logout
│   │   ├── NavbarPlain.tsx                    → navbar of the privacy page
│   │   ├── Footer.tsx                         → footer with the privacy link
│   │   ├── Logo.tsx, icons.tsx                → logo and icons
│   │   ├── SearchBar.tsx                      → search field of the DataTable
│   │   ├── GradeSourceAttributeSelector.tsx   → combobox of the grade attributes
│   │   ├── InputErrorMessage.tsx              → error text below form fields (login, signup)
│   │   ├── ui/                                → 24 shadcn/ui building blocks (button, dialog, table, data-table, …)
│   │   └── views/Page.tsx                     → PageView: frame of the course views and privacy (breadcrumbs, title, actions)
│   ├── hooks/                                 → 14 files: 12 backend hooks, logout, toasts (see section 2 of the main page)
│   ├── _temp_types/                           → 15 hand-written type files for the backend responses (see section 3.4 of the main page)
│   ├── lib/utils.ts                           → cn(): merges Tailwind classes
│   ├── routes.ts                              → paths of the auth pages
│   └── types.ts                               → Action, NonEmptyArray
├── utils/
│   ├── auth.ts, auth-server.ts                → auth header from the cookie (browser / server)
│   └── string.ts                              → title(): capitalises a word
├── public/                                    → logo.svg; login-image.webp (imported on the login page, not shown);
│                                                next.svg, vercel.svg (unused)
├── test/spec.test.ts                          → Jest example test
├── cypress/                                   → Cypress example spec, fixture and support files
├── .github/workflows/check.yml                → CI: yarn lint and tsc
└── configuration files                        → see the table below
```

| Configuration file | Purpose |
|---|---|
| `package.json`, `yarn.lock` | Dependencies and scripts (`dev`, `build`, `start`, `lint`, `format`, `test`, `cypress:open`); the lockfile is Yarn 1 (`yarn lockfile v1`) |
| `.nvmrc` | Node version 20.16.0 |
| `sample.env` | Template for `.env` with `BACKEND_BASE_URI` |
| `next.config.js` | Loads `.env`, passes `BACKEND_BASE_URI` to the app, imports SVGs as components (`@svgr/webpack`) |
| `tsconfig.json` | TypeScript in `strict` mode, aliases `@/*` and `@public/*` |
| `index.d.ts` | Allows importing `.svg` and `.webp` files in TypeScript |
| `tailwind.config.js` | Active Tailwind configuration: theme, colours, animations, plugin `tailwindcss-animate`. `components.json` also points to this file |
| `tailwind.config.ts` | Not used: the installed Tailwind 3.4.1 loads `tailwind.config.js` first |
| `postcss.config.js` | Runs Tailwind and Autoprefixer |
| `components.json` | Configuration for adding shadcn/ui components |
| `.eslintrc.json` | ESLint rules |
| `jest.config.ts` | Jest; compiles TypeScript with `@swc/jest` |
| `cypress.config.ts` | Cypress for end-to-end and component tests (Next.js with webpack) |
| `Dockerfile` | Image based on `node:20.16.0`; runs `yarn` and `yarn build` |
| `docker-compose.yml` | Starts the image with `yarn start` and `.env`; port 8003 → 3000 |
| `.gitignore` | Keeps `.env`, `node_modules/` and `.next/` out of git |
| `README.md` | Setup guide (without the install step `yarn`) |

**Conventions:**

- **Route groups:** `( … )` folders add no URL segment. They are used in three ways:
  - **Areas with their own layout:** `(app)` and `(marketing)`.
  - **Grouping without a layout:** `(accounts)` groups `accounts/`, which has its own layout, and `authenticate/`. `(providers)` holds the provider files.
  - **Helper files inside route folders:** `(hooks)`, `(components)` and `(table)`.
- **Hook files:**
  - `src/hooks/` uses kebab-case (`use-generate-teams.tsx`).
  - Hooks inside route folders use camelCase (`useCourse.tsx`, `useSetupSteps.tsx`, `useStudents.tsx`). `useCourse.tsx` and `useStudents.tsx` also contain their context provider (`CourseProvider`, `StudentsProvider`).
- **Hook pattern:**
  - Query hooks call `useQuery` with the API path as query key and return the query result.
  - Mutation hooks wrap `useMutation` and return a function `<action>Async` (e.g. `generateTeamsAsync`, `loginAsync`) together with the mutation state.
  - They show toasts on success and on error. Exception: the login shows a toast only on errors. The two import hooks also show a toast when the import starts.
- **Component files:**
  - Most own components outside `components/ui/` are PascalCase (`Navbar.tsx`, `SetupStepDetailCard.tsx`).
  - Kebab-case or lowercase: `components/ui/*`, `students/(table)/*`, all `columns.tsx`, `(providers)/*` and `components/icons.tsx`.
  - Next.js file names are lowercase by convention: `page.tsx`, `layout.tsx`, `not-found.tsx`, `route.ts`.
  - The column definitions of a table sit in a `columns.tsx` next to its page.
- **Server and client components:** 6 of the 19 route files are client components (`"use client"`):
  - the login page, the course layout, setup, students, team sets and the team set detail page
  - the other 13 route files run on the server, among them the start page, Profiles (which renders client components) and Project Sets
- **Code style:** enforced by ESLint ([.eslintrc.json](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/.eslintrc.json)) and checked in CI:
  - no semicolons, 2-space indentation, 1TBS braces
  - double quotes in JSX, trailing commas in multi-line lists, newline at the end of a file
  - the code also uses double quotes for imports (408 of 410 import statements in `src/` and `utils/`; the two exceptions are the `@radix-ui/react-icons` imports in the Profiles editor), although ESLint does not enforce it
- **Import aliases:**
  - `@/*` → `src/*` and `@public/*` → `public/*` ([tsconfig.json](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/tsconfig.json)). `@public/*` is used in two files (`Logo.tsx`, login page).
  - `utils/` sits outside `src/` and is imported with relative paths, e.g. `../../../../../../../utils/string` in the team set detail page.

---

## 2. Key files in detail


| File | Purpose |
|---|---|
| [(providers)/query-client-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/query-client-provider.tsx) | Central fetch logic for almost all backend calls (lines 18–65). `defaultQueryFn` (GET) takes the path from `queryKey[0]`. The mutation hooks call `defaultMutationFn` (POST). Both send the request to `${BACKEND_BASE_URI}/api/v1/<path>/`, appending the trailing slash, with `Authorization: Token …` when the cookie exists. `defaultMutationFn` throws an error for failed responses (usually the parsed response body). `defaultQueryFn` does not check the status, so an error response is stored as data |
| [(providers)/auth-user-provider.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28providers%29/auth-user-provider.tsx) | `AuthUserContext` and the hook `useAuthUser()`, which the navbar and the logout use. Loads `accounts/users/me/` through `use-auth-user-query.ts` and redirects to the login if nobody is logged in (lines 35–47). It only wraps the `(app)` routes, so its code for sending logged-in users away from the login and signup pages never runs |
| [hooks/use-auth-user-query.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-auth-user-query.ts) | Hook with its own fetch function `authUserQueryFn` for `accounts/users/me/` (not the default `queryFn`). It returns the user, or `null` on HTTP 401, and only runs when the cookie exists. The start page calls `authUserQueryFn` on the server |
| [course/[courseId]/(hooks)/useCourse.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/%28hooks%29/useCourse.tsx) | `CourseProvider` loads `GET courses/{id}/` and provides `courseId`, name, sections and `lmsType` through `useCourse()`. Shows an empty page while loading and "Course not found" if no course comes back (lines 35–69). `useCourse()` is used by the navbar, the setup, students and team set pages and 9 of the hooks |
| [hooks/use-generate-teams.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/hooks/use-generate-teams.tsx) | `POST courses/{id}/generate_teams/` with `{ attribute }`. On success a toast with a "View" button opens the new team set; on error a toast asks to relaunch Teamable from Canvas (lines 21–54) |
| [course/[courseId]/components/GenerateTeamsAttributeSelector.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/components/GenerateTeamsAttributeSelector.tsx) | Dialog around `GradeSourceAttributeSelector`. "Generate teams" stays disabled until an attribute is selected, then calls `onSubmit({ attribute })` and closes the dialog. Used in the onboarding and on the team sets page |
| [components/GradeSourceAttributeSelector.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/components/GradeSourceAttributeSelector.tsx) | Searchable combobox over `GET courses/{id}/grade-attributes/` (through `use-grade-attributes.ts`); shows "Loading attributes..." while loading |
| [setup/(hooks)/useSetupSteps.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx) | Wizard logic: the 5 steps, the actions of steps 1, 3 and 5, and the dialog of step 5 (lines 19–101). The current step is derived from `OnboardingProgress` (lines 103–111); the step texts are in lines 156–211 |
| [students/(hooks)/useStudents.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/students/%28hooks%29/useStudents.tsx) | `StudentsProvider` and `useStudents()` (lines 108–344): search, sorting (`first_name`, `last_name`, `id`), page and page size, synced with the URL query. Only the current page is loaded from the backend (server-side pagination) |
| [components/views/Page.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/components/views/Page.tsx) | `PageView`: frame of the course views and privacy with breadcrumbs, title and action buttons (on small screens in a menu) |
| [components/ui/data-table.tsx](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/components/ui/data-table.tsx) | Generic table: search on one column, sorting, optional pagination, click on a row (`rowAction`). Used by team sets, the team set detail and project sets. The student list uses its own table with server-side pagination (`students/(table)/student-table.tsx`) |
| [(accounts)/authenticate/route.ts](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28accounts%29/authenticate/route.ts) | Entry point after the LMS launch: reads `token` and `path` from the URL, sets the cookie `token` (path `/`) and redirects to `path` (lines 7–18) |
| [_temp_types/](https://github.com/Teamable-Analytics/teamable-frontend/tree/98ce1c8/src/_temp_types) | Types of the backend responses (see [section 3.4](frontend-inventory.md#34-data-shapes)) |

---

## 3. Known gaps (complete list)


What is missing or not working in the current code:

| Topic | Current state |
|---|---|
| **Views** | |
| Profiles | Question editor without backend. "Save Question" only shows an `alert()` in both forms; nothing is stored. Not linked |
| Project Sets | Not working (see [section 4.1](frontend-inventory.md#41-overview-of-all-views)). Not linked. In the backend, `TeamSetTemplateViewSet` exists but is not registered as a URL |
| Onboarding | Steps 2 (define student profile) and 4 (project setup) are disabled and show placeholder text ([useSetupSteps.tsx:164-196](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/setup/%28hooks%29/useSetupSteps.tsx#L164-L196)) |
| Student list | The section filter ([student-table-toolbar.tsx:34-40](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/students/%28table%29/student-table-toolbar.tsx#L34-L40)) and the section column ([columns.tsx:63-90](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/%28app%29/course/%5BcourseId%5D/students/%28table%29/columns.tsx#L63-L90)) are commented out. The backend still filters by the URL parameter `sections` |
| Results | Generated teams can only be viewed, not edited, renamed or deleted. `dnd-kit` is installed but only used in the Profiles editor |
| **Team generation** | |
| Input | The UI sends only one grade attribute. Profiles, projects and relationships are not used |
| Algorithm | The grade import already split the students at the median into "Above Average" and "Below Average". `generate_teams` puts the above group first and the below group last and pairs the students from both ends of this list. When 3 students are left, they form one team of three ([team_generation.py:68-100](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/services/team_generation.py#L68-L100)). The team sets are named "Study Buddy - {n}" (line 70), the teams "Team {n}" |
| Edge cases | Students without a response for the chosen attribute are not placed in a team and appear as unassigned in this team set. If no student has a response, an empty team set is created. If exactly one student has one, the backend fails (`pop()` on an empty list), and the UI shows the error toast |
| External API | A request to an external team generation API is commented out (lines 29–66). It uses the same endpoint (`api/generate/teams/`) and the same top-level payload keys (`algorithm_options`, `students`, `team_generation_options`) as the API in the `algorithms` repository. The commented code contains a hard-coded API key (line 32) |
| Generate teams dialog | After reopening, the dialog sends the previously selected attribute although the combobox shows "Select attribute..." (see [section 3.3](frontend-inventory.md#33-flow-generate-teams)) |
| **Courses and access** | |
| Multiple courses | No course selection. The LMS launch opens the course it was started from ([teamable-backend lti/views.py:149-151](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/lti/views.py#L149-L151)). The start page `/` always opens the user's first course ([page.tsx:13](https://github.com/Teamable-Analytics/teamable-frontend/blob/98ce1c8/src/app/page.tsx#L13)); this affects the email login, the signup and every click on "Home" or the logo |
| Students in several courses | The student import looks up existing students only by `lms_id`, not by course ([course_member.py:109-110](https://github.com/Teamable-Analytics/teamable-backend/blob/183a36c/src/app/models/course_member.py#L109-L110)). Importing a student who is already in another Teamable course therefore moves them to the new course |
| Student access | There is no student UI. Students who launch Teamable from Canvas see "Course not found", because the backend only lets instructors, staff and superusers open a course |
| Authentication | `/authenticate` without `token` or `path` returns a plain object instead of a `Response` and fails with a server error. There is no logout endpoint; the DRF token never expires. The course views are only protected in the browser (no `middleware.ts`) |
| **LMS** | |
| Organization without LMS type | If `organization.lms_type` is `null`, `title(lmsType)` throws an error and the team set detail page crashes (see section 4 of [Frontend: LMS integration](frontend-lms-integration.md)) |
| Hard-coded Canvas | "Canvas" is hard-coded in the wizard texts, in five error toasts and in the privacy text. The privacy text also still says "Study Buddy" |
| **Code base** | |
| Types | Written by hand, not checked at runtime. Four fields differ from the backend responses (see [section 3.4](frontend-inventory.md#34-data-shapes)). The backend has no OpenAPI schema (no `drf-spectacular` or `drf-yasg` in `requirements.txt`) |
| Error handling of queries | `defaultQueryFn` does not check the HTTP status, so error responses are stored as data (see [section 3.2](frontend-inventory.md#32-how-data-moves)). Apart from "Course not found", no view shows a message when loading data fails |
| Tests | No tests for the app code. The only tests are two placeholders: Jest checks `expect(1).toBe(1)`, and the Cypress spec only opens `example.cypress.io`. CI runs lint and `tsc` on every push, no tests and no build |
