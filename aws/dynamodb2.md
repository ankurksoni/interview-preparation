**DynamoDB - Part 2: Pagination and Index-Based Querying (LSI & GSI)**

In this part, we will explore how pagination works in DynamoDB, and how to handle it effectively when using Local Secondary Indexes (LSI) and Global Secondary Indexes (GSI). We'll use a Books example to illustrate all key points.

---

## Table Schema: Books

**Table Name:** Books

**Primary Key:**

- Partition Key: book_id (string)
- Sort Key: publishedYear (number)

**Attributes:**

- title (string)
- author (string)
- genre (string)
- rating (number)

**LSI:**

- Index Name: LSI1
- Partition Key: book_id
- Sort Key: rating

**GSI:**

- Index Name: GSI1
- Partition Key: genre
- Sort Key: rating

---

## Pagination Basics

DynamoDB returns paginated responses when a query or scan request exceeds 1 MB in size. To retrieve more data, you use the `LastEvaluatedKey` from the previous response as the `ExclusiveStartKey` in the next request.

### Basic Pagination Query

```ts
const res1 = await dynamo.send(new QueryCommand({
  TableName: 'Books',
  KeyConditionExpression: 'book_id = :b',
  ExpressionAttributeValues: {
    ':b': { S: 'B101' }
  },
  Limit: 5
}));

const res2 = await dynamo.send(new QueryCommand({
  TableName: 'Books',
  KeyConditionExpression: 'book_id = :b',
  ExpressionAttributeValues: {
    ':b': { S: 'B101' }
  },
  Limit: 5,
  ExclusiveStartKey: res1.LastEvaluatedKey
}));
```

---

## Pagination with LSI

LSIs allow an alternate sort key for the same partition key. In our case, we want to get all versions of the same book sorted by rating instead of publishedYear.

### Query Using LSI

```ts
const res = await dynamo.send(new QueryCommand({
  TableName: 'Books',
  IndexName: 'LSI1',
  KeyConditionExpression: 'book_id = :b',
  ExpressionAttributeValues: {
    ':b': { S: 'B101' }
  },
  Limit: 3
}));
```

### Paginate on LSI

The LastEvaluatedKey will contain:

```json
{
  "book_id": { "S": "B101" },
  "rating": { "N": "4.5" }
}
```

Use it like this:

```ts
const nextPage = await dynamo.send(new QueryCommand({
  TableName: 'Books',
  IndexName: 'LSI1',
  KeyConditionExpression: 'book_id = :b',
  ExpressionAttributeValues: {
    ':b': { S: 'B101' }
  },
  Limit: 3,
  ExclusiveStartKey: res.LastEvaluatedKey
}));
```

---

## Pagination with GSI

GSIs support querying on completely different partition/sort keys. We use GSI1 to fetch books by genre and sort by rating.

### Query Using GSI

```ts
const res = await dynamo.send(new QueryCommand({
  TableName: 'Books',
  IndexName: 'GSI1',
  KeyConditionExpression: 'genre = :g',
  ExpressionAttributeValues: {
    ':g': { S: 'Fantasy' }
  },
  Limit: 4
}));
```

### LastEvaluatedKey Example for GSI

```json
{
  "genre": { "S": "Fantasy" },
  "rating": { "N": "4.2" },
  "book_id": { "S": "B123" }
}
```

Here, `book_id` is a projected key from the base table.

### Next Page Using GSI

```ts
const nextPage = await dynamo.send(new QueryCommand({
  TableName: 'Books',
  IndexName: 'GSI1',
  KeyConditionExpression: 'genre = :g',
  ExpressionAttributeValues: {
    ':g': { S: 'Fantasy' }
  },
  Limit: 4,
  ExclusiveStartKey: res.LastEvaluatedKey
}));
```

---

## Notes and Best Practices

- Always inspect the structure of `LastEvaluatedKey` before using it.
- Ensure all required keys are passed exactly when paginating.
- Be careful with `FilterExpression`, as it is applied after the pagination limit.
- Preferably `Query` over `Scan` for efficient pagination.

---

## Interview Q&A on DynamoDB Pagination, LSI, and GSI

Below are frequently asked interview questions on DynamoDB pagination and indexing, with clear labels for questions and answers:

## 🧠 Interview Q&A on DynamoDB Pagination, LSI, and GSI

**QUESTION:** What is the default data size limit for a single DynamoDB query or scan?  
**ANSWER:** 1 MB of data per request. If more data is available, DynamoDB returns a `LastEvaluatedKey` for pagination.

**QUESTION:** How does DynamoDB handle pagination?  
**ANSWER:** DynamoDB returns a `LastEvaluatedKey` when not all items fit in the response. This key can be used as `ExclusiveStartKey` in the next request to continue from where it left off.

**QUESTION:** What is the role of `LastEvaluatedKey`?  
**ANSWER:** It indicates the last item returned in the result. It is used to continue pagination in the next request.

**QUESTION:** How do you use `ExclusiveStartKey` in pagination?  
**ANSWER:** It is passed as a parameter to a query or scan request to continue reading from the last evaluated key.

**QUESTION:** What is the difference between Query and Scan in DynamoDB?  
**ANSWER:** Query searches items using a partition key and optional sort key; Scan reads every item in the table and filters results.

**QUESTION:** What happens if the response exceeds 1MB and you haven't specified a limit?  
**ANSWER:** DynamoDB truncates the response at 1MB and includes a `LastEvaluatedKey` to allow continuation.

**QUESTION:** What is the difference between GSI and LSI?  
**ANSWER:** GSI can have a completely different partition and sort key; LSI shares the same partition key as the base table but allows a different sort key.

**QUESTION:** When would you use an LSI over a GSI?  
**ANSWER:** When you want an alternative sort order for the same partition key and need strong consistency in reads.

**QUESTION:** Can you paginate through a GSI? How does the key structure affect it?  
**ANSWER:** Yes, you can paginate. The `LastEvaluatedKey` will include the GSI keys and any projected base table attributes.

**QUESTION:** What keys are included in `LastEvaluatedKey` for an LSI query?  
**ANSWER:** The partition key (from the base table) and the LSI sort key.

**QUESTION:** What keys are included in `LastEvaluatedKey` for a GSI query?  
**ANSWER:** GSI partition key, GSI sort key, and any projected primary key attributes.

**QUESTION:** How does a `FilterExpression` affect pagination?  
**ANSWER:** It is applied after the items are read, so fewer items than the `Limit` may be returned, and pagination may continue.

**QUESTION:** Does `Limit` apply before or after the filter expression?  
**ANSWER:** `Limit` applies before filtering, so filtering is done on the limited result set.

**QUESTION:** What issues can occur if you don't include all keys in `ExclusiveStartKey`?  
**ANSWER:** DynamoDB will throw a validation error, and the query will fail.

**QUESTION:** Can a query return no results even though `LastEvaluatedKey` is set? Why?  
**ANSWER:** Yes, if the items fetched are filtered out by `FilterExpression`.

**QUESTION:** How can you paginate results from a frontend application using DynamoDB?  
**ANSWER:** Store and use the `LastEvaluatedKey` as a cursor in frontend APIs to request the next page.

**QUESTION:** What are projected attributes in GSI, and why do they matter in pagination?  
**ANSWER:** Projected attributes are copied from the base table to the GSI and are required to reconstruct `LastEvaluatedKey`.

**QUESTION:** What performance trade-offs exist between using GSI and scanning?  
**ANSWER:** GSIs are much faster and more efficient for targeted queries. Scans are slower and consume more throughput.

**QUESTION:** How do you enforce strong consistency with LSI queries?  
**ANSWER:** LSI queries automatically support strong consistency if you set `ConsistentRead: true`.

**QUESTION:** Give a scenario where paginating through an LSI would be more beneficial than through the base table.  
**ANSWER:** When querying multiple versions of the same book (same `book_id`), sorted by rating instead of publishedYear.

...