# Chapter 2: Core Concepts

## 2.1 Basic Vocabulary

### 2.1.1 Table

- A table is similar to a RDBMS table or MongoDB collection
- A grouping of records that conceptually belong together
- Unlike a RDBMS, the records in DynamoDB can and often do represent different types of entities
- The table is *schemaless* meaning that the table will not enforce the shape of your data

### 2.1.2 Item

- A single record in a DynamoDB table
- A row/document in other DBs

### 2.1.3 Attributes

- An item is made up of attributes
- An attribute is similar to a column in a RDBMS table
  - Or a property on an object
- When you write an item, each attribute is given a specific type

#### Attribute Types

- Scalars
  - A single simple value
    - String
    - Number
    - Binary
    - Boolean
    - Null
- Complex
  - Groupings with arbitrary attributes nested
    - Lists - An array
    - Maps - An object
- Sets
  - Represents multiple unique values
  - Each element must be the same type
  - Three set types
    - String sets
    - Number sets
    - Binary sets 

### 2.1.4 Primary Keys

- DynamoDB is schemaless, but it is not without structure
- All tables must have a primary key declared
- The key can be a "simple" single value or a "composite" key, consisting of two values
- Each item in the table must include the primary key
  - If you try to write an item without it, it will be rejected
- Each item is uniquely identifiable by its primary key
  - If you try to write an item using a primary key that already exists, it will (by default) overwrite the existing item or reject if configured to do so
- Primary keys are the primary driver for data modeling in DynamodDB

### 2.1.5 Secondary Indexes

- Primary keys are the main drivers of the access patterns in DDB
- Secondary indexes allow you to query on different aspects of the data
- When you create a secondary index, you specify the primary keys for the secondary index
- AWS copies the items from the main table to the secondary index

## 2.2 A Deeper Look: Primary keys and secondary indexes

### 2.2.1 Types of primary keys

- **Simple Primary Keys** - consist of a single element called a partition key
- **Composite Primary Keys** - consist of two elements, a partition key and a sort key
- A simple PK only allows you to fetch one item at a time
- Composite keys allow you to get all items with the same partition key
  - Good for handling relations between items in the data and retrieving multiple items at once
### 2.2.2 Types of secondary indexes

- When defining a secondary index, you specify the key schema of the index
- The key schema is similar to the primary key
  - You specify the partition key and sort key (if desired)
- Two kinds of secondary indexes
  - **Local Secondary Index (LSI)** 
  - **Global Secondary Index (GSI)**
- LSI uses the same partition key as your table's primary key but a different sort key
  - Nice to filter by a different property
  - You can use strongly-consistent or eventually-consistent (default) reads
  - Must be created when table is created
- GSI uses any attributes you want for the partition key and sort key
  - More flexible
  - Need to provision additional throughput for the secondary index
    - This is because the read/write throughput for the index is separate from the table
  - Can only use eventually-consistent reads
  - Can be created after the table

## 2.3 The importance of item collections

- **Item Collections** - Groups of items that share the same partition key in either the base table or a GSI
- All items with the same partiion key will be kept on the same storage node
  - This is important for performance reasons
- Allows you to use the `Query` operation to retrieve multiple items