# Anbindung an Blockbrain

Dieses Dokument hält fest, wie der n8n-gehostete GA-MCP-Server aus Blockbrain heraus
genutzt wird — und vor allem die Diagnose, die dabei teuer erarbeitet wurde. Der
größte Teil des Aufwands ging nicht in die Konfiguration, sondern in die Widerlegung
einer falschen Fehlermeldung.

---

## 1. Konfiguration, die funktioniert

| Feld in Blockbrain | Wert |
|---|---|
| Server URL | die **Production-URL des `mcpTrigger`**, ohne Suffix — kein `/sse`, kein `/messages` |
| Authentifizierung | **API Key** |
| Auth Header Name | `Authorization` |
| Auth Header Template | `Bearer {{apiKey}}` |
| API Key | der **reine** Token, ohne `Bearer `-Präfix |

Das Template setzt das Präfix selbst. Steht `Bearer` zusätzlich im Key-Feld, sendet
Blockbrain dauerhaft `Bearer Bearer <token>` und n8n weist mit 403 ab.

n8n-seitig gehört derselbe Token in eine `httpBearerAuth`-Credential, die am
`mcpTrigger` unter `authentication: bearerAuth` hängt.

Dass die Blockbrain-UI nach dem Speichern **„SSE"** als Transport anzeigt, ist ohne
Bedeutung — der tatsächliche Aufruf ist Streamable HTTP am Basispfad, passend zu
`mcpTrigger` v2 (siehe [`README.md`](README.md), Abschnitt 4).

---

## 2. Der Testknopf lügt

**Befund:** *Test Streaming Connection* schlägt fehl, obwohl der Zugang funktioniert.

```
Connection Failed
Failed to connect to MCP server: Streamable HTTP error:
Error POSTing to endpoint: Authorization data is wrong!
```

Ursache: der Client hinter diesem Knopf identifiziert sich als
`blockbrain-connection-tester/1.0.0` und sendet **überhaupt keinen
`Authorization`-Header**. Dreimal gemessen, jedes Mal identisch. Die **echte
Agent-Laufzeit** von Blockbrain sendet ihn korrekt — der erste reale Prompt hat
`get_account_summaries` fehlerfrei aufgerufen.

> **Ein roter Verbindungstest ist kein Beweis für einen kaputten Zugang.** Bei einem
> mit Bearer-Auth abgesicherten `mcpTrigger` ist er das erwartete Verhalten. Verifiziert
> wird die Integration mit einem echten Prompt, nicht mit dem Knopf.

---

## 3. Warum n8n's 403 nichts unterscheidet

`Authorization data is wrong!` ist n8n's wörtliche Ablehnung. Der Grund ist im
Quellcode nur eine Gleichheitsprüfung — für einen **fehlenden** und einen **falschen**
Header identisch:

```ts
// packages/nodes-base/nodes/Webhook/utils.ts
if (headers.authorization !== `Bearer ${expectedToken}`) {
    throw new WebhookAuthorizationError(403);   // → "Authorization data is wrong!"
}
```

Der `bearerAuth`-Zweig hat **kein** separates 401 für „Header fehlt". Damit sagt die
Meldung genau nichts darüber, welche der beiden Ursachen vorliegt — und das war der
Grund, weshalb die Diagnose so lange gedauert hat.

Was sie immerhin ausschließt:

- **Der Pfad ist korrekt.** Ein unbekannter Pfad ergibt auf dieser Instanz
  `404 The requested webhook "…" is not registered`. Ein Auth-Fehler entsteht erst,
  *nachdem* der Webhook aufgelöst wurde.
- **Der Reverse Proxy entfernt den Header nicht.** Ein gewöhnlicher HTTP-Client bekam
  über dieselbe öffentliche URL einen vollständigen Handshake (200, `mcp-session-id`).
- **Es ist kein Credential-Entschlüsselungsproblem.** Das ergäbe **500**
  (`No authentication data defined on node!`), nicht 403.

Zweite Falle bei der Fehlersuche: **eine am Webhook-Layer mit 403 abgewiesene Anfrage
erzeugt keine Execution.** In den n8n-Executions steht von den fehlgeschlagenen
Blockbrain-Versuchen nichts — beide MCP-Server-Workflows hatten null Executions,
während `mode: webhook` auf dieser Instanz sonst protokolliert wird. Es gibt also kein
Log, in dem man nachsehen könnte.

---

## 4. Wie man es auseinanderhält: Mitschnitt-Webhook

Da die Meldung ambivalent ist und keine Execution entsteht, bleibt nur, den
eingehenden Header selbst zu messen. Vorgehen:

1. Temporärer Workflow mit `n8n-nodes-base.webhook`, **eigener Zufallspfad**,
   `authentication: none`, nimmt GET und POST an.
2. Ein `code`-Node gibt einen **Fingerprint** aus, nicht den Token: Länge,
   „beginnt mit `Bearer `", Länge des Tokenteils, erste und letzte 6 Zeichen,
   Zeichencodes des ersten und letzten Zeichens (deckt Whitespace auf), Anzahl der
   `Bearer`-Vorkommen, sowie die Liste **aller** eingegangenen Headernamen.
3. **Zubringer A:** ein `httpRequest`-Node mit der fraglichen Bearer-Credential ruft
   diesen Webhook auf → zeigt, was **n8n gespeichert** hat.
4. **Zubringer B:** die Webhook-URL kurzzeitig als Server-URL in Blockbrain eintragen
   und den Test auslösen → zeigt, was **Blockbrain sendet**.

Zwei Fingerprints, ein Vergleich. Das Ergebnis war eindeutig: A lieferte
`header_length 57` (= `Bearer ` + 50), ein einziges `Bearer`, kein Whitespace, kein
`__n8n_BLANK_VALUE`. B lieferte **kein `authorization` in der Headerliste** — und
`user-agent: blockbrain-connection-tester/1.0.0`.

Wichtig für die Aussagekraft: den API Key in Blockbrain vor der Messung **nicht** neu
eintragen, sonst ist das Beweismaterial weg.

**Danach aufräumen.** Ein Webhook ohne Auth, der Header mitschneidet, darf nicht
dauerhaft offenstehen. Der Zufallspfad begrenzt das Risiko, ersetzt aber kein
Unpublish + Archivieren. Der Mitschnitt-Workflow dieser Sitzung
(`MCP_GA_97_TokenFingerprint`) ist archiviert, der auf ihn zeigende Probe-Node wurde
aus `MCP_GA_99_MCP_Probe` entfernt.

---

## 5. Blockbrain lädt nur die Top 5 von 82 Tools

Blockbrain stellt einem Agenten nicht alle registrierten MCP-Tools zur Verfügung,
sondern wählt per **Keyword-Relevanz** eine Teilmenge aus. Im ersten echten Aufruf
stand in der Antwort wörtlich:

> *only the top 5 by keyword relevance out of 82 available tools*

Gezogen wurden `get_account_summaries`, `get_property_details`,
`list_google_ads_links`, `run_realtime_report`, `ga4_hints`. **`run_report` war nicht
dabei** — für die gestellte Frage („Welche Properties habe ich?") korrekt, aber es
zeigt das Prinzip.

**Die Folge für die Sprache der Beschreibungen.** Die Tool-Descriptions dieser
Portierung sind englisch (wie im Original), die Prompts des Nutzers sind deutsch.
„Google Analytics" traf wörtlich; *„Wie viele Sitzungen hatte ich letzte Woche?"*
trifft `run_report` nicht, weil in der Description `sessions` steht und nicht
`Sitzungen`.

Deshalb trägt **jedes** Tool im Server am Ende seiner Description eine Zeile
`Deutsche Stichwörter: …`. Die englische Anleitung bleibt vollständig und steht vorn —
die Schlagwörter kommen hinten dran und dienen nur dem Retrieval. `run_report` hat die
längste Liste, weil es das Tool ist, das für Berichtsfragen zwingend gefunden werden
muss.

Wer diesen Server unter einer anderen Prompt-Sprache betreibt, muss diese Zeile
anpassen — sie ist eine Retrieval-Maßnahme, keine fachliche Dokumentation.

---

## 6. Zwei Nebenbeobachtungen

### Blockbrain präfixt die Tool-Namen

Im Blockbrain-Transkript heißen die Tools
`google_analytics_mcp_7b8ff0_get_account_summaries` usw. Die Regel
**„Node-Name = Tool-Name"** bleibt auf MCP-Protokollebene gültig — `tools/list` liefert
weiterhin `get_account_summaries`. Blockbrain überlagert die Namen nur clientseitig, um
Tools mehrerer Server auseinanderzuhalten. Ein Node im Server darf trotzdem nicht
umbenannt werden.

### Abweichende `protocolVersion`

Blockbrain fragt `protocolVersion: 2025-11-25` an, n8n antwortet `2024-11-05`. Das hat
nicht gestört; festgehalten für den Fall künftiger Protokollfehler.

---

## 7. Offenes Risiko: Queue-Modus und mehrere Webhook-Replicas

Die Instanz läuft im **Queue-Modus**. Eine MCP-Session ist **instanzgebunden**: der
`initialize`-Handshake legt eine Session auf genau dem Prozess an, der den Request
angenommen hat, und die `mcp-session-id` ist nur dort bekannt.

**Konsequenz:** Laufen mehrere Webhook-Replicas hinter dem Reverse Proxy, müssen alle
Requests eines Clients auf **dieselbe** Replica gehen — per Session-Affinität, oder
indem es nur eine Webhook-Replica gibt. Andernfalls landet ein Folgerequest auf einem
Prozess, der die Session nicht kennt.

Bisher **nicht aufgetreten** — hier als Risiko dokumentiert, nicht als Befund. Das
Symptom wäre sporadisches Abbrechen mitten in einer Tool-Sequenz, nicht ein
reproduzierbarer Fehler beim Verbinden.

---

## 8. Was verifiziert ist

| Prüfung | Ergebnis |
|---|---|
| Streamable HTTP + Bearer am Endpunkt | 200 mit Handshake und `mcp-session-id` |
| Aufruf ohne Token | 403 — die Absicherung greift |
| `MCP_GA_98_Selftest` | 14/14 PASS, `fail: 0` |
| Echter Prompt aus Blockbrain | `get_account_summaries` liefert 3 Accounts / 14 Properties |
| Anzahl sichtbarer Tools | 11 im `tools/list` des Servers |

Der eine Punkt, der aus einer echten Nutzung noch nicht belegt ist: ob eine **deutsche
Berichtsfrage** `run_report` tatsächlich in die Top 5 zieht. Die Schlagwortzeile aus
Abschnitt 5 ist dafür eine begründete Maßnahme, aber bis zur Messung eine Wette.
