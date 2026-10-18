# Optionen zur Datenübertragung

## Ziel

Wege finden und vergleichen, wie Kurs- und Studierendendaten aus einem LMS in das Teamable-System gelangen, ohne das System fest an ein LMS zu binden.

## Kriterien

| Kriterium | Leitfrage |
| --- | --- |
| Usability | Wie einfach ist es für Dozenten? Wie viele manuelle Schritte? |
| Aktualität | Sind die Daten automatisch aktuell (z. B. bei späten Einschreibungen)? |
| Umsetzungsaufwand | Im Projektzeitraum machbar? |
| Flexibilität | Funktioniert es mit mehreren LMS? |
| Abhängigkeiten | Braucht es Admin-Freigaben, Tokens oder eine Installation im LMS? |

## Optionen

### API-Anbindung

Das System ruft die Daten per Token direkt von der Schnittstelle des LMS ab (Moodle Web Services, Canvas REST API).

- **Vorteile:**
    - Gute Usability: nach einmaliger Einrichtung keine manuellen Schritte, Lehrende wählen nur den Kurs
    - Daten sind immer aktuell
- **Nachteile:**
    - Pro LMS ein eigener Connector (Moodle-API ≠ Canvas-API) -> Adapter Pattern?
    - Token bzw. Freigabe der Web Services durch Admins nötig
    - Abhängig von API-Änderungen des LMS

### CSV-Export/-Import

Dozenten exportieren die Teilnehmerliste aus dem LMS (z. B. Moodle: Teilnehmer/innen → „Herunterladen als CSV“) und laden sie im System hoch.

- **Vorteile:**
    - Wenig Aufwand, keine Freigaben nötig
    - Funktioniert mit jedem LMS
- **Nachteile:**
    - Schlechte Usability: Lehrende müssen Export suchen, Datei speichern, hochladen – bei jeder Änderung erneut, Teams dann manuell erstellen
    - Liste ist veralten, sobald sich die Kursteilnahme ändert
    - Spaltenformate unterscheiden sich je LMS -> Sparen uns nicht wirklich die Customization pro LMS


### LMS-Plugin

Erweiterung, die im LMS installiert wird und die Daten an das System übergibt.

- **Vorteile:**
    - Sehr gute Usability: alles direkt in Moodle, kein Wechsel in ein anderes System
    - Voller Zugriff auf Moodle-Daten (Teilnehmer, Gruppen, Aktivitäten)
- **Nachteile:**
    - Nur für Moodle, für jedes weitere LMS ein eigenes Plugin
    - Hoher Aufwand: PHP und Moodle-Plugin-Framework, bestehendes Tool müsste neu gebaut oder angebunden werden
    - Installation nur durch das Rechenzentrum, Anpassungen bei jedem Moodle-Update

### LTI (Learning Tools Interoperability)

Einbindung als externes Tool über den LTI-Standard, der von mehreren LMS unterstützt wird.

- **Vorteile:**
    - Gute Usability: Dozenten und Studierende klicken nur eine Aktivität im Kurs, kein eigener Login
    - Teilnehmerliste ist immer aktuell (über NRPS)
    - Ein Standard für Moodle, Canvas, ILIAS usw.
    - Bestehendes Tool spricht bereits LTI (Canvas), daher vermutlich nur Anpassungen nötig
- **Nachteile:**
    - Tool muss im LMS registriert werden (Admin bzw. Rechenzentrum)
    - Liefert nur Login, Kurs und Teilnehmer, Fragebogen muss in Teamable laufen
    - Unterschiede je LMS müssen getestet werden, Canvas-spezifische Teile müssen ersetzt werden

## Gewünschter Workflow (LTI)

**Dozenten:**

1. Fügen „Teambildung“ als Aktivität im Moodle-Kurs hinzu.
2. Öffnen sie und erstellen den Fragebogen in Teamable.
3. Starten die Teambildung, sobald die Studierenden geantwortet haben.

**Studierende:**

1. Öffnen „Teambildung“ im Moodle-Kurs.
2. Füllen den Fragebogen aus und kehren zu Moodle zurück.

Login, Kurs und Teilnehmerliste kommen automatisch über LTI. Niemand braucht einen eigenen Account.

## Vergleichstabelle

| Option | Usability | Aktualität | Aufwand | Flexibilität | Abhängigkeiten |
| --- | --- | --- | --- | --- | --- |
| API-Anbindung | gut | gut | mittel | mittel | Admin-Freigabe, Token |
| CSV-Export/-Import | schlecht | schlecht | gering | hoch | keine |
| LMS-Plugin | sehr gut | gut | hoch | gering | Installation durch Rechenzentrum |
| LTI | gut | gut | gering–mittel | hoch | Tool-Registrierung im LMS |

## Fazit/Empfehlung

**LTI ist der empfohlene Weg.** Das Tool benutzt bereits LTI in Canvas (zur Auth). Statt einer neuen Anbindung muss es also vor allem für Moodle angepasst werden. LTI bietet dabei die beste Kombination aus Usability (kein eigener Login), aktuellen Daten und Unabhängigkeit vom LMS.

- **CSV** ist einfach umzusetzen, verschiebt die Arbeit aber auf Dozenten. 
- **Moodle-API** ergänzend, für das, was LTI nicht abdeckt (z. B. Teams als Moodle-Gruppen anlegen)
- **Moodle-Plugin** scheidet aus: sehr aufwendig und nur für Moodle, wir müssten fast alles neu bauen

## Quellen

- [LTI 1.3 Core Specification (1EdTech)](https://www.imsglobal.org/spec/lti/v1p3/)
- [LTI Names and Role Provisioning Services (NRPS)](https://www.imsglobal.org/spec/lti-nrps/v2p0)
- [LTI Assignment and Grade Services (AGS)](https://www.imsglobal.org/spec/lti-ags/v2p0)
- [Moodle Docs: LTI and Moodle](https://docs.moodle.org/en/LTI_and_Moodle)
- [Moodle Docs: External tool](https://docs.moodle.org/en/External_tool)
- [Moodle Docs: Using web services](https://docs.moodle.org/en/Using_web_services)
- [Moodle Developer Docs: Plugin types](https://moodledev.io/docs/apis/plugintypes)
- [Canvas REST API](https://canvas.instructure.com/doc/api/)
