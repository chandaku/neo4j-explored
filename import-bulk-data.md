# Import Bulk Data neo4j
With Neo4j 3.5
Using Json to bulk import data, following commands help

https://medium.com/@faaizhussain/loading-json-in-neo4j-1e3e396fd219

`WITH "file:///sample-topology.json" as url 
CALL apoc.load.json(url) YIELD value  
UNWIND keys(value) AS key  
RETURN key, apoc.meta.type(value[key]);`

You may find this exception

##### Failed to invoke procedure `apoc.load.json`: Caused by: java.lang.RuntimeException: Import from files not enabled,
set apoc.import.file.enabled=true in your neo4j.conf


Query To Load complete JSON file nodes

Create Nodes 

`WITH "file:///sample-topology.json" as url 
CALL apoc.load.json(url) YIELD value 
UNWIND  value.nodeInstances as data
MERGE (nodeItem:Node {id: data.id, importKey:data.importKey, nodeTemplateName: data.nodeTemplateName, nodeTemplateId:data.nodeTemplateId, name: data.name})`

Create Relations

`WITH "file:///sample-topology.json" as url 
CALL apoc.load.json(url) YIELD value 
UNWIND value.relationshipInstances as data
MATCH
  (a:Node),
  (b:Node)
WHERE a.id = data.srcId AND b.id = data.dstId
MERGE (b)-[r:CONNECTED {type:data.relationship.relationshipType}]->(a)
RETURN type(r)`


Delete All relations Neo4J
`start r=relationship(*) 
delete r;`

In cypher3.5, start is deprecated.

You can use this cypher to delete all relationships

`match ()-[r]->() delete r;`

### With above JSON, a process remain continued to load 250MB of data for hours so it did not worth much and following is admin-import utility with neo4j that helps to import huge data very quickly


Fist of all find something to convert data to a csv file and I used jq for the purpose

Started using jq to convert json feeds to CSV

` jq -r '["id","importKey","name","nodeTemplateId", "nodeTemplateName"],(.[].nodeInstances|flatten[]|[.id, .importKey, .name, .nodeTemplateId, .nodeTemplateName])|@csv' sample-topology_1.json >all_nodes.csv`

`LOAD CSV WITH HEADERS FROM 'file:///all_nodes.csv' AS row
WITH row WHERE row.id IS NOT NULL
CREATE (nodeItem:Node {id: row.id, importKey:row.importKey, nodeTemplateName: row.nodeTemplateName, nodeTemplateId:row.nodeTemplateId, name: row.name})`



`jq -r '["srcId","destId","relationType","percentageRelationship", "weightedRelationship"],(.[].relationshipInstances|flatten[]|[.srcId, .dstId, .relationship.relationshipType, .relationship.percentageRelationship, .relationship.weightedRelationship])|@csv' sample-topology_1.json >all_relations.csv`


`LOAD CSV WITH HEADERS FROM 'file:///all_relations.csv' AS row
MERGE (nodeItem:Node {id: row.id, importKey:row.importKey, nodeTemplateName: row.nodeTemplateName, nodeTemplateId:row.nodeTemplateId, name: row.name})`

version-neo 4.4
`bin/neo4j-admin import --database=sie --nodes=/import/all_nodes.csv`

version-neo5.6.0

Install JQ if you don't have 


`bin/neo4j-admin database import full --nodes=/import/all_nodes.csv sie`

Test JQ command
`jq -r '[.[].nodeInstances[]]|unique_by(.id)' sample-topology.json`

Following JQ command will generate unique nodes information CSV from json fed file
`jq -r '["id:ID(Node)","importKey","name","nodeTemplateId", "nodeTemplateName",":LABEL"],([.[].nodeInstances[]]|unique_by(.id)[]|[.id, .importKey, .name, .nodeTemplateId, .nodeTemplateName, "Node"])|@csv' sample-topology_1.json >all_nodes.csv`

Following JQ command will generate relations of nodes information CSV from json fed file

`jq -r '["id:START_ID(Node)","id:END_ID(Node)",":TYPE"],(.[].relationshipInstances|flatten[]|[.srcId, .dstId, .relationship.relationshipType])|@csv' sample-topology_1.json >all_relations.csv`

Import nodes and relations in neo4j from admin import tool
`bin/neo4j-admin database import full --nodes=/import/all_nodes.csv --relationships=/import/all_relations.csv sie`

You might see that sie database is not created in UI and you will need to create manually from UI, then delete folder inside database/sie and transaction/sie

After this do import again as above and it may show data 
neo4j stop
neo4j start


https://neo4j.com/docs/operations-manual/current/tutorial/neo4j-admin-import/
#### Some blog references:

https://neo4j.com/blog/bulk-data-import-neo4j-3-0/

https://neo4j.com/blog/import-10m-stack-overflow-questions/

https://neo4j.com/docs/operations-manual/4.0/tutorial/neo4j-admin-import/

https://neo4j.com/docs/operations-manual/5/tutorial/neo4j-admin-import/

https://neo4j.com/docs/status-codes/current/errors/all-errors/?utm_source=GPPC&utm_campaign=*NA%20-%20Search%20-%20Tier%200&utm_content=187497217113&utm_term=neo4j&gclid=EAIaIQobChMIkOvI6PXh2AIVgrfACh2NSQGfEAAYASAAEgJYx_D_BwE

#### To update nodes attribute use pattern matching and setting data for nodes

MATCH (n:Node {id: 'ee31d9cc-533c-49d2-91aa-2883f642f9c6'})-[r:Required]-(m:Node)
SET m.state = 'Impacted'
return m, r, n