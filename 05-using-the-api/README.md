# Chapter 5: Using the API

## 5.1 Expression names and values

#### Sample Query

```javascript
const items = client.query({
  TableName: 'ActorTracker',
  KeyConditionExpression: '#actor = :actor AND #movie BETWEEN :a AND :m',
  ExpressionAttributeNames: {
    '#actor': 'Actor',
    '#movie': 'Movie'
  },
  ExpressionAttributeValues: {
    ':actor': { 'S': 'Tom Hanks' },
    ':a': { 'S': 'A' },
    ':m': { 'S': 'M' },
  }
});
```

### KeyConditionExpression

- Two types of placeholders
  - symbol starting with `#` (e.g. `#actor`)
    - Called "Expression Attribute Names"
    - Represent the name of the attribute that you're evaluating in the request
    - Not required to use them
      - Unless your attribute name is also a DynamoDB reserved word
    - May as well just use them
    - There are 573 reserved words (e.g. "Bucket", "Name", "Timestamp", "Count")
  - symbol starting with `:` (e.g. `:actor`)
    - Called "Expression Attribute Value"
    - Used to represent the value of the attribute that you are evaluating
    - Similar to parameterized queries in SQL
    - When defining the value, you must also specify the type because it is not inferred

## 5.2 Don't use an ORM

- There are multiple ways to model the data
- Sometimes you have multiple object types in the same table
- Fetching objects is dependent on the design of the primary key
- There's no long SQL queries to abstract away
- Use the DocumentClient instead
  - Just has a slightly trimmed down api with less boilerplate
- Also use DynamoDB Toolbox by Jeremy Daly
  - Helps you define entity types and map them to a table
  - Simplify boilerplate

## 5.3 Understand the optional properties on individual requests

There are optional properties you can add to your requests:

- ConsistentRead
- ScanIndexForward
- ReturnValues
- ReturnConsumedCapacity
- ReturnItemCollectionMetrics

### 5.3.1 ConsistentRead

- Can specify a strongly consistent read by setting `ConsistentRead: true` in the query
- The default is eventually consistent
- This property is available on the following API operations
  - GetItem
  - BatchGetItem
  - Query
  - Scan
- FYI: Strongly consistent reads use double the capacity of eventually consistent reads
- You can use strong consistency on an LSI but NOT a GSI
  - Those are always eventually consistent, as the data is replicated into the index

### 5.3.2 ScanIndexForward

- Only available on the `Query` operation
- Controls the direction that DDB reads your sort key
  - Ascending (default) = `True`
  - Descending = `False`

### 5.3.3 ReturnValues

- Allows you to configure the payload that is sent as the return value for write operations
  - Lets you return specified attributes for create/update/delete operations
- Applies to the following operations:
  - `PutItem`
  - `UpdateItem`
  - `DeleteItem`
  - `TransactionWriteItem`
    - For this operation, the param is `ReturnValuesOnConditionCheckFailure` instead of `ReturnValues`
- By default, DDB doesn't return any attributes for these operations
- Examples
  - A counter you want to get the new value from
  - You might want to keep the attributes of an item you deleted, so you can tell the client what it was

#### Available `ReturnValues` options

- **NONE** - Return no attributes from the item (Default)
- **ALL_OLD** - Return *all* the attributes from the item as it looked *before* the operation was applied
- **UPDATED_OLD** - Return *only the updated* attributes from the item as it looked *before* the operation was applied
- **ALL_NEW** - Return *all* the attributes from the item as it looks *after* the operation was applied
- **UPDATED_NEW** - Return *only the updated* attributes from the item as it looks *after* the operation was applied

### 5.3.4 ReturnConsumedCapacity

- Returns data about the capacity units that were used by the request
- Can specify the following options
  - **ReturnConsumedCapacity=INDEXES**
    - Includes detailed information on the Table and any indexes
  - **ReturnConsumedCapacity=TOTAL**
    - Only returns the summary of the capacity consumed

```JSON
{
  "ConsumedCapacity": {
    "TableName": "Ur Table",
    "CapacityUnits": 123,
    "ReadCapacityUnits": 123,
    "WriteCapacityUnits": 123,
    "GitHub": {
      "ReadCapacityUnits": 123,
      "WriteCapacityUnits": 123,
      "CapacityUnits": 123
    },
    "GlobalSecondaryIndexes": {
      "GSI1": {
        "ReadCapacityUnits": 123,
        "WriteCapacityUnits": 123,
        "CapacityUnits": 123
      }
    }
  }
}
```

- Useful when you're load testing your table to understand which access patterns are heavy
- Or if you're passing on the cost of the table to your customers

### 5.3.5 ReturnItemCollectionMetrics

- Gives you the size of an Item Collection
- An LSI cannot exceed 10GB
- You can configure the writes to give you the `ReturnItemCollectionMetrics`
  - This can alert you in advance
- If you don't use an LSI, this isn't really that useful

```JSON
{
  "ItemCollectionMetrics": {
    "ItemCollectionKey": {
      "S": "USER#djheru"
    },
    "SizeEstimateRangeGB": [
      123
    ]
  }
}
```