## Table of Contents
- [Disk Storage](#Disk-Storage)
- [Query Language](#Query-Language)
- [Data Explorer](#Data-Explorer)
- [SDK / Library Support](#SDK-/-Library-Support)
- [Security](#Security)
- [Other Advantages](#Other-Advantages)
- [Operation / Maintainence](#Operation-/-Maintainence)
- [Consulting Contact](#Consulting-Contact)


## Disk Storage
### Neo4j connect data as it stores it - All linked lists
- Properties are stored as a linked list, each holding a key and value and pointing to the next property. Each node or relationship references its first property record. 
- The Nodes also reference the first relationship in its relationship chain. 
- Each Relationship references its start and end node. It also references the previous and next relationship record for the start and end node respectively as doubly linked list.
![](2019-08-27-14-57-49.png)

### Cosmos DB - All JSON documents
```json
{
    "id": "5cc4d110-1081-4f53-ac37-75b9ed7c68c4",
    "label": "person",
    "type": "vertex",
    "properties": {
      "Email": [
        {
          "id": "900e65ff-f194-463c-bdd4-6fbe17f48fc2",
          "value": "ChristieC@M365x791395.OnMicrosoft.com"
        }
      ],
      "Address": [
        {
          "id": "ac75af63-a49b-4ec2-a588-6d75609df14d",
          "value": "9257 Towne Center Dr., Suite 400"
        }
      ],
      "Birthday": [
        {
          "id": "d9afa4b9-1382-4107-9ef6-cddf1c5290f9",
          "value": "2019-07-19T00:00:00Z"
        }
      ],
```

## Query Language 
### Usage (both are easy to read and learn)
- Cypher is SQL like - You declare what you want to see, and the engine figures out how to get that data for you. 
```
MATCH p = (e:Employee {EmployeeId: 858})-[:Manages*]->(EmployeeUnder) 
where EmployeeUnder.Title = "Software Engineer"
return count(p), order by length(p)
```

- Gremlin - you specify how to traverse the graph
```
g.V("18df261b")
    .repeat(out("manages"))
    .emit()
    .outE("employedAt")
    .groupCount()
    .by("Title") 
```
### Performance (unclear)

- [This stack overflow disucssion](https://stackoverflow.com/questions/13824962/neo4j-cypher-vs-gremlin-query-language)  has a lot contradict opinions, and opinions seem to changed from 2012 to now.
> "Cypher query may go wild into the graph in a wrong direction first. I have not done much with Gremlin, but I could imagine you get much more execution control with Gremlin."

> "Use cypher for query and gremlin for traversal. You will see the response timing yourself."

> "When all I want is to query data, I go with Cypher as it is more readable and easier to maintain. Gremlin is the fallback when a limitation is reached."

- [This article](https://dzone.com/articles/neo4j-client-now-supports) says Gremlin is efficient in simple query, but Cyper is efficient in complex traversals where back tracking is required.

## Data Explorer
- Custom node coloring, sizing
- Show properties values when hovered
- Frequent queries management
- Json view, download as CSV
![](2019-08-27-16-49-09.png)

## SDK / Library Support
### C# and .NET
#### Neo4j .NET Client
```java
User users = client.Cypher
    .Match("(p:Person)")
    .Return(p => p.As<User>())
    .Results.ToList();
```
- Support type casting
- Support fluent API
#### Neo4j .NET Driver
```java
List<User> users = new List<User>();
using (var session = _driver.Session())
{
    session.ReadTransaction(tx =>
    {
        var result = tx.Run("MATCH (p:Person) RETURN p");
        foreach(var record in result)
        {
            var nodeProps = JsonConvert.SerializeObject(record[0].As<INode>().Properties);
            users.Add(JsonConvert.DeserializeObject<User>(nodeProps));
        }
    });
}
```
- No support type casting, need to de-serialize on our own
- Query is a string in source code

#### Gremlin .NET Driver
```java
var graph = new Graph();
var g = graph.Traversal().WithRemote(new DriverRemoteConnection(new GremlinClient(_server)));

return g.V().hasLabel("person").has("age", gt(40)).ToList()
```
- Gremlin has fluent API, but cosmosDB does not support it yet, public preview Dec, 2019

```java
public async Task<PersonVertex> GetPersonByID(string id)
{
    var response = (await ExecuteReadQuery<dynamic>($"g.V('{id}').as('person').out('hasSkill').store('skills').in().has('id', '{id}').in('manages').as('manager').select('person', 'skills', 'manager').limit(1)"))[0];
    PersonVertex person = JsonConvert.DeserializeObject<PersonVertex>(response["person"]);
}
```
- No support type casting, need to de-serialize on our own
- Query is a string in source code
- We talk to Gremlin directly as additional layer to query CosmosDB as a graph, but if our CosmosDB run into issues, Gremlin layer might also abstract the error return by CosmosDB, which hinder trouble shooting.

### Spark
#### Write graph to Neo4j - Clean syntax
```scala
Neo4jDataFrame.mergeEdgeList(sc, peopleSkillsDF, ("People", Seq("PersonId")),("HasSkill", Seq("Strength")),("Skills", Seq("SkillId", "Skill")))
```

#### Write graph to CosmosDB - No Gremlin support
- Need to construct the required GraphSON schema on our own (not a big deal)
- Spark notebook doesn't know anything about gremlin, it just write it as GraphSON schema
```scala
def toCosmosDBEdges(g: GraphFrame, labelColumn: String, partitionKey: String = "") : DataFrame = {
  var dfEdges = g.edges
  
  if (!partitionKey.isEmpty()) {
    dfEdges = dfEdges.alias("e")
      .join(g.vertices.alias("sv"), $"e.src" === $"sv.id")
      .join(g.vertices.alias("dv"), $"e.dst" === $"dv.id")
      .selectExpr("e.*", "sv." + partitionKey, "dv." + partitionKey + " AS _sinkPartition")
  }
  
  dfEdges = dfEdges
    .withColumn("id", udfUrlEncode(concat_ws("_", $"src", col(labelColumn), $"dst")))
    .withColumn("_isEdge", lit(true))
    .withColumn("_vertexId", udfUrlEncode($"src"))
    .withColumn("_sink", udfUrlEncode($"dst"))
    .withColumnRenamed(labelColumn, "label")
    .drop("src", "dst")
  
  dfEdges
}
```
#### Read graph from Neo4j in spark to perform analysis - Comprehensive support
- You can declare the queries or patterns you want to use, ``cypher(query,[params])``, ``nodes(query,[params])``, ``rels(query,[params])`` as direct query, or use pattern ``pattern(("Label1","prop1",("REL","prop"),("Label2","prop2))``

- Define load size
``partitions(n)``, ``batch(size)``, ``rows(count)`` for parallelism

- Choose to load graph as these datatypes:
``loadRowRdd``, ``loadNodeRdds``, ``loadRelRdd``, ``loadRdd[T]``
``loadDataFrame``, ``loadDataFrame(schema)``
``loadGraph[VD,ED]``
``loadGraphFrame[VD,ED]``

- The perform analytical spark operations

#### Read graph from Cosmos DB in spark - No Gremlin support
- No Gremlin support for reading CosmosDB document as a graph

### Python
- Neo4j Python Driver + ``Py2Neo`` Python Client

## Role Level Security (Enterprice Edition)
### Property-level access control 
> "You can use role-based, database-wide, property blacklists to limit which properties a user can read. (Deprecated)". New methods for this functionality will be provided in an upcoming release

The user user-1 will be unable to read the property propertyA.
```
dbms.security.property_level.enabled=true
dbms.security.property_level.blacklist=\
  roleX=propertyA;\
  roleY=propertyB,propertyC

CALL dbms.security.addRoleToUser('roleX', 'user-1')
```

### SubGraph Access Control
> It is possible to restrict a user’s read or write to specified portions of the graph. For example, a user can be allowed to read, but not write, nodes with specific labels and relationships with certain types.
```
CALL dbms.security.createRole('accounting')

CALL dbms.security.addRoleToUser('accounting', 'billsmith')
```

## Other Advantages
- Community Support
- Official Documentation
- ACID constrain (write safe)
- Indexing
- APOC






## Neo4j Cluster Operation and Maintainence 
- Azure Market Place

![](2019-08-27-17-41-23.png)

casaul cluster
replicating db across three cluster so scale out for query they all see the same data with ACID gurantee ...
master handle the write operation
spread workload 可以自己設定
user management is per machine, but with halin can apply to all machines
configuration diff across cluster
show all response
cost
ramp up 
learning curve
maintainability
Administration: Hanlin
snapshot cache

Neo4j underlying structure storage
Monitoring, 
Causal Cluster Architecture and Upgrade a Causal Cluster

Some of the features offered by Azure Cosmos DB are:

Fully managed with 99.99% Availability SLA
Elastically and highly scalable (both throughput and storage)
Predictable low latency: <10ms @ P99 reads and <15ms @ P99 fully-indexed writes
On the other hand, Neo4j provides the following key features:
intuitive, using a graph model for data representation
reliable, with full ACID transactions
durable and fast, using a custom disk-based, native storage engine
Neo4j
Managing the cluster on our own
Halin  

### Backup
### Real Time Monitoring
Halin
Halin won't do real time alert

Grafana
if you want something like 7 days or a month, you should use an external monitoring solution such as datadog, stackdriver, grafana, or a similar product. Rather than getting metrics into your browser, they should be pushed to a third-party service where the metrics are stored in a specialized timeseries database, and then you can set up alerts and so on.
### PRICE AND PERFORMANCE

## Consulting Contact 
- Shawn.Elliott@microsoft.com (Cloud architect appear in Neo4j-Azure video)
- Brian.Sherwin@microsoft.com did a comparison of Neo4j vs Cosmos DB Gremlin according to Hassaan.Ahmed@microsoft.com's article.