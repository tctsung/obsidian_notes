---
created: 2025-10-26T21:09
updated: 2025-10-26T21:19
---
### Intro
Query language for graph database
- Entity == Node
- Nodes are linked by <span style="color:rgb(255, 0, 0)">uni-direction</span> relationship
- each node & relationship can have <span style="color:rgb(255, 0, 0)">key-val pair</span> properties

### Basic Unit
- `(:nodes)-[:ARE_CONNECTED_TO]->(:otherNodes)`
	 - <span style="color:rgb(255, 0, 0)">(): node</span>
	 - <span style="color:rgb(255, 0, 0)">[]: relationship</span>
- node:
	 - **syntax:** `(<variable name> : <node type> <properties>)`
		  - **variable name**: human readable label, ideally unique but not necessary
		   - useful in MERGE/CREATE to further define relationship
		  - **node type**: category/<span style="color:rgb(255, 0, 0)">label</span> of the node, usually shared by multiple nodes
		  - **properties**: key-val pair metadata for node & relationship
	 - Ex. (Wenny:Person {name: "Wenny", age: null, active: true})
- relationship
	 - **syntax:** `[<variable name> : <relationship type> {k1: 'val1', k2: 'val2'}]`
	 - variable name is often skipped, less needed
	 - one direction, if need bi-direction, create two relationship
	 - Ex. [:WORKS_AT {since: 2024, citation: "http://xxx.com"}]
### Data type
**Basic Types**
- String: "Alice", 'Bob' (case-sensitive)
- Integer: 42, -10
- Float: 3.14, -2.5
- Boolean: <span style="color:rgb(255, 0, 0)">true</span>, <span style="color:rgb(255, 0, 0)">false</span>
- missing data: <span style="color:rgb(255, 0, 0)">null</span>

**Collection Types**
- List: `[1, 2, 3]`, `["a", "b", "c"]`
- Map: {name: "Alice", age: 30}

**Temporal Types**
- Date: `date("2023-01-01")`
- DateTime: `datetime("2023-01-01T10:30:00")`
- Time: `time("10:30:00")`
- Duration: `duration("P1Y2M")`

**Spatial Types**
- Point: `point({x: 1.0, y: 2.0})`

### CRUD
##### Def
- **CREATE:** create new nodes/relationship, <span style="color:rgb(255, 0, 0)">allow duplicates</span>
- **MERGE:** <span style="color:rgb(255, 0, 0)">create if not exist</span>, no duplicates
 - usually, only use MERGE
 - can MERGE node & relationship & property
- **SET:** <span style="color:rgb(255, 0, 0)">update</span> properties on existing nodes
##### CREATE
- can create node & relationship at the same time
 - suitable for simple relationship
```cypher
MERGE (Wenny:Person {name: "Wenny", age: null, active: true})-[:WORKS_AT {since: 2020}]->(PipiCat:Company {name: "PipiCat"})
```

- can separate node & relationship creation
	- suitable for complex graph & more readable
```cypher
// Create nodes first
MERGE (Wenny:Person {name: "Wenny", age: null, active: true})
MERGE (PipiCat:Company {name: "PipiCat"})

// Then relationship
MERGE (Wenny)-[:WORKS_AT {since: 2020}]->(PipiCat)
```
##### SET
- <span style="color:rgb(255, 0, 0)">SET</span> can update/create new property
 - if in same statement (;), then no need to MATCH
```cypher
MATCH (w:Person {name: "Wenny"})
SET w.age = 18
SET w.active = false
```
- <span style="color:rgb(255, 0, 0)">ON CREATE SET</span>
	- add property only if a new node was created
```cypher
MERGE (p:Person {name: "Deron"})
ON CREATE SET p.created = timestamp(), p.status = "new"
```
##### DELETE
```cypher
// delete a node
MATCH (p:Person {name: "John"})
DELETE p

// delete relationships
MATCH (a)-[r:KNOWS]-(b)
WHERE a.name = "Alice"
DELETE r
```
## Query: MATCH
##### Basics
- MATCH == SELECT in SQL
- semicolon (`;`) sep session, like SQL
- syntax: `MATCH (<variable name>:<node label>) RETURN <variable name>`
	 - variable name only lives in current session
- MATCH a graph/specific node type
 - return <span style="color:rgb(255, 0, 0)">graph | json</span> format as array
```cypher
// return all nodes & relationship in graph format
MATCH (n) RETURN n   // this might only return node in new versions
MATCH (n)-[r]-(m) RETURN n, r, m   // this will return whole graph
```
- return nodes with specific label/node type
	- relationship btw the label will also be returned
	 - eg. `(wenny) -[]-> (deron)`

```cypher
// return one label & relationship ONLY of the label:
MATCH (n:Person) RETURN n
```
- MATCH specific property of specific label
	 - syntax:`MATCH (n:<label>) RETURN n.<p1>, n.<p2>`
 - return <span style="color:rgb(255, 0, 0)">tabular format</span>
```cypher
MATCH (n:Person) RETURN n.name AS first_name, n.height
```
##### Multiple labels
- if MATCH all labels, it became same as MATCH (n) RETURN n
- use <span style="color:rgb(255, 0, 0)">RETURN to define the node labels to include in output</span>
 - if need node & relationship:
 `RETURN <node variable name>, <relationship variable name>, <node 2>`
```cypher
// return labels & relationships btw them
MATCH p = (n:Person) -[w:WORKS_FOR] -> (c:company)
RETURN p
```
##### WHERE: filter by property
- ideally, we should create the unique ID by ourselves using a specific property
- syntax:
	 - =: equal
	 - <>: not equal
	 - multiple conditions: AND , OR
	 - `NOT`: flip condition
```cypher
// Method 1: WHERE
MATCH (n:Person)
WHERE NOT (n.name <> "Deron" AND n.height <= 2)
 OR n.name = "Wenny"
RETURN n

// Method 2: use {} directly
MATCH (n:Person {name = "Deron", age = 27})
RETURN n
```
##### LIMIT | SKIP
- `SKIP`: skip the first k matches
 - should always pair with ORDER BY
```cypher
// Eg. get the second tallest person
MATCH (n:Person)
RETURN n
ORDER BY n.height DESC
SKIP 1 LIMIT 1
```
### Advanced MATCH:
##### relationship
- use (node) -[relationship]->(node) with WHERE condition
- can filter by relationship | node property
```cypher
MATCH (n:PLAYER) -[r:PLAYS_FOR]-> (t:TEAM)
WHERE t.name in ["LA Lakers", "Dallas Mavericks"]
 AND r.salary >= 3000000
// return all node & relationship to see full subgraph:
RETURN n, r, t
```
##### Traverse

| Syntax               | Desc                                 |
| -------------------- | ------------------------------------ |
| `[:<type>*]`         | ==**unbounded**==/any len of path    |
| `[:<type>*<j>..<k>]` | path w len ==**range from**== j to k |
| `[:<type>*..<k>]`    | path w len ==**up to**== k           |
| `[:<type>*<k>]`      | path w len ==**exactly**== k         |
```cypher
// Find connections within 3 hops
MATCH (p:Person)-[:KNOWS*..3]-(connected)
WHERE p.name = "Deron"
RETURN connected

// Find a friend's friend
MATCH (p:Person)-[:KNOWS*2]-(connected)
WHERE p.name = "Deron"
RETURN connected
```
##### Shortest path
- find the shortest path of two nodes
- can define weights for the relationship using Dijkstra ([source](https://neo4j.com/docs/graph-data-science/current/algorithms/dijkstra-single-source/))
	- can set diff weights for diff relationship, and minimize the total weight
```cypher
// find the shortest path btw 2 nodes:
MATCH (start:Person {name: "Alice"}), (end:Person {name: "Bob"})
MATCH p = shortestPath((start)-[:KNOWS*]-(end))
RETURN p

// return all possible paths, in list format
MATCH (start:Person {name: "Alice"}), (end:Person {name: "Bob"})
MATCH p = allShortestPaths((start)-[:KNOWS*]-(end))
RETURN p
```
### Aggregation/Summary Stats
- TBD
