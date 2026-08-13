# Google Analytics MCP Server → n8n

Diese Dokumentation beschreibt die Portierung des in diesem Repository enthaltenen
**Python-MCP-Servers** auf eine **n8n-gehostete, remote erreichbare MCP-Server-Instanz**.

Der Python-Code des Repositories wird dabei **nicht verändert**. Es ist eine Portierung,
kein Patch.

---

## 1. Warum überhaupt portieren

Der Original-Server ist ein **lokaler Prozess mit stdio-Transport**
(`analytics_mcp/server.py:32`) und liest Google-Credentials als **Application Default
Credentials vom Dateisystem** (`analytics_mcp/tools/client.py:83`). Daraus folgt:

- er läuft nur auf dem Rechner, auf dem er installiert ist
- er ist nicht teambar und nicht zentral betreibbar
- er ist aus Automatisierungen (n8n-Workflows, andere Agenten) nicht erreichbar

Ziel der Portierung: derselbe Tool-Katalog, aber als **authentifizierter HTTP-Endpunkt**,
mit den Credentials im Secret-Store der Automatisierungsplattform statt auf einer
Entwicklermaschine.

---

## 2. Analyse des Originals

### 2.1 Architektur

| Aspekt | Befund |
|---|---|
| MCP-Layer | `mcp.server.lowlevel.Server` + Google-ADK `FunctionTool` — **kein FastMCP** |
| Tool-Deklaration | **keine** `@mcp.tool`-Dekoratoren; Schemata werden aus Signatur + Docstring generiert |
| Transport | stdio |
| Auth | ADC, ein Scope: `https://www.googleapis.com/auth/analytics.readonly` |
| Clients | `admin_v1beta`, `admin_v1alpha`, `data_v1beta`, `data_v1alpha` (gRPC/Protobuf) |
| Tool-Anzahl | **9** (`analytics_mcp/coordinator.py:74-84`) |

Die Registrierung patcht anschließend drei ADK-Eigenheiten nachträglich
(`coordinator.py:95-152`): `additionalProperties` wird auf Boolean gezwungen
(Claude-Desktop-Bug), ein leeres `inputSchema` wird ergänzt
(ADK-Bug google/adk-python#948), und `required` wird für drei Report-Tools **manuell**
gesetzt, weil ADK es aus den Defaults nicht korrekt ableitet.

### 2.2 Wo die eigentliche Qualität steckt

Der Code ist schmal. Echte Eigenlogik gibt es nur an drei Stellen:

1. **`construct_property_rn()`** (`tools/utils.py:22-44`) — normalisiert `123`,
   `"123"`, `" 123 "`, `"properties/123"` auf `properties/123`.
2. **Funnel-Schritt-Expansion** (`tools/reporting/funnel.py:146-159`) — akzeptiert je
   Schritt zwei Formen und expandiert die Kurzform `{"event": "purchase"}`.
3. **Custom-Feld-Filter** (`tools/reporting/metadata.py:495-504`) — holt das komplette
   Metadata-Objekt und filtert clientseitig auf `custom_definition == true`.

Der überwiegende Teil des Nutzens liegt dagegen in den **generierten Tool-Descriptions**.
`coordinator.py:58-71` überschreibt die Descriptions der vier Report-Tools mit mehreren
Kilobyte Prompt-Text aus `metadata.py`: Filter-, OrderBy-, DateRange- und
Funnel-Beispiele plus 55 Zeilen Erklärung (`_FILTER_NOTES`, `metadata.py:225-281`), warum
`dimension_filter` und `metric_filter` unabhängig angewendet werden und welche komplexen
Bedingungen deshalb *nicht* in einem Request ausdrückbar sind.

**Wer diese Descriptions beim Nachbau weglässt, bekommt einen Server, der formal
funktioniert, dessen Tools ein LLM aber falsch aufruft.**

### 2.3 Der zentrale Fallstrick: snake_case vs. camelCase

Der Docstring von `run_report` sagt es selbst (`tools/reporting/core.py:97-102`):

> *"the reference docs … all use camelCase field names, but field names passed to this
> method should be in snake_case since the tool is using the protocol buffers (protobuf)
> format."*

Grund: `data_v1beta.DateRange(dr)`, `FilterExpression(...)`, `OrderBy(...)` übergeben
Dicts **positional** an proto-plus-Konstruktoren, die gegen Proto-Feldnamen
(snake_case) auflösen. Ein camelCase-Key führt dort zu `ValueError`.

Die REST-APIs erwarten das Gegenteil: **camelCase**. Diese Portierung spricht REST
(siehe `deviations.md`), toleriert aber snake_case in der Eingabe und schreibt um.

---

## 3. Zielarchitektur in n8n

```
                    ┌─────────────────────────────┐
   MCP-Client  ───▶ │ MCP_GA_00_Server            │
   (Bearer)         │ mcpTrigger v2, bearerAuth   │
                    │ 11 Tools via ai_tool        │
                    └──────────────┬──────────────┘
                                   │  (10× toolWorkflow, 1× toolCode)
                    ┌──────────────▼──────────────┐
                    │ MCP_GA_20 … MCP_GA_29       │
                    │ je Tool ein Sub-Workflow:   │
                    │ Normalize → Gate → Fetch    │
                    │        → Shape → Respond    │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │ MCP_GA_10_Fetch             │
                    │ EINZIGE Stelle mit:         │
                    │ Credential, API-Hosts,      │
                    │ Timeout, Pagination         │
                    └──────────────┬──────────────┘
                                   │
              analyticsdata.googleapis.com  ·  analyticsadmin.googleapis.com
                        v1beta / v1alpha
```

### 3.1 Warum HTTP und nicht der n8n-Google-Analytics-Node

`n8n-nodes-base.googleAnalytics` v2 hat **genau zwei** Operations: `report:get` und
`userActivity:search` (letzteres Universal-Analytics-only und damit funktionslos).

**8 der 9 Original-Tools sind damit nicht baubar** — kein Realtime, keine Admin-API, kein
`getMetadata`, kein Funnel, kein Conversions-Report.

Und selbst `run_report` nicht sinnvoll: `metricsGA4` / `dimensionsGA4` sind
`fixedCollection`-Strukturen. Ein LLM kann die *Anzahl* der Einträge nicht verändern, nur
Werte in zur Designzeit festgelegten Slots füllen. Beliebige Metrik- und
Dimensionslisten sind damit nicht darstellbar.

Deshalb laufen **alle** Aufrufe als REST über einen HTTP-Request-Node mit
`authentication: predefinedCredentialType` und
`nodeCredentialType: googleAnalyticsOAuth2`.

### 3.2 Warum Sub-Workflows statt flacher HTTP-Tool-Nodes

Weil es echte serverseitige Logik zu replizieren gibt (siehe 2.2) und weil Fehler als
**strukturierte Daten** zurückkommen sollen, nicht als rohe Google-Errors:

```json
{
  "ok": false,
  "tool": "run_report",
  "errors": [{
    "error": "invalid_parameter",
    "parameter": "dimensions",
    "received": "10 Dimensionen",
    "allowed": "maximal 9",
    "hint": "Die GA4 Data API erlaubt höchstens 9 Dimensionen pro Request. Teile die Abfrage auf."
  }],
  "applied_filters": { "property_id": "321804543", "...": "..." }
}
```

`applied_filters` ist immer dabei. Damit bleibt „keine Daten gefunden" von „der Parameter
kam nie an" unterscheidbar — eine Unterscheidung, die das Original nicht anbietet.

Das entspricht auch dem Verhalten des Originals auf Protokollebene: `coordinator.py:174-183`
fängt **jede** Exception und gibt sie als `{"error": "..."}` in einer *erfolgreichen*
MCP-Antwort zurück. Ein Tool-Call schlägt dort nie hart fehl.

### 3.3 Die Fetch-Schicht

`MCP_GA_10_Fetch` ist der einzige Ort, der die Credential, die beiden API-Hosts, das
Timeout (120 s statt der 10 s Default — für `runReport` über große Zeiträume zu knapp)
und die Pagination kennt.

Vertrag — Eingaben alle `string`, Rückgabe immer **genau ein Item**:

| Eingabe | Werte |
|---|---|
| `api` | `data` \| `admin` |
| `version` | `v1beta` \| `v1alpha` |
| `path` | Ressourcenpfad ohne führenden Slash, z. B. `properties/321804543:runReport` |
| `method` | `GET` \| `POST` |
| `body_json` | JSON-Objekt als String (nur POST) |
| `query_json` | JSON-Objekt als String → Querystring |

Rückgabe: `{ ok, status, error, reason, message, pages, request, data }`.
`reason` trägt bei Google-Fehlern den `details[].reason` — also `SERVICE_DISABLED`,
`ACCESS_TOKEN_SCOPE_INSUFFICIENT` usw., was die Diagnose entscheidend abkürzt.

**Pagination:** `Fetch GET` hat sie dauerhaft aktiv, mit
`completeExpression: {{ !$response.body.nextPageToken }}`. Fehlt das Feld — etwa bei
`properties/{id}/metadata` — stoppt sie nach dem ersten Request. Für die Admin-Listen
folgt sie allen Seiten, und `Respond` konkateniert jedes Array-Feld. Der Python-Server
materialisiert per GAPIC-Pager ebenfalls alle Seiten; **ohne** das schneidet ein Nachbau
bei mehr als 50 Accounts stillschweigend ab.

Die Data-API-Reports paginieren **nicht** so: v1beta kennt kein `nextPageToken`, dort
läuft es über `limit`/`offset` gegen `rowCount` und bleibt Sache des aufrufenden LLM —
genau wie im Original.

### 3.4 Warum alle Parameter `string` sind

n8n weist bei `workflowInputs` vom Typ `number` ungültige Werte **am Protokollrand** ab.
Der eigene strukturierte Fehler wäre damit unerreichbar und das LLM bekäme eine
n8n-Meldung statt eines verwertbaren Hinweises. Deshalb sind auch Zahl-Parameter
(`limit`, `offset`) als `string` deklariert und werden in `Normalize Args` geprüft.

---

## 4. Betrieb

### Transport: Streamable HTTP am Basispfad

`mcpTrigger` **v2** registriert **Streamable HTTP direkt am Basispfad**. Gemessen:

| Aufruf | Ergebnis |
|---|---|
| `POST <basis>` mit Token | **200**, JSON-RPC-Handshake, `mcp-session-id`-Header |
| `POST <basis>` ohne Token | **403** `Authorization data is wrong!` |
| `GET <basis>/sse` | **404** nicht registriert |
| `POST <basis>/messages` | **404** nicht registriert |

> **Nicht mit dem SSE-Muster verwechseln.** `/sse` + `/messages` gilt für den älteren
> `mcpTrigger` **v1.1**. Wer die v1.1-URLs gegen einen v2-Trigger konfiguriert, bekommt
> 404 mit dem irreführenden Hinweis *„The workflow must be active"* — obwohl der
> Workflow aktiv ist.

Der Server meldet sich mit `protocolVersion 2024-11-05` und
`capabilities.tools`.

- **Alle Callees müssen publiziert sein** — der Server *und* `MCP_GA_10_Fetch` *und* alle
  `MCP_GA_2x_*`. Ein unpublizierter Callee bricht den Tool-Aufruf mit
  *„Workflow is not active and cannot be executed"* ab, ohne den Grund zu nennen.
- **Der Tool-Name ist der Node-Name.** Die elf Tool-Nodes im Server heißen exakt wie die
  Tools des Python-Servers. Ein Node umbenennen benennt das Tool um und bricht jeden
  Client, der es schon kennt.

### Zwei Fallstricke der Tool-Nodes im Server

Beide betreffen nur die `toolWorkflow`-Nodes im Server, nicht die Sub-Workflows — und
beide sind deshalb für den Selbsttest **unsichtbar**.

**1. Zwei aufeinanderfolgende schließende geschweifte Klammern brechen eine
`$fromAI`-Beschreibung.** Die Werte in `workflowInputs` sind n8n-Ausdrücke der Form
`={{ … $fromAI('name', 'beschreibung') … }}`. Enthält die Beschreibung ein JSON-Beispiel,
das auf `}}` endet, liest n8n dort das **Ende des Ausdrucks**. Der Rest wird Literaltext,
der Ausdruck bleibt unvollständig, und **jeder** Aufruf des Tools antwortet nur noch:

```
There was an error: "invalid syntax"
```

Der Sub-Workflow startet dabei nie. Die Execution gilt trotzdem als *success*, weil der
Fehler als reguläre Tool-Antwort zurückgeht — es gibt also keinen roten Eintrag in der
Executionliste, an dem man es merkt.

Am 12.08.2026 hat genau das `run_report` vollständig blockiert: die Beispiele in
`dimension_filter` und `metric_filter` endeten auf `}}}` bzw. `}}}}`. Der Fix ist, die
schließenden Klammern durch Leerzeichen zu trennen — JSON erlaubt das:

```
{"filter": {"fieldName": "eventName", "stringFilter": {"matchType": "EXACT", "value": "purchase"} } }
```

**2. Der `mcpTrigger` erklärt jeden `$fromAI`-Parameter als `required`.** Ein `tools/call`,
der die optionalen Felder wegläßt, scheitert mit *„Received tool input did not match
expected schema"*. Echte Clients senden sie als Leerstring; `Normalize Args` behandelt
Leerstring wie „nicht gesetzt", deshalb fällt das im Betrieb nicht auf. Wer den Endpunkt
mit einem eigenen Client anspricht, muss **alle** Parameter mitsenden.

### `tools/list` beweist nicht, dass ein Tool funktioniert

Der `run_report`-Defekt hat drei Prüfungen gleichzeitig überlebt:

| Prüfung | Ergebnis trotz kaputtem Tool |
|---|---|
| `MCP_GA_98_Selftest` | 14/14 PASS — er ruft die **Sub-Workflows** direkt auf und sieht die Server-Nodes nicht |
| `tools/list` am Endpunkt | vollständige Liste mit 11 Tools — der Defekt entsteht erst beim Aufruf |
| Executionliste | lauter *success* — der Fehler ist eine Tool-Antwort, kein Fehlschlag |

Nur ein echter **`tools/call`** deckt diese Fehlerklasse auf. `MCP_GA_99_MCP_Probe` führt
ihn seither aus (Node `B2`): `initialize` → `notifications/initialized` → `tools/call` auf
`run_report`, mit der `mcp-session-id` aus der `initialize`-Antwort.

### Quoten

Standard-Property: 200 000 Tokens/Tag, 40 000/Stunde, 10 gleichzeitige Requests.
Analytics 360 jeweils das Zehnfache. Ein typischer Report kostet unter 10 Tokens.
`return_property_quota: "true"` liefert die Restkontingente in der Antwort.

`_FILTER_NOTES` im Original rät explizit: **wenige breite Reports plus clientseitiges
Nachfiltern** statt vieler kleiner Requests.

---

## 5. Weiterführend

- [`tool-mapping.md`](tool-mapping.md) — die 11 Tools → REST-Endpunkt → n8n-Workflow,
  plus die snake_case ↔ camelCase-Feldmatrix
- [`deviations.md`](deviations.md) — jede bewusste Abweichung vom Python-Server mit
  Begründung
- [`blockbrain.md`](blockbrain.md) — Anbindung an Blockbrain als MCP-Client, inklusive
  der Diagnose des irreführenden Verbindungstests
- [`agent-core-prompt.md`](agent-core-prompt.md) — der Core-Prompt des Google-Analytics-Agents,
  kanonische Fassung mit Begründung und Testfragen
- [`workflows/README.md`](workflows/README.md) — Inventar der erzeugten Workflows
