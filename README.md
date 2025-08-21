# Multi-Entity Process Graph (MEPG) in Neo4j

This repository contains a reproduction of the model described in **[Multi-Dimensional Event Data in Graph Databases](https://link.springer.com/10.1007/s13740-021-00122-1)**, using **Neo4j** as the graph database.  
We follow the paper’s idea of representing events, entities, and directly-follows (DF) relations across multiple entity types.

---

## 🔎 Data & Preprocessing
- Source log: `BPI2017.csv`  
- Each row = one `Event` with attributes:
  - `case` (Application ID)
  - `OfferID`
  - `Resource`
  - `Activity`
  - `completeTime` (HH:MM:SS format, later converted to `Timestamp`)

**Preprocessing notes:**
- If timestamps are only durations (e.g., `0:51:15`), convert them into valid ISO8601 datetimes before import (e.g., `2025-01-01T00:51:15`).
- CSV cleaning can be done manually or via Python/Pandas/Excel.

---

## 🗂 Data Model

### Event Layer
- `(:Event)` nodes  
- Properties: `{Activity, Timestamp, Appl, oID, Resource}`  

### Entity Layer
- `(:Application)`, `(:Offer)`, `(:Resource)` nodes  
- Reified entities (e.g., `(:AO)`) represent relationships between entities (Application–Offer, etc.)  
- Relationships:
  - `(:Event)-[:CORR]->(:Entity)`  
  - `(:Entity)-[:REL {Type:"Reified"}]->(:AO)`  

### Class Layer
- `(:Class)` nodes group events by Activity or Resource  
- `(:Event)-[:OBSERVED]->(:Class)`  

### Behavioral Layer
- Directly-Follows at event level:  
  `(:Event)-[:DF {EntityType:"Application"|"Offer"|"Resource"|"AO"}]->(:Event)`  
- Aggregated Directly-Follows at class level:  
  `(:Class)-[:DF_C {EntityType:"Application"|...}]->(:Class)`

---

## ⚙️ Main Steps

### 1. Load Events
```cypher
LOAD CSV WITH HEADERS FROM "file:///BPI2017.csv" AS line
CREATE (:Event {
  EventId: line.EventID,
  EventType: line.event,
  Timestamp: datetime(line.timeStamp),
  Appl: line.case,
  Action: line.Action,
  EventOrigin: line.EventOrigin,
  oID: line.OfferID,
  FWAmount: line.FirstWithdrawalAmount,
  NOofTerms: line.NumberOfTerms,
  Accepted: line.Accepted,
  CreditScore: line.CreditScore,
  OfferedAmount: line.OfferedAmount,
  MonthlyCost: line.MonthlyCost,
  Resource: line.resource 
});
### 2. Create Entities
```cypher
// Application
MATCH (e:Event)
MERGE (a:Entity:Application {id: e.Appl})
MERGE (e)-[:CORR]->(a);

// Offer
MATCH (e:Event)
WHERE e.oID IS NOT NULL
MERGE (o:Entity:Offer {id: e.oID})
MERGE (e)-[:CORR]->(o);

// Resource
MATCH (e:Event)
WHERE e.Resource IS NOT NULL
MERGE (r:Entity:Resource {id: e.Resource})
MERGE (e)-[:CORR]->(r);

### 3. Reify Relations (e.g., AO)
```cypher
MATCH (a:Application)<-[:CORR]-(e:Event)-[:CORR]->(o:Offer)
WITH a, o
MERGE (ao:Entity:AO {appId:a.id, offerId:o.id, type:"AO"})
MERGE (a)-[:REL {Type:"Reified"}]->(ao)
MERGE (o)-[:REL {Type:"Reified"}]->(ao)
MERGE (e)-[:CORR]->(ao);


### 4. Directly-Follows (DF) Relations
```cypher
// Application perspective
MATCH (a:Application)<-[:CORR]-(e:Event)
WITH a, e ORDER BY a.id, e.Timestamp
WITH a, collect(e) AS events
UNWIND range(0, size(events)-2) AS i
WITH events[i] AS e1, events[i+1] AS e2, a
MERGE (e1)-[:DF {EntityType:"Application"}]->(e2);

### 5. Event Classes & Observations
```cypher
// Resource Classes
MATCH (e:Event)
MERGE (c:Class:Resource {id:e.Resource, type:"Resource"})
MERGE (e)-[:OBSERVED]->(c);

// Activity Classes
MATCH (e:Event)
MERGE (c:Class:Activity {id:e.Activity, type:"Activity"})
MERGE (e)-[:OBSERVED]->(c);

### 6.Aggregated DF (DF_C)
```cypher
MATCH (e1:Event)-[df:DF]->(e2:Event),
      (e1)-[:OBSERVED]->(c1:Class),
      (e2)-[:OBSERVED]->(c2:Class)
MERGE (c1)-[dfc:DF_C {EntityType: df.EntityType}]->(c2)
ON CREATE SET dfc.count = 1
ON MATCH SET dfc.count = dfc.count + 1;




