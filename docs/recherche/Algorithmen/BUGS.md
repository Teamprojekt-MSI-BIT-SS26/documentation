# Offene Bugs: Was behoben werden muss, damit alles aus den Papern funktioniert

Gefunden durch Lesen des Codes, nicht durch Ausführen. Stand: 20.10.2026.
Welche Paper-Vorgabe jeweils betroffen ist, steht in [PAPER_UMSETZUNG.md](PAPER_UMSETZUNG.md).

## Übersicht

| # | Bug | Folge | Dringlichkeit |
|---|---|---|---|
| 1 | API und CLI lesen jede Priorität als Tokenism-Priorität ein | Diversitäts-, Projekt- und Social-Prioritäten führen zu einem Fehler | 🔴 hoch |
| 2 | Die Validierung löscht `priority_type` aus den Request-Daten | Verhindert, dass Bug 1 allein behoben werden kann; außerdem fehlt der Typ in den API-Logs | 🔴 hoch |
| 3 | Schema und Klasse von `RequirementPriority` passen nicht zusammen; `criteria` lässt sich über die API nirgends setzen | Projekt-Anforderungs-Prioritäten lassen sich nicht erzeugen; über die API gilt immer „eine Person reicht“ | 🔴 hoch |
| 4 | Das Schema von `DiversityPriority` erlaubt `max_num_choices` nicht | Merkmale konzentrieren (z. B. Zeitfenster) ist über die API unmöglich | 🔴 hoch |
| 5 | Enum-Werte werden beim Einlesen nicht umgewandelt | Projekt-Präferenzen verhalten sich falsch | 🟠 mittel |
| 6 | Freund/Feind-Schalter werden ignoriert | Die Dashboard-Option aus Paper 1 hat keine Wirkung | 🟠 mittel |
| 7 | Zwei Methoden im Validator sind doppelt definiert | Zwei Prüfungen finden nie statt | 🟠 mittel |
| 8 | Die Gewichte wirken beim Priority-Algorithmus nur auf die Startlösung | Ohne passende Priorität gehen z. B. erfüllte Projektwünsche beim Hill Climbing wieder verloren | 🟠 mittel |
| 9 | Fehlende Antworten zählen als eigene Kategorie | Diversitätswerte sind leicht verfälscht | 🟡 niedrig |

Die Bugs 1 bis 5 betreffen dasselbe Thema (Prioritäten einlesen) und sollten zusammen behoben werden.

**Betroffen ist nur `algorithm_type: "priority"`.** Bei `weight`, `social` und `random` laufen die Codestellen der Bugs 1 bis 5 nicht.

---

## 1. API und CLI lesen jede Priorität als Tokenism-Priorität ein 🔴

**Wo:** [algorithm_options.py:166-175](algorithms/ai/interfaces/algorithm_options.py#L166-L175)

`PriorityAlgorithmOptions.parse_json` erzeugt für jede Priorität ein `TokenizationPriority`-Objekt, egal welcher `priority_type` gesendet wurde.

| Gesendeter Typ | Ergebnis |
|---|---|
| `tokenization` | ✅ funktioniert |
| `diversity` | `direction` fehlt → `TokenizationConstraintDirection(None)` → `ValueError` |
| `project_requirement`, `project_preference`, `social_preference` | `strategy` fehlt → `DiversifyType(None)` → `ValueError` |

Über die API ergibt das Status 500. Betroffen ist auch die CLI ([runs/cli.py:52](runs/cli.py#L52)), weil sie dieselbe Methode aufruft.

## 2. Die Validierung löscht `priority_type` aus den Request-Daten 🔴

**Wo:** [algorithm_options.py:224](algorithms/ai/interfaces/algorithm_options.py#L224)

`validate_priority` ruft `priority_data.pop("priority_type")` auf und verändert dabei das Original-Dict. Der Data Loader kopiert die Daten nur flach ([generate_teams_data_loader.py:42](algorithms/api/utils/generate_teams_data_loader.py#L42)), die Prioritäts-Dicts sind also dieselben Objekte.

**Zusammenhang mit Bug 1:** Im Moment fällt das kaum auf, weil `parse_json` den Typ sowieso ignoriert. Wird aber nur Bug 1 behoben, findet `parse_json` keinen `priority_type` mehr und bricht bei **jeder** Priorität ab, auch bei `tokenization`. Beide Bugs müssen also zusammen behoben werden.

**Schon jetzt sichtbare Folge:** Die API protokolliert die Anfrage erst nach dem Einlesen ([generate_teams.py:30-37](algorithms/api/views/generate_teams.py#L30-L37)), und zwar mit denselben Dict-Objekten. In den gespeicherten Logs fehlt deshalb bei jeder Priorität der `priority_type`.

Die CLI ist nicht betroffen, weil sie den Validator nicht aufruft.

## 3. Schema und Klasse von `RequirementPriority` passen nicht zusammen 🔴

**Wo:** [priority.py:197-222](algorithms/ai/priority_algorithm/priority/priority.py#L197-L222)

Das Schema verlangt ein **Pflichtfeld** `criteria`, die Klasse hat aber keine Felder und nur den leeren Konstruktor `RequirementPriority()`:

| Anfrage | Ergebnis |
|---|---|
| ohne `criteria` | ❌ Der Validator lehnt ab (Status 400) |
| mit `criteria` | ✅ Der Validator lässt sie durch, dann `TypeError: unexpected keyword argument 'criteria'` |

Keine Anfrage erfüllt beide Stellen gleichzeitig. Im Moment verdeckt Bug 1 dieses Problem.

**Zusätzlich lässt sich `criteria` über die API auch nicht pro Anforderung setzen:** `ProjectRequirement` hat zwar ein Feld `criteria` ([project/dataclass.py:19](algorithms/dataclasses/project/dataclass.py#L19)). Das Anforderungs-Schema im Validator kennt aber nur `attribute`, `operator` und `value` ([generate_teams_validator.py:281-289](algorithms/api/validators/generate_teams_validator.py#L281-L289)), und der Serializer liest auch nur diese drei Felder ([project/serializer.py:22-26](algorithms/dataclasses/project/serializer.py#L22-L26)).

Über die API gilt deshalb immer `someone_meets_requirement` (eine Person im Team reicht). Für die Paper-Ergebnisse genügt das, weil *Project Coverage* in Paper 2 genau diese Regel verwendet. Die Varianten „alle“ (`everyone_meets_requirement`) und „n Personen“ (`some_number_of_members_meet_requirement`) sind aber nicht erreichbar.

## 4. Das Schema von `DiversityPriority` erlaubt `max_num_choices` nicht 🔴

**Wo:** Schema in [priority.py:186-194](algorithms/ai/priority_algorithm/priority/priority.py#L186-L194), Prüfung in [priority.py:147-152](algorithms/ai/priority_algorithm/priority/priority.py#L147-L152)

Für `strategy: "concentrate"` verlangt die Klasse das Feld `max_num_choices` und bricht ohne dieses Feld mit einem `ValueError` ab. Das Schema kennt aber nur `attribute_id` und `strategy` und lehnt alle anderen Felder ab.

| Anfrage | Ergebnis |
|---|---|
| `concentrate` ohne `max_num_choices` | Validator ✅, dann `ValueError` in der Klasse |
| `concentrate` mit `max_num_choices` | ❌ Der Validator lehnt ab (Status 400) |

Merkmale konzentrieren, z. B. Studierende mit gemeinsamen Zeitfenstern zusammenlegen, ist über die API deshalb nicht möglich. Paper 2 verwendet genau das in Szenario 2 und 3 sowie bei beiden echten Kursen. `diversify` funktioniert, sobald Bug 1 behoben ist.

## 5. Enum-Werte werden beim Einlesen nicht umgewandelt 🟠

**Wo:** [priority.py:141-145](algorithms/ai/priority_algorithm/priority/priority.py#L141-L145) und [priority.py:226-245](algorithms/ai/priority_algorithm/priority/priority.py#L226-L245)

Die JSON-Daten enthalten Strings (`"diversify"`, `"exclude"`). Die Klassen vergleichen aber mit Enums. Ein Beispiel: `"exclude" == PreferenceDirection.EXCLUDE` ergibt `False` ([priority.py:241](algorithms/ai/priority_algorithm/priority/priority.py#L241)). Ein „Exclude“ würde deshalb stillschweigend wie „Include“ behandelt.

## 6. Freund/Feind-Schalter werden ignoriert 🟠

**Wo:** [algorithm_options.py:69-70](algorithms/ai/interfaces/algorithm_options.py#L69-L70), weitergereicht in [priority_algorithm.py:149-150](algorithms/ai/priority_algorithm/priority_algorithm.py#L149-L150) und [social_algorithm.py:272-273](algorithms/ai/social_algorithm/social_algorithm.py#L272-L273)

`friend_behaviour` und `enemy_behaviour` (`enforce`, `invert`, `ignore`) werden eingelesen und weitergegeben, aber kein Algorithmus liest sie aus. Die Schalter aus dem Dashboard in Paper 1 haben daher keine Wirkung.

## 7. Zwei Methoden im Validator sind doppelt definiert 🟠

**Wo:** [generate_teams_validator.py](algorithms/api/validators/generate_teams_validator.py)

In Python ersetzt eine zweite Methode mit demselben Namen die erste. Die erste Definition wird nie ausgeführt.

| Methode | Erste Definition (wird nie ausgeführt) | Zweite Definition (gilt) | Fehlende Prüfung |
|---|---|---|---|
| `_validate_student_attributes` | [Z. 100-130](algorithms/api/validators/generate_teams_validator.py#L100-L130) | [Z. 193-210](algorithms/api/validators/generate_teams_validator.py#L193-L210) | Ob die Merkmals-Schlüssel Zahlen und eindeutig sind und ob die Merkmale existieren |
| `_validate_student_relationships` | [Z. 132-152](algorithms/api/validators/generate_teams_validator.py#L132-L152) | [Z. 212-228](algorithms/api/validators/generate_teams_validator.py#L212-L228) | Ob die Beziehungs-Schlüssel eindeutig sind |

`_validate_student_ids_unique`, `_validate_student_project_preferences_exist` und `_validate_student_project_preferences` sind ebenfalls doppelt vorhanden, aber beide Versionen sind identisch.

Ob die Prüfung „Merkmal existiert“ überhaupt gewollt ist, ist offen. Sie verlangt, dass jedes Merkmal einer Person auch in einer Projektanforderung vorkommt. Geschlecht ohne passende Anforderung wäre dann nicht erlaubt. Vermutlich wurde die Methode deshalb ersetzt.

## 8. Die Gewichte wirken beim Priority-Algorithmus nur auf die Startlösung 🟠

**Wo:** [priority_algorithm.py:50-56](algorithms/ai/priority_algorithm/priority_algorithm.py#L50-L56) (Startlösung), [priority_algorithm.py:85-93](algorithms/ai/priority_algorithm/priority_algorithm.py#L85-L93) (Auswahl), [custom_dataclasses.py:40-59](algorithms/ai/priority_algorithm/custom_dataclasses.py#L40-L59) (Score)

`PriorityAlgorithmOptions` erbt alle Optionen von `WeightAlgorithmOptions`, z. B. `preference_weight`. Diese Gewichte werden aber nur einmal verwendet, um mit dem Weight-Algorithmus die Startlösung zu erzeugen. Beim Hill Climbing zählen **nur** die `priorities`.

- **Ohne Prioritäten** (`priorities: []`) hat jede Variante den Score 0. Beim Sortieren kommen die neuen Varianten bei gleichem Score vor die alten. Bis zu 1500 Runden lang wird also zufällig getauscht, und die gute Startlösung geht verloren.
- **Mit anderen Prioritäten** passiert dasselbe bei allen Tauschen, die diese Prioritäten nicht verändern. Projektwünsche können sich dabei unbemerkt verschlechtern.

Das ist aus dem Code abgeleitet und noch nicht durch einen Testlauf bestätigt.

## 9. Fehlende Antworten zählen als eigene Kategorie 🟡

**Wo:** [diversity_utility.py:45](algorithms/ai/weight_algorithm/utility/diversity_utility.py#L45) (laut Kommentar im Code so gewollt)

Studierende ohne Antwort (`-1`) werden wie eine eigene Merkmalsausprägung gezählt. Dadurch wirken Teams diverser, als sie sind.

---

## Kein Bug, aber auffällig

- **Robinhood-Mutation** ([robinhood.py:117-151](algorithms/ai/priority_algorithm/mutations/robinhood.py#L117-L151)): Findet sie keine Verbesserung, gibt sie den zuletzt ausprobierten Zustand zurück statt des Ausgangszustands. Das Ergebnis wird dadurch nicht schlechter, weil das beste Team-Set ohnehin erhalten bleibt. Es kostet aber Rechenzeit. Diese Mutation steht nicht im Paper und ist standardmäßig nicht aktiv.
- **Score-Cache** ([custom_dataclasses.py:43](algorithms/ai/priority_algorithm/custom_dataclasses.py#L43)): `if self.score:` behandelt einen berechneten Score von 0 wie „noch nicht berechnet“. Ein Score von 0 wird deshalb jedes Mal neu berechnet. Das Ergebnis bleibt richtig, es kostet nur Rechenzeit.
- **Keine Tests:** Für `parse_json` und `validate_priority` gibt es keine Tests.
