# Bewusste Abweichungen vom Python-Server

Diese Liste existiert, damit später nachvollziehbar bleibt, was Absicht war und was
Zufall. Alles hier ist eine **Entscheidung**, keine Ungenauigkeit.

---

## A. Formatabweichung

### A1 Responses in camelCase statt snake_case

Das Original serialisiert Antworten mit
`proto_to_dict(..., preserving_proto_field_name=True)` (`tools/utils.py:47-51`) und
liefert damit **snake_case** (`dimension_headers`, `row_count`, `property_quota`).
Diese Portierung spricht REST und liefert **camelCase** (`dimensionHeaders`, `rowCount`,
`propertyQuota`).

*Begründung:* bewusst gewählt. Eine Rückkonvertierung camel→snake wäre eine zusätzliche
Fehlerquelle in jedem Tool, und camelCase ist genau das Format, das ein LLM aus Googles
offizieller REST-Doku kennt.

*Konsequenz:* ein Client, der gegen die snake_case-Ausgabe des Python-Servers gebaut
wurde, muss seine Feldzugriffe anpassen. Die Tool-**Namen** und die
Parameter-**Namen** bleiben identisch, nur verschachtelte Payloads und Antworten
unterscheiden sich.

*Eingaben bleiben tolerant:* `start_date`, `field_name`, `and_group` usw. werden
akzeptiert und rekursiv nach camelCase umgeschrieben.

### A2 Zwei handgeschriebene Response-Keys umbenannt

`get_custom_dimensions_and_metrics` liefert im Original `custom_dimensions` /
`custom_metrics` (`metadata.py:505-508`) — hier `customDimensions` / `customMetrics`,
konsistent zu A1.

### A3 Antwort-Envelope

Jedes Tool antwortet in einem Envelope statt als nackter Wert:

```json
{ "ok": true, "tool": "<name>", "applied_filters": {...}, "<payload>": ... }
```

*Begründung:* Voraussetzung für „Fehler sind Daten" (B-Abschnitt). `applied_filters`
macht „nichts gefunden" von „Parameter kam nie an" unterscheidbar.

---

## B. Korrigierte Defekte des Originals

### B1 `run_realtime_report` akzeptiert kein `offset` mehr

`RunRealtimeReportRequest` hat **kein** `offset`-Feld. Das Original setzt es dennoch
(`realtime.py:158-159`); jeder Aufruf mit `offset` läuft dort in einen `AttributeError`,
der als `{"error": ...}` beim LLM landet.

*Fix:* Der Parameter ist gar nicht deklariert.

### B2 Realtime-Description ohne `date_ranges`-Hints

Die Realtime-Description des Originals enthält `date_ranges`-Beispiele
(`realtime.py:65-66`), obwohl das Tool keinen solchen Parameter hat — ein
Copy-Paste-Artefakt, das ein LLM zu ungültigen Aufrufen verleitet.

*Fix:* entfernt. Die Description nennt stattdessen das feste ~30-Minuten-Fenster.

### B3 Keine Verweise auf nicht existierende Tools

Die Docstrings verweisen auf **sieben Tools, die es nicht gibt**:
`get_standard_dimensions`, `get_dimensions`, `get_standard_metrics`, `get_metrics`
(`core.py:117-124`), `run_report_dimension_filter_hints`,
`run_report_metric_filter_hints`, `run_report_order_bys_hints`
(`realtime.py:107-123`). Ein LLM ruft sie auf und bekommt
*„Tool 'x' not implemented by this server."* (`coordinator.py:185-187`).

*Fix:* Die Verweise zeigen auf real existierende Tools —
`get_custom_dimensions_and_metrics`, `ga4_hints`, `check_compatibility`. Die drei
`*_hints`-Verweise waren offenbar für genau so ein Tool gedacht; `ga4_hints` erfüllt
diese Ankündigung nun.

### B4 Falsche Funnel-Keys werden gemeldet statt ignoriert

Das Original prüft `if funnel_breakdown and "breakdown_dimension" in funnel_breakdown`
(`funnel.py:173`) und ignoriert sonst **stillschweigend**. Wer
`{"dimension": "deviceCategory"}` schickt, bekommt einen Funnel *ohne* Breakdown und
merkt es nicht. Gleiches für `next_action_dimension` (`funnel.py:180`).

*Fix:* `invalid_parameter` mit Nennung des erwarteten Keys. Abgedeckt durch
Selbsttest-Fall C12.

### B5 Admin-Listen paginieren vollständig

Das Original iteriert die GAPIC-Pager komplett durch. Ein naiver REST-Nachbau würde bei
mehr als 50 Einträgen (Default `pageSize`) stillschweigend abschneiden.

*Fix:* `MCP_GA_10_Fetch` folgt allen `pageToken`-Seiten und konkateniert jedes
Array-Feld. `pages` in der Antwort sagt, wie viele Seiten geholt wurden.

---

## C. Ergänzungen ohne Original-Äquivalent

### C1 `check_compatibility` (Tool 10)

`POST DA/v1beta/{P}:checkCompatibility`. Prüft eine Dimensions-/Metriken-Kombination
vorab, ohne Report-Quota zu verbrauchen.

*Begründung:* Das Original hat kein Tool, mit dem ein Agent Standardfelder prüfen kann.
Erfundene Feldnamen fallen dort erst beim Report auf und kosten Tokens plus Quota.

*Verdichtung:* Die API liefert **alle** passenden Felder — bei der Referenz-Property
372 Dimensionen und 81 Metriken. Das unverändert an ein LLM zu geben wäre teuer und
nutzlos. Zurückgegeben werden ein Urteil je angefragtem Feld, ein
`allRequestedCompatible`-Boolean, eine `unresolved`-Liste und je 40 Beispielnamen.

### C2 `ga4_hints` (Tool 11)

`toolCode` mit echtem JSON-Schema-`enum` über `topic`. Liefert die Filter-, OrderBy-,
DateRange- und Funnel-Beispiele plus `_FILTER_NOTES` — übersetzt nach camelCase.

*Begründung:* Diese Texte sind mehrere Kilobyte. Im Original stehen sie in den
Tool-Descriptions und landen bei **jedem** `tools/list` im Kontext. Als abrufbares Tool
werden sie nur bezahlt, wenn sie gebraucht werden. Außerdem versprechen die
Original-Docstrings ohnehin solche Tools (siehe B3).

### C3 `conversionActions` in `get_custom_dimensions_and_metrics`

Zusätzlich zu `DA/v1beta/{P}/metadata` wird `DA/v1alpha/{P}/metadata` abgefragt und
dessen `conversions`-Array als `conversionActions` durchgereicht.

*Begründung:* Das Original fragt nur v1beta ab, wo dieses Feld **nicht existiert**.
`run_conversions_report` kann dort deshalb praktisch nur `conversion_actions: []`
(= alle) sinnvoll füllen. Bei der Referenz-Property liefert die Ergänzung sofort eine
echte ID.

*Fehlertoleranz:* Schlägt der v1alpha-Aufruf fehl, bleibt das Tool `ok: true` und setzt
`conversionsNote`. Ein Alpha-Problem darf die Custom-Felder nicht mitkippen.

### C4 API-Limits werden vorab geprüft

9 Dimensionen, 10 Metriken, 4 `date_ranges`, `limit` ≤ 250 000, 4 `segments`,
`funnel_next_action.limit` 1–5. Das Original prüft nichts davon.

### C5 Conversions-Allowlists werden erzwungen

Die 12 erlaubten Dimensionen und 12 erlaubten Metriken stehen im Original **nur als
Prosa** in der Description (`conversions.py:44-74`) und werden nirgends validiert.
Hier sind sie harte Prüfungen. Abgedeckt durch Selbsttest-Fall C13.

### C6 Realtime-Regeln werden erzwungen

Custom Metrics sind in Realtime verboten, Custom Dimensions nur user-scoped
(`customUser:`). Das Original dokumentiert das nur im Text und lässt die API scheitern.

### C7 Leere Ergebnisse werden benannt

Liefert ein Tool legitim eine leere Liste oder null Zeilen, wird ein `note`-Feld
ergänzt („der Aufruf war erfolgreich, das Ergebnis ist echt leer"). Betrifft
`list_google_ads_links`, `list_property_annotations`, `run_realtime_report`,
`run_conversions_report`.

*Begründung:* Das Original gibt ein nacktes `[]` zurück — ein LLM kann daran nicht
erkennen, ob nichts existiert oder ob die Berechtigung fehlt. Das ist die direkte
Anwendung der Regel „strukturierte Antwort statt leerer Liste".

### C8 Statusabhängige `hint`-Felder

Bei v1alpha-Tools (`list_property_annotations`, `run_funnel_report`,
`run_conversions_report`) wird bei 400/403/404 ein `hint` ergänzt, der einen
Alpha-/Verfügbarkeitsvorbehalt von einem Auth-Problem unterscheidet.

Bei `run_conversions_report` entscheidet der **Text der Google-Meldung**, welcher Hinweis
greift: enthält er „incompatible", zeigt der Hinweis auf `check_compatibility`, sonst auf
die Feature-Verfügbarkeit. Diese Unterscheidung wurde nach einem Live-Test nachgezogen,
bei dem ein pauschaler Verfügbarkeitshinweis eine reine Metrik-Inkompatibilität
falsch erklärt hätte.

### C9 `get_account_summaries` liefert Zähler mit

Zusätzlich zum unveränderten `accountSummaries`-Array gibt das Tool aus:

```json
{ "accounts": 3, "properties": 14,
  "propertiesPerAccount": [ { "account": "accounts/…", "displayName": "…", "properties": 11 } ] }
```

Die Description sagt dem aufrufenden Modell ausdrücklich, dass es **diese Zahlen
verwenden** soll statt selbst zu zählen.

*Begründung:* nachgezogen nach einem gemessenen Fehlerfall. Ein echter MCP-Client hat am
2026-08-12 alle 14 Properties korrekt aufgelistet, sie in seiner Zusammenfassung aber
als „13 Properties" bezeichnet — er musste selbst zählen und hat sich verzählt. Das
Original liefert keine Anzahl, und `accounts` allein deckt den häufigsten Fall („wie
viele Properties habe ich?") nicht ab.

Abgeleitete Felder wurden vorher bewusst vermieden, um die Signaturparität zum
Python-Server zu halten. Hier liegt ein konkreter Fehler vor, der genau aus dieser
Parität entsteht — das rechtfertigt die Abweichung. Das `accountSummaries`-Array selbst
bleibt **unverändert** 1:1, die Zähler kommen daneben.

---

## D. Strenger als das Original

### D1 `property_id`-Prüfung

`construct_property_rn()` akzeptiert im Original auch Werte, die praktisch immer Fehler
sind:

| Eingabe | Original | Diese Portierung |
|---|---|---|
| `-5` (int) | `properties/-5` | `invalid_parameter` |
| `True` (bool) | `properties/1` | `invalid_parameter` |
| `"１２３"` (Fullwidth) | `properties/123` | `invalid_parameter` |
| `"properties/123/456"` | `properties/456` ⚠️ | `invalid_parameter` |

Geprüft wird strikt gegen `^\d+$`. Führende Nullen werden wie im Original entfernt
(`"00012345"` → `properties/12345`).

Der letzte Fall ist im Original ein stiller Fehlgriff: `split("/")[-1]` nimmt das letzte
Segment und liefert eine **andere** Property als angegeben.

---

## E. Nicht portiert

### E1 `prevent_stdio_inheritance()`

`tools/client.py:59-75` patcht `subprocess.Popen`, damit der von `google.auth.default()`
gestartete `gcloud` nicht die stdio-Handles des MCP-Transports erbt — ein
Windows-Deadlock-Workaround. Ohne stdio-Transport und ohne ADC gegenstandslos.

### E2 Custom User-Agent

Das Original sendet `analytics-mcp/<version>` (`client.py:45-47`). Funktional nicht
erforderlich; nicht übernommen.

### E3 Quota-Project-Header

Das Original setzt `x-goog-user-project` implizit über die ADC-Datei. Bei einer
n8n-OAuth-Credential entfällt das; die Quota wird dem Projekt des OAuth-Clients
zugerechnet.

### E4 Die vom Original ebenfalls nicht exponierten API-Features

Unverändert nicht verfügbar, um die Signatur identisch zu halten: `cohortSpec`,
`comparisons`, `metricAggregations`, `keepEmptyRows`, `dimensionExpression`,
`metric.expression`, `minuteRanges` (Realtime), `isOpenFunnel`,
`funnelVisualizationType`, `withinDurationFromPriorStep`, `isDirectlyFollowedBy`, sowie
der `filter`-Query-Parameter der Annotations-Liste.

Anmerkung: `withinDurationFromPriorStep` und `isDirectlyFollowedBy` werden im Original
**verworfen, selbst wenn sie übergeben werden** (`funnel.py:161-163` baut jeden Schritt
nur aus `name` und `filter_expression`).

---

## F. Nicht umgesetzt

### F1 Metadata-Cache

Erwogen und **verworfen**. `get_custom_dimensions_and_metrics` holt bei jedem Aufruf das
komplette Metadata-Objekt (mehrere Hundert KB). Ein Cache mit TTL wäre möglich, hätte
aber Invalidierungsfragen aufgeworfen, die den Nutzen nicht rechtfertigen.

### F2 SDK-Quellen der Workflows

Die validierten n8n-Workflow-SDK-Quellen sind **nicht** in diesem Repository abgelegt.
Die live erzeugten Workflows sind die Quelle der Wahrheit; siehe
[`workflows/README.md`](workflows/README.md) für das Inventar und den Export-Weg.
