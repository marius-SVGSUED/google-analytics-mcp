# Workflow-Inventar

Alle Workflows liegen im n8n-Projekt des Betreibers im Ordner
`MCP_CustomServers / MCP_GoogleAnalytics`.

## Erzeugte Workflows

| Workflow | Rolle | Nodes |
|---|---|---|
| `MCP_GA_00_Server` | MCP-Endpunkt: `mcpTrigger` v2, `authentication: bearerAuth`, 11 Tools via `ai_tool` | 13 |
| `MCP_GA_10_Fetch` | Gemeinsame HTTP-Schicht — **einzige** Stelle mit Google-Credential, API-Hosts, Timeout, Pagination | 11 |
| `MCP_GA_20_get_account_summaries` | Tool 1 | 5 |
| `MCP_GA_21_get_property_details` | Tool 2 | 9 |
| `MCP_GA_22_list_google_ads_links` | Tool 3 | 9 |
| `MCP_GA_23_list_property_annotations` | Tool 4 (Admin v1alpha) | 9 |
| `MCP_GA_24_get_custom_dimensions_and_metrics` | Tool 5 (zwei Metadata-Abrufe) | 10 |
| `MCP_GA_25_run_report` | Tool 6 | 9 |
| `MCP_GA_26_run_realtime_report` | Tool 7 | 9 |
| `MCP_GA_27_run_funnel_report` | Tool 8 (Data v1alpha) | 9 |
| `MCP_GA_28_run_conversions_report` | Tool 9 (Data v1alpha) | 9 |
| `MCP_GA_29_check_compatibility` | Tool 10 (Ergänzung) | 9 |
| `MCP_GA_98_Selftest` | 14 Fälle über alle Tools, davon 5 Negativfälle | 17 |
| `MCP_GA_99_AuthProbe` | Diagnose: 11 HTTP-Probes über alle GA-API-Flächen | 14 |

`ga4_hints` (Tool 11) ist ein `toolCode`-Node **im Server** und hat keinen eigenen
Sub-Workflow.

## Aufbau eines Tool-Sub-Workflows

Alle `MCP_GA_2x_*` folgen demselben Muster:

```
Start (executeWorkflowTrigger)
  → Normalize Args (code)      Validierung, property_id-Normalisierung,
  → Args OK? (if)              snake→camel, Limitprüfungen
      ├── true  → Fetch (executeWorkflow → MCP_GA_10_Fetch, waitForSubWorkflow)
      │            → Respond (code)
      └── false → Return Error (code)

Manual Test (manualTrigger) → Test Inputs (set) → Normalize Args
```

Der `Manual Test`-Pfad existiert, weil `test_workflow` HTTP-Nodes pinnt. Ein echter
Live-Test läuft deshalb über diesen Trigger.

## Export

Die live erzeugten Workflows sind die Quelle der Wahrheit. Export-Wege:

- **n8n-UI:** Workflow öffnen → Menü → *Download* liefert die Workflow-JSON
- **n8n-API / MCP:** `get_workflow_details` je Workflow-ID

Die SDK-Quellen sind absichtlich nicht mitversioniert — siehe
[`../deviations.md`](../deviations.md), Abschnitt F2.

## Wiederherstellung

Jeder Workflow trägt eine Versionshistorie mit sprechenden Versionsnamen. Über
`get_workflow_history` und `restore_workflow_version` lässt sich jeder Stand
zurückholen, inklusive der dokumentierten Korrekturschritte
(z. B. „Distinguish incompatibility from availability in 400 hint").

## Abhängigkeitsreihenfolge beim Publizieren

```
MCP_GA_10_Fetch          zuerst
MCP_GA_20 … MCP_GA_29    dann
MCP_GA_00_Server         zuletzt
```

Ein unpublizierter Callee bricht den Tool-Aufruf mit *„Workflow is not active and cannot
be executed"* ab, ohne den Grund zu nennen.
