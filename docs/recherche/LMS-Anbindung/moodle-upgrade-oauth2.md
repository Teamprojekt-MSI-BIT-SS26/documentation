# Teams aus Teamable als Gruppen in Moodle

!!! info "Stand"
    25. November 2026, HTWG-Moodle 4.5. Geprüft gegen die Moodle-Doku, den Moodle-Code (Branches `MOODLE_405_STABLE` und `MOODLE_503_STABLE`) und die IMS-Spezifikationen. Aussagen ohne Kennzeichnung sind mit dem verlinkten Beleg gestützt. Eigene Schlüsse sind mit *Rückschluss* markiert.

    Verwandte Seiten: [Moodle-API](moodle-api.md), [Moodle-Token: kritische Punkte](moodle-api-key-problem.md).

## Das Problem

**Wir wollen die generierten Teams aus Teamable als Gruppen in einen Moodle-Kurs schreiben.** Die Studierenden sind bereits in den Kurs eingeschrieben; Teamable soll sie nur in Gruppen einsortieren.

Was heute schon geht und was fehlt:

| Schritt | Weg | Status |
|---|---|---|
| Wer öffnet Teamable? (Person, Kurs, Rolle) | LTI-1.3-Launch | geht auf 4.5 |
| Teilnehmende lesen | LTI Advantage NRPS mit dem LTI-Schlüssel von Teamable | geht auf 4.5 |
| **Gruppen anlegen und Studierende zuordnen** | klassische Web Services | **offen** |

Das Schreiben der Gruppen ist der einzige offene Punkt. Warum es so schwierig ist:

- Die Funktionen zum Anlegen von Gruppen und Zuordnen von Mitgliedern sind klassische Web Services: `core_group_create_groups` („Creates new groups.“), `core_group_add_group_members` („Adds group members.“), `core_group_create_groupings`, `core_group_assign_grouping`. Sie verlangen `moodle/course:managegroups` und sind in 4.5 und 5.3 unverändert vorhanden. ([services.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/lib/db/services.php), [services.php 5.3](https://github.com/moodle/moodle/blob/MOODLE_503_STABLE/public/lib/db/services.php))
- Klassische Web Services brauchen ein **Web-Service-Token**. Web-Service-Tokens *„work with the existing web service functions“*, die neuen REST-API-Tokens *„work with the REST API only“*. ([REST API tokens 5.3](https://docs.moodle.org/503/en/REST_API_tokens))
- Die schreibenden Gruppenfunktionen sind **nicht** für Aufrufe aus der Browser-Sitzung freigegeben (kein `'ajax' => true`; nur lesende wie `core_group_get_course_groups` haben das). Die Moodle-Anmeldung der Lehrperson allein reicht also nicht. (gleiche Quelle)
- Sie gehören **nicht** zum Dienst der Moodle-App (`moodle_mobile_app`); ihre Einträge haben keinen `services`-Schlüssel. Ein App-Token kann keine Gruppen anlegen. (gleiche Quelle)
- **LTI kann Gruppen nur lesen.** Der IMS-Standard für Gruppen definiert nur Lesezugriffe: *„There are two actions defined for this service, both of which are HTTP GET requests.“* ([IMS LTI Course Groups 1.0](https://www.imsglobal.org/spec/lti-gs/v1p0))

Kurz: Für jeden automatischen Weg braucht Teamable entweder ein Web-Service-Token der Lehrperson oder Code in Moodle, und beides geht nur mit dem Rechenzentrum (RZ). Die einzige Ausnahme ist eine Browser-Erweiterung (Lösungsweg E).

---

## Lösungswege im Überblick

| | Lösungsweg | Automatisch | Was das RZ tun muss | Hauptproblem | Bewertung |
|---|---|---|---|---|---|
| A | Web Service + Token der Lehrperson | ja | Dienst anlegen, `createtoken` und `rest:use` auf Systemebene vergeben | Rechte auf Systemebene; dauerhaft gültiges Token in der Teamable-Datenbank | **empfohlen** |
| B | Token über `login/token.php` mit dem Moodle-Passwort | ja | wie A (Dienst, `createtoken`, `rest:use`) | Teamable verarbeitet das Moodle-Passwort | **zweitbeste Option** |
| B2 | Wie B, aber im Browser: Token nur im Frontend, nicht in der Datenbank | ja | wie A | Moodle-Passwort wird weiterhin auf einer Teamable-Seite eingegeben | **ersetzt B**, falls `token.php` genutzt wird |
| C | CSV aus Teamable + Plugin `local_groupimport` | halb (Lehrperson lädt Datei hoch) | Plugin installieren und pflegen | Fremd-Plugin; ein Schritt von Hand | **Alternative ohne Token** |
| D | Eigenes LTI-Plugin, das Gruppen schreibt | ja | eigenen Code installieren und pflegen | kein Standard, Pflege bei jedem Upgrade | keine Option: zu aufwändig |
| E | Browser-Erweiterung bedient die Moodle-Seiten | ja | nichts | fragil, Installation bei jeder Lehrperson | Notlösung |
| E2 | Bookmarklet bedient die Moodle-Seiten | ja (ein Klick im Kurs) | nichts | fragil wie E, evtl. durch CSP blockiert | Notlösung, für einen Pilot leichter als E |
| F | CSV-Import mit Moodle-Bordmitteln | nein | nichts | nur Gruppennamen, Zuordnung von Hand | Rückfallebene |
| G | Upgrade auf 5.3 + OAuth2 | – | Upgrade | keine Gruppen-Endpunkte in der neuen REST API | **gleiches Problem wie heute** |

---

## A. Web Service + Token der Lehrperson (empfohlen)

**Ablauf**

1. Das RZ legt einen Web Service „Teamable“ an. Darin legt es fest, welche **Funktionen** das Token aufrufen darf, z. B. `core_group_create_groups` (vollständige Liste: [Anfrage an das RZ](moodle-api.md#request-to-the-computing-centre)).
2. Öffnet eine berechtigte Lehrperson *Einstellungen → Sicherheitsschlüssel*, erzeugt Moodle automatisch ein Token für diesen Dienst. ([Security keys 4.5](https://docs.moodle.org/405/en/Security_keys), [webservice/lib.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/webservice/lib.php), Funktion `generate_user_ws_tokens`)
3. Die Lehrperson kopiert das Token einmal in Teamable. Teamable speichert es in seiner **Datenbank**.
4. Beim Export ruft das Teamable-Backend mit dem gespeicherten Token die Web Services auf, legt die Gruppen an und ordnet die Studierenden zu.

**Was das Token darf**

Das Token gibt keine eigenen Rechte. Moodle prüft bei jedem Aufruf die Rechte der Lehrperson im jeweiligen Kurs. Das Token darf also die Schnittmenge aus:

- den Funktionen im Dienst (legt das RZ fest) und
- den Rechten der Lehrperson im Kurs (z. B. `moodle/course:managegroups`).

Studierende können mit einem solchen Token keine Gruppen anlegen (lokal getestet, siehe [Students cannot create groups](moodle-api-key-problem.md#students-cannot-create-groups)).

**Voraussetzungen beim RZ**

Damit Moodle das Token erzeugt, muss das RZ Web Services aktivieren und den Dienst für alle Nutzenden freigeben (nicht auf bestimmte Personen beschränkt). Außerdem braucht die Lehrperson zwei Rechte: `moodle/webservice:createtoken`, das standardmäßig nur *Manager* haben ([access.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/lib/db/access.php)), und `webservice/rest:use`, das standardmäßig keiner Rolle zugeordnet ist ([rest/db/access.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/webservice/rest/db/access.php)). Beide prüft Moodle im Systemkontext. ([webservice/lib.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/webservice/lib.php))

*Rückschluss:* Die Kursrolle *Teacher* reicht nicht. Das RZ muss Lehrenden **zwei Rechte auf Systemebene** geben (`createtoken` und `rest:use`), z. B. über eine eigene Systemrolle oder für alle angemeldeten Nutzenden. Im zweiten Fall könnten auch Studierende Tokens erzeugen, aber keine Gruppen anlegen.

**Widerspruch, mit dem RZ zu klären:** Auf der HTWG-Seite Sicherheitsschlüssel erscheint bereits ein Token-Bereich ([createtoken is probably already active](moodle-api-key-problem.md#createtoken-is-probably-already-active-at-htwg)). Sind die Rechte schon vergeben, muss das RZ nur noch den Dienst anlegen.

**Probleme**

- **Rechte auf Systemebene.** Das RZ muss Lehrenden `createtoken` und `rest:use` systemweit geben (siehe „Voraussetzungen beim RZ“). `createtoken` ist von Moodle mit mehreren Risiken gekennzeichnet (`RISK_CONFIG`, `RISK_DATALOSS`, `RISK_SPAM`, `RISK_PERSONAL`, `RISK_XSS`, [access.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/lib/db/access.php)); hier wird das RZ am ehesten zögern.
- **Token von Hand kopieren.** Ein zusätzlicher Schritt für Lehrende; Teamable muss erklären, wo das Token zu finden ist.
- **Token läuft nicht ab.** Beim automatischen Erzeugen setzt Moodle kein Ablaufdatum (`validuntil` wird nicht gesetzt). ([webservice/lib.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/webservice/lib.php)) Das Token gilt, bis die Lehrperson es auf der Seite Sicherheitsschlüssel zurücksetzt.
- **Token gilt für alle Kurse der Lehrperson** und alle Funktionen des Dienstes, nicht nur für den Kurs, aus dem Teamable gestartet wurde.
- **Speicherung in der Teamable-Datenbank.** *Rückschluss:* Wegen der drei Punkte davor ist das Token ein dauerhaft gültiger Schlüssel zu allen Kursen der Lehrperson. Daraus folgt:
    - Das Token muss in der Datenbank **verschlüsselt** gespeichert werden; der Schlüssel liegt nicht in der Datenbank.
    - Teamable sollte das Token nur für den Kurs verwenden, aus dem es per LTI gestartet wurde (Kurs-ID aus dem Launch).
    - Lehrende brauchen in Teamable eine Möglichkeit, das Token zu löschen; zusätzlich können sie es in Moodle zurücksetzen.
    - Wird das Token in Moodle zurückgesetzt, lehnt Moodle Aufrufe mit `invalidtoken` ab. ([webservice/lib.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/webservice/lib.php)) Teamable muss das erkennen und die Lehrperson um ein neues Token bitten.
    - Speicherung und Zweck sind mit dem Datenschutz der HTWG abzustimmen.
- **Keine Transaktion.** Das Schreiben besteht aus mehreren Aufrufen nacheinander (Gruppierung anlegen, Gruppen anlegen, zuordnen, Mitglieder hinzufügen). Bricht es mittendrin ab, bleiben halb angelegte Gruppen zurück. (siehe [Implications for Teamable](moodle-api.md#implications-for-teamable))
- **Eindeutige Gruppennamen.** Gruppennamen müssen im ganzen Kurs eindeutig sein; ein zweiter Export mit denselben Teamnamen schlägt fehl. (gleiche Quelle)

## B. Token über `login/token.php` mit dem Moodle-Passwort

**Ablauf**

1. Wie bei Lösungsweg A legt das RZ den Dienst „Teamable“ an.
2. Die Lehrperson gibt in Teamable ihren Moodle-Benutzernamen und ihr Passwort ein.
3. Teamable schickt beides an `login/token.php` und bekommt dafür ein Token für den Dienst. Das Token speichert Teamable in seiner Datenbank wie bei Lösungsweg A.

Belege: `login/token.php` verlangt `username`, `password` und `service`, meldet die Person mit `authenticate_user_login` an und erzeugt das Token mit `generate_token_for_current_user`. ([login/token.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/login/token.php)) Für einen eigenen Dienst wird ein neues Token nur erzeugt, wenn die Person `moodle/webservice:createtoken` hat und kein Site-Admin ist; es gilt standardmäßig 12 Wochen (`validuntil = timecreated + tokenduration`). ([util.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/lib/external/classes/util.php)) An der HTWG funktioniert der Endpunkt mit den normalen Zugangsdaten (getestet mit dem Dienst `moodle_mobile_app`, siehe [Moodle-Token: kritische Punkte](moodle-api-key-problem.md#approach-4-logintokenphp-works-but-needs-the-moodle-password)).

**Vorteile**

- **Kein Kopieren:** Die Lehrperson muss das Token nicht in Moodle suchen.
- **Token läuft ab:** Anders als das Token von der Seite Sicherheitsschlüssel gilt es standardmäßig 12 Wochen.

**Probleme**

- **Teamable verarbeitet das Moodle-Passwort.** Die Lehrperson gibt ihr Passwort in einer fremden Anwendung ein. *Rückschluss:* Teamable muss das Passwort nicht speichern, sieht es aber bei jeder Anmeldung. Ist das Moodle-Passwort das zentrale HTWG-Passwort, betrifft das alle HTWG-Dienste. Außerdem gewöhnt es Lehrende daran, ihr Passwort auf fremden Seiten einzugeben. Datenschutz und RZ werden das voraussichtlich nicht akzeptieren.
- **Gleiche Voraussetzungen wie Lösungsweg A:** Dienst, `createtoken` und `rest:use` auf Systemebene.
- **Erneute Eingabe:** Nach Ablauf des Tokens muss die Lehrperson ihr Passwort erneut eingeben.

## B2. Token über `login/token.php` im Browser (nur im Frontend)

**Ablauf**

1. Wie bei Lösungsweg A legt das RZ den Dienst „Teamable“ an.
2. Die Lehrperson klickt in Teamable auf „Nach Moodle exportieren“ und gibt ihren Moodle-Benutzernamen und ihr Passwort ein.
3. Das JavaScript im Browser schickt beides **direkt** an `login/token.php` (mit `service=teamable`). Das Teamable-Backend sieht weder Passwort noch Token.
4. Moodle antwortet mit `token` und `privatetoken`. Das Frontend verwirft `privatetoken` sofort und hält `token` nur im Arbeitsspeicher (JS-Variable, kein `localStorage`, kein Cookie).
5. Das Frontend prüft mit `core_webservice_get_site_info`, ob die Moodle-Nutzer-ID zur Person aus dem LTI-Launch passt.
6. Das Frontend holt die Teams (mit Moodle-Nutzer-IDs) vom Teamable-Backend und ruft die Gruppenfunktionen direkt bei Moodle auf (`webservice/rest/server.php`).

Belege: Beide Endpunkte erlauben Aufrufe von fremden Domains. `login/token.php` sendet `Access-Control-Allow-Origin: *` ([login/token.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/login/token.php)), der REST-Server ebenfalls, mit dem Kommentar *„This allow to receive requests done by Web Workers or webapps in different domains.“* ([webservice/rest/locallib.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/webservice/rest/locallib.php), Funktion `send_headers`). Existiert für Person und Dienst schon ein gültiges Token, gibt `token.php` dasselbe Token zurück, statt ein neues zu erzeugen. ([util.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/lib/external/classes/util.php), Funktion `generate_token_for_current_user`) Im Quellcode geprüft, noch nicht aus dem Browser getestet.

**Vorteile gegenüber B**

- **Kein Passwort auf dem Teamable-Server.** Es geht vom Browser direkt zu Moodle und landet nicht in Logs, Fehlerberichten oder Proxys von Teamable.
- **Kein Token in der Teamable-Datenbank.** *Rückschluss:* Teamable ist kein lohnendes Angriffsziel mehr (keine Sammlung von Tokens aller Lehrenden). Verschlüsselung, Schlüsselverwaltung, „Token löschen“ und die Behandlung von `invalidtoken` für gespeicherte Tokens entfallen.
- **Weniger Abstimmung mit dem Datenschutz:** Das Backend verarbeitet weder Passwort noch Token.

**Probleme**

- **Das Passwort-Problem von B bleibt.** Für die Lehrperson sieht es genauso aus: ein Passwortfeld auf einer fremden Seite. *Rückschluss:* Das JavaScript stammt von Teamable. Bei einer XSS-Lücke, einem manipulierten npm-Paket oder Deployment könnte es das Passwort mitlesen. Auch der Gewöhnungseffekt bleibt. RZ und Datenschutz werden voraussichtlich ähnlich skeptisch sein wie bei B.
- **„Nur im Browser“ heißt nicht „weg“.** Moodle speichert das Token in seiner Datenbank, und es bleibt bis zum Ablauf (standardmäßig 12 Wochen) gültig, auch wenn der Tab geschlossen wird. Wer es aus dem Browser stiehlt (z. B. per XSS), kann es genauso nutzen.
- **Export hängt am offenen Tab.** Die Aufrufe laufen nacheinander im Browser. Wird der Tab mittendrin geschlossen, bleiben halb angelegte Gruppen zurück (siehe „Keine Transaktion“ bei A). Kein Export im Hintergrund, keine Wiederholung durch den Server.
- **Zweiter Codepfad.** Der Canvas-Export läuft im Backend (Python), der Moodle-Export läge im Frontend (TypeScript).
- **Häufigere Eingabe:** Das Passwort wird bei jeder Export-Sitzung (nach jedem Neuladen) erneut abgefragt, nicht nur alle 12 Wochen.
- **Gleiche Voraussetzungen wie Lösungsweg A:** Dienst, `createtoken` und `rest:use` auf Systemebene.

*Rückschluss:* B2 ist in fast allen Punkten besser als B. Wenn `token.php` genutzt wird, dann in dieser Form. Gegenüber A bleibt die Abwägung: kein Passwort, dafür ein dauerhaft gültiges Token in der Teamable-Datenbank (A), oder kein Token bei Teamable, dafür ein Passwortfeld (B2).

## C. CSV aus Teamable + Plugin `local_groupimport`

**Ablauf**

1. Teamable bietet die Teams als CSV-Datei zum Herunterladen an (Kennung der Person und Gruppenname, Gruppierung optional).
2. Die Lehrperson lädt die Datei im Moodle-Kurs hoch. Das Plugin legt fehlende Gruppen an und ordnet die bereits eingeschriebenen Studierenden zu.

Belege: Das Plugin *„allows teachers to import groups and enrol students into them using a CSV file, directly from within a course.“* Kennung: Benutzername, E-Mail, ID-Nummer oder freigegebene Profilfelder. Berechtigung: *„The course tools require `moodle/course:managegroups`.“* Versionen: *„Certified installation and upgrade support for Moodle 4.5 through 5.3“*. Es legt keine Personen an und schreibt niemanden in Kurse ein: *„EasyStud never creates users or enrols users into a course.“* ([GitHub](https://github.com/Kestryk/moodle-local_groupimport))

Warum ein Plugin nötig ist: Der CSV-Import von Moodle legt nur Gruppen an, ohne Mitglieder (siehe Lösungsweg F).

**Vorteile**

- **Kein Token, keine Rechte für Teamable.** Teamable speichert nichts, womit es auf Moodle zugreifen könnte. Die Lehrperson lädt die Datei mit ihren eigenen Rechten hoch. *Rückschluss:* Für das RZ und den Datenschutz deutlich weniger Risiko als Lösungsweg A.
- **Wiederholter Export.** Das Plugin kennt zwei Strategien: *„Keep current placements adds only missing memberships and assignments.“* und „Synchronise listed groups“. ([GitHub](https://github.com/Kestryk/moodle-local_groupimport)) *Rückschluss:* Ein zweiter Export mit denselben Teamnamen scheitert nicht, anders als bei Lösungsweg A.

**Probleme**

- **Kennung der Studierenden.** Teamable bekommt die Teilnehmenden über NRPS. Das liefert die Moodle-Nutzer-ID, `idnumber`, Rollen und Status, Name und E-Mail aber *„only if the tool's privacy settings allow it“*. (siehe [LTI services](moodle-api.md#lti-services-nrps-ags)) Die Moodle-Nutzer-ID steht nicht in der Liste der Kennungen, die das Plugin laut README erkennt.
    - *Rückschluss:* Damit die CSV funktioniert, muss das RZ beim Einrichten von Teamable als LTI-Tool die Weitergabe der E-Mail erlauben, oder die `idnumber` muss an der HTWG gepflegt sein (bisher unbekannt, siehe [Open questions](moodle-api.md#open-questions)).
- **Das RZ muss ein Fremd-Plugin installieren** und bei jedem Upgrade mitziehen.
- **Ein Schritt bleibt Handarbeit:** Herunterladen und Hochladen.
- **Pflegezustand unklar:** Die Plugin-Seite auf moodle.org war nicht automatisch abrufbar; Pflege und Lizenz im Browser prüfen.

## D. Eigenes LTI-Plugin, das Gruppen schreibt

**Ablauf**

Moodle setzt LTI-Dienste als Subplugins um, z. B. *„An implementation of the LTI Memberships, Names and Role Provisioning and Course Groups services“*. ([celtic-project Plugin](https://github.com/celtic-project/moodle-ltiservice_memberships_celtic)) *Rückschluss:* Ein eigenes Subplugin könnte Teamable mit seinem LTI-Schlüssel Gruppen schreiben lassen, nur in Kursen, in denen Teamable eingebunden ist, ohne Token der Lehrenden.

**Probleme**

- **Kein Standard, eigener Scope:** Der IMS-Standard für Gruppen ist nur lesend (siehe oben). Das Plugin wäre eine Eigenentwicklung mit einem eigenen Scope (z. B. „Gruppen schreiben“), den nur Teamable und dieses Plugin kennen. Zum Vergleich: Der NRPS-Dienst definiert nur den lesenden Scope `contextmembership.readonly`. ([memberships.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/mod/lti/service/memberships/classes/local/service/memberships.php))
- **Wer handelt?** Bei Lösungsweg A handelt Teamable mit den Rechten der Lehrperson, und Moodle prüft bei jedem Aufruf `moodle/course:managegroups`. Beim LTI-Dienst handelt das Tool ohne Person. *Rückschluss:* Das Plugin muss selbst sicherstellen, dass Teamable nur in seinem Kurs Gruppen anlegt und nur Studierende dieses Kurses zuordnet. Die Basisklasse bietet dafür Bausteine (`check_tool()` prüft Access Token und Scopes, `is_allowed_in_context($typeid, $courseid)` prüft, ob das Tool im Kurs erlaubt ist), die Sicherheitsverantwortung für das Schreiben liegt aber beim Plugin und damit bei uns. ([service_base.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/mod/lti/classes/local/ltiservice/service_base.php))
- **Eigener Code in Moodle.** Das RZ muss das Plugin installieren, den Code prüfen, den Dienst in der Konfiguration des Teamable-Tools einschalten (`get_configuration_options()`, gleiche Quelle) und das Plugin bei jedem Upgrade mitziehen. Für ein RZ, das bei Updates zurückhaltend ist, die größte Hürde.
- **Entwicklungsaufwand** für das Team (PHP, Moodle-Plugin-Struktur).

## E. Browser-Erweiterung

**Ablauf**

1. Die Erweiterung holt die Teams mit der Teamable-Anmeldung der Lehrperson aus Teamable. *Rückschluss:* Sie braucht dafür Zugriff auf Teamable und auf `moodle.htwg-konstanz.de`.
2. Pro Team legt sie über die normale Moodle-Seite `group/group.php` eine Gruppe an. Die Seite prüft `moodle/course:managegroups` und legt die Gruppe über ein Moodle-Formular an (`group_form`, danach `groups_create_group`). ([group/group.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/group/group.php))
3. Danach ordnet sie über `group/members.php` alle Mitglieder des Teams auf einmal zu. Die Seite erwartet die Gruppen-ID (`group`), die ausgewählten Personen (`addselect`) und `add`, prüft den Sitzungsschlüssel (`confirm_sesskey()`) und fügt alle ausgewählten Personen hinzu (`get_selected_users()`, dann `groups_add_member`). ([group/members.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/group/members.php))

Web Services aus der Sitzung aufzurufen geht nicht, weil die Gruppenfunktionen dafür nicht freigegeben sind (siehe oben).

**Probleme** (*Rückschluss*)

- **Fragil:** Die Erweiterung hängt am HTML der Moodle-Seiten und kann bei jedem Moodle-Update brechen. Am empfindlichsten ist Schritt 2: Nach dem Anlegen muss die Erweiterung die ID der neuen Gruppe aus der Antwort von Moodle auslesen, um in Schritt 3 die Mitglieder zuzuordnen.
- **Zuordnung über die Moodle-Nutzer-ID:** `members.php` erwartet Moodle-Nutzer-IDs. Die liefert NRPS bereits (siehe [LTI services](moodle-api.md#lti-services-nrps-ags)); anders als bei Lösungsweg C ist keine E-Mail nötig.
- **Anzahl der Aufrufe:** Pro Team etwa zwei Seitenaufrufe (anlegen, Mitglieder zuordnen), bei 20 Teams also etwa 40.
- **Verteilung und Installation:** Die Erweiterung müsste über die Erweiterungs-Stores von Chrome und Firefox veröffentlicht oder von Hand installiert werden. Jede Lehrperson muss sie installieren; auf dienstlichen Geräten ist das evtl. gesperrt.
- **Akzeptanz:** Das RZ könnte es als Umgehung sehen; offen ansprechen.
- Vorteil: der einzige automatische Weg ganz ohne RZ, ohne Token-Speicherung und ohen Moodle Daten Eingabe. Die Erweiterung kann nur, was die Lehrperson auch von Hand darf.

## E2. Bookmarklet

Ein Bookmarklet ist ein Lesezeichen, dessen Adresse nicht `https://…`, sondern `javascript:…` ist. Beim Anklicken führt der Browser den Code **auf der gerade offenen Seite** aus. Auf der Moodle-Seite heißt das: mit der Sitzung der Lehrperson und dem Sitzungsschlüssel (`M.cfg.sesskey`), als hätte Moodle den Code selbst geladen. Das Bookmarklet macht also dasselbe wie die Erweiterung (Schritte 2 und 3 bei Lösungsweg E), nur ohne Installation.

Der gesamte Code steckt im Lesezeichen selbst; es wird nichts nachgeladen:

```
javascript:(async()=>{const s=M.cfg.sesskey; /* Gruppen anlegen, Mitglieder zuordnen … */})()
```

**Ablauf**

1. Teamable zeigt einen Link `<a href="javascript:(…)">Teams nach Moodle</a>`. Die Lehrperson zieht ihn einmal in ihre Lesezeichenleiste; der Browser speichert dabei den Code.
2. Die Lehrperson öffnet in Teamable den Export. Die Teams werden übergeben (siehe „Woher kommen die Teams?“).
3. Im Moodle-Kurs klickt sie auf das Lesezeichen. Der Code legt die Gruppen über `group/group.php` an und ordnet die Mitglieder über `group/members.php` zu.

**Eigenschaften**

- **Fester Code:** Es läuft nur der Code, den die Lehrperson beim Reinziehen gespeichert hat. Für ein Update muss sie das Lesezeichen neu ziehen.
- **Klein:** Der Code muss minifiziert in die Adresse passen.

**Woher kommen die Teams?**

Der Code läuft als Moodle-Seite und kann nicht einfach die Teamable-Anmeldung nutzen.

- **Zwischenablage:** Teamable kopiert die Teams (mit Moodle-Nutzer-IDs) als JSON; das Bookmarklet liest sie ein. Einfach, aber ein Handgriff mehr.
- **Einmal-ID in der Adresse:** Der Export-Button in Teamable öffnet den Moodle-Kurs mit `#teamable=<einmal-id>`. Das Bookmarklet liest die ID und holt die Teams damit bei Teamable ab. Die ID gilt kurz und nur einmal; Teamable muss dafür CORS für die Moodle-Domain erlauben. Das ist kein Moodle-Token, nur ein Abholschein für die Teamdaten.

**Unterschied zur Erweiterung**

| | Erweiterung | Bookmarklet |
|---|---|---|
| Einrichten | Über Chrome Web Store / Firefox Add-ons (mit Prüfung) | Link in die Lesezeichenleiste ziehen |
| Auf Dienstgeräten | Oft per Richtlinie gesperrt | Selten gesperrt |
| Start | Kann selbst einen Button auf Moodle-Seiten einblenden | Nur per Klick auf das Lesezeichen |
| Teams von Teamable holen | Einfach, darf Anfragen an beide Seiten ohne CORS-Grenzen schicken | Umständlicher (siehe oben) |
| Updates | Automatisch über den Store | Lesezeichen neu ziehen |
| Content Security Policy (CSP) der Seite | Gilt für die Erweiterung weitgehend nicht | Kann blockieren |
| Bauaufwand | Höher (Manifest V3, Veröffentlichung, Chrome und Firefox) | Gering (eine JS-Datei) |
| Zugriff | Dauerhaft auf die freigegebenen Seiten | Nur im Moment des Klicks |

**Gleich wie bei der Erweiterung:** kein RZ, kein Token, kein Passwort; kann nur, was die Lehrperson auch von Hand darf; fragil bei Moodle-Updates; Akzeptanz beim RZ offen.

*Rückschluss:* Für einen Prototyp oder Pilot mit wenigen Lehrenden reicht ein Bookmarklet (schnell gebaut, sofort nutzbar). Für den Dauerbetrieb ist die Erweiterung bequemer und unabhängig von der CSP.

**Offen:** Ob die HTWG-Moodle eine CSP setzt, die Bookmarklets blockiert. Lässt sich mit einem harmlosen Test-Bookmarklet an der HTWG prüfen. Lokal noch nicht getestet.

## F. CSV-Import mit Moodle-Bordmitteln

**Ablauf**

Lehrende dürfen Gruppen per CSV importieren: *„Teachers are only allowed to upload groups in courses they are authorized to edit.“* Spalten: `groupname` (Pflicht), optional `groupidnumber`, `description`, `enrolmentkey`, `groupingname`. ([Import groups 4.5](https://docs.moodle.org/405/en/Import_groups))

**Vorteile**

- Geht heute schon auf Moodle 4.5: kein RZ, kein Token, kein Plugin.

**Probleme**

- **Keine Spalte für Mitglieder.** Die Studierenden ordnet die Lehrperson pro Gruppe über „Add/remove users“ von Hand zu. ([Groups 4.5](https://docs.moodle.org/405/en/Groups))
- **Aufwand der Handarbeit.** Die Seite `group/members.php` übernimmt alle ausgewählten Personen auf einmal (`get_selected_users()`). ([group/members.php 4.5](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/group/members.php)) Die Lehrperson markiert also einmal pro Gruppe die Mitglieder in der Liste, nicht jede Person einzeln. Bei 20 Teams sind das 20 Durchgänge.
- Damit geht der Nutzen von Teamable beim Übertragen weitgehend verloren; nur als Rückfallebene.

## G. Upgrade auf Moodle 5.3 + OAuth2

!!! warning "Ergebnis"
    Auch nach einem Upgrade auf 5.3 stehen wir vor demselben Problem: Gruppen lassen sich weiterhin nur über die klassischen Web Services anlegen, und dafür braucht es ein Web-Service-Token (Lösungsweg A) oder einen der anderen Wege. OAuth2 hilft erst, wenn eine spätere Moodle-Version Gruppen-Endpunkte in der neuen REST API anbietet.

**Was 5.3 neu bringt**

Ab 5.3 kann das RZ Teamable als OAuth2-Client registrieren. Die Lehrperson meldet sich bei Moodle an und bestätigt den Zugriff, ähnlich wie heute bei Canvas. Ein OAuth-2-Client *„is an external app [...] that an administrator registers so it can connect to Moodle's new REST API“*. Access Tokens gelten 1 Stunde, Refresh Tokens 1 Monat. ([OAuth 2 clients 5.3](https://docs.moodle.org/503/en/OAuth_2_clients))

**Probleme**

- **OAuth2 gilt nur für die neue REST API**, nicht für die Web Services (siehe oben).
- **Die neue REST API kann noch wenig:** *„In Moodle 5.3, only a few parts of Moodle are available through the REST API. More will be added in later releases.“* ([REST API tokens 5.3](https://docs.moodle.org/503/en/REST_API_tokens)) *Rückschluss:* Wir haben keinen Gruppen-Endpunkt gefunden. Eine offizielle Liste gibt es nicht; Moodle verweist auf die SwaggerUI der jeweiligen Installation (*Site administration → Development → Moodle REST API UI*). Prüfbar auf einer 5.3-Testinstanz.
- **In 4.5** gibt es die neue REST API (`r.php`) schon, aber nur mit zwei Routen: Templates und Nutzereinstellungen. ([templates.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/lib/classes/route/api/templates.php), [preferences.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/user/classes/route/api/preferences.php))
- **Zeitpunkt offen:** Gruppen-Endpunkte kämen erst mit einer späteren Moodle-Version und bräuchten ein weiteres Upgrade.

### Hintergrund: Support-Daten

| Version | Bugfixes bis | Sicherheitsupdates bis | LTS |
|---|---|---|---|
| 4.5 (HTWG) | 6. Oktober 2025 (beendet) | 4. Oktober 2027 | ja |
| 5.3 | 4. Oktober 2027 | 1. Oktober 2029 | ja |

Quelle: [Moodle releases](https://moodledev.io/general/releases#version-support). Auf 5.3 kann direkt aktualisiert werden: *„You can only upgrade to Moodle 5.3 from Moodle 4.4 or later.“* ([Upgrading 5.3](https://docs.moodle.org/503/en/Upgrading#Check_the_requirements)) Serveranforderungen: [Release Notes 5.3](https://moodledev.io/general/releases/5.3#server-requirements).

*Rückschluss:* Nach dem 4. Oktober 2027 würden neu bekannte Sicherheitslücken in 4.5 nicht mehr geschlossen. Das ist ein Argument für das Upgrade, aber unabhängig von Teamable: Am Schreiben der Gruppen ändert das Upgrade nichts.

---

## Ausgeschlossene Wege

| Weg | Warum nicht | Beleg |
|---|---|---|
| Technischer Nutzer mit globalem Token | Zugriff auf alle Kurse; wird das RZ voraussichtlich nicht erlauben | [Moodle-Token: kritische Punkte](moodle-api-key-problem.md#approach-1-the-technical-user-has-permissions-across-all-of-moodle) |
| LTI Course Groups (IMS) | Nur lesend | [IMS LTI Course Groups 1.0](https://www.imsglobal.org/spec/lti-gs/v1p0) |
| Token über den Login der Moodle-App (`launch.php`) | Stellt Tokens für jeden aktivierten Dienst aus, leitet aber an `moodlemobile://` weiter, solange das RZ `forcedurlscheme` nicht leert (*„Leave the field empty if you want to allow any app.“*); das beträfe die App-Sicherheit der ganzen Instanz. *Rückschluss:* Eine Web-Anwendung kann ein eigenes URL-Schema nicht ohne Weiteres empfangen; Zweckentfremdung des App-Logins | [launch.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/admin/tool/mobile/launch.php), [util.php](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/lib/external/classes/util.php), [tool_mobile Sprachdatei](https://github.com/moodle/moodle/blob/MOODLE_405_STABLE/admin/tool/mobile/lang/en/tool_mobile.php) |

---

## Empfehlung und nächste Schritte

1. **Lösungsweg A (erste Wahl)** beim RZ anfragen: Dienst „Teamable“ anlegen. Vorher klären, ob Lehrende `moodle/webservice:createtoken` und `webservice/rest:use` auf Systemebene schon haben.
    - Für den Dienst **nur die Gruppenfunktionen** beantragen (`core_group_create_groupings`, `core_group_create_groups`, `core_group_assign_grouping`, `core_group_add_group_members`). *Rückschluss:* Ein gestohlenes Token könnte dann keine Noten oder Umfrageantworten lesen.
2. Das Token verschlüsselt in der Teamable-Datenbank speichern und nur für den per LTI gestarteten Kurs verwenden.
3. **Lösungsweg B (zweitbeste Option):** Token über `login/token.php` mit dem Moodle-Passwort, umgesetzt als **B2** (nur im Browser, ohne Speicherung in der Teamable-Datenbank).
4. **Lösungsweg C:** Plugin `local_groupimport` vorschlagen.
5. Ohne RZ bleibt **F** als Rückfallebene; **E** oder **E2** nur, wenn das RZ einverstanden ist.
6. **Lösungsweg D** (eigenes LTI-Plugin) verfolgen wir nicht: zu aufwändig.

- [ ] Beim RZ klären: Sind `createtoken` und `rest:use` für Lehrende auf Systemebene vergeben? Wird der Dienst „Teamable“ angelegt?
- [ ] Falls nicht: Kann `local_groupimport` installiert werden? Pflege und Lizenz auf moodle.org prüfen.
- [ ] Speicherung des Tokens mit dem Datenschutz der HTWG abstimmen.
- [ ] Optional: SwaggerUI einer 5.3-Testinstanz auf Gruppen-Endpunkte prüfen.
- [ ] Optional: Mit einem harmlosen Test-Bookmarklet prüfen, ob die HTWG-Moodle eine CSP setzt, die Bookmarklets blockiert.
