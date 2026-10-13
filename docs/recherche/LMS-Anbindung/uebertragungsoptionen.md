# Optionen zur Datenübertragung

!!! info "Spike 1 – Evaluierung der Schnittstellen- und Integrationspfade"
    **Verantwortlich:** Jonas
    **Status:** In Arbeit

    **DoD:** Vergleichstabelle mit Vor- und Nachteilen je Option liegt vor.

## Ziel

Mögliche Wege sammeln, wie Kurs- und Studierendendaten aus einem LMS (Canvas, Moodle) in das Teambildungs-System gelangen, sodass das System nicht fest an ein bestimmtes LMS gebunden ist. Die Optionen werden nach Umsetzungsaufwand, Flexibilität und Abhängigkeiten bewertet.

## Bewertungskriterien

| Kriterium | Leitfrage |
| --- | --- |
| Umsetzungsaufwand | Wie viel Entwicklungsaufwand ist im Projektzeitraum nötig? |
| Flexibilität | Funktioniert die Option mit mehreren LMS bzw. ohne LMS? |
| Abhängigkeiten | Welche Rechte, Freigaben oder externen Systeme werden benötigt (z. B. Admin-Zugang, API-Token, Plugin-Installation)? |

## Optionen

### API-Anbindung

Direkter Abruf der Daten über die Schnittstelle des LMS (z. B. Moodle Web Services, Canvas REST API).

- **Vorteile:**
- **Nachteile:**

### CSV-Export/-Import

Lehrende exportieren Kurs- bzw. Teilnehmerlisten aus dem LMS und laden die Datei in das System hoch.

- **Vorteile:**
- **Nachteile:**

### LMS-Plugin

Erweiterung, die im LMS installiert wird und die Daten an das System übergibt.

- **Vorteile:**
- **Nachteile:**

### LTI (Learning Tools Interoperability)

Einbindung als externes Tool über den LTI-Standard, der von mehreren LMS unterstützt wird.

- **Vorteile:**
- **Nachteile:**

## Vergleichstabelle

| Option | Umsetzungsaufwand | Flexibilität | Abhängigkeiten | Vorteile | Nachteile |
| --- | --- | --- | --- | --- | --- |
| API-Anbindung | | | | | |
| CSV-Export/-Import | | | | | |
| LMS-Plugin | | | | | |
| LTI | | | | | |

## Fazit

## Quellen
