# Optionen zur Datenübertragung

!!! info "Spike 1 – Evaluierung der Schnittstellen- und Integrationspfade"
    **Verantwortlich:** Jonas
    **Status:** In Arbeit

    **DoD:** Vergleichstabelle mit Vor- und Nachteilen je Option liegt vor.

## Ziel

Wege vergleichen, wie Kurs- und Studierendendaten aus einem LMS (Canvas, Moodle) in das Teambildungs-System gelangen, ohne das System fest an ein LMS zu binden.

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

Das System ruft die Daten per Token direkt von der Schnittstelle des LMS ab (z. B. Moodle Web Services, Canvas REST API).

- **Vorteile:**
    - Gute Usability: nach einmaliger Einrichtung keine manuellen Schritte, Lehrende wählen nur den Kurs
    - Daten sind immer aktuell
- **Nachteile:**
    - Pro LMS ein eigener Connector (Moodle-API ≠ Canvas-API)
    - Token bzw. Freigabe der Web Services durch Admins nötig
    - Abhängig von API-Änderungen des LMS

### CSV-Export/-Import

Lehrende exportieren die Teilnehmerliste aus dem LMS (z. B. Moodle: Teilnehmer/innen → „Herunterladen als CSV“) und laden sie im System hoch.

- **Vorteile:**
    - Wenig Aufwand, keine Freigaben nötig
    - Funktioniert mit jedem LMS
- **Nachteile:**
    - Schlechte Usability: Lehrende müssen Export suchen, Datei speichern, hochladen – bei jeder Änderung erneut
    - Liste ist veralten, sobald sich die Kursteilnahme ändert
    - Die Spaltenformate unterscheiden sich auch je LMS, Fehler durch Kodierung/Trennzeichen

### LMS-Plugin

Erweiterung, die im LMS installiert wird und die Daten an das System übergibt.

- **Vorteile:**
- **Nachteile:**

### LTI (Learning Tools Interoperability)

Einbindung als externes Tool über den LTI-Standard, der von mehreren LMS unterstützt wird.

- **Vorteile:**
- **Nachteile:**

## Workflows

### Moodle-API

Teamable ruft Kursdaten und Fragebogen-Antworten per Token aus Moodle ab. Der Fragebogen läuft in Moodle (Aktivität „Feedback“).

**Dozenten:**

1. Loggen sich in Teamable ein (eigener Login) und verknüpfen einmalig ihr Moodle-Konto (Token).
2. Wählen in Teamable den Kurs aus. Teamable holt die Teilnehmerliste über die API.
3. Legen in Moodle einen Fragebogen (Feedback-Aktivität) an.
4. Starten in Teamable die Teambildung. Teamable holt die Antworten über die API.
5. Optional: Teams werden über die API als Moodle-Gruppen zurückgeschrieben.

**Studierende:**

1. Füllen den Fragebogen in Moodle aus.
2. Sehen Teamable nie.

**Fazit:** Studierende bleiben in Moodle, aber Dozenten brauchen einen eigenen Login und die Lösung funktioniert nur mit Moodle.

### LTI

Teamable wird als Aktivität in den Moodle-Kurs eingebunden. Login und Kurskontext kommen über LTI, der Fragebogen läuft in Teamable.

**Dozenten:**

1. Fügen im Moodle-Kurs die Aktivität „Teambildung“ (LTI-Tool) hinzu.
2. Klicken darauf und sind automatisch in Teamable eingeloggt, ohne eigenen Login.
3. Erstellen beim ersten Aufruf den Fragebogen (oder übernehmen eine Vorlage). Er ist automatisch dieser Aktivität im Kurs zugeordnet.
4. Sehen, wer schon ausgefüllt hat (Abgleich mit der Teilnehmerliste über NRPS).
5. Starten die Teambildung mit den gespeicherten Antworten.

**Studierende:**

1. Klicken im Moodle-Kurs auf die Aktivität „Teambildung“.
2. Sind automatisch eingeloggt und sehen nur den Fragebogen.
3. Füllen ihn aus, schicken ihn ab und kehren zu Moodle zurück.

**Fazit:** Kein eigener Login für irgendwen, keine Moodle-API nötig, funktioniert mit jedem LMS, das LTI unterstützt. Dafür muss Teamable den Fragebogen und eine Oberfläche für Studierende selbst bereitstellen.

!!! note "Offen"
    Wie frei Dozenten den Fragebogen gestalten dürfen, hängt von den Pflicht-Eingabedaten des Algorithmus ab (Spike 2).

## Vergleichstabelle

| Option | Usability | Aktualität | Aufwand | Flexibilität | Abhängigkeiten |
| --- | --- | --- | --- | --- | --- |
| API-Anbindung | gut | gut | mittel | mittel | Admin-Freigabe, Token |
| CSV-Export/-Import | schlecht | schlecht | gering | hoch | keine |
| LMS-Plugin | | | | | |
| LTI | | | | | |

## Fazit

## Quellen
