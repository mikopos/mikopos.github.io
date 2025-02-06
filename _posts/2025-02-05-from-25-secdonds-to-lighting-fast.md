---
layout: post
section-type: post
has-comments: true
title: From 50-Seconds Queries to Lightning-Fast - My Epic Journey Optimizing MongoDB
category: tech
tags: [ "mongoDB", "optimization", "large", "datasets", "java", "springBoot" ]
---

[//]: # (# From 25-Second Queries to Lightning-Fast: My Advanced Journey Optimizing MongoDB Aggregation, Paging, and Sorting)


> **TL;DR**: I transformed a painfully slow MongoDB query (about 50 seconds) into a near-instant (sub-10ms) operation. This article showcases the entire process—from devising compound indexes, streamlining aggregation pipelines, and handling pagination and sorting, to exploring in-memory versus database-based sorting and employing Java virtual threads for efficient concurrency. If you’ve faced similar bottlenecks or timeouts, read on.

---

## Table of Contents

1. [Prologue – Realizing the Problem](#prologue--realizing-the-problem)
2. [Chapter 1 – Early Challenges](#chapter-1--early-challenges)
3. [Chapter 2 – Key Concepts: Indexing, \$sort, \$skip, and $limit](#chapter-2--key-concepts-indexing-sort-skip-and-limit)
4. [Chapter 3 – Experimenting with Batching](#chapter-3--experimenting-with-batching)
5. [Chapter 4 – Searching for the Ideal Approach](#chapter-4--searching-for-the-ideal-approach)
6. [Chapter 5 – Full-Text Queries and TextCriteria](#chapter-5--full-text-queries-and-textcriteria)
7. [Chapter 6 – Java Concurrency: From Fixed Threads to Virtual Threads](#chapter-6--java-concurrency-from-fixed-threads-to-virtual-threads)
8. [Chapter 7 – The Final Optimized Solution](#chapter-7--the-final-optimized-solution)
9. [Epilogue – Performance Gains & Future Directions](#epilogue--performance-gains--future-directions)
10. [Appendix – Visuals & Code Examples](#appendix--visuals--code-examples)

---

## Prologue – Realizing the Problem

It all started when the application went live and the datasets from some thousands enlarged to many millions. 
Initially, I believed that the queries would execute quickly, however user feedback indicated significant 
timeouts. In practice, the aggregation pipeline was taking nearly 50 seconds to return a single paginated 
set of results even for the last day. While some indexes were in place, the query plan consistently resorted
to a massive **`COLLSCAN`** (collection scan), highlighting several issues:

1. The existing indexes did not align well with the filtering or sorting.
2. `$skip + $limit` pagination can degrade performance without proper indexing.
3. Sorting is only efficient when MongoDB leverages the appropriate index.

This set the stage for an in-depth exploration of MongoDB and Spring Data, specifically focusing on
performance improvements and concurrency strategies.

---

## Chapter 1 – Early Challenges

### Scenario
We employed an aggregator pipeline of this form:

```javascript
db.users.aggregate([
  { "$match": { "$and": [...] } },
  { "$sort": { "date.dateTime": -1 } },
  { "$skip": 0 },
  { "$limit": 50 }
]).explain("executionStats");
```

Although it looked straightforward—especially given that `date.dateTime` was indexed—
`explain("executionStats")` painted a different picture:

```json
"stage": "SORT",
"nReturned": 50,
"executionTimeMillisEstimate": 50957
```

At almost 51 seconds, the pipeline scanned thousands of documents, applying a sort in memory, and 
then skipped to the relevant offset. This indicated that the index on `date.dateTime` alone was 
insufficient or mismatched.

### Lesson #1
Whenever your **sort** fields do not perfectly match the underlying index, MongoDB defaults to an 
in-memory sort, which is by far computationally expensive and explains the protracted runtime.

---

## Chapter 2 – Key Concepts: Indexing, \$sort, \$skip, and $limit

### $sort
- **Index-based**: If your sort field is covered by an index, MongoDB avoids an in-memory sort.
- **In-memory**: If no relevant index exists, MongoDB allocates resources to perform a separate sorting phase, often scanning large portions of the collection.

### \$skip and $limit
- **$limit**: Typically low cost if the data is already sorted using an index.
- **$skip**: Can cause substantial overhead if MongoDB must read many documents to skip them, unless the sort is handled directly through the index.

### Compound Index
A multi-field index is invaluable if you filter or sort on multiple fields. For instance:

```javascript
db.users.createIndex(
  { "userId": 1, "status": 1, "date.dateTime": -1 },
  { name: "userId_status_date.dateTime_1" }
);
```

By aligning the fields used for filtering (`userId`, `status`) and sorting (`date.dateTime`), you can vastly improve performance.

---

## Chapter 3 – Experimenting with Batching

After addressing the issue with indexes, I assumed that apart from `pagination` `batching` will 
reduce overhead by retrieving data in segments. so I ended up with an implementation like the below one:

```java
int batchSize = 10000;
long totalProcessed = 0;

while (true) {
    Aggregation batchAgg = Aggregation.newAggregation(
        match,
        sort,
        Aggregation.skip(totalProcessed),
        Aggregation.limit(batchSize),
        project
    );
    List<Doc> batchResults = template.aggregate(batchAgg, ...).getMappedResults();
    // ...
    totalProcessed += batchResults.size();
}
```

**Advantages**:
- Potentially mitigates memory usage for massive datasets.
- Suitable for data migrations or large-scale batch analytics.

**Drawbacks**:
- Introduces complexity.
- Fails to fix underlying index mismatches if each sub-query still scans large swaths of data.

Ultimately, batching proved not to be the primary solution for a query returning just 50 results. 
Correctly configured indexes remained the decisive factor.

---

## Chapter 4 – Searching for the Ideal Approach

I evaluated various techniques in the twilight of the `batching` failure:

1. **Aggregation Pipeline**
    - Allows `$match`, `$sort`, `$skip`, `$limit`, `$project`, and `$lookup`.
    - A misaligned or incomplete index results in an expensive sort step.

2. **Streaming with `find()`**
    - Simpler in many respects, especially with `.hint()` to force an index.
    - Less flexible for sophisticated transformations.

3. **In-memory Sorting**
    - Fetch results and sort them in Java if the user wants a sort that is not covered by existing indexes.
    - Practical only for relatively small datasets or infrequent usage.

### Lesson #2
Choose the approach that aligns with your data volume and transformation needs. For large datasets 
or performance-critical queries, ensure `$sort` leverages a compound index.

---

## Chapter 5 – Full-Text Queries and TextCriteria

Then came the request for a **free-text search** on text field and I have to introduce
Spring Data MongoDB **`TextCriteria`**:

```java
TextCriteria textCriteria = TextCriteria.forDefaultLanguage().matching("hello");
```

However, `TextCriteria` implements `CriteriaDefinition`, not `Criteria`, meaning it cannot directly 
plug into methods like `new Criteria().andOperator(...)`. So I must either:

- Apply two `$match` stages, one for standard filters and one for text search.
- Carefully merge them if you want a single `$match`.

You also need a **text index**:

```javascript
db.users.createIndex({ "description": "text"});
```


### Lesson #3
Text-based queries require a dedicated text index. Merging them with other filter criteria demands 
careful handling, since `TextCriteria` is not the same as `Criteria`.

---

## Chapter 6 – Java Concurrency: From Fixed Threads to Virtual Threads

Our query logic involved two parallel tasks:
1. The aggregator pipeline.
2. A count query to determine the total records.

Initially, we used:

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
Future<List<Doc>> resultsPromise = ...;
Future<Long> countPromise = ...;
```

While functional, scaling it up increased concerns about thread blocking. 
The advent of **Project Loom** (Java 19+ preview, or Java 21 stable) offered **virtual threads**:

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // aggregator + count in parallel
}
```

Virtual threads consume fewer resources, allowing you to maintain a direct, blocking style without 
saturating a limited pool of OS threads. A very good choice for containerized applications.

### Lesson #4
On Java 21 or the Java 19 preview, virtual threads can greatly simplify concurrency. 
They handle large volumes of blocking tasks without the traditional overhead of a large pool of
platform threads.

---

## Chapter 7 – The Final Optimized Solution

Ultimately, the following methods converged into our cohesive fix:

1. **Compound Index**
    - Incorporates all filter fields (`userId`, `status`) plus the sort field (`date.dateTime`).
    - Example:
      ```javascript
      db.users.createIndex(
        { "userId": 1, "status": 1, "sdate.dateTime": -1 },
        { name: "userId_status_date.dateTime_1" }
      );
      ```

2. **Aggregation Pipeline**
    - `$match` to filter, `$sort` utilizing the index, `$skip` and `$limit` for pagination, `$project` for specifying fields.

3. **Parallel Execution**
    - Using either a fixed thread pool or virtual threads to run both the aggregator and the count query simultaneously.

4. **Separate TextCriteria**
    - If free-text search is required, handle it in a distinct `$match` stage or merge carefully.

5. **In-memory Sorting**
    - Used only if a user requests sorting on an unindexed field; otherwise, rely on the index.

**Result**: We reduced a 50+ second operation to under 20ms or even single-digit milliseconds on 
large datasets. The dreaded `COLLSCAN` or in-memory `SORT` was effectively avoided.

---

## Epilogue – Performance Gains & Future Directions

We benchmarked:
- **Original aggregator**: ~50 seconds.
- **With a proper compound index**: ~8 ms.
- **Adding parallel aggregator + count**: near-instant for small result sets.
- **Text search**: minimal overhead if the text index is well-defined.

**Possible Next Steps**:
- **Partial indexes** if you only care about recent or frequently accessed data.
- **Collation** for case-insensitive sorting.
- **Sharding** once the dataset outgrows a single node.
- **Advanced concurrency** (further exploiting virtual threads).

**Core Message**: Rely on `explain("executionStats")` to confirm the aggregator pipeline is properly 
indexed. A suitable compound index and an efficient pipeline can drastically enhance performance.

---

## Appendix – Visuals & Code Examples

### Diagram Suggestions

1. **Aggregator Pipeline Flow**
   ```
   [$match] -> [$sort] -> [$skip] -> [$limit] -> [results]
   ```
   Illustrate where MongoDB might do an `IXSCAN` or `COLLSCAN`.

2. **Compound Index**
   Demonstrate how `(userId, status, date.dateTime)` in one index covers both filtering and sorting.

3. **Virtual Threads**
   Sketch how multiple I/O-blocking tasks fit on a small pool of OS threads when using Loom.

### Sample Code: Parallel Aggregation & Count

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<List<Doc>> resultsPromise = executor.submit(() ->
        template.aggregate(aggregation, "users", Doc.class).getMappedResults()
    );

    Future<Long> countPromise = executor.submit(() ->
        template.count(query, Doc.class)
    );

    List<Doc> results = resultsPromise.get();
    long totalCount = countPromise.get();
    // Do something with results and totalCount
}
```

Here, both the aggregator and the count query run concurrently. This approach cuts waiting time in 
half for many scenarios.

---

## Final Words

What began as a routine query quickly escalated into a 50-second bottleneck. By leveraging compound 
indexes, concurrency (even via virtual threads), and occasionally in-memory sorting for edge cases, 
we achieved a robust and scalable solution.

If you find yourself dealing with `$sort` bottlenecks, heavy `$skip` usage, or text search integration 
dilemmas, I hope this guide helps you refine your indexing strategy, concurrency approach, and 
aggregator design. Best of luck, and may your queries always be fast!

