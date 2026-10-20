# Paper-Vorgaben und ihre Umsetzung im Code

Verglichen wurde der Code mit zwei Papern:

- **Paper 1:** Hui et al. (2022), *Design Guidelines for a Team Formation and Analytics Software*
- **Paper 2:** Hui et al. (2025), *Diversity Considerations in Team Formation Design, Algorithm, and Measurement*

Grundlage ist das Lesen des Codes. Ausgeführt wurde er nicht. Stand des Codes: 20.10.2026.

**Legende:** ✅ umgesetzt · ⚠️ teilweise oder abweichend · ❌ fehlt oder funktioniert nicht

Zu allen Einträgen mit ⚠️ oder ❌ steht in [BUGS.md](BUGS.md), was zu tun ist.

## Priority-Algorithmus (Paper 2)

| # | Vorgabe im Paper | Status | Wo im Code |
|---|---|---|---|
| 1 | **Hill Climbing:** viele Team-Sets parallel verbessern, pro Runde die besten K behalten | ✅ | [priority_algorithm.py:78-93](algorithms/ai/priority_algorithm/priority_algorithm.py#L78-L93) |
| 2 | Parameter `MAXITER`, `SPREAD`, `K` | ✅ | [algorithm_config.py:58-63](algorithms/ai/interfaces/algorithm_config.py#L58-L63) (`MAX_ITERATE`, `MAX_SPREAD`, `MAX_KEEP`) |
| 3 | Werte aus dem Paper: `MAXITER = 250`, `SPREAD = 100`, `K = 30` | ⚠️ | Standardwerte im Code: 1500 / 3 / 3, dazu ein Zeitlimit von 30 s (`MAX_TIME`), [algorithm_config.py:59-62](algorithms/ai/interfaces/algorithm_config.py#L59-L62) |
| 4 | **Startlösung:** zufällig oder mit dem Weight-Algorithmus (GRR) | ✅ | [priority_algorithm.py:41-59](algorithms/ai/priority_algorithm/priority_algorithm.py#L41-L59), Einstellung `START_TYPE` |
| 5 | **Verbesserungsschritt:** zwei zufällige Studierende zwischen Teams tauschen | ✅ | [random_swap.py:17](algorithms/ai/priority_algorithm/mutations/random_swap.py#L17), standardmäßig gesetzt in [algorithm_config.py:79](algorithms/ai/interfaces/algorithm_config.py#L79) |
| 6 | **Gewichtete Prioritäten:** jede höhere Priorität zählt mehr als alle niedrigeren zusammen | ✅ | [scoring.py:10](algorithms/ai/priority_algorithm/scoring.py#L10) (25 Bewertungsstufen), [scoring.py:53-55](algorithms/ai/priority_algorithm/scoring.py#L53-L55) (Multiplikatoren 25ⁿ) |
| 7 | **Tokenism vermeiden** (z. B. nie genau eine Frau allein im Team), Formel mit θ = 0,2 | ✅ | [priority.py:27-100](algorithms/ai/priority_algorithm/priority/priority.py#L27-L100), θ in Zeile 73 |
| 8 | **Diversität** über den Gini-Simpson-Index (Blau-Index) pro Team | ⚠️ | [priority.py:141-184](algorithms/ai/priority_algorithm/priority/priority.py#L141-L184), Index in [diversity_utility.py:44-53](algorithms/ai/weight_algorithm/utility/diversity_utility.py#L44-L53). Die volle Punktzahl ist nie erreichbar (Bug 8). |
| 9 | **Konzentrieren** von Merkmalen (z. B. gemeinsame Zeitfenster) | ✅ | [priority.py:154-181](algorithms/ai/priority_algorithm/priority/priority.py#L154-L181) |
| 10 | Prioritätstypen Projekt-Anforderungen, Projekt-Präferenzen und Freunde/Feinde | ✅ im Algorithmus · ❌ über die API | Klassen: [priority.py:197-299](algorithms/ai/priority_algorithm/priority/priority.py#L197-L299). Über die API kommen sie nicht an (Bugs 1 bis 4). |

## Dashboard-Optionen (Paper 1)

| # | Vorgabe im Paper | Status | Wo im Code |
|---|---|---|---|
| 11 | Gewichtete Kriterien im Weight-Algorithmus (Anforderungen, Social, Diversität, Präferenzen) | ✅ | [algorithm_options.py:62-113](algorithms/ai/interfaces/algorithm_options.py#L62-L113), [weight_algorithm/](algorithms/ai/weight_algorithm/) |
| 12 | Schalter für Freunde/Feinde: *einbeziehen*, *umkehren*, *ignorieren* | ❌ | Die Optionen werden eingelesen ([algorithm_options.py:69-70](algorithms/ai/interfaces/algorithm_options.py#L69-L70)), aber nirgends ausgewertet (Bug 5) |

## Metriken zur Auswertung (Paper 2)

| # | Metrik im Paper | Status | Wo im Code |
|---|---|---|---|
| 13 | Intra-Team-Heterogenität (Gini-Simpson) | ✅ | [average_gini_index.py](benchmarking/evaluations/metrics/average_gini_index.py), [average_gini_index_multi_attribute.py](benchmarking/evaluations/metrics/average_gini_index_multi_attribute.py) |
| 14 | Inter-Team-Homogenität | ✅ | [cosine_similarity.py](benchmarking/evaluations/metrics/cosine_similarity.py) |
| 15 | Solo Status (Tokenism) | ✅ | [average_solo_status.py](benchmarking/evaluations/metrics/average_solo_status.py) |
| 16 | Abdeckung der Projekt-Anforderungen | ✅ | [average_project_requirements_coverage.py](benchmarking/evaluations/metrics/average_project_requirements_coverage.py) |
| 17 | Gemeinsame Zeitfenster | ✅ | [average_timeslot_coverage.py](benchmarking/evaluations/metrics/average_timeslot_coverage.py) |
| 18 | Social Satisfaction | ⚠️ | [average_social_satisfied.py](benchmarking/evaluations/metrics/average_social_satisfied.py). Der Code zählt Teams, in denen *alle* zufrieden sind. Im Paper-Text steht „Anteil zufriedener Mitglieder“. Die Zahlen im Paper (z. B. 44,4 % = 4 von 9 Teams) passen zum Code, ungenau ist also der Paper-Text. |

## Vergleich mit anderen Algorithmen (Paper 2)

| # | Vorgabe im Paper | Status | Wo im Code |
|---|---|---|---|
| 19 | Vergleichsalgorithmen DRR und Group Matcher | ❌ | Fehlen im Repository; auch [enums.py:32-36](algorithms/dataclasses/enums.py#L32-L36) kennt sie nicht (Bug 6) |
| 20 | Benchmark-Skripte, mit denen die Tabellen und Grafiken entstanden sind | ❌ | [benchmarking/runs/](benchmarking/runs/) ist vorhanden, die Skripte importieren aber Module, die es nicht mehr gibt (Bug 6) |
