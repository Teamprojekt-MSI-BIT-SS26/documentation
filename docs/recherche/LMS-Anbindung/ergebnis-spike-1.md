# Ergebnis Spike 1

## Ausgangslage: Canvas

- **Anmeldung:** LTI 1.3. Lehrende öffnen Teamable direkt aus dem Canvas-Kurs.
- **Zugriff auf Daten:** OAuth2. Die Lehrperson erlaubt Teamable einmalig den Zugriff in ihrem Namen.
- **Gelesen** (Canvas REST API): Teilnehmerliste, Opt-in-Quiz, Punktzahlen aus Aufgaben
- **Geschrieben** (Canvas REST API): Opt-in-Quiz, Teams als Gruppen

→ [Bisherige Canvas-Anbindung](bisherige-canvas-anbindung.md)

## Moodle an der HTWG

- **LTI 1.3** liefert Login, Kurs, Rolle und Teilnehmerliste (NRPS), aber keine Gruppen
- **API bzw. Web-Services** sind aktiv und decken Teilnehmer, Noten und Gruppen ab
- **Lücken:** kein OAuth2 (Token manuell oder technischer Nutzer), Umfragen nicht per API anlegbar

→ [Moodle-API](moodle-api.md)

## Optionen im Vergleich

| Option | Vorteile | Nachteile |
| --- | --- | --- |
| API-Anbindung | Automatisch, Daten aktuell | Pro LMS eigener Connector, Token nötig (Auth schwierig) |
| CSV-Export/-Import | Einfach, kein Admin nötig | Manuell für Dozenten, Daten veralten |
| LMS-Plugin | Alles direkt in Moodle, keine Auth Probleme | Nur Moodle, hoher Aufwand, Quellcode im Rechenzentrum anfassen |
| **LTI** | Kein eigener Login, Standard für alle LMS, Teamable kann es schon | Keine Gruppen, Tool-Registrierung nötig |

→ [Optionen zur Datenübertragung](uebertragungsoptionen.md)

## Entscheidung

LTI ist der Standard über alle LMS hinweg und der Login läuft bereits darüber, deshalb versuchen wir soweit wie möglich mit LTI zu arbeiten.

Funktionen wie z.B. der Gruppen-Export aus Teamable in Moodle brauchen aber eventuell die Moodle-API -> Brauchen Moodle API und ggf. anderen Token dafür nochmal


| Zweck | Weg |
| --- | --- |
| Login | LTI 1.3 |
| Studierenden-Import | LTI 1.3 (NRPS) |
| Punktzahl, Umfragenergebnis, Gruppen-Export | Noch unklar |


!!! question "Offen: Token für die Moodle API"
    Wie Teamable den Token bekommt, ist noch zu klären:

    1. Kommt er direkt mit dem LTI-Login?
    2. Müssen wir einen eigenen Ablauf implementieren?
    3. Wird er manuell übergeben (z. B. technischer Nutzer durch das Rechenzentrum)?
