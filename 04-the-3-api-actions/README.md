# Chapter 4: The Three API Action Types

- The API for DynamoDB is small but powerful
- Split into 3 categories
  - Item-based actions
    - Operates on specific items
  - Queries
    - Operates on an item collection
  - Scans
    - Operates on an entire table
## 4.1 Item-based actions

1. **GetItem** - Reads a single item from a table
2. **PutItem** - Writes an item to the table, overwriting any existing item with the same key
3. **UpdateItem** - Updates an item. Can create it if it doesn't exist, can add/remove/modify attributes on an existing item
4. **DeleteItem** - Removes an item from the table

#### Rules for Item-Based Actions

- The full primary key must be specified in the request
- All updates must use an item-based action
- All item-based actions must be performed on the main table, not a secondary index

### 4.1.1 Batch Actions and Transactions

- Used for reading and writing multiple DynamoDB items in a single request
- Can still be considered "Item-Based" even though they operate on multiple items
  - You have to specify the exact items on which you want to operate
- The separate requests are split up when the API call hits the DDB router
  - Just saves you from having to make multiple API calls

#### Differences Between Batch and Transactional Actions

- In a batch API request, the reads or writes will succeed or fail independently
  - The failure of one request will not affect the rest of the batch
- In a transactional API action, all reads/writes fail or succeed together
  - The failure to write one item with cause the other writes to be rolled back

## 4.2 Query

- Allows you to retrieve multiple items with the same partition key
- This is powerful when modeling and retrieving data that includes relations
- For example, the following table
  - Table Name: ActorTracker
  - Primary Key
    - Partition Key: Actor Name
    - Sort Key: Movie Name
  - Attributes
    - Role (character name)
    - Year
    - Genre
- You can perform a query by partition key to get all the movies for a given actor
- You can use the sort key to get the movies for an actor with names between A-E for instance
- You may want to query by movie though, so you can create a GSI with the keys flipped
  - Partition key: Movie Name
  - Sort Key: Actor Name
- Now you can query by Movie and sort/filter by Actor

## 4.3 Scan

- Scan returns ALL of the records in the table
- If the table is larger than 1MB, you will have to paginate to get all results
  - The first request will include a pagination key, which you will return to get the next page of data
- Times when you might want to use a scan
  - When you have a very small table
  - When you're exporting all of the data from your table
  - When you have a specific sparse secondary index that expects a scan (discussed in CH 19)

## 4.4 How DynamoDB enforces efficient queries

- The API is designed to limit "bad queries"
  - "Bad queries" are queries that degrade in performance at scale
- Partitions are used to shard data across multiple machines
- Sharding is done on the basis of the partition key
- No matter how large the table becomes, searching using the partition key is a constant time operation to find the item or item collection
- Query operations only allow you to fetch a contiguous block of items within a particular collection
  - You can do `>=`, `<=`, `begins_with()` or `between`, but not `contains()` or `ends_with()`
  - Like a phonebook, it's easy to find items that start with a given string or are between two listings, but not to find names that end with "son" for example
- You are limited to returning 1MB of data per operation
  - To get more, you must paginate