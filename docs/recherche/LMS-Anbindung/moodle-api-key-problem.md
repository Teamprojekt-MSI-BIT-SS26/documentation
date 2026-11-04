# Moodle-Token: Kritische Punkte

Gilt für Moodle 4.5 (HTWG). Tests wurden in einer lokalen Moodle-Instanz gemacht, nicht bei der HTWG.

## Die drei Wege

| # | Weg | Rechte des Tokens | Token nötig |
|---|---|---|---|
| 1 | Service-Token (technischer User) | Alle Kurse | ja, einmalig im Backend |
| 2 | Persönliches Token (Lehrperson) | Nur die Kurse der Lehrperson | ja, pro Lehrperson |
| 3 | CSV-Export | - | nein |


## Weg 1: Der technische User hat Rechte über ganz Moodle

Er braucht `webservice/rest:use` und `moodle/course:managegroups`. Zusätzlich muss er den Kurs betreten dürfen, durch Einschreibung oder `moodle/course:view`. Das ist ein Lesezugriff auf Kursinhalte. Eine reine Gruppenrolle gibt es nicht.

Einschätzung: Das werden die Admins wahrscheinlich nicht erlauben

## Weg 2: Das Token lässt sich nicht automatisch holen

- Es gibt keine Moodle-Funktion, die das Token eines Users herausgibt, und `login/token.php` bräuchte das Moodle-Passwort.
- Die Lehrperson kopiert es einmal unter *Einstellungen → Sicherheitsschlüssel* und fügt es in Teamable ein.
- Es braucht einen Dienst „Teamable“ in Moodle (mit den Gruppenfunktionen, z. B. `core_group_create_groups`).
- Jeder mit dem Recht `moodle/webservice:createtoken` bekommt dann automatisch ein Token für diesen Dienst, sobald er die Sicherheitsschlüssel-Seite öffnet. Das gilt für Dienste, die nicht auf bestimmte Nutzer beschränkt sind.

- Einschätzung: Evtl. erlauben das die Admins

## `createtoken` ist bei der HTWG vermutlich schon aktiv

Der Token-Kasten auf der Sicherheitsschlüssel-Seite erscheint nur, wenn Web-Services an sind und der User `createtoken` hat. Dort stehen sechs Dienste, der Dienst „Teamable“ könnte der siebte sein.

## Studierende können keine Gruppen anlegen

Getestet in der lokalen Instanz mit einem Studenten und einem Teamable-Token:

| Fall | Ergebnis |
|---|---|
| Student ohne `webservice/rest:use` | abgelehnt (`missing capability: webservice/rest:use`) |
| Student **mit** `rest:use` | Gruppe anlegen abgelehnt (`Manage groups`), Gruppen auflisten ebenfalls |

Das Token gibt keine zusätzlichen Rechte. Jede Funktion prüft die Rechte des Users, und Studierende haben `moodle/course:managegroups` in keinem Kurs.

## Offen bei der HTWG

- Ob die Admins uns den Dienst „Teamable“ mit den Gruppenfunktionen anlegen lassen
