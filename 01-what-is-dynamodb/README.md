# Chapter 1: What is DynamoDB?

## Misconceptions about DDB

### 1. DynamoDB is just a key-value store

DDB can handle relationships (1-m and m-m) just fine. It can also handle querying, filtering and pagination. Chapters 18-22 will cover more complex patterns and strategies.

### 2. DynamoDB doesn't scale

Lol who thinks this? Amazon uses it for all of their Tier 1 services. Lyft uses it as well as many other hyperscale applications. That's its main use case. People may think this due to the fact that it's possible to misuse it in a way that will make it less scalable, for example using Scan operations for access patterns, or placing all of your data into hot partitions.

### 3. DynamoDB is only for enormous scale

DDB has a very flexible pricing model that is extremely cheap for small scale applications. It also has several aspects that fit well within a serverless ecosystem such as the HTTP connection model, pay-per-use pricing and IAM permissions model

### 4. You can't use DynamoDB if your data model will change

It's true that it is very important to understand your application's access patterns before you begin modeling your table. However, this doesn't mean that you can't change your access patterns over time. 

You can make additive changes for many use cases, and there are strategies to modify existing records with migrations.

### 5. You don't neeed a schema when using DynamoDB

DDB and other NoSQL databases are often called "schemaless", and while DynamoDB doesn't enforce the schema, you still want to have schemas in your application otherwise you will go crazy.

---
## 1.1 Key properties of DDB

### DynamoDB is a fully-managed, NoSQL database provided by Amazon Web Services

"NoSQL" doesn't tell us what it is, only what it isn't

#### What it is: 

- Key value store
  - Basically a distributed hash table/dictionary/map
  - Each element is uniquely identifiable by a key
  - All get/update/delete operations based on the key
  - Fast, consistent performance regardless of data set size
  - Can only retrieve one record per request
- Wide-column store
  - Like a hash table where the value for each record is a B-tree
  - Allows you to perform range queries
  - Similar to a bookshelf full of phone books, each for a different city
    - "Give me all entries with names between A-C in Lincoln, NE"
- Provides nearly infinite scaling with no performance degradation
  - Most operations in single-digit milliseconds
  - Also provides an in-memory cache (DAX) which can provide sub-millisecond responses
  - Table size is virtually unbounded
  - Most importantly, response time does not degrade with table size
- HTTP connection model
  - All requests to DDB are sent as HTTP requests, instead of long-lived TCP connections
  - HTTP is slightly slower than a persistent TCP connection
    - But not when you factor in overhead of setting up the initial connection
    - Also, holding those connections requires DB server resources
  - HTTP allows for virtually unlimited throuput
    - As long as you pay for it
  - This connection model works really well for serverless/ephemeral compute
- IAM Authentication
  - Instead of username/password
  - Convenient when you're using AWS compute services
    - The AWS compute services can take on IAM Roles instead of managing secrets
  - Provides a granular permission system that is standard AWS workflow
  - Allows permissions for specific actions on tables (e.g. GetItem), so that we can easily enforce least privilege access (e.g splitting read and write side lambdas in CQRS)
- IAC friendly
  - Easy to specify the Table definitions in the infrastructure code
    - Defining indexes
    - Adding users
- Flexible pricing model
  - Usually you have to provision DB instances
    - At the very least, you have to provision RAM, CPU, disk, etc
  - With DynamoDB you specify the throughput using Read Capacity Units and Write Capacity Units
    - A Read Capacity Unit (RCU) is either:
      - One strongly consistent read per second
      - Two eventually-consistent reads per second
      - Up to 4KB in size for either
    - A Write Capacity Unit (WCU) is writing one item per second up to 1KB in size
  - This allows you to tweak/set your read and write throughput separately
  - You can scale dynamically up/down as needed throughout the day
  - You can also choose to use the On-Demand pricing model
    - You pay per-request rather than provisioning throughput capacity
    - The per-request price is higher than provisioned
    - It's still cheap if your scale is smaller
    - It can also be cheaper if workload is very spiky
  - You can switch between pricing models as well, starting with On-Demand and then switching to provisioned when you understand your traffic
- Change data capture with DynamoDB Streams
  - Transactional log of each write in the table!
  - Can access the log programmatically
  - Good for event-driven architectures
- Managed
  - It's managed

## 1.2 When to use DynamoDB

### Hyperscale Applications

- When RDBMS systems were initially developed, storage space was expensive, so relational DBs optimized for storage.
- Data normalization was the focus, to avoid duplicating information
- In the early 2000's storage cost dropped dramatically
- The internet changed the needs of application data storage
- Because of the scale of internet applications like 2004 Amazon.com Cyber Monday, RDBMSs were hitting a wall
  - The load from performing JOIN queries was too high
  - Strong Consistency at scale is expensive, and not always necessary
  - Consistency also required single server instances to scale

### Hyper-Ephemeral Compute (Serverless)

- HTTP connection allows better scaling 
- No long initialization like TCP connections
  - The connection isn't likely to last because the compute is ephemeral
  - This negates the advantage of connection pools
- DynamoDB uses a shared Request Router to direct data to a specific shard
- No need to set up network partitioning/VPCs
- Uses IAM to protect access

### Other Use Cases

#### Most OLTP applications

- OLTP applications have end users writing small amounts of data with high frequency
  - DynamoDB features fast, consistent performance
- This is in contrast to OLAP applications, where you're processing analysis of data sets
  - A better fit would be to use DynamoDB change capture to send the data to a more suitable store

#### Caching

- Can be used to store the results of frequently-accessed queries from different DBs or other operations
- Not quite as fast as Redis, but fast

#### Simple Data Models

- Key-value lookups and the like
- E.g. session data
- The next step above a cache

## 1.3 Comparisons to other databases

### DynamoDB vs Relational DBs

- RDBMS benefits:
  - Well-known
  - Strong tooling
  - Basically universal
  - Support in many web application frameworks
  - More flexible data model
- DynamoDB benefits:
  - Scale much larger than RDBMSs
  - More compatible with ephemeral compute

### DynamoDB vs MongoDB

- MongoDB benefits
  - More flexible data model
  - More index types (text, geospatial, multi-key indexes)
- DynamoDB benefits
  - MongoDB does not scale as well when using these features
  - Managed service more integrated with AWS than MongoDB or DocumentDB

### DynamoDB vs Apache Cassandra

- Cassandra Benefits
  - Pretty similar to DynamoDB
  - Wide-column data model
- DynamoDB benefits
  - It's better than Cassandra
  - Managed service
  - Scalability