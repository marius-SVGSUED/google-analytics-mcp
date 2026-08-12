# Tool-Mapping: Python → REST → n8n

`DA` = `https://analyticsdata.googleapis.com`
`AD` = `https://analyticsadmin.googleapis.com`
`{P}` = auf `properties/<ziffern>` normalisierte Property-ID

> **`{P}` gehört immer in den URL-Pfad, nie in den Body.** Der häufigste Nachbau-Fehler
> ist ein doppeltes Präfix: `.../properties/properties/123:runReport` → 400.

---

## Die 11 Tools

| # | Tool | Quelle im Repo | HTTP | Endpunkt | n8n-Workflow |
|---|---|---|---|---|---|
| 1 | `get_account_summaries` | `admin/info.py:31-38` | GET | `AD/v1beta/accountSummaries` | `MCP_GA_20_get_account_summaries` |
| 2 | `get_property_details` | `admin/info.py:62-78` | GET | `AD/v1beta/{P}` | `MCP_GA_21_get_property_details` |
| 3 | `list_google_ads_links` | `admin/info.py:41-59` | GET | `AD/v1beta/{P}/googleAdsLinks` | `MCP_GA_22_list_google_ads_links` |
| 4 | `list_property_annotations` | `admin/info.py:81-110` | GET | `AD/v1alpha/{P}/reportingDataAnnotations` | `MCP_GA_23_list_property_annotations` |
| 5 | `get_custom_dimensions_and_metrics` | `reporting/metadata.py:477-508` | GET | `DA/v1beta/{P}/metadata` **+** `DA/v1alpha/{P}/metadata` | `MCP_GA_24_get_custom_dimensions_and_metrics` |
| 6 | `run_report` | `reporting/core.py:82-176` | POST | `DA/v1beta/{P}:runReport` | `MCP_GA_25_run_report` |
| 7 | `run_realtime_report` | `reporting/realtime.py:80-165` | POST | `DA/v1beta/{P}:runRealtimeReport` | `MCP_GA_26_run_realtime_report` |
| 8 | `run_funnel_report` | `reporting/funnel.py:85-199` | POST | `DA/v1alpha/{P}:runFunnelReport` | `MCP_GA_27_run_funnel_report` |
| 9 | `run_conversions_report` | `reporting/conversions.py:101-190` | POST | `DA/v1alpha/{P}:runReport` | `MCP_GA_28_run_conversions_report` |
| 10 | `check_compatibility` | — *(Ergänzung)* | POST | `DA/v1beta/{P}:checkCompatibility` | `MCP_GA_29_check_compatibility` |
| 11 | `ga4_hints` | — *(Ergänzung)* | — | rein lokal, kein Netzwerk | `toolCode` im Server |

Zwei Endpunkte sind leicht zu verwechseln:

- **Tool 8** hat einen eigenen Endpunkt (`:runFunnelReport`) und existiert **nur** auf
  v1alpha. Es gibt kein `v1beta:runFunnelReport`.
- **Tool 9** hat **keinen** eigenen Endpunkt. Es ist `v1alpha:runReport` mit gesetztem
  `conversionSpec` — derselbe Methodenname wie Tool 6, aber andere Version.

---

## Pagination — zwei völlig verschiedene Modelle

| Fläche | Modell | Wer paginiert |
|---|---|---|
| Admin-Listen (Tools 1, 3, 4) | `pageSize` / `pageToken` → `nextPageToken` | `MCP_GA_10_Fetch`, automatisch, alle Seiten |
| Data-API-Reports (Tools 6, 7, 9) | `limit` / `offset` gegen `rowCount` | das aufrufende LLM — v1beta hat kein Page-Token |

---

## Pflichtparameter

| Tool | Pflicht |
|---|---|
| `get_account_summaries` | – |
| `get_property_details` | `property_id` |
| `list_google_ads_links` | `property_id` |
| `list_property_annotations` | `property_id` |
| `get_custom_dimensions_and_metrics` | `property_id` |
| `run_report` | `property_id`, `date_ranges`, `dimensions`, `metrics` |
| `run_realtime_report` | `property_id`, `dimensions`, `metrics` |
| `run_funnel_report` | `property_id`, `funnel_steps` |
| `run_conversions_report` | `property_id`, `date_ranges`, `dimensions`, `metrics`, `conversion_spec` |
| `check_compatibility` | `property_id` + mindestens eines von `dimensions` / `metrics` |
| `ga4_hints` | `topic` |

---

## API-Limits (vor dem Call geprüft)

| Limit | Wert |
|---|---|
| Dimensionen pro Request | **9** |
| Metriken pro Request | **10** |
| `date_ranges` pro Request | **4** |
| `limit` | ≤ **250 000** (Default 10 000) |
| `segments` (Funnel) | **4** |
| `funnel_next_action.limit` | **1–5** |

Das Original prüft **keines** davon — dort kostet jede Überschreitung einen
API-Roundtrip und ein `INVALID_ARGUMENT`.

---

## Feldmatrix snake_case ↔ camelCase

Diese Portierung sendet und liefert **camelCase**, toleriert aber snake_case in der
Eingabe und schreibt rekursiv um.

| Original (protobuf) | REST / diese Portierung |
|---|---|
| `date_ranges` | `dateRanges` |
| `start_date` / `end_date` | `startDate` / `endDate` |
| `dimension_filter` / `metric_filter` | `dimensionFilter` / `metricFilter` |
| `field_name` | `fieldName` |
| `string_filter` / `numeric_filter` | `stringFilter` / `numericFilter` |
| `in_list_filter` / `empty_filter` / `between_filter` | `inListFilter` / `emptyFilter` / `betweenFilter` |
| `match_type` / `case_sensitive` | `matchType` / `caseSensitive` |
| `and_group` / `or_group` / `not_expression` | `andGroup` / `orGroup` / `notExpression` |
| `int64_value` / `double_value` | `int64Value` / `doubleValue` |
| `from_value` / `to_value` | `fromValue` / `toValue` |
| `order_bys` | `orderBys` |
| `dimension_name` / `metric_name` / `order_type` | `dimensionName` / `metricName` / `orderType` |
| `currency_code` | `currencyCode` |
| `return_property_quota` | `returnPropertyQuota` |
| `filter_expression` | `filterExpression` |
| `funnel_event_filter` / `funnel_field_filter` | `funnelEventFilter` / `funnelFieldFilter` |
| `event_name` | `eventName` |
| `funnel_parameter_filter_expression` | `funnelParameterFilterExpression` |
| `event_parameter_name` | `eventParameterName` |
| `conversion_spec` | `conversionSpec` |
| `conversion_actions` / `attribution_model` | `conversionActions` / `attributionModel` |
| **Response:** `dimension_headers` / `metric_headers` | `dimensionHeaders` / `metricHeaders` |
| **Response:** `dimension_values` / `metric_values` | `dimensionValues` / `metricValues` |
| **Response:** `row_count` / `property_quota` | `rowCount` / `propertyQuota` |
| **Response:** `funnel_table` / `funnel_visualization` | `funnelTable` / `funnelVisualization` |
| **Response:** `display_name` / `property_summaries` | `displayName` / `propertySummaries` |
| **Response:** `custom_definition` / `api_name` / `ui_name` | `customDefinition` / `apiName` / `uiName` |

### Zwei Ausnahmen von der reinen Namensregel

**`funnel_breakdown` / `funnel_next_action` sind nicht bloß umbenannt, sondern
verschachtelt.** Das Tool nimmt eine flache Dimension, die API braucht ein
Dimension-Objekt:

```
Eingabe : {"breakdown_dimension": "deviceCategory"}
Gesendet: {"breakdownDimension": {"name": "deviceCategory"}}
```

**Datumsformate unterscheiden sich je Fläche.** `annotationDate` (Tool 4) ist ein
`google.type.Date`-Objekt `{year, month, day}` — nicht der `YYYY-MM-DD`-String, den
`dateRanges` verwendet.

---

## Werte sind Strings

In jeder Report-Antwort sind **alle** `dimensionValues[].value` und
`metricValues[].value` Strings — auch Ganzzahlen und Fließkommazahlen.
`metricHeaders[].type` sagt, wie zu parsen ist: `TYPE_INTEGER`, `TYPE_FLOAT`,
`TYPE_SECONDS`, `TYPE_CURRENCY` usw.

Das gilt auch für das `date`-Dimensionsformat: `"20260805"`, nicht `"2026-08-05"`.

---

## Empfohlener Aufruf-Flow

```
1. get_account_summaries                    → Property-IDs entdecken
2. get_property_details                     → timeZone, currencyCode, serviceLevel
3. get_custom_dimensions_and_metrics        → gültige customEvent:/customUser:-Felder
                                              + conversionActions-IDs
4. check_compatibility                      → Feldkombination absichern (billig)
5. ga4_hints                                → Filter-/OrderBy-/Funnel-Syntax nachschlagen
6. run_report | run_realtime_report | run_funnel_report | run_conversions_report
7. list_property_annotations                → Ausreißer im Zeitverlauf erklären
   list_google_ads_links                    → prüfen, ob Ads-Daten existieren können
```

Ohne Schritt 3 halluziniert ein LLM regelmäßig Custom-Feldnamen. Ohne Schritt 4 sind
`pageViews` (korrekt: `screenPageViews`) und `users` (korrekt: `totalUsers` bzw.
`activeUsers`) die häufigsten Fehlgriffe.
