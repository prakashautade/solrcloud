# [SolrCloud](https://lucene.apache.org/solr/guide/7_5/solrcloud.html)
  - Apache Solr has the capability to setup a cluster of Solr servers. This cluster is called as SolrCloud.
# Why SolrCloud ?
  - Distribute index across multiple machines to overcome the physical disk limit.
  - Near Real Time (NRT) search and incremental indexing.
  - Query distribution and load balancing.
  - Fault tolerant.
# SolrCloud concepts
<img src="https://github.com/prakashautade/solrcloud/blob/master/SolrCloud.png" alt="SolrCloud" width="600" height="375">
<img src="https://github.com/prakashautade/solrcloud/blob/master/SolrCloudDetail.png" alt="SolrCloudDetail" width="600" height="375">

### Collection
- Collection is logical set of documents and its indices.
### Shards
- A Shard is a logical partition of the collection. It contains a subset of documents from the collection.
- When collection is too large for one node, We can break it up and store it in multiple shards.
- Every document in a collection is contained in exactly one Shard.
        Which shard contains each document in a collection depends on the overall "Sharding" strategy for that collection.
        For example, We might have a collection where the "country" field of each document determines which shard it is part of so documents from the same country are co-located.
### Leaders and Replicas
- We can keep the physical copy of shard, this is called as replica.
- In SolrCloud there are no masters or slaves. Instead, every shard consists of at least one physical replica, exactly one of which is a leader.
- Leaders are automatically elected
- If a leader goes down, one of the other replicas is automatically elected as the new leader.
### Core
- Core is the physical set of documents and associated indices
- It provides single replica
### Node
- A node is an instance of the Solr, running on a application server (eg Tomcat or Jetty).
- It may host zero, one, or many cores, which may be part of one or many distinct collections.
- It coordinates with other nodes via ZooKeeper.
- It can transparently proxy requests to other nodes as needed, and combines query results.
### Zookeeper
- Configuration management
- Zookeepers are central repository for solrcloud configuration
- You can consider it as distributed filesystem which can be accessed by all solr nodes in the cluster.
- So if you change any config file you just need to inform or upload it to zookeeper and not on every node in the cluster.
- Leader election
- One more important responsibility of zookeeper is to keep an eye on all solr nodes in the cluster
- If leader goes down zookeeper perform leader election and select one of the replica as leader.

### How SolrCloud works ?
#### Document indexing
- When a document is sent to a Solr node for indexing, the system first determines which Shard that document belongs to Then which node is currently hosting the leader for that shard.
- The document is then forwarded to the current leader for indexing.
- The leader forwards the update to all of the other replicas.
#### Document search
- When a Solr node receives a search request.
- The request is routed behind the scenes to a replica of a shard that is part of the collection being searched.
- The chosen replica acts as an aggregator:
- It creates internal requests to randomly chosen replicas of every shard in the collection.
- Coordinates the responses.
- Issues any subsequent internal requests as needed (for example, to refine facets values, or request additional stored fields)
- Constructs the final response for the client.
#### Write Side Fault Tolerance
- SolrCloud is designed to replicate documents to ensure redundancy for your data, and enable you to send update requests to any node in the cluster.
- That node will determine if it hosts the leader for the appropriate shard, and if not it will forward the request to the the leader, which will then forward it to all existing replicas.
- If the leader goes down, another replica can take its place.
- Recovery
  - A Transaction Log is created for each node so that every change to content or organization is noted.
  - The log is used to determine which content in the node should be included in a replica.
  - When a new replica is created, it refers to the Leader and the Transaction Log to know which content to include. If it fails, it retries.
  - Since the Transaction Log consists of a record of updates, it allows for more robust indexing because it includes redoing the uncommitted updates if indexing is interrupted.
  - If a leader goes down, it may have sent requests to some replicas and not others.
  - So when a new potential leader is identified, it runs a synch process against the other replicas. If this is successful, everything should be consistent, the leader registers as active, and normal actions proceed.
  - If a replica is too far out of sync, the system asks for a full replication/replay-based recovery.
  - If an update fails because cores are reloading schemas and some have finished but others have not, the leader tells the nodes that the update failed and starts the recovery procedure.
  - This architecture enables you to be certain that your data can be recovered in the event of a disaster, even if you are using Near Real Time Searching.
#### Read Side Fault Tolerance
- In a SolrCloud cluster each individual node load balances read requests across all the replicas in a collection.
- You still need a load balancer on the 'outside' that talks to the cluster.
- Or you need a smart client which understands how to read and interact with Solr’s metadata in ZooKeeper and only requests the ZooKeeper ensemble’s address to start discovering to which nodes it should send requests. (Solr provides a smart Java SolrJ client called CloudSolrClient.)
- Even if some nodes in the cluster are offline or unreachable, a Solr node will be able to correctly respond to a search request as long as it can communicate with at least one replica of every shard, or one replica of every relevant shard if the user limited the search via the shards or _route_ parameters.
- The more replicas there are of every shard, the more likely that the Solr cluster will be able to handle search results in the event of node failures.
