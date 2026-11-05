# Local setup: Moodle + Teamable via LTI

!!! info "What this is about"
    This guide rebuilds the proof of concept from the Moodle spike locally: Teamable is opened from a Moodle course via **LTI 1.3** and imports the students via **NRPS** (Names and Role Provisioning Service).

    **Code:** `teamable-backend`, branch `local-moodle-lti-nrps-import`. The branch is a spike state and will not be merged into `master`.

    **Tested with:** Moodle 4.5.4 (Docker image `brandkern/moodle:4.5-latest`), Windows 11, Python 3.10, Node 20. The HTWG Moodle runs on 4.5.14, which is the same LTS version.

## Overview

At the end, three services run at the same time:

| Service | Runs in | Address |
| --- | --- | --- |
| Moodle 4.5 + MariaDB | Docker | `http://localhost:8008` |
| Teamable backend | locally with Python | `http://localhost:8000` |
| Teamable frontend | locally with Node | `http://localhost:3000` |

!!! warning "Always use `localhost`"
    Always open Moodle, the backend and the frontend in the browser via `localhost`, never via `127.0.0.1` or `host.docker.internal`. Moodle only knows itself as `http://localhost:8008`. This address is part of the LTI message and must be exactly the same everywhere.

**Prerequisites:** Docker Desktop, Python 3.10 or newer, Node 20, Git

## 1. Start Moodle

The backend branch contains the file `docker-compose.moodle.yml`. It starts Moodle together with a MariaDB database:

```yaml
services:
  db:
    image: mariadb:10.11
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      MARIADB_ROOT_PASSWORD: root
      MARIADB_DATABASE: moodle
      MARIADB_USER: moodle
      MARIADB_PASSWORD: moodle
    volumes:
      - db-data:/var/lib/mysql

  moodle:
    image: brandkern/moodle:4.5-latest
    depends_on:
      - db
    ports:
      - "8008:80"
    environment:
      DB_HOST: db
      DB_NAME: moodle
      DB_USER: moodle
      DB_PASS: moodle
      WWWROOT: http://localhost:8008
    volumes:
      - moodledata:/var/www/moodledata

volumes:
  db-data:
  moodledata:
```

Start Docker Desktop, then run in the `teamable-backend` folder:

```bash
docker compose -f docker-compose.moodle.yml up -d
```

Then open `http://localhost:8008`. The **installation wizard** appears: accept the license, go through the system check, create an admin account.

!!! note
    The `brandkern/moodle` image only contains Moodle, no database. If it is started without the compose file, "Database connection failed" appears. On the very first start, MariaDB needs a few seconds. If the message appears anyway, wait briefly and reload.

## 2. Set up the backend

```powershell
git clone https://github.com/Teamprojekt-MSI-BIT-SS26/teamable-backend.git
cd teamable-backend
git checkout local-moodle-lti-nrps-import
python -m venv venv
.\venv\Scripts\python.exe -m pip install -r requirements.txt
copy sample.env .env
```

The `.env` from `sample.env` already fits. The important part is `DEBUG=TRUE`: the backend then uses SQLite instead of Postgres and skips the check of the LTI deployment ID.

Create the database and an admin account for the Django admin:

```powershell
cd src
..\venv\Scripts\python.exe manage.py migrate
..\venv\Scripts\python.exe manage.py createsuperuser
```

### Generate a key pair for Teamable

Moodle needs Teamable's public key to verify requests from Teamable. Teamable keeps the private key. Both are stored in `src/lti/config/`. The folder is listed in `.gitignore`.

=== "PowerShell"

    In the `src` folder:

    ```powershell
    mkdir lti\config
    ..\venv\Scripts\python.exe -c "from cryptography.hazmat.primitives.asymmetric import rsa; from cryptography.hazmat.primitives import serialization as s; k=rsa.generate_private_key(public_exponent=65537, key_size=2048); open('lti/config/private.key','wb').write(k.private_bytes(s.Encoding.PEM, s.PrivateFormat.TraditionalOpenSSL, s.NoEncryption())); open('lti/config/public.key','wb').write(k.public_key().public_bytes(s.Encoding.PEM, s.PublicFormat.SubjectPublicKeyInfo))"
    ```

=== "Git Bash"

    In the `src` folder:

    ```bash
    mkdir -p lti/config
    openssl genrsa -out lti/config/private.key 2048
    openssl rsa -in lti/config/private.key -pubout -out lti/config/public.key
    ```

### Start the backend

```powershell
..\venv\Scripts\python.exe manage.py runserver
```

Keep the terminal open. In debug mode, the backend prints all data that Moodle sends during the LTI launch and the import here.

`http://localhost:8000/` shows a 404 page. This is expected, because the backend has no start page. The Django admin is available at `http://localhost:8000/admin/`.

## 3. Register Teamable in Moodle

In Moodle as admin: *Site administration → Plugins → Activity modules → External tool → Manage tools → **configure a tool manually***

| Field | Value |
| --- | --- |
| Tool name | `Teamable` |
| Tool URL | `http://localhost:8000/lti/launch/` |
| LTI version | **LTI 1.3** |
| Public key type | **RSA key** |
| Public key | complete content of `src/lti/config/public.key`, including `-----BEGIN PUBLIC KEY-----` |
| Initiate login URL | `http://localhost:8000/lti/login/` |
| Redirection URI(s) | `http://localhost:8000/lti/launch/` |
| Custom parameters | two lines, see below |
| Tool configuration usage | Show in activity chooser and as a preconfigured tool |
| Default launch container | **New window** |
| Services → IMS LTI Names and Role Provisioning | **Use this service to retrieve members' information as per privacy settings** |
| Privacy → Share launcher's name with tool | **Always** |
| Privacy → Share launcher's email with tool | **Always** |

Custom parameters, both lines:

```text
canvas_course_id=$Context.id
canvas_user_id=$User.id
```

Some fields only appear after clicking **"Show more..."**.

!!! note "Why Canvas parameters?"
    The LTI launch in Teamable currently expects the Canvas-specific fields `canvas_course_id` and `canvas_user_id`. Moodle fills them via the placeholders with the Moodle course ID and the Moodle user ID. This is only a bridge for the spike. Moodle also sends the same values in the LTI standard fields `context.id` and `sub`.

!!! note "Why 'New window' and an RSA key?"
    Django forbids embedding in an iframe by default. With the pasted RSA key, Moodle does not need to reach Teamable over the network. Moodle blocks requests to `localhost` by default.

After saving, click the list icon **"View configuration details"** on the Teamable entry in the tool list. It shows the **Platform ID**, **Client ID** and **Deployment ID**. They are needed in the next step.

## 4. LTI configuration in Teamable

Create the file `src/lti/config/lti_config_settings.json` and enter the Client ID and Deployment ID from Moodle:

```json
{
  "http://localhost:8008": [
    {
      "default": true,
      "client_id": "<CLIENT-ID>",
      "auth_login_url": "http://localhost:8008/mod/lti/auth.php",
      "auth_token_url": "http://localhost:8008/mod/lti/token.php",
      "key_set_url": "http://localhost:8008/mod/lti/certs.php",
      "key_set": null,
      "private_key_file": "private.key",
      "public_key_file": "public.key",
      "deployment_ids": ["<DEPLOYMENT-ID>"]
    }
  ]
}
```

- The top-level key must match the Platform ID from Moodle **exactly**: `http://localhost:8008`, without a trailing `/`.
- In Moodle, the Deployment ID is the internal number of the tool. For the first registered tool, it is usually `1`.
- The file is read again on every launch. The backend does not need to be restarted.

## 5. Create an organization in Teamable

At `http://localhost:8000/admin/` → *Organizations* → *Add*:

| Field | Value |
| --- | --- |
| Name | `HTWG Test` |
| LMS type | **MOODLE** |
| LMS API URL | `http://localhost:8008` |

!!! warning
    Without an organization, the LTI launch fails, because every course is attached to the first organization. If the type is `CANVAS`, "Import students" runs the Canvas import and fails. The LMS API URL must match the Platform ID from Moodle, because Teamable uses it to find the LTI configuration for the import.

## 6. Test course and test students in Moodle

**Create a course:** In Moodle, create a course with the short name `Test`.

**Create students:** Save the following content as `test-students.csv`, **encoding UTF-8**:

```csv
username,password,firstname,lastname,email,idnumber,course1,role1
anna.mueller,Test1234!,Anna,Müller,anna.mueller@example.com,100001,Test,student
ben.schmidt,Test1234!,Ben,Schmidt,ben.schmidt@example.com,100002,Test,student
clara.weber,Test1234!,Clara,Weber,clara.weber@example.com,100003,Test,student
david.fischer,Test1234!,David,Fischer,david.fischer@example.com,100004,Test,student
emma.wagner,Test1234!,Emma,Wagner,emma.wagner@example.com,100005,Test,student
felix.becker,Test1234!,Felix,Becker,felix.becker@example.com,100006,Test,student
```

| Column | Meaning |
| --- | --- |
| `course1` | Short name of the course the user is enrolled in |
| `role1` | `student` enrolls the person as a student |
| `idnumber` | ID number in Moodle, at the university typically the student ID number (Matrikelnummer). Arrives at Teamable via NRPS as `lis_person_sourcedid`. |

Upload it at *Site administration → Users → Accounts → **Upload users***: choose the file, CSV delimiter **comma**, encoding **UTF-8**, confirm the preview.

**Add Teamable to the course:** In the course `Test`, turn on edit mode → *Add an activity or resource* → **Teamable** → save.

## 7. Start the frontend

```powershell
git clone https://github.com/Teamprojekt-MSI-BIT-SS26/teamable-frontend.git
cd teamable-frontend
copy sample.env .env
npm install -g yarn
yarn install
yarn dev
```

`sample.env` already contains `BACKEND_BASE_URI=http://localhost:8000`. The frontend then runs at `http://localhost:3000`. `yarn install` takes a few minutes the first time.

## 8. Test

1. In Moodle, open the **Teamable** activity in the course `Test`. It opens in a new window.
2. You land directly in the Teamable frontend in your course, without a separate login.
3. Click **"Import students"**.
4. Under **Students**, the six test students are listed with first name, last name and Student ID (student ID number).

To verify:

- In the `runserver` terminal, the LTI data appears during the launch, and the raw data from Moodle appears under "NRPS members:" during the import.
- At `http://localhost:8000/admin/`, *Course members* lists the students with the role *Student*.

## Troubleshooting

| Error | Cause and solution |
| --- | --- |
| Moodle: "Database connection failed" | Moodle was started without a database. Start it with `docker-compose.moodle.yml`. Wait briefly on the first start. |
| `http://localhost:8000/` shows 404 | Expected, the backend has no start page. |
| "LTI disabled" | `.env` is missing, or the lines `LTI_CONF_FOLDER` and `LTI_CONF_FILE_NAME` are missing. |
| "iss http://localhost:8008 [client_id=…] not found in settings" | Client ID or Platform ID in `lti_config_settings.json` is missing or does not match exactly. |
| "Missing canvas_course_id/canvas_user_id custom claims" | Custom parameters in Moodle are missing or incomplete. Check both lines. |
| Redirect to `canvas.docker` ("This site can't be reached") | The launch was not recognized as a Moodle launch. Check that the branch is checked out. |
| IntegrityError for `organization` | The organization in Teamable is missing (step 5). |
| "Unable to import students. Please relaunch Teamable from your Canvas dashboard …" | The organization is not set to `MOODLE`, or NRPS is not enabled on the Moodle tool. The message is hard-coded in the frontend and always mentions Canvas. |
| Students page shows "undefined" | Branch is not up to date. The fix is in `src/app/views/course.py` (see below). |

Reset Moodle completely (deletes all Moodle data):

```bash
docker compose -f docker-compose.moodle.yml down -v
```

## Code changes

Compared to `master`, the branch contains:

| File | Change |
| --- | --- |
| `docker-compose.moodle.yml` | New: local Moodle 4.5 with MariaDB |
| `src/app/models/organization.py` | New LMS type `MOODLE` |
| `src/app/models/course.py` | New field `lms_memberships_url`: NRPS address of the course roster, saved on every launch |
| `src/app/migrations/0009_moodle_lms_type_and_memberships_url.py` | Migration for both model changes |
| `src/app/moodle/import_students.py` | New: student import via NRPS, counterpart to `src/app/canvas/import_students.py` |
| `src/lti/views.py` | For a launch from Moodle, the Canvas OAuth redirect is skipped. Instead, the NRPS address is saved and the user is redirected directly to the frontend. |
| `src/app/views/course.py` | "Import students" chooses the Moodle or Canvas import based on the LMS type. Also fixed: `students/` returned course data instead of student data, which caused "undefined" in the frontend. This bug also exists on `master`. |

**Access to Moodle:** Teamable obtains the token for NRPS from Moodle itself on every import (`/mod/lti/token.php`). To do so, it signs the request with its private key (LTI Advantage, client credentials). Only the key pair and the LTI configuration are stored, no token. An OAuth2 flow with an "Authorize" click like in Canvas is not needed.

## Known limitations

- Only the **student import** is implemented. Team export, opt-in quiz and grade import do not work with Moodle.
- The **Canvas parameters** in Moodle are a bridge. A real implementation should use the LTI standard fields `sub` and `context.id`.
- Every course is attached to the **first organization** (`Organization.objects.first()`). Multiple LMS instances at the same time are not supported.
- All **error messages in the frontend** still refer to Canvas.

