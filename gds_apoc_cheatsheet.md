# GDS & APOC Cheatsheet — Cypher für AML, Sanktions- & Betrugsanalyse

*Praktisches Nachschlagewerk für die Graph Data Science Library (GDS) und APOC in Neo4j. Jedes Beispiel ist in sich geschlossen: es **erstellt** ein Mikro-Datensample, **führt** einen GDS-Algorithmus oder eine APOC-Prozedur **aus** und **löscht** die Daten am Ende wieder — du kannst also jede Zelle isoliert ausprobieren, ohne deinen Hauptgraphen zu verändern.*

> Dieses Dokument ergänzt die [README](README.md) aus dem Sanktions-Beispiel und nutzt dasselbe `:Entity`-Datenmodell (`id`, `name`, `country`, `sanctioned`, `location_x`/`location_y`).

---

## Inhalt

0. [⚠ Zuerst lesen: GDS/APOC in einem Dashboard verwenden](#0-zuerst-lesen)
1. [Wie ruft man GDS und APOC überhaupt auf? — `CALL`, `YIELD`, `RETURN`](#1-call-yield-return)
2. [Die `DISTINCT`-Regel zum Merken](#2-distinct)
3. [APOC & Geo: Distanz berechnen und 50-km-Radius filtern](#3-geo)
4. [Die wichtigsten GDS-Algorithmen — was sagen sie im AML-Kontext aus?](#4-algorithmen-erklaert)
5. [5 Mikro-Beispiele: Daten erstellen → GDS ausführen → Daten löschen](#5-mikrobeispiele)

---

<a name="0-zuerst-lesen"></a>
## 0. ⚠ Zuerst lesen: Worauf du achten musst, wenn du GDS-Cypher für ein Dashboard verwendest

Das ist die wichtigste Seite des ganzen Dokuments. Wer GDS-Queries direkt in ein Dashboard (Neodash, Grafana, eine React-App über den Bolt-Treiber, …) einbaut, tappt fast immer in dieselben Fallen:

**1. GDS rechnet NICHT auf deinem echten Graphen, sondern auf einer In-Memory-Kopie.**
Bevor ein Algorithmus laufen kann, musst du eine *Projektion* anlegen (`gds.graph.project(...)`). Diese Kopie liegt im RAM. Das hat zwei Konsequenzen:
- **Sie wird nicht automatisch aktualisiert.** Ändert sich deine Datenbank (neue Transaktion kommt rein), sieht der projizierte Graph das erst, wenn du ihn *neu* projizierst. Ein Dashboard, das „live" wirken soll, zeigt sonst veraltete Scores.
- **Sie belegt Speicher, bis du sie freigibst.** Immer mit `gds.graph.drop('name')` aufräumen. Sonst summiert sich bei jedem Dashboard-Refresh eine neue Projektion → Out-of-Memory.

**2. Namenskollision bei parallelen Nutzern.**
Projektionen haben einen globalen Namen. Öffnen zwei Analysten dasselbe Dashboard und beide führen `gds.graph.project('myGraph', ...)` aus, bekommt der zweite den Fehler *„A graph with name 'myGraph' already exists"*. Lösung: eindeutige Namen (z. B. mit Session-ID/Zeitstempel) oder vor der Projektion prüfen:
```cypher
CALL gds.graph.exists('myGraph') YIELD exists
WHERE exists
CALL gds.graph.drop('myGraph') YIELD graphName
RETURN graphName
```

**3. `.write` schreibt in die Datenbank — im Dashboard fast immer falsch.**
GDS-Algorithmen haben mehrere Ausführungsmodi (siehe Kapitel 1). `.write` schreibt die Ergebnisse als Knoteneigenschaften zurück in die DB und braucht dafür eine **Schreib-Transaktion**. Viele Dashboards laufen aber in Read-Only-Transaktionen → Fehler. Für die reine Anzeige nimm **`.stream`** (liefert Ergebnisse nur zurück, ändert nichts).

**4. Live-Neuberechnung bei jedem Refresh ist teuer.**
Für große Graphen ist Projektion + Louvain bei jedem Panel-Reload sekundenlang. Zwei saubere Muster:
- **Vorberechnen (Batch):** Nachts einmal mit `.write` die Scores als Property (`e.pageRank`, `e.community`) in die DB schreiben. Das Dashboard liest dann nur noch billige `MATCH ... RETURN e.pageRank` — blitzschnell und read-only.
- **On-the-fly nur für kleine Subgraphen:** Projiziere per Cypher-Projektion nur den gerade betrachteten Ausschnitt.

> **Merksatz fürs Dashboard:** *Projizieren → Streamen → Droppen.* Schreiben (`.write`) gehört in den nächtlichen Batch-Job, nicht in ein Panel.

---

<a name="1-call-yield-return"></a>
## 1. Wie ruft man GDS und APOC überhaupt auf? — `CALL`, `YIELD`, `RETURN`

GDS- und APOC-Funktionen sind keine normalen Cypher-Keywords, sondern **Prozeduren** (bzw. Funktionen). Es gibt genau zwei Aufrufarten.

### a) Funktionen — direkt im Ausdruck verwendbar

Eine *Funktion* gibt einen einzelnen Wert zurück und darf überall stehen, wo ein Wert erwartet wird (in `RETURN`, `WHERE`, `SET`, …). Kein `CALL`, kein `YIELD` nötig:

```cypher
RETURN apoc.text.clean('  Álpha Círcuit GmbH  ') AS normalisiert
// → "alpha circuit gmbh"
```

### b) Prozeduren — brauchen `CALL` und meist `YIELD`

Eine *Prozedur* kann mehrere Zeilen und mehrere Spalten zurückgeben. Muster:

```
CALL <prozedur>(<argumente>)   // ruft die Prozedur auf
YIELD spalteA, spalteB          // benennt, WELCHE Rückgabespalten du weiterverwenden willst
RETURN spalteA, spalteB         // gibt sie aus (oder verarbeite sie weiter mit WHERE/WITH/...)
```

Konkretes GDS-Beispiel (Zentralität streamen):

```cypher
CALL gds.degree.stream('myGraph')       // Prozedur mit Argument (Name der Projektion)
YIELD nodeId, score                      // diese zwei Spalten will ich
RETURN gds.util.asNode(nodeId).name AS name, score   // nodeId → echten Knoten auflösen
ORDER BY score DESC
```

**Drei Dinge, die man am Anfang oft falsch macht:**

| Frage | Antwort |
|---|---|
| Wann brauche ich `YIELD`? | Immer bei einer Prozedur (`CALL`), deren Ergebnis du weiterverwenden willst. Nur wenn `CALL` die *allerletzte* Klausel ist, darf `YIELD` entfallen. |
| Warum liefert GDS `nodeId` statt des Knotens? | Aus Performancegründen arbeitet GDS intern mit internen IDs. Mit `gds.util.asNode(nodeId)` holst du den echten Knoten zurück. |
| Was bedeuten `stream` / `stats` / `mutate` / `write`? | Der **Modus** des Algorithmus — siehe unten. |

### Die vier GDS-Ausführungsmodi (extrem wichtig)

Jeder GDS-Algorithmus existiert in bis zu vier Varianten. Nur das Suffix ändert sich:

| Suffix | Was es tut | Wann verwenden |
|---|---|---|
| `.stream` | Gibt das Ergebnis Zeile für Zeile zurück, **ändert nichts**. | Dashboards, Ad-hoc-Analyse, Read-Only. |
| `.stats` | Gibt nur eine Zusammenfassung zurück (z. B. Anzahl Communities), keine Details. | Schneller Überblick / Health-Check. |
| `.mutate` | Schreibt das Ergebnis in die **In-Memory-Projektion** (nicht in die DB). | Wenn ein zweiter Algorithmus auf dem Ergebnis des ersten aufbauen soll. |
| `.write` | Schreibt das Ergebnis als **Property in die echte DB** zurück. | Nächtlicher Batch, damit das Dashboard später nur noch liest. |

```cypher
CALL gds.pageRank.stream('myGraph')  YIELD nodeId, score    // nur anzeigen
CALL gds.pageRank.write('myGraph', {writeProperty: 'pageRank'})  // in DB speichern
```

### APOC — dieselbe Logik

```cypher
// APOC-Prozedur (mit CALL + YIELD):
CALL apoc.path.expandConfig(startNode, {relationshipFilter: 'TRANSFERRED_TO>', maxLevel: 4})
YIELD path
RETURN path

// APOC-Funktion (direkt im Ausdruck):
RETURN apoc.coll.sum([480000, 510000, 490000]) AS gesamtvolumen
```

Faustregel: **Steht ein Wert direkt in `RETURN`/`WHERE`? → Funktion (kein CALL). Kommen mehrere Zeilen zurück? → Prozedur (`CALL … YIELD …`).**

---

<a name="2-distinct"></a>
## 2. Die `DISTINCT`-Regel zum Merken

`DISTINCT` verwirrt, weil es an manchen Stellen erlaubt ist und an anderen nicht. Der Grund ist simpel, sobald man **eine einzige Regel** verinnerlicht:

> **`DISTINCT` gehört an genau zwei Stellen — und sonst nirgends:**
> 1. **Direkt hinter `RETURN` oder `WITH`** → dedupliziert *ganze Zeilen*.
> 2. **Als erstes Wort in den Klammern einer Aggregatsfunktion** → `count(DISTINCT x)`, `collect(DISTINCT x)`, `sum(DISTINCT x)`.

Warum funktioniert es **nicht** in `WHERE`? Weil `WHERE` einzelne Zeilen *filtert* (ja/nein), aber nichts *zusammenfasst*. „Eindeutig" ist erst dann eine sinnvolle Operation, wenn man mehrere Werte zu einem zusammenzieht — und das passiert nur beim Projizieren (`RETURN`/`WITH`) oder beim Aggregieren (`count`/`collect`/`sum`/…).

### Die vier Fälle nebeneinander

```cypher
// ✅ 1) Hinter RETURN — jedes Land nur einmal:
MATCH (e:Entity) RETURN DISTINCT e.country

// ✅ 2a) In count() — Anzahl VERSCHIEDENER Empfängerländer:
MATCH (:Entity)-[:TRANSFERRED_TO]->(t:Entity)
RETURN count(DISTINCT t.country) AS anzahl_laender

// ✅ 2b) In collect() — Liste OHNE Duplikate (ja, das ist erlaubt!):
MATCH (s:Entity)-[:TRANSFERRED_TO]->(t:Entity)
RETURN s.name, collect(DISTINCT t.country) AS belieferte_laender

// ❌ 3) In WHERE — SYNTAXFEHLER, hier gibt es kein DISTINCT:
// MATCH (e:Entity) WHERE DISTINCT e.country = 'Russia' RETURN e   // ✗ funktioniert nicht
```

> **Wenn du „eindeutig" in einer Bedingung brauchst,** dann willst du eigentlich vorher deduplizieren oder gruppieren. Beispiel: „Firmen, die an *mehr als 2 verschiedene* sanktionierte Länder geliefert haben":
> ```cypher
> MATCH (s:Entity)-[:TRANSFERRED_TO]->(t:Entity {sanctioned: true})
> WITH s, count(DISTINCT t.country) AS laender   // erst aggregieren…
> WHERE laender > 2                               // …dann filtern
> RETURN s.name, laender
> ```
> Das ist der Schlüssel: **`DISTINCT` beim Aggregieren, das Filtern danach mit `WHERE` auf dem Ergebnis.**

---

<a name="3-geo"></a>
## 3. APOC & Geo: Distanz berechnen und im 50-km-Radius filtern

Unser Datenmodell speichert Koordinaten als `location_x` (Längengrad) und `location_y` (Breitengrad). Für Distanzrechnungen wandeln wir sie zuerst in einen echten **`point`-Typ** um — dann liefert die Funktion `point.distance()` die Entfernung in **Metern** (Luftlinie, auf dem WGS-84-Ellipsoid).

### Schritt 1 — Koordinaten in Point-Nodes umwandeln (einmalig)

```cypher
MATCH (e:Entity)
WHERE e.location_x IS NOT NULL AND e.location_y IS NOT NULL
SET e.location = point({longitude: e.location_x, latitude: e.location_y})
```

### Schritt 2 — Alle Entitäten im 50-km-Radius um einen Punkt finden

```cypher
// Bezugspunkt: Frankfurt am Main (Finanzplatz)
WITH point({longitude: 8.6821, latitude: 50.1109}) AS frankfurt
MATCH (e:Entity)
WHERE e.location IS NOT NULL
  AND point.distance(e.location, frankfurt) <= 50000   // 50 km = 50.000 Meter
RETURN e.name,
       e.country,
       round(point.distance(e.location, frankfurt) / 1000, 1) AS entfernung_km
ORDER BY entfernung_km
```

> **Warum `point.distance` und nicht ein APOC-Aufruf für die reine Distanz?** In modernen Neo4j-Versionen (4.x/5.x) ist `point.distance()` nativ, schneller und ohne Extra-Plugin verfügbar — für die reine Entfernung ist das die empfohlene Wahl. APOC glänzt bei den *umliegenden* Aufgaben (siehe unten).

### Wo APOC beim Geo-Thema wirklich hilft: Geocoding

Oft hast du keine Koordinaten, sondern nur eine Adresse. `apoc.spatial.geocodeOnce` holt die Koordinaten aus einer Adresse (benötigt einen konfigurierten Geocoding-Provider) — danach rechnest du wieder mit `point.distance`:

```cypher
CALL apoc.spatial.geocodeOnce('Bahnhofstrasse 1, Zürich') YIELD location
WITH point({longitude: location.longitude, latitude: location.latitude}) AS ziel
MATCH (e:Entity)
WHERE e.location IS NOT NULL
  AND point.distance(e.location, ziel) <= 50000
RETURN e.name, round(point.distance(e.location, ziel) / 1000, 1) AS km
ORDER BY km
```

**AML-Nutzen:** Räumliche Nähe ist ein Risikosignal. Häufen sich mehrere Briefkastenfirmen unter derselben Adresse oder im selben engen Radius (klassisch: ein „Firmenhotel", in dem hunderte Shells registriert sind), ist das ein starker Cluster-Indikator.

---

<a name="4-algorithmen-erklaert"></a>
## 4. Die wichtigsten GDS-Algorithmen — was sagen sie im AML-Kontext WIRKLICH aus?

„Zeigt, wie zentral ein Knoten ist" ist nutzlos, wenn man nicht weiß, *welche Art* von Zentralität und *was das über einen Verdachtsfall aussagt*. Hier die gängigen Algorithmen mit ihrer konkreten AML-Bedeutung.

### Zentralitäts-Algorithmen — „Wer ist wichtig, und warum?"

| Algorithmus | Was er misst (simpel) | Was es im AML/Sanktions-Kontext bedeutet |
|---|---|---|
| **Degree Centrality** | Wie viele direkte Kanten hat ein Knoten (rein/raus). | **Aktivitätsniveau.** Ein Konto mit auffällig vielen ein- und ausgehenden Transfers ist ein Verteil-/Sammelknoten — typisch für *Placement* (Bargeld-Einspeisung über viele kleine Zuflüsse) oder einen „Money Mule"-Hub. |
| **PageRank** | Wie wichtig ein Knoten ist, gewichtet danach, wie wichtig die Knoten sind, die *auf ihn zeigen*. | **Risikovererbung.** Anders als Degree zählt PageRank nicht nur *Anzahl*, sondern *Qualität* der Verbindungen. Ein Konto, das Geld von vielen ohnehin verdächtigen Konten erhält, „erbt" deren Risiko — auch wenn es selbst wenige Kanten hat. Ideal, um verdeckte Sammelkonten zu finden. |
| **Betweenness Centrality** | Wie oft ein Knoten auf dem *kürzesten Pfad* zwischen zwei anderen liegt. | **Gatekeeper / Flaschenhals.** Das ist der wichtigste Wert für Layering: Ein Knoten mit hoher Betweenness ist die *Brücke*, über die Geld zwischen sonst getrennten Clustern fließt — genau die Shell, deren Wegfall den ganzen Ring zerschneidet. Prime-Ziel für Ermittler. |
| **Closeness Centrality** | Wie nah (über wie wenige Hops) ein Knoten alle anderen im Schnitt erreicht. | **Reichweite / Steuerungsfähigkeit.** Ein Akteur mit hoher Closeness kann das ganze Netzwerk schnell erreichen — häufig der eigentliche Organisator/UBO hinter mehreren Shells, nicht die Shell selbst. |

**Der Kernunterschied in einem Satz:** *Degree = wie viel Verkehr, PageRank = wie wichtiger Verkehr, Betweenness = wie unverzichtbar als Durchgangsstation, Closeness = wie gut vernetzt in die Breite.*

### Community-Detection-Algorithmen — „Wer gehört zusammen?"

| Algorithmus | Was er tut (simpel) | Was es im AML-Kontext bedeutet |
|---|---|---|
| **Weakly Connected Components (WCC)** | Teilt den Graphen in völlig **getrennte Inseln** — Knoten, die über *irgendeinen* Pfad verbunden sind, landen in derselben Komponente. | **Grobe Ring-Trennung / Datenqualität.** Findet komplett isolierte Teilnetze. Nützlich als erster Schnitt: „Welche Konten hängen überhaupt mit unserer sanktionierten Zielentität zusammen?" |
| **Louvain** | Findet **dicht verbundene Gruppen** (viele Kanten innen, wenige nach außen) und optimiert die „Modularität". Hierarchisch. | **Ringfindung.** Das ist *der* Geldwäsche-Algorithmus. Eine Louvain-Community ist typischerweise ein Ring aus Shells + wirtschaftlich Berechtigten, die untereinander viel Geld bewegen, aber nach außen abgeschottet sind. Deckt Strukturen auf, die kein einzelner Pfad zeigt. |
| **Label Propagation (LPA)** | Schnellere, gröbere Community-Findung: jeder Knoten übernimmt iterativ das häufigste Label seiner Nachbarn. | **Schneller Erst-Cluster** auf sehr großen Graphen, wenn Louvain zu teuer ist. Weniger stabil (leicht zufällig), aber gut für ein grobes Vorclustering. |

### Ähnlichkeit — „Welche Knoten verhalten sich gleich?"

| Algorithmus | Was er tut | AML-Bedeutung |
|---|---|---|
| **Node Similarity** | Vergleicht Knoten anhand ihrer *gemeinsamen Nachbarn* (Jaccard-Ähnlichkeit). | **Strohmann-Erkennung.** Zwei Shells, die von denselben Personen kontrolliert werden und an dieselben Ziele überweisen, sind strukturell fast identisch — ein starker Hinweis auf koordinierte, künstlich aufgeteilte Strukturen (Smurfing). |

---

<a name="5-mikrobeispiele"></a>
## 5. Fünf Mikro-Beispiele: Daten erstellen → GDS ausführen → Daten löschen

Jedes Beispiel steht für sich. Es legt ein winziges, benanntes Datensample (`:Demo`-Label) an, damit es **nicht mit deinen echten Daten kollidiert**, projiziert es, führt genau einen Algorithmus aus, zeigt das Ergebnis — und räumt am Ende Projektion *und* Daten wieder auf.

> **Bitte in dieser Reihenfolge ausführen:** (1) Daten anlegen, (2) projizieren, (3) Algorithmus, (4) aufräumen. In einer Jupyter-/Browser-Zelle kannst du jeden Block einzeln laufen lassen.

---

### Beispiel 1 — Degree Centrality: Wer ist der Sammel-/Verteilknoten?

**Idee:** `M` (Mule-Hub) empfängt von vier Zuflüssen und leitet an ein Ziel weiter. Degree macht ihn sofort sichtbar.

```cypher
// (1) Daten anlegen
CREATE (a:Demo {name:'Zufluss A'}), (b:Demo {name:'Zufluss B'}),
       (c:Demo {name:'Zufluss C'}), (d:Demo {name:'Zufluss D'}),
       (m:Demo {name:'Mule-Hub M'}), (z:Demo {name:'Ziel Z'})
CREATE (a)-[:PAYS]->(m), (b)-[:PAYS]->(m),
       (c)-[:PAYS]->(m), (d)-[:PAYS]->(m), (m)-[:PAYS]->(z);
```
```cypher
// (2) In-Memory-Projektion (UNDIRECTED, damit rein+raus zählt)
CALL gds.graph.project('demo1', 'Demo',
     {PAYS: {orientation: 'UNDIRECTED'}});
```
```cypher
// (3) Degree streamen
CALL gds.degree.stream('demo1')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS knoten, score AS grad
ORDER BY grad DESC;
```
**Erwartung:** `Mule-Hub M` hat Grad 5, alle anderen 1. → Der Hub sticht klar heraus.
```cypher
// (4) Aufräumen: Projektion + Daten löschen
CALL gds.graph.drop('demo1');
MATCH (n:Demo) DETACH DELETE n;
```

---

### Beispiel 2 — PageRank: Risikovererbung statt bloßer Anzahl

**Idee:** Zwei Konten haben *gleich viele* eingehende Kanten. Aber eines wird von einem hoch-verbundenen Cluster gespeist. PageRank unterscheidet sie, Degree nicht.

```cypher
// (1) Daten anlegen: Cluster (h1..h3) → wichtig; Randknoten (r) → x; beide Ziele je 1 Zufluss extra
CREATE (h1:Demo {name:'Hub H1'}), (h2:Demo {name:'Hub H2'}), (h3:Demo {name:'Hub H3'}),
       (wichtig:Demo {name:'Konto WICHTIG'}), (r:Demo {name:'Randknoten'}),
       (x:Demo {name:'Konto UNWICHTIG'})
CREATE (h1)-[:PAYS]->(h2), (h2)-[:PAYS]->(h3), (h3)-[:PAYS]->(h1),  // dichter Cluster
       (h1)-[:PAYS]->(wichtig), (h2)-[:PAYS]->(wichtig),           // aus dem Cluster gespeist
       (r)-[:PAYS]->(x);                                           // nur ein schwacher Zufluss
```
```cypher
// (2) Projektion (gerichtet — PageRank folgt der Flussrichtung)
CALL gds.graph.project('demo2', 'Demo', 'PAYS');
```
```cypher
// (3) PageRank streamen
CALL gds.pageRank.stream('demo2')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS knoten, round(score, 4) AS pagerank
ORDER BY pagerank DESC;
```
**Erwartung:** `Konto WICHTIG` bekommt einen deutlich höheren PageRank als `Konto UNWICHTIG`, obwohl beide „nur Empfänger" sind — weil sein Geld aus dem wichtigen Cluster stammt. **Genau das übersieht eine reine Zählung.**
```cypher
// (4) Aufräumen
CALL gds.graph.drop('demo2');
MATCH (n:Demo) DETACH DELETE n;
```

---

### Beispiel 3 — Betweenness Centrality: Die Brücke (Gatekeeper) finden

**Idee:** Zwei Gruppen sind nur über eine einzige Shell `B` verbunden. Alles Geld zwischen den Gruppen muss durch `B` — das ist der Layering-Flaschenhals.

```cypher
// (1) Daten anlegen: Gruppe links (l1,l2) — BRÜCKE B — Gruppe rechts (r1,r2)
CREATE (l1:Demo {name:'Links 1'}), (l2:Demo {name:'Links 2'}),
       (b:Demo {name:'BRÜCKE B'}),
       (r1:Demo {name:'Rechts 1'}), (r2:Demo {name:'Rechts 2'})
CREATE (l1)-[:PAYS]->(l2), (l2)-[:PAYS]->(b),
       (b)-[:PAYS]->(r1), (r1)-[:PAYS]->(r2);
```
```cypher
// (2) Projektion
CALL gds.graph.project('demo3', 'Demo', 'PAYS');
```
```cypher
// (3) Betweenness streamen
CALL gds.betweenness.stream('demo3')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS knoten, score AS betweenness
ORDER BY betweenness DESC;
```
**Erwartung:** `BRÜCKE B` hat die mit Abstand höchste Betweenness. Entfernt man sie, zerfällt das Netz in zwei Hälften — der ideale Ermittlungs-Angriffspunkt.
```cypher
// (4) Aufräumen
CALL gds.graph.drop('demo3');
MATCH (n:Demo) DETACH DELETE n;
```

---

### Beispiel 4 — Louvain: Zwei getrennte Geldwäsche-Ringe automatisch clustern

**Idee:** Zwei dichte Dreiecke sind nur durch eine einzige schwache Kante verbunden. Louvain erkennt sie trotzdem als *zwei* Communities.

```cypher
// (1) Daten anlegen: Ring 1 (a,b,c) dicht, Ring 2 (d,e,f) dicht, 1 schwache Brücke c→d
CREATE (a:Demo {name:'Ring1-A'}), (b:Demo {name:'Ring1-B'}), (c:Demo {name:'Ring1-C'}),
       (d:Demo {name:'Ring2-D'}), (e:Demo {name:'Ring2-E'}), (f:Demo {name:'Ring2-F'})
CREATE (a)-[:PAYS]->(b), (b)-[:PAYS]->(c), (c)-[:PAYS]->(a),   // Ring 1
       (d)-[:PAYS]->(e), (e)-[:PAYS]->(f), (f)-[:PAYS]->(d),   // Ring 2
       (c)-[:PAYS]->(d);                                       // schwache Brücke
```
```cypher
// (2) Projektion (UNDIRECTED — Communities sind richtungsunabhängig)
CALL gds.graph.project('demo4', 'Demo',
     {PAYS: {orientation: 'UNDIRECTED'}});
```
```cypher
// (3) Louvain streamen
CALL gds.louvain.stream('demo4')
YIELD nodeId, communityId
RETURN communityId, collect(gds.util.asNode(nodeId).name) AS mitglieder
ORDER BY communityId;
```
**Erwartung:** Zwei Communities — {Ring1-A,B,C} und {Ring2-D,E,F}. Louvain trennt die Ringe trotz der Brücke, weil *innerhalb* jedes Rings viel mehr Kanten liegen als dazwischen. **Das ist Ringfindung ohne vorheriges Wissen über die Ringe.**
```cypher
// (4) Aufräumen
CALL gds.graph.drop('demo4');
MATCH (n:Demo) DETACH DELETE n;
```

---

### Beispiel 5 — WCC (Weakly Connected Components): Isolierte Netze trennen

**Idee:** Zwei völlig unverbundene Gruppen. WCC gibt jeder Gruppe eine eigene Komponenten-ID — der schnellste Weg, „was hängt überhaupt zusammen?" zu beantworten.

```cypher
// (1) Daten anlegen: Insel 1 (p,q,s) und komplett getrennte Insel 2 (t,u)
CREATE (p:Demo {name:'Insel1-P'}), (q:Demo {name:'Insel1-Q'}), (s:Demo {name:'Insel1-S'}),
       (t:Demo {name:'Insel2-T'}), (u:Demo {name:'Insel2-U'})
CREATE (p)-[:PAYS]->(q), (q)-[:PAYS]->(s),
       (t)-[:PAYS]->(u);
```
```cypher
// (2) Projektion (UNDIRECTED)
CALL gds.graph.project('demo5', 'Demo',
     {PAYS: {orientation: 'UNDIRECTED'}});
```
```cypher
// (3) WCC streamen
CALL gds.wcc.stream('demo5')
YIELD nodeId, componentId
RETURN componentId, collect(gds.util.asNode(nodeId).name) AS mitglieder
ORDER BY componentId;
```
**Erwartung:** Zwei Komponenten — {Insel1-P,Q,S} und {Insel2-T,U}. Damit filterst du in Sekunden heraus, welche Konten *überhaupt* mit einer Zielentität in Verbindung stehen — und welche man ignorieren kann.
```cypher
// (4) Aufräumen
CALL gds.graph.drop('demo5');
MATCH (n:Demo) DETACH DELETE n;
```

---

## Schnellreferenz (Spickzettel)

```cypher
// — Aufruf-Muster —
CALL gds.<algo>.stream('graph')  YIELD nodeId, score      // anzeigen, nichts ändern
CALL gds.<algo>.write('graph', {writeProperty:'x'})       // in DB schreiben (Batch!)
RETURN apoc.<funktion>(...)                                // APOC-Funktion direkt im Ausdruck

// — Projektion verwalten —
CALL gds.graph.project('g', 'Label', 'REL')               // native Projektion
CALL gds.graph.project('g','Label',{REL:{orientation:'UNDIRECTED'}})
CALL gds.graph.exists('g')  YIELD exists                  // vor Kollision prüfen
CALL gds.graph.drop('g')                                  // IMMER aufräumen

// — nodeId → Knoten —
gds.util.asNode(nodeId).name

// — Geo —
point({longitude:x, latitude:y})                          // Point erzeugen
point.distance(p1, p2)                                    // Entfernung in Metern

// — DISTINCT-Regel —
RETURN DISTINCT ...            // ✅ ganze Zeile
count(DISTINCT x)             // ✅ in Aggregat
collect(DISTINCT x)          // ✅ in Aggregat
// WHERE DISTINCT ...        // ❌ gibt es nicht → erst WITH/aggregieren, dann WHERE
```

---

*Alle Daten und Namen sind fiktiv und dienen ausschließlich der technischen Veranschaulichung. Getestet mit Neo4j 5.x, GDS 2.x und APOC Core.*
