# Sharding data collections with MongoDB


## Objective

The objective of this exercise is to illustrate the concept of [sharding][1], a database partitioning technique for storing large data collections across multiple database servers. For this purpose, you will work with [MongoDB][2], a _document oriented database management system_ supporting different sharding strategies.


## Requirements
+ [Docker Toolbox][3] (or [Docker CE][4] if supported by your OS)
+ [Hands-on material][5] (github repository)


## Installation
Download and unzip the [hands-on material][5].

Open a terminal and execute the following instructions inside the ```dxlab-sharding``` directory:

```bash
docker-compose pull
docker-compose build
```

This will download the software required for this exercise (as [docker images][7]): 
+ [MongoDB][14]
+ [Jupyter Notebook][6]
You can verify this by executing the following instruction:

```bash
docker images
```

That’s all. You are ready for the exercise.

## Introduction to MongoDB & Sharded Clusters

### MongoDB

As stated before, MongoDB is a [document oriented database management system][8] (i.e., you insert/get/update/delete documents).

In MongoDB, documents are represented using [JSON][9], a _text-based data format_. The following example illustrates a JSON document containing information about the city of Washigton:

```javascript
{ 
  "city" : "WASHINGTON", 
  "loc" : [ -69.384237, 44.269281 ], 
  "pop" : 1261, 
  "state" : "ME"
}
```

Internally, MongoDB organizes documents in **collections**, and collections in **databases**. A mongo database may contain multiple collections.

MongoDB does not support SQL; queries are expressed as programs (see [examples][15]). 


### Sharded Clusters

MongoDB supports sharding via a _sharded cluster_. A sharded cluster is composed of the following components: 
+ **Shard**(s): store data.
+ **Query router**(s): redirect queries/operations to the appropriate shard (or shards).
+ **Config server**(s): store cluster’s metadata. Query router(s) uses this metadata to select appropriate shards.

![cluster-architecture]

In MongoDB, sharding is enabled on a _per-collection_ basis. When enabled, MongoDB distributes the documents of the collection across the shards (i.e. mongo servers) of a cluster.

![sharded-collection]

## Preparing a Sharded Cluster  
For simplicity, you will start working with a _partially configured cluster_. The cluster will be composed of:
+ 1 query router
+ 1 config server
+ 3 shards
Start the cluster as follows:

```bash
# Start docker containers
docker-compose up -d
```
Cluster components run inside a virtual network as [docker containers][10]. You can list containers (and their IPs) as follows: 

```bash
# List containers 
docker ps

# List containers IPs
docker network inspect dxlabsharding_default
```

### Cluster configuration

As shown in the figure, only the **query router** and **config server** are configured as part of a cluster. Let's complete the cluster by adding a shard server. 

![docker-cluster-img]

Enter the cluster environment:

```bash 
docker-compose run --rm cli
```

Connect to the **query router**:

```bash
mongo --host queryrouter.docker
```

Add **shard 1** to cluster:

```javascript 
// Change database 
use admin

// Add shard to cluster 
db.runCommand({ 
  addShard: "shard1.docker", 
  name: "shard1" 
})
```
You can verify the **cluster status** as follows:

```javascript 
sh.status()
```

Disconnect now from query router (```ctr + c```). 


> #### Q2. Which is the important information reported by _sh.status()_?


### Inserting and Querying data

At this point, your cluster does not contain any data (i.e., you have not created any collection or database). The following example illustrates how to import data to your cluster:

```bash
mongoimport \
  --host queryrouter.docker \
  --db mydb \
  --collection cities \
  --file ./cities.txt
```
This imports the content of **cities.txt** into collection **cities** of database **mydb**. Verify this by connecting to the query router: 

```bash
# Connect to query router
mongo --host queryrouter.docker
```

Explore the database:

```javascript
// List databases
show dbs

// Change to database mydb
use mydb 

// List collections
show collections
```

Run some queries over **cities**:

```javascript
// SELECT * FROM Cities
db.cities.find().pretty()

// SELECT COUNT(*) FROM Cities
db.cities.count()
```

> #### Q2. Describe in natural language the content of the collection 

## Sharding Database Collections

Recall that in MongoDB, sharding is enabled on a _per-collection basis_. When enabled, MongoDB uses the [shard key][11] (an attribute that must exists in every document), for partitioning and distributing the collection documents across the cluster. 
 
MongoDB uses two kinds of partitioning strategies:

+ **Range based partitioning**: data is partitioned into intervals ```[min, max]``` called [chunks][12]. Chunks limits depend on the domain of the shard key.

+ **Hash based partitioning**: data is partitioned using a [hash function][13].

In what follows, you will experiment with both kind of strategies.

### Range Based Partitioning

Create a new collection:

```javascript
db.createCollection("cities1")
show collections
```

Enable sharding on this collection using attribute **state** as shard key:

```javascript
sh.enableSharding("mydb") 
sh.shardCollection("mydb.cities1", { "state": 1} )
```

Verify the cluster state:

```javascript
sh.status()
```

> #### Q3. How many chunks did you create? Which are their associated ranges? Include a screen copy of the results of the command in your answer to support your answer.


Populate the collection using _mydb.cities_:

```javascript
db.cities.find().forEach( 
   function(doc) {
      db.cities1.insert(doc); 
   }
)
```

Verify the new cluster state:

```javascript
sh.status()
```

> #### Q4. How many chunks are there now? Which are their associated ranges? Which changes can you identify in particular? Include a screen copy of the results of the command in your answer to support your answer.


### Hash Based Partitioning

Create a new collection:

```javascript
db.createCollection("cities2")
show collections
```

Enable sharding on this collection using also attribute **state** as shard key:

```javascript
sh.enableSharding("mydb") 
sh.shardCollection(
  "mydb.cities2", { "state": "hashed" } 
)
```

Verify the cluster state:

```javascript
sh.status()
```

> #### Q5. How many chunks did you create? What differences do you see with respect to the same task in the range sharding strategy? Include a screen copy of the results of the command in your answer to support your answer. 


Populate the collection using _mydb.cities_:

```javascript
db.cities.find().forEach( 
   function(doc) {
      db.cities2.insert(doc); 
   }
)
```

Verify the new cluster state:

```javascript
sh.status()
```

> #### Q6. How many chunks are there now? Include a screen copy of the results of the command in your answer to support your answer. Compare the result with respect to the range sharding. Include a screen copy of the results of the command in your answer to support your answer.


## Balancing Data Across Shards 

Independently of the selected partition strategy, when a shard server has too many chunks c(ompared to other shards in the cluster), MongoDB automatically redistributes the chunks across shards. This process is called [cluster balancing][16]. 

![sharding-migrating]

Let's trigger the balancing process by adding more shards to your cluster.

```javascript
use admin
db.runCommand( { addShard: "shard2.docker", name: "shard2" } )
db.runCommand( { addShard: "shard3.docker", name: "shard3" } ) 

```

Wait a few seconds. Then, check again the cluster status and compare to the previous result.

```javascript
sh.status()
```

> #### Q7. Draw the new configuration of the cluster and label each element (router, config server and shards) with its corresponding IP.


## Guiding Partitioning Using Tags

MongoDB also supports tagging a _range_ of shard key values. Some advantages of using tags are: 

+ Isolation of data.
+ Colocation of shards in geographical related regions.

In what follows, you will use tags for isolating some cities into specific shard servers.

Associate tags to shards: 

```javascript
sh.addShardTag("shard1", "CA") 
sh.addShardTag("shard2", "NY") 
sh.addShardTag("shard3", "Others")
```

Create, populate and enable sharding on a new collection:

```javascript
use mydb

db.createCollection("cities3") 
sh.shardCollection("mydb.cities3", { "state": 1} ) 

db.cities.find().forEach( 
   function(doc) { 
      db.cities3.insert(doc); 
   } 
)
```

Define and associate _key ranges_ ```[from, to)``` to shards:

```javascript
sh.addTagRange("mydb.cities3", { state: MinKey }, { state: "CA" }, "Others") 
sh.addTagRange("mydb.cities3", { state: "CA" }, { state: "CA_" }, "CA") 
sh.addTagRange("mydb.cities3", { state: "CA_" }, { state: "NY" }, "Others") 
sh.addTagRange("mydb.cities3", { state: "NY" }, { state: "NY_" }, "NY") 
sh.addTagRange("mydb.cities3", { state: "NY_" }, { state: MaxKey }, "Others")
```

Analize new cluster state:


```javascript
sh.status()
```

> #### Q8. Analyze the results and explain the logic behind this tagging strategy. Connect to the shard that contains the data about California, and count the documents. Do the same operation with the other shards. Is the sharded data collection complete with respect to initial one? Are shards orthogonal?


Not sure about data distribution? Try the  [sharding notebook](http://localhost:8888/notebooks/geo-visualization.ipynb).

## Uninstalling

Disconnect from query router (```ctr + c```).

Disconnect from cli (```ctr + d```).

Stop containers and remove docker images:

```bash
docker-compose down
docker rmi -f mongo:3.0 jupyter/base-notebook dxlabsharding_jupyter 
```

[1]: https://en.wikipedia.org/wiki/Shard_(database_architecture)
[2]: https://en.wikipedia.org/wiki/MongoDB
[3]: https://www.docker.com/products/docker-toolbox
[4]: https://store.docker.com/search?offering=community&q=&type=edition
[5]: https://github.com/javieraespinosa/dxlab-sharding
[6]: https://jupyter.org
[7]: https://www.docker.com/what-docker
[8]: https://en.wikipedia.org/wiki/Document-oriented_database
[9]: http://json.org/
[10]: https://www.docker.com/what-container
[11]: https://docs.mongodb.com/manual/core/sharding-shard-key/#shard-key
[12]: https://docs.mongodb.com/manual/core/sharding-data-partitioning/
[13]: https://en.wikipedia.org/wiki/Hash_function
[14]: https://www.mongodb.com
[15]: https://docs.mongodb.com/manual/reference/sql-comparison/
[16]: https://docs.mongodb.com/manual/core/sharding-balancer-administration/

[sharded-collection]: http://espinosa-oviedo.com/big-data-visualization/wp-content/uploads/sites/7/2017/10/sharded-collection.png

[cluster-architecture]: http://espinosa-oviedo.com/big-data-visualization/wp-content/uploads/sites/7/2017/10/sharded-cluster-production-architecture.png

[docker-cluster-img]: http://espinosa-oviedo.com/big-data-visualization/wp-content/uploads/sites/7/2017/10/docker-cluster.png

[sharding-migrating]: http://espinosa-oviedo.com/big-data-visualization/wp-content/uploads/sites/7/2017/10/sharding-migrating.png

