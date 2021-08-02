# Chapter 6: DynamoDB Expressions

- Expressions are statements that operate on your items
- Similar to SQL
- There are five types of expressions in DynamoDB
  1. **Key Condition Expressions** 
      - Used in Query operations to describe which items to return
      - For read-based operations
  1. **Filter Expressions** 
      - Used in `Query` and `Scan` operations to describe which items to return *after finding items that match the Key Condition Expression*
      - For read-based operations
  1. **Projection Expressions** 
      - Used in all read operations to describe which attributes to return on the items that were read
      - For read-based operations
  1. **Condition Expressions** 
      - Used in write operations to assert the existing condition of an item before writing to it
      - For write-based operations
  1. **Update Expressions** 
      - Used in the `UpdateItem` operation to describe the nature of the update to an existing item
      - For update operations only

## 6.1 Key Condition Expressions

- This is the main Expression you'll use
- Used on every Query to select items
- It's how you specify the Partition key and Sort key
- **Can only be used on the Primary Key attributes**
- Can use operators like `=`, `<`, `>`, `BETWEEN` etc on the sort key
  - The sort key is sorted according to whether the type is a number or a string

## 6.2 Filter Expressions

- Helpful but limited
- Can be used in `Query` and `Scan` operations
- It removes items response that match your `KeyConditionExpression` but not the filter condition
- **Can be applied to ANY attribute, not just the primary key**

### Potential Hazards

When you perform a Query or Scan, DDB performs the following steps

1. Read items from table
1. If there's a FilterExpression, remove items that don't match
1. Return your items

#### BUT!

- Query and Scan operations only return a maximum of 1MB of data
- *This limit is applied in step 1, before the FilterExpression is applied*
- You might have to page through many empty requests to get the data you're looking for

### Useful Scenarios

- Reducing response payload size
  - If you're just going to filter it on the client anyway...
- Better validation of the TTL expiry
  - The TTL expiration/removal isn't instant
  - You can filter the collection to ensure that you don't get items that should be expired, but haven't been removed yet by DDB

## 6.3 Projection Expressions

- Allows you to filter out or include specific item attributes
- Similar idea to MongoDB projections
- Like filter expressions, they're evaluated after the items are read from the table and the 1MB limit is reached
- If you have large-sized attributes that you're excluding from the projection, you might still have to do some paginating to get the result set you're looking for
  - If this is the case, you might want to create a secondary index with a custom projection that only copies certain attributes into the index

## 6.4 Condition Expressions

- For write operations
- Describes the required state of the item before the update will be performed
- If the ConditionExpression evaluates to false, the operation will be cancelled
- Possible uses:
  - Avoid overwriting an existing item when using PutItem
  - Prevent an UpdateItem operation from violating some constraint (e.g. AccountBalance < 0>)
  - Assert that a given user is the owner of an item before updating or deleting
- Includes the operators `>`, `<`, `=`, `BETWEEN`
- Includes functions as well
  - `attribute_exists()`
    - Assert that a given attribute exists
  - `attribute_not_exists()`
    - Assert that a given attribute doesn't exist
  - `attribute_type()`
    - Assert that the attribute is of the given type
  - `begins_with()`
    - Assert that the value begins with the given substring
  - `contains()`
    - Assert that the string value contains a given substring, *OR*
    - Assert that the Set value contains a given value
  - `size()`
    - Assert that the value is of a given size
    - Varies depending on the type of the value
      - For Lists, Maps, Sets - The number of elements in the set
      - For Strings - The length of the string
      - For Binary - The number of bytes in the value

### 6.4.1 Preventing overwrites or checking for uniqueness

- The PutItem operation will insert an item into the table, overwriting any existing item with the same primary key
- You can prevent this by using the `attribute_not_exists()` operator
- Example: You have a "Users" table and you don't want to overwrite an existing user

```
result = dynamodb.put_item(
  TableName='Users',
  Item={
    "Username": { "S": "cooldude123" },
    "Name": { "S": "Jim Smith" },
    "Created": { "S": datetime.datetime.now().isoformat() }
  },
  ConditionExpression: "attribute_not_exists(#username)",
  ExpressionAttributeNames={
    "#username": "Username"
  }
)
```

- This will only allow the write if no items exist with the specified username
  
### 6.4.2 Limiting in-progress items

- Maybe you have a table that is tracking the number of items in a group
  - For example a set of background workers that are processing a queue
  - You don't want more than 10 jobs at a time running in a particular state
- Example

```
result = dynamodb.update_item(
  TableName='WorkQueue',
  Key={
    "PK": { "S": "Tracker" }
  },
  ConditionExpression: "size(#inprogress) <= 10",
  UpdateExpression="Add #inprogress :id",
  ExpressionAttributeNames={
    "#inprogress": "InProgress"
  },
  ExpressionAttributeValues={
    ":id": { "SS": ["some-job-id" ] }
  }
)
```

- This uses a string set attribute to keep track of the 10 or fewer jobs in progress

### 6.4.3 Asserting user permissions on an item

- Maybe you have items that can be read by people that don't have permissions to write them
- Before making an update, you want to ensure the user has admin permissions
- Example

```
result = dynamodb.update_item(
  TableName='',
  Key={ "PK": { "S": 'Amazon' } }
  ConditionExpression="contains(#a, :user)",
  UpdateExpression="Set #st :type",
  ExpressionAttributeNames={
    "#a": "Admins",
    "#st": "SubscriptionType"
  },
  ExpressionAttributeValues={
    ":user": { "S": 'Jeff Bezos' },
    ":type": { "S": 'Pro' }
  }
)
```

- This uses a string set of admins to hold the list of admins
- It checks that the list contains the username of the current user before making the update

### 6.4.4 Checks across multiple items

- Similar to above, but in this case you have to check a separate item for the list of admins
- We can use `TransactWriteItem` to make up to 10 operations in a single request
  - These can be write operations like `PutItem`, `DeleteItem`, etc
  - Or they can be a `ConditionCheck`
- We can use a `ConditionCheck` to assert details about an item
- In this example, we want to delete their company's account
  - We need to ensure that they are an admin first
  - The admin users are stored in a separate item
- Example:

```
result = dynamodb.transact_write_items(
  TransactItems=[
    {
      "ConditionCheck": {
        "Key": { "PK": { "S": "Admins#<orgId>" } }
        "TableName": "SaasApp",
        ConditionExpression: "contains(#a, :user)",
        ExpressionAttributeNames: {
          "#a": "Admins",
        },
        ExpressionAttributeValues={
          ":user": { "S": <username> }
        }
      }
    },
    {
      "Delete": {
        "Key": { "PK": { "S": "Billing<orgId>" } },
        "TableName: "SaasApp"
      }
    }
  ]
)
```

- First it checks that the user is an Admin
- Then it runs the delete operation
## 6.5 Update Expressions

- `UpdateItem` differs from `PutItem`
  - With UpdateItem, attributes you don't specify will not be altered
  - With PutItem, the item is completely replaced by the new payload
    - Any existing attributes not in the payload will be removed
- With UpdateItem, you express the changes in the update using the following 4 verbs:
  1. **SET**
      - Used for adding or overwriting an attribute on an item
      - Can also be used to add or subtract from an attribute with a Number type
  1. **REMOVE**
      - Deletes an attribute from an item
      - Remove a nested property from a list or map type value
  1. **ADD**
      - Add to a number attribute
      - Add an element to a set
  1. **DELETE**
      - Remove an element from a set
- You can use multiple operations for a single verb
  - State the verb once, and use commas to separate the clauses
  - e.g. `UpdateExpression="SET Name = :name, UpdatedAt = :updatedAt"`
- You can use any combination of these verbs in a single update
  - Don't need to use commas to separate verbs
  - e.g. `UpdateExpression="SET UpdatedAt = :updatedAt REMOVE InProgress"`


### 6.5.1 Updating or setting an attribute on an item

- Example

```
result = dynamodb.update_item(
  TableName='Users',
  Key={
    "Username": { "S": "python_fan" }
  }
  UpdateExpression="SET #picture :url",
  ExpressionAttributeNames={
    "#picture": "ProfilePictureUrl"
  },
  ExpressionAttributeValues={
    ":url": { "S": <http://....> }
  }
)
```

### 6.5.2 Deleting an attribute from an item

- Example - Removing the profile picture from the example above

```
result = dynamodb.update_item(
  TableName='Users',
  Key={
    "Username": { "S": "python_fan" }
  }
  UpdateExpression="REMOVE #picture"
  ExpressionAttributeNames={
    "#picture": { "S": "ProfilePictureUrl" }
  }
)
```

### 6.5.3 Incrementing a numeric value

- Example - Increment a counter

```
result = dynamodb.update_item(
  TableName='PageViews',
  Key={
    "Page": { "S": "ContactUsPage" }
  }
  UpdateExpression="SET #views = #views + :inc",
  ExpressionAttributeNames={
    "#views": "PageViews"
  }, 
  ExpressionAttributeValues={
    ":inc": { "N": "1" }
  }
)
```

### 6.5.4 Adding a nested property

- Example - Updating the value of a phone number stored in a Map attribute

```
result = dynamodb.update_item(
  TableName='Users',
  Key={
    "Username": { "S": "python_fan" }
  }
  UpdateExpression="SET #phone.#mobile :cell",
  ExpressionAttributeNames={
    "#phone": "PhoneNumbers",
    "#mobile": "MobileNumber"
  },
  ExpressionAttributeValues={
    ":cell": { "S": <some phone number> }
  }
)
```

This will set the mobile number if you have an object like this as the Map

```
PhoneNumbers = {
  HomeNumber: "212-555-1212"
}
```

### 6.5.5 Adding and removing from a set

- Example - Adding a new administrator to the set

```
result = dynamodb.update_item(
  TableName="SaasApp",
  Key={ "PK": { "S": "Admins#<orgId>" } }
  UpdateExpression="ADD #a :user",
  ExpressionAttributeNames={
    "#a": "Admins"
  },
  ExpressionAttributeValues={
    ":user": { "SS": [ <someUser> ] }
  }
)
```
- The REMOVE operation is the same, with `REMOVE` in place of `ADD`
- You can ADD/REMOVE multiple values using the array in ExpressionAttributeValues