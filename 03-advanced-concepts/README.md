# Chapter 3: Advanced Concepts

## 3.1 DynamoDB Streams

- DynamoDB creates a stream of data containing a record of each change to an item
- The stream can be processed by AWS Lambda or other compute services
- Can be used to copy the data to a different store for analytics purposes

## 3.2 Time-to-live (TTL)

- Use TTL to automatically delete items after a certain amount of time
  - The amount of time is not precise
- Store a Unix timestamp of the desired expiration time in an attribute
- Tell DDB which attribute to use for expiry
- You don't have to set the TTL for every item in the table

## 3.3 Partitions

- Partitions are the core storage units underlying a DynamoDB table
- To support scale, the data is sharded across multiple server instances
- When a request comes in, the DDB request router uses the partition key in the request to determine which partition to use
  - It performs a hash function to indicate the server where the data is stored
- DynamoDB can add additional nodes as the data scales up
- DDB now uses *adaptive capacity* to dynamically assign throughput around to the different shards of the table

## 3.4 Consistency

- Consistency refers to whether a particular read operation recieves every write operation that took place before the read
- This is due to the workflow of how DynamoDB handles data
  - DDB shards the data across multiple partitions
  - When you write to DDB, a request router determines which node to write to based on the partition key
  - The primary node of a partition holds the canonical data for the items in that node
  - When a write comes in, the primary node commits the write and copies to one of two secondary nodes
    - This ensures that the wriet is saved in the event of a loss of a single node
  - The node responds to the client that the write was successful, then asynchronously replicates the write to a third node
  - These secondary nodes can serve read requests to alleviate pressure on the primary node
  - The issue is that it's possible for a read request for data that has been written will come through and be handled by a secondary node that has not received the write data from the primary node yet
  - With strong consistency, it's guaranteed that a read will reflect all writes that took place previously
  - With eventual consistency, it's possible that a write has not been replicated to the read node yet
  - You can control this by passing the `ConsistentRead=True` in the API call
  - A strongly-consistent read uses double the read capacity of an eventually consistent read
  - A GSI will always be eventually consistent

## 3.5 DynamoDB Limits

### 3.5.1 Item Size Limits

- A single DDB item is limited to 400KB

### 3.5.2 Query and Scan Request Size Limits

- `Query` and `Scan` are the two "get many" API operations
- They will read a maximum of 1MB from your table
- This limit is applied before any filter expressions are evaluated
- If you need more, you can paginate

### 3.5.3 Partition Throughput Limits

- A single partition has a miximum of 3000 RCUs or 1000 WCUs (per second)
- Thus, 3000 reads per second (or 1000 writes per second) *for a given partition key* to hit the limit

### 3.5.4 Item Collection Limits

- For LSI indexes with item collections, a single item collection cannot be more than 10GB
- Obvs, this is if you have many, many items with the same partition key
- Your writes will get rejected if you run out of space
- Does not apply to GSIs
  - They are automatically distributed under the hood

## 3.6 Overloading Keys and Indexes

- This is a data modeling concept more than a feature of DDB
- You can include different types of entities in one table
- Use generic names for the partition key and sort key (e.g. "PK" and "SK")
  - This allows you to be flexibile and store multiple entities in the same table
- Use prefixes for the values for PK and SK
  - This helps avoid overlap between different item types