# Moodle Token: Critical Points

Applies to Moodle 4.5 (HTWG). Tests were done in a local Moodle instance, not at HTWG.

## The three approaches

| # | Approach | Token's permissions | Token needed |
|---|---|---|---|
| 1 | Service token (technical user) | All courses | yes, once in the backend |
| 2 | Personal token (teacher) | Only the teacher's courses | yes, per teacher |
| 3 | CSV export | - | no |


## Approach 1: The technical user has permissions across all of Moodle

It needs `webservice/rest:use` and `moodle/course:managegroups`. In addition, it must be allowed to enter the course, either through enrolment or `moodle/course:view`. This is read access to course content. There is no pure group-only role.

Assessment: The admins will probably not allow this.

## Approach 2: The token cannot be retrieved automatically

- There is no Moodle function that returns a user's token, and `login/token.php` would need the Moodle password.
- The teacher copies it once under *Preferences → Security keys* and pastes it into Teamable.
- A "Teamable" service is needed in Moodle (with the group functions, e.g. `core_group_create_groups`).
- Anyone with the capability `moodle/webservice:createtoken` then automatically receives a token for this service as soon as they open the security keys page. This applies to services that are not restricted to specific users.

- Assessment: The admins might allow this.

## `createtoken` is probably already active at HTWG

The token box on the security keys page only appears if web services are enabled and the user has `createtoken`. Six services are listed there; the "Teamable" service could be the seventh.

## Students cannot create groups

Tested in the local instance with a student and a Teamable token:

| Case | Result |
|---|---|
| Student without `webservice/rest:use` | rejected (`missing capability: webservice/rest:use`) |
| Student **with** `rest:use` | creating a group rejected (`Manage groups`), listing groups also rejected |

The token grants no additional permissions. Every function checks the user's permissions, and students do not have `moodle/course:managegroups` in any course.

## Open at HTWG

- Whether the admins will let us set up the "Teamable" service with the group functions
