# Ergebnis Spike 1

## Ausgangslage: Canvas

- **Anmeldung:** LTI 1.3. Lehrende öffnen Teamable direkt aus dem Canvas-Kurs.
- **Zugriff auf Daten:** OAuth2. Die Lehrperson erlaubt Teamable einmalig den Zugriff in ihrem Namen.
- **Gelesen** (Canvas REST API): Teilnehmerliste, Opt-in-Quiz, Punktzahlen aus Aufgaben
- **Geschrieben** (Canvas REST API): Opt-in-Quiz, Teams als Gruppen

→ [Bisherige Canvas-Anbindung](bisherige-canvas-anbindung.md)

## Moodle an der HTWG

| | Kann | Kann nicht |
| --- | --- | --- |
| **LTI 1.3** | Login aus dem Kurs, Kurs und Rolle, Teilnehmerliste (NRPS) | Noten, Umfrageantworten lesen, Gruppen anlegen |
| **Moodle-API (Web Services)** | Teilnehmende, Noten, Umfrageantworten lesen, Gruppen anlegen. An der HTWG bereits aktiv. | Umfragen anlegen |

**Lücken und Lösung:**

- **Kein OAuth2:** Teamable bekommt den Token für die Moodle-API nicht automatisch wie bei Canvas.
    - → Token je Lehrperson (einmal kopieren) oder technischer Nutzer, beides über einen Service beim Rechenzentrum.
- **Umfragen nicht per API anlegbar:** Teamable kann das Opt-in nicht selbst erstellen.
    - → Die Lehrperson legt die Umfrage in Moodle an, Teamable liest die Antworten.

→ [Moodle-API](moodle-api.md)

## Optionen im Vergleich

Wie kommen Kurs- und Studierendendaten aus einem LMS in Teamable, ohne an ein bestimmtes LMS gebunden zu sein? Diese Wege gibt es grundsätzlich:

| Option | Vorteile | Nachteile |
| --- | --- | --- |
| **LTI** | Standard in allen LMS, Login ohne eigenes Konto, Teamable nutzt es schon bei Canvas | Liefert nur Login und Teilnehmerliste. Tool muss im LMS registriert werden. |
| **LMS-API** (z. B. Moodle Web Services, Canvas REST API) | Automatisch, alle Daten, auch Gruppen zurückschreiben | Pro LMS eigene Anbindung, Token nötig |
| **CSV-Export/-Import** | Einfach, funktioniert mit jedem LMS, keine Freigaben nötig | Manuell für Lehrende, Daten veralten, Gruppen nur von Hand zurück |
| **LMS-Plugin** | Alles direkt im LMS, keine Token-Probleme | Nur für ein LMS, hoher Aufwand, Rechenzentrum muss es installieren und pflegen |

→ [Optionen zur Datenübertragung](uebertragungsoptionen.md)

## Entscheidung

Wir kombinieren zwei Wege: **LTI** ist der Standard über alle LMS und wird schon für den Login genutzt. Was LTI nicht kann, läuft über die **Moodle-API**.

| Zweck | Weg |
| --- | --- |
| Login | LTI 1.3 |
| Studierenden-Import | LTI 1.3 (NRPS) |
| Punktzahlen, Umfrageergebnisse, Gruppen-Export | Moodle-API (Web Services) |

!!! question "Offen: Token für die Moodle-API"
    Der LTI-Login liefert keinen Token, und Moodle 4.5 hat kein OAuth2. Realistisch sind ein Token je Lehrperson (einmal kopieren) oder ein technischer Nutzer. Beides braucht einen Service beim Rechenzentrum.
