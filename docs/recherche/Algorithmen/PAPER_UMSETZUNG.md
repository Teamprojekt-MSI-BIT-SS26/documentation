# Paper-Vorgaben und ihre Umsetzung im Code

Verglichen wurde der Code mit zwei Papern:

- **Paper 1:** Hui et al. (2022), *Design Guidelines for a Team Formation and Analytics Software*
- **Paper 2:** Hui et al. (2025), *Diversity Considerations in Team Formation Design, Algorithm, and Measurement*

Grundlage ist das Lesen des Codes. Ausgeführt wurde er nicht. Stand des Codes: 20.10.2026.

**Legende:** ✅ umgesetzt · ⚠️ teilweise oder abweichend · ❌ fehlt oder funktioniert nicht

Zu allen Einträgen mit ⚠️ oder ❌ steht in [BUGS.md](BUGS.md), was zu tun ist.

**Cluster:**
1. [Projektanforderungen](#1-projektanforderungen)
2. [Priority-Bewertung](#2-priority-bewertung)
3. [Tokenism-Funktion](#3-tokenism-funktion)
4. [Diversitätsmetriken](#4-diversitätsmetriken)
5. [Allgemein](#5-allgemein)

## Überblick: Wie die Algorithmen zusammenwirken

Die Gesamtlogik lässt sich wie folgt verstehen:

- Der **Priority-Algorithmus** ist der eigentliche Optimierungsalgorithmus. Er erzeugt Teamzuordnungen, bewertet sie mit Prioritäten und verbessert sie iterativ durch kleine Tauschschritte (`Hill Climbing`). Die einzelnen Prioritäten wie Skills, Projektwünsche, Diversität oder Tokenism sind dabei als eigene Regeln in der Priority-Logik eingebaut.
- Der **Weight-Algorithmus** dient vor allem als **Startlösung** oder als alternative Bewertungsgrundlage. In der Praxis wird er als GRR (Greedy Round Robin) genutzt, um eine bereits gute erste Einteilung zu erzeugen, bevor der Priority-Algorithmus weiter optimiert.
- Der **Random-Algorithmus** bildet die Vergleichsbasis: Er liefert eine einfache, nicht optimierte Einteilung und zeigt damit, wie viel durch die gezielten Prioritäten gewonnen wird.
- Die **Metriken** und Evaluationsfunktionen bewerten anschließend die Qualität der Einteilung: z. B. Projektabdeckung, soziale Zufriedenheit, Diversität, gemeinsame Zeiten und Laufzeit.

Kurz gesagt: **Weight oder Random liefern eine erste Lösung, der Priority-Algorithmus verbessert diese Lösung, und die Metriken prüfen am Ende, wie gut das Ergebnis wirklich ist.**

Damit wird klar, dass die einzelnen Teile nicht isoliert stehen, sondern als Ablauf zusammenarbeiten:

1. Eingaben definieren die Kriterien.
2. Eine Startlösung wird erzeugt.
3. Der Priority-Algorithmus bewertet und verbessert sie.
4. Die Metriken messen das Ergebnis und zeigen, ob die Vorgaben erfüllt wurden.

---

## 1. Projektanforderungen

| # | Vorgabe im Paper | Quelle | Status | Algorithmus | Wo im Code |
|---|---|---|---|---|---|
| 1.1 | **Skills auf Projektanforderungen abstimmen:** Jedes Projekt hat Anforderungen (z. B. „kann Python“). Ein Team erfüllt eine Anforderung, wenn seine Mitglieder die nötigen Fähigkeiten mitbringen. | P1 3.1.2, P2 3.1 | ✅ | Priority | [priority.py:197-212](algorithms/ai/priority_algorithm/priority/priority.py#L197-L212) (`RequirementPriority`), Regeln *jemand / alle / n Mitglieder* in [project/dataclass.py:19-80](algorithms/dataclasses/project/dataclass.py#L19-L80) |
| 1.2 | **Projekt-Präferenzen:** Studierende geben eine Rangfolge ihrer Wunschprojekte an. Je höher das zugeteilte Projekt in ihrer Liste steht, desto besser. | P1 3.1.2 | ✅ im Algorithmus · ❌ über die API | Priority | [priority.py:226-245](algorithms/ai/priority_algorithm/priority/priority.py#L226-L245) (`ProjectPreferencePriority`). Über die API funktioniert das nicht (Bugs 1, 2, 4). |
| 1.3 | **Projektanforderungen als Priorität über die API:** Die Anforderungen sollen sich wie jedes andere Kriterium als Priorität einstellen lassen. | P2 3.1 | ❌ | Priority | [algorithm_options.py:166-175](algorithms/ai/interfaces/algorithm_options.py#L166-L175), [priority.py:214-222](algorithms/ai/priority_algorithm/priority/priority.py#L214-L222) (Bugs 1 bis 3) |
| 1.4 | **Metrik *Project Coverage*:** Anteil der Anforderungen, die mindestens ein Teammitglied erfüllt, gemittelt über alle Projekte. | P2 4.4 | ✅ | Metrik / Benchmark | [average_project_requirements_coverage.py:16-40](benchmarking/evaluations/metrics/average_project_requirements_coverage.py#L16-L40) |

## 2. Priority-Bewertung

| # | Vorgabe im Paper | Quelle | Status | Algorithmus | Wo im Code |
|---|---|---|---|---|---|
| 2.1 | **Geordnete Kriterienliste:** Die Lehrperson legt fest, welche Kriterien in welcher Reihenfolge wichtig sind. Ein wichtigeres Kriterium zählt immer mehr als alle unwichtigeren zusammen. | P2 3, 3.1 | ✅ | Priority | [priority_algorithm.py:86-92](algorithms/ai/priority_algorithm/priority_algorithm.py#L86-L92) sortiert nach dem Score |
| 2.2 | **`satisfy()`:** Für jedes Kriterium wird ein Erfüllungsgrad zwischen 0 und 1 berechnet und über alle Teams gemittelt. | P2 3.1 | ✅ | Priority | [scoring.py:38-50](algorithms/ai/priority_algorithm/scoring.py#L38-L50) |
| 2.3 | **`normalize()` mit B Stufen:** Der Erfüllungsgrad wird in eine von B ganzzahligen Stufen eingeteilt. Winzige Unterschiede (z. B. 0,451 und 0,4512) zählen dadurch nicht. | P2 3.1 | ✅ | Priority | [scoring.py:10](algorithms/ai/priority_algorithm/scoring.py#L10) (B = 25), [scoring.py:24-35](algorithms/ai/priority_algorithm/scoring.py#L24-L35) |
| 2.4 | **`score()`:** Gesamtwert = Summe aus Stufe × B^(j−1). Durch die Potenzen von B dominiert das wichtigste Kriterium. | P2 3.1 | ✅ | Priority | [scoring.py:53-56](algorithms/ai/priority_algorithm/scoring.py#L53-L56) (Multiplikatoren) |
| 2.5 | **Prioritätstypen:** Projektanforderungen, soziale Präferenzen, Merkmale konzentrieren, Merkmale diversifizieren und Tokenism vermeiden. | P2 3.1 | ✅ im Algorithmus · ❌ über die API | Priority | Klassen: [priority.py:27-299](algorithms/ai/priority_algorithm/priority/priority.py#L27-L299). Über die API kommt nur der Tokenism-Typ an (Bugs 1 bis 4). |
| 2.6 | **Metrik *Priority Satisfaction*:** Gewichtete Summe der Erfüllungsgrade mit B = 2. Wird nur intern zum Testen der Parameter verwendet. | P2 3.3 | ✅ | Metrik / Benchmark | [priority_satisfaction.py:39-69](benchmarking/evaluations/metrics/priority_satisfaction.py#L39-L69) (`exponent_base = 2`) |

## 3. Tokenism-Funktion

| # | Vorgabe im Paper | Quelle | Status | Algorithmus | Wo im Code |
|---|---|---|---|---|---|
| 3.1 | **Tokenism vermeiden:** Eine Minderheit (z. B. Frauen) wird so verteilt, dass in jedem Team entweder keine oder mindestens m Personen davon sind. So ist niemand die „einzige“ im Team. | P2 3.1.3 | ✅ | Priority | [priority.py:27-48](algorithms/ai/priority_algorithm/priority/priority.py#L27-L48) (`TokenizationPriority`, m = `threshold`) |
| 3.2 | **Stückweise Bewertungsfunktion f(x):** x = Anzahl der Minderheit im Team, n = Teamgröße. Es gilt f(0) = θ, f(0 < x < m) = 0, f(m) = 1. Danach fällt die Funktion linear von 1 − θ auf f(n) = θ, damit nicht alle in einem Team landen. | P2 3.1.3 | ✅ | Priority | [priority.py:79-93](algorithms/ai/priority_algorithm/priority/priority.py#L79-L93). Die Formel entspricht exakt dem Paper. |
| 3.3 | **θ = 0,2:** Mindestwert, den ein Team ohne Minderheit oder mit zu vielen davon bekommt. θ muss kleiner als 0,5 sein, sonst steigt die Funktion statt zu fallen. | P2 3.1.3 | ✅ | Priority | [priority.py:73](algorithms/ai/priority_algorithm/priority/priority.py#L73) |
| 3.4 | **Alternative: Minderheiten konzentrieren** (Szenario 3): Statt zu verteilen, werden Minderheiten bewusst zusammengelegt, ebenfalls um Tokenism zu vermeiden. | P2 4.1 | ✅ | Priority | [priority.py:95-100](algorithms/ai/priority_algorithm/priority/priority.py#L95-L100) (Variante `CONCENTRATE` mit `MAX_OF`) |

## 4. Diversitätsmetriken

| # | Vorgabe im Paper | Quelle | Status | Algorithmus | Wo im Code |
|---|---|---|---|---|---|
| 4.1 | **Merkmale als Bit-Vektor:** Jede Ausprägung eines Merkmals bekommt ein eigenes Bit (ist-Frau, ist-Mann, …). So ist kein Wert „näher“ an einem anderen, was bei Zahlencodes wie 1/2/3 der Fall wäre. | P2 3.1.1 | ✅ | Priority | [priority/utils.py:6-20](algorithms/ai/priority_algorithm/priority/utils.py#L6-L20), für die Metriken [cosine_similarity.py:14-42](benchmarking/evaluations/metrics/cosine_similarity.py#L14-L42) |
| 4.2 | **Diversifizieren im Score über den Gini-Simpson-Index:** Der Index misst, wie gemischt ein Team bei einem Merkmal ist, also die Wahrscheinlichkeit, dass zwei zufällig gezogene Mitglieder verschieden sind. | P2 3.1.2 | ✅ | Priority | [priority.py:183-184](algorithms/ai/priority_algorithm/priority/priority.py#L183-L184), Index in [diversity_utility.py:44-53](algorithms/ai/weight_algorithm/utility/diversity_utility.py#L44-L53). Wie im Paper erreicht der Index nie den Wert 1 (siehe Verbesserung 8 in BUGS.md). |
| 4.3 | **Konzentrieren (Clustern) von Merkmalen:** Gleiche Ausprägungen kommen ins selbe Team, z. B. Studierende mit gemeinsamen Zeitfenstern. | P1 3.1.2, P2 3.1 | ✅ | Priority | [priority.py:155-181](algorithms/ai/priority_algorithm/priority/priority.py#L155-L181) (Skalarprodukt der Bit-Vektoren) |
| 4.4 | **Metrik *Intra-Heterogenität*:** Wie verschieden sind die Mitglieder *innerhalb* eines Teams? Berechnet als Kosinus-Differenz 1 − cos(θ) zwischen den Bit-Vektoren (0 = identisch, 1 = maximal verschieden), gemittelt über alle Teams. | P2 4.4 | ✅ | Metrik / Benchmark | [cosine_similarity.py:109-124](benchmarking/evaluations/metrics/cosine_similarity.py#L109-L124) (`AverageCosineDifference.calculate`) |
| 4.5 | **Metrik *Inter-Homogenität*:** Sind alle Teams *ähnlich* divers? Berechnet als Standardabweichung der Intra-Heterogenität über alle Teams. Kleiner ist besser. | P2 4.4 | ✅ | Metrik / Benchmark | [cosine_similarity.py:126-131](benchmarking/evaluations/metrics/cosine_similarity.py#L126-L131) (`calculate_stdev`). Die Standardabweichung der Ähnlichkeit ist mathematisch gleich der Standardabweichung der Differenz. |
| 4.6 | **Metrik *Solo Status*:** Anzahl der Personen, die als einzige ihrer Minderheit im Team sind, geteilt durch alle Studierenden. Misst, ob Tokenism vermieden wurde. | P2 4.4 | ✅ | Metrik / Benchmark | [average_solo_status.py:26-52](benchmarking/evaluations/metrics/average_solo_status.py#L26-L52) |

## 5. Allgemein

| # | Vorgabe im Paper | Quelle | Status | Algorithmus | Wo im Code |
|---|---|---|---|---|---|
| 5.1 | **Hill Climbing:** Ein Suchverfahren, das von einer Startlösung ausgeht und sie Schritt für Schritt durch kleine Änderungen verbessert. Das Paper verfolgt dabei mehrere Lösungswege gleichzeitig, um nicht in einem lokalen Optimum stecken zu bleiben. | P2 3, Alg. 1 | ✅ | Priority | [priority_algorithm.py:71-97](algorithms/ai/priority_algorithm/priority_algorithm.py#L71-L97) |
| 5.2 | **Parameter `MAXITER`, `SPREAD`, `K`:** `MAXITER` = Anzahl der Verbesserungsrunden (Tiefe der Suche). `SPREAD` = Anzahl der Varianten pro Lösung und Runde. `K` = Anzahl der besten Lösungen, die in die nächste Runde kommen. | P2 3, 3.3 | ✅ | Priority | [algorithm_config.py:58-63](algorithms/ai/interfaces/algorithm_config.py#L58-L63) (`MAX_ITERATE`, `MAX_SPREAD`, `MAX_KEEP`) |
| 5.3 | **Werte aus dem Paper:** `MAXITER = 250`, `SPREAD = 100`, `K = 30`. Mit diesen Werten wurden die Ergebnisse in Paper 2 erzeugt. | P2 3.3, 5.1 | ⚠️ | Priority | Standardwerte im Code: 1500 / 3 / 3, dazu ein Zeitlimit von 30 s (`MAX_TIME`), [algorithm_config.py:59-62](algorithms/ai/interfaces/algorithm_config.py#L59-L62) (Bug 7) |
| 5.4 | **Startlösung:** Die Suche beginnt mit einer zufälligen Einteilung oder mit dem Ergebnis von GRR (Greedy Round Robin, dem Weight-Algorithmus). GRR liefert schon eine fast optimale Startlösung. | P2 3.3 | ✅ | Weight / Random | [priority_algorithm.py:41-59](algorithms/ai/priority_algorithm/priority_algorithm.py#L41-L59) (`START_TYPE`) |
| 5.5 | **`improve()`:** Zwei zufällige Studierende aus verschiedenen Teams werden getauscht. Weil die Ausgangslösung erhalten bleibt, wird das Ergebnis nie schlechter. | P2 3.2 | ✅ | Priority | [random_swap.py:17](algorithms/ai/priority_algorithm/mutations/random_swap.py#L17), standardmäßig gesetzt in [algorithm_config.py:79](algorithms/ai/interfaces/algorithm_config.py#L79) |
| 5.6 | **Weight-Algorithmus mit Gewichtungen:** Im Dashboard stellt die Lehrperson ein, wie stark soziale Präferenzen, Projekt-Präferenzen, Projektanforderungen und Diversität zählen. | P1 3.1.2 | ✅ | Weight | [algorithm_options.py:62-113](algorithms/ai/interfaces/algorithm_options.py#L62-L113), [weight_algorithm/](algorithms/ai/weight_algorithm/) |
| 5.7 | **Soziale Präferenzen:** Studierende nennen Freunde (gleiches Team) und Feinde (anderes Team). Der Algorithmus versucht, diese Wünsche zu erfüllen. | P1 3.1.2, P2 3.1 | ✅ im Algorithmus · ❌ über die API | Priority | [priority.py:259-290](algorithms/ai/priority_algorithm/priority/priority.py#L259-L290) (`SocialPreferencePriority`). Über die API funktioniert das nicht (Bug 1). |
| 5.8 | **Schalter für Freunde/Feinde:** Die Wünsche können *beachtet*, *umgekehrt* oder *ignoriert* werden. | P1 3.1.2 | ❌ | Priority / API | Die Optionen werden eingelesen ([algorithm_options.py:69-70](algorithms/ai/interfaces/algorithm_options.py#L69-L70)), aber nirgends ausgewertet (Bug 5) |
| 5.9 | **Metrik *Social Satisfaction*:** Anteil der Mitglieder, deren Wünsche erfüllt sind (mindestens ein Freund und kein Feind im Team), gemittelt über alle Teams. | P2 4.4 | ⚠️ | Metrik / Benchmark | Die Paper-Definition für einzelne Mitglieder gibt es als `has_friend_and_no_enemies` ([student_calculations.py:46](benchmarking/evaluations/metrics/utils/student_calculations.py#L46)). Die Metrik ([average_social_satisfied.py:15-19](benchmarking/evaluations/metrics/average_social_satisfied.py#L15-L19)) zählt aber pro Team nur ja oder nein, nicht den Anteil der Mitglieder. Die 44,4 % im Paper (= 4 von 9 Teams) passen zu dieser Zählweise. |
| 5.10 | **Metrik *Common Times*:** Anteil der Zeitfenster, an denen *alle* Teammitglieder können, gemittelt über alle Teams. | P2 4.4 | ✅ | Metrik / Benchmark | [average_timeslot_coverage.py:45-57](benchmarking/evaluations/metrics/average_timeslot_coverage.py#L45-L57) |
| 5.11 | **Metrik *Run Time*:** Wie lange braucht ein Algorithmus für eine Einteilung? | P2 4.4, 5.1 | ✅ | Metrik / Benchmark | [simulation.py:161-163](benchmarking/simulation/simulation.py#L161-L163) |
| 5.12 | **Vergleichsalgorithmen:** Random als Untergrenze, DRR (Double Round Robin, ordnet Skills Projekten zu), GRR und Group Matcher (vermeidet Tokenism in Lerngruppen). | P2 4.3 | ⚠️ | Random / Weight / DRR / Group Matcher | Random und GRR: [random_algorithm/](algorithms/ai/random_algorithm/), [weight_algorithm/](algorithms/ai/weight_algorithm/). DRR und Group Matcher fehlen; auch [enums.py:32-36](algorithms/dataclasses/enums.py#L32-L36) kennt sie nicht (Bug 6). |
| 5.13 | **Offenes Evaluations-Framework:** Klassen werden simuliert oder als echte Daten eingelesen, alle Algorithmen laufen über mehrere Durchläufe, und die Ergebnisse werden mit den Metriken verglichen. | P2 4 | ⚠️ | Benchmarking / Evaluation | Das Framework existiert ([benchmarking/simulation/](benchmarking/simulation/), [benchmarking/data/](benchmarking/data/)). Die Skripte für die Paper-Szenarien in [benchmarking/runs/](benchmarking/runs/) importieren aber Module, die es nicht mehr gibt (Bug 6). |
