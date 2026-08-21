# BALM-Unternehmenssuche (VUDat) → n8n

Automatisierung der öffentlichen **Unternehmenssuche des Bundesamtes für Logistik und
Mobilität (BALM)** — der Recherchemaske über die **Verkehrsunternehmensdatei (VUDat)**,
das zentrale elektronische Register der Güterkraft- und Personenverkehrsunternehmen.

Einstiegsseite: <https://www.balm.bund.de/DE/Service/Unternehmenssuche/suche_node.html>

Ziel: eine strukturierte Liste aus *Unternehmensname* + *PLZ oder Ort* abfragen und je
gefundenes Unternehmen eine Zeile in einer Tabelle anlegen.

---

## 1. Analyse der Webseite

Die Seite läuft auf dem **Government Site Builder (GSB)**, der Content-Management-Lösung
der Bundesverwaltung. Das ist der entscheidende Befund, denn GSB-Suchformulare sind
serverseitig gerendert.

### 1.1 Das Suchformular

Auf der Einstiegsseite liegen drei `<form>`-Elemente. Zwei davon sind die globale
Servicesuche des Portals (Kopf- und Overlay-Variante), das dritte ist die gesuchte:

```html
<form name="unternehmenssuche"
      action="SiteGlobals/Forms/Suche/Unternehmenssuche_Formular.html"
      method="get"
      enctype="application/x-www-form-urlencoded">
  <input type="hidden" name="nn"           value="540550" />
  <input type="hidden" name="resourceId"   value="564846" />
  <input type="hidden" name="input_"       value="540550" />
  <input type="hidden" name="pageLocale"   value="de" />
  <input name="unternehmensName" type="text" maxlength="100" aria-required="true" />
  <input name="plz_ort"          type="text" maxlength="100" aria-required="true" />
  <button type="submit" name="submit" value="Suchen">Suchen</button>
</form>
```

Daraus folgt die günstigste mögliche Ausgangslage für eine Automatisierung:

| Aspekt | Befund |
|---|---|
| Methode | **GET** — die Suche ist als URL vollständig darstellbar und wiederholbar |
| Authentifizierung | **keine** |
| CSRF-Token | **keins** |
| Captcha | **keins** |
| JavaScript | **nicht erforderlich** — Treffer stehen im ausgelieferten HTML |
| Cookies | werden gesetzt (`GSB_CAE_SESSIONID`), sind aber **nicht nötig** |
| Antwortformat | HTML (kein JSON-Endpunkt vorhanden) |

Es braucht also **keinen Browser und kein Headless-Chrome** — ein einzelner
HTTP-Request-Node genügt.

### 1.2 Der Abfrage-Endpunkt

```
GET https://www.balm.bund.de/SiteGlobals/Forms/Suche/Unternehmenssuche_Formular.html
      ?pageLocale=de
      &unternehmensName=<Name>
      &plz_ort=<PLZ oder Ort>
      &submit=Suchen
```

Beispiel (Testfall):

```
…Unternehmenssuche_Formular.html?pageLocale=de&unternehmensName=R%C3%BCdinger&plz_ort=Krautheim&submit=Suchen
```

**Die Hidden-Felder `nn`, `resourceId` und `input_` sind verzichtbar.** Sie wurden
gegengeprüft: die Abfrage funktioniert mit und ohne sie. Sie werden bewusst *nicht*
mitgesendet, weil sie deploymentspezifische GSB-Knoten-IDs sind — sie ändern sich, wenn
BALM die Seite umbaut, und ein veralteter Wert wäre eine unnötige Bruchstelle.

> **Beobachtung am Rande:** Mit und ohne `resourceId`/`input_` liefert dieselbe
> Suchanfrage teils unterschiedliche Zehnerausschnitte derselben Treffermenge. Für die
> Kappungsgrenze (1.4) ist das ohne Belang, für Suchen mit ≤ 9 Treffern ebenfalls.

### 1.3 Struktur der Antwort

Die Trefferliste steht in `<section class="c-search-result">`:

```html
<section class="c-search-result">
  <h2 class="c-search-result__count-headline">
    <span class="c-search-result__count-headline-text">2</span>
    <span class="c-search-result__count-headline-subtext">Ergebnisse</span>
  </h2>
  <ul class="c-search-result__list">
    <li class="c-search-result-teaser c-search-result-teaser--unternehmen">
      <a class="c-search-result-teaser__link"
         href="DE/Service/EEREinzelansicht/einzelansicht_node.html?idUnternehmen=16709"></a>
      <h3 class="c-search-result-teaser__headline">
        <span class="c-search-result-teaser__headline-text">Rüdinger Spedition GmbH</span>
      </h3>
      <div class="c-search-result-teaser__content"><p>74238 Krautheim</p></div>
    </li>
    …
  </ul>
</section>
```

Extraktionsvertrag je `<li>`:

| Feld | Herkunft |
|---|---|
| Unternehmensname | `.c-search-result-teaser__headline-text` |
| PLZ + Ort | `.c-search-result-teaser__content > p` |
| Unternehmens-ID | `href`, Query-Parameter `idUnternehmen` |
| Detailseite | `href`, relativ zu `https://www.balm.bund.de/` |

Zwei Fallstricke beim Parsen:

1. **HTML-Entities.** Namen enthalten numerische *und* benannte Entities —
   `&#034; MOBILe Hubtechnik &#034;`, `Alpina Umzüge &amp; Transport GmbH`. Ohne
   Dekodierung landen die Rohsequenzen in der Tabelle.
2. **PLZ/Ort ist ein Freitextfeld.** Neben `74238 Krautheim` kommt auch
   `12623 Berlin, Stadt` vor. Die Trennung läuft über `^(\d{4,5})\s+(.*)$`; das Rohfeld
   wird zusätzlich unverändert mitgeschrieben.

### 1.4 Grenzen der Quelle — geprüft, nicht vermutet

**Die Suche liefert maximal 10 Treffer und hat keine Blätterfunktion.**

Gegengeprüfte Kandidatenparameter, alle ohne Wirkung:

| Parameter | Ergebnis |
|---|---|
| `pageNo=1` / `pageNo=2` / `pageNo=50` | identische erste 10 Treffer, kein Versatz |
| `resultsPerPage=50`, `hitsPerPage=50`, `pageSize=50` | unverändert 10 Treffer |
| `gtp=564846_list%3D2` (GSB-Blättermuster) | unverändert 10 Treffer |
| `gtp=565772_list%3D2` | Trefferliste wird gar nicht mehr gerendert |

Belegt wird die Kappung dadurch, dass die Abfrage *Transport / Berlin* bei
unterschiedlichen Parametersätzen **verschiedene** Zehnermengen zurückgibt — die
Gesamtmenge ist also nachweislich größer als 10, während die Kopfzeile „10 Ergebnisse"
meldet. Die Zahl in der Kopfzeile ist damit **die Anzahl der ausgelieferten Treffer, nicht
die Gesamttrefferzahl.**

Konsequenz für den Betrieb: Bei genau 10 Treffern ist das Ergebnis als **abgeschnitten**
zu behandeln und die Suche zu verfeinern (PLZ statt Ortsname, vollständigerer
Firmenname). Der Workflow markiert diesen Fall selbst.

Weitere dokumentierte Einschränkung der Seite: *„Eine Suche mit Sonderzeichen oder
Platzhaltern ist nicht möglich."* Es gibt also kein `Rüding*`.

### 1.5 Verhalten in Randfällen

Alle vier Fälle liefern **HTTP 200** — der Statuscode taugt nicht zur Fallunterscheidung,
sie muss am HTML erfolgen.

| Fall | Antwort | Erkennungsmerkmal |
|---|---|---|
| Treffer vorhanden | Kopfzahl > 0, `<li>`-Teaser | Teaser vorhanden |
| Kein Treffer | Kopfzahl `0`, keine Teaser | Kopfzahl = 0 |
| Pflichtfeld leer | Formular-Fehlerseite ohne Ergebnisbereich | `Fehler im Formular` bzw. `class="formError"` |
| Unerwartete Seite | keine Kopfzahl auffindbar | Kopfzahl nicht parsebar |

---

## 2. Der Workflow

**`BALM Unternehmenssuche (VUDat) → Tabelle`**, Workflow-ID `8Aff5QK1N3gS8QFI`.

```
Manuell starten
  → Suchliste (Eingabe)          Code: Liste aus { unternehmensname, plz_ort }
  → BALM Unternehmenssuche       HTTP GET, 1 Request je Listeneintrag
      ├── Treffer auslesen       Code: 1 Item je gefundenem Unternehmen
      │     → Zeile in Tabelle schreiben   Data Table, insert
      └── Abfrage-Protokoll      Code: 1 Statuszeile je Abfrage
```

### 2.1 Warum kein Loop-Node

Der HTTP-Request-Node läuft von sich aus einmal pro Eingabe-Item. Die Drosselung
übernimmt seine Batching-Option (`batchSize: 1`, `batchInterval: 1500` — also eine
Anfrage pro 1,5 s gegen eine öffentliche Behördenseite).

Ein `splitInBatches`-Loop wäre hier sogar **schädlich**: Eine Abfrage ohne Treffer
erzeugt keine Items, die Loop-Rückkopplung bekäme keine Daten und die Schleife bliebe
stehen. Ohne Loop existiert dieses Problem nicht.

### 2.2 Zuordnung Antwort → Abfrage

Beide Code-Nodes lesen die Abfrageparameter über
`$('Suchliste (Eingabe)').all()[i]` positionsgleich zur Antwort. Das ist zulässig, weil
der HTTP-Node **genau ein** Ausgabe-Item je Eingabe-Item erzeugt — auch im Fehlerfall,
denn er ist auf `onError: continueRegularOutput` gesetzt. Die Reihenfolge bleibt damit
unter allen Umständen erhalten.

Zusätzlich: `retryOnFail` mit 3 Versuchen und 2 s Abstand gegen Netzwerkaussetzer.

### 2.3 Zwei Ausgänge, mit Absicht

Der Auftrag lautet „je gefundenes Ergebnis eine neue Zeile". Abfragen **ohne** Treffer
erzeugen deshalb korrekt keine Tabellenzeile — dürfen aber auch nicht stillschweigend
verschwinden. Darum hängt am HTTP-Node ein zweiter Zweig:

`Abfrage-Protokoll` liefert je Abfrage genau eine Statuszeile mit
`status` ∈ {`ok`, `keine_treffer`, `treffer_ggf_abgeschnitten`, `eingabe_fehler`,
`unerwartete_seite`, `http_fehler`}, `treffer_anzahl`, `http_status` und
`fehlermeldung`.

Damit bleibt „nichts gefunden" von „nie abgefragt" unterscheidbar. Soll das Protokoll
dauerhaft persistiert werden, genügt ein weiterer Data-Table-Node an diesem Zweig.

### 2.4 Zieltabelle

Data Table **`BALM Unternehmenssuche Ergebnisse`**, ID `d8qGdBU1RdLftkmI`.

| Spalte | Typ | Inhalt |
|---|---|---|
| `abgefragt_am` | date | Zeitpunkt des Abrufs |
| `abfrage_unternehmensname` | string | Eingabewert Name |
| `abfrage_plz_ort` | string | Eingabewert PLZ/Ort |
| `unternehmensname` | string | Treffer, Entities dekodiert |
| `plz` | string | aus PLZ/Ort abgetrennt |
| `ort` | string | aus PLZ/Ort abgetrennt |
| `plz_ort` | string | Rohfeld der Seite |
| `id_unternehmen` | string | `idUnternehmen` der Detailseite |
| `detail_url` | string | absolute URL der Einzelansicht |
| `treffer_anzahl` | number | Treffer dieser Abfrage |
| `moeglicherweise_abgeschnitten` | boolean | `true` bei ≥ 10 Treffern |
| `quelle_url` | string | reproduzierbare Such-URL |

`plz` und `ort` sind bewusst `string`: Eine PLZ ist eine Kennung, keine Zahl — führende
Nullen dürfen nicht verloren gehen.

### 2.5 Eingabe austauschen

Die Liste steht als Array im Code-Node `Suchliste (Eingabe)`. Die Folge-Nodes erwarten
ausschließlich die zwei Felder `unternehmensname` und `plz_ort` — der Node ist damit
1:1 gegen einen Data-Table-, Google-Sheets-, Excel- oder Webhook-Node austauschbar,
ohne dass sonst etwas angepasst werden muss.

---

## 3. Testnachweis

### 3.1 Vereinbarter Testfall

Eingabe `Rüdinger` / `Krautheim` → **2 Treffer**, 2 Tabellenzeilen:

| unternehmensname | plz | ort | id_unternehmen |
|---|---|---|---|
| Rüdinger Spedition GmbH | 74238 | Krautheim | 16709 |
| Rüdinger Verkehrsbetriebe e. K. | 74238 | Krautheim | 21239 |

Protokoll: `status: ok`, `treffer_anzahl: 2`, `http_status: 200`.

### 3.2 Härtetest über eine Mehrzeilen-Liste

Eine Liste mit vier Einträgen in einem Lauf, um Reihenfolge, Zuordnung und
Fallunterscheidung gemeinsam zu prüfen:

| Eingabe | Status | Treffer | Tabellenzeilen |
|---|---|---|---|
| `Rüdinger` / `Krautheim` | `ok` | 2 | 2 |
| `Zzzqqxyzfirma` / `Krautheim` | `keine_treffer` | 0 | 0 |
| `Transport` / `Berlin` | `treffer_ggf_abgeschnitten` | 10 | 10, alle mit `moeglicherweise_abgeschnitten: true` |
| *(leer)* / `Krautheim` | `eingabe_fehler` | — | 0 |

Insgesamt 12 Zeilen geschrieben. Mitgeprüft und bestätigt: Entity-Dekodierung
(`" MOBILe Hubtechnik " Kranarbeiten und Transporte GmbH`,
`Alpina Umzüge & Transport GmbH`) sowie die PLZ/Ort-Trennung bei
`12623 Berlin, Stadt`.

Die Testzeilen wurden danach entfernt; die Tabelle enthält nur noch die zwei Zeilen des
Testfalls aus 3.1.

---

## 4. Rechtlicher und betrieblicher Rahmen

- Abgefragt wird ein **öffentliches Register** über seine öffentliche Suchmaske; es
  werden keine Zugangsbeschränkungen umgangen.
- Die Drosselung auf eine Anfrage pro 1,5 s ist Absicht. Wer die Liste auf mehrere
  hundert Einträge ausbaut, sollte das Intervall erhöhen, nicht senken.
- Die Automatisierung hängt an **HTML-Klassennamen**. Ein Redesign der BALM-Seite bricht
  die Extraktion. Symptom wäre `status: unerwartete_seite` oder plötzlich 0 Treffer bei
  bekannten Unternehmen — beides im Protokoll sichtbar, weshalb es existiert.
- BALM bietet keinen offiziellen JSON-/API-Zugang zur VUDat. Für regelmäßige Abfragen
  größeren Umfangs ist die Anfrage eines Datenzugangs beim BALM der sauberere Weg als
  eine Ausweitung dieses Workflows.

## 5. Ausbaustufe: Detailseiten

Jeder Treffer trägt seine `id_unternehmen` und die URL der Einzelansicht
(`DE/Service/EEREinzelansicht/einzelansicht_node.html?idUnternehmen=…`). Damit lässt sich
der Workflow ohne Umbau um eine Anreicherung erweitern: ein zweiter HTTP-Node auf
`detail_url` je Tabellenzeile. Bewusst **nicht** Teil dieser Ausbaustufe, weil die
Aufgabenstellung Name und PLZ/Ort verlangt — und weil jede Detailabfrage die Last auf
der Behördenseite mit der Trefferzahl multipliziert.
