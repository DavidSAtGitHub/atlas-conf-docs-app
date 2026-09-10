# Storing Data in DynamoDB

DynamoDB is Amazon's fully managed NoSQL database service. It stores data as **items** (similar to rows) inside **tables**, where each item is a collection of **attributes** (similar to columns). Unlike traditional databases, items in the same table don't need to share the same attributes — the schema is flexible per item.

## Core Concepts

**Tables, Items, and Attributes** A table holds items, and each item is essentially a JSON-like document with key-value pairs. An attribute can be a scalar (string, number, boolean), a set, a list, or a nested map. Only the primary key attributes must be defined upfront; everything else is schemaless.

**Primary Keys** Every item is uniquely identified by a primary key, which comes in two forms:

- _Partition key (simple)_ — a single attribute whose hash determines which physical partition stores the item.
- _Partition key + sort key (composite)_ — items sharing a partition key are stored together, sorted by the sort key. This enables efficient range queries within a partition.

**Partitioning** DynamoDB automatically distributes data across partitions based on the partition key's hash. This is what makes it horizontally scalable, but it also means your access patterns must align with your keys — a poorly chosen partition key creates "hot partitions" that bottleneck performance.

**Secondary Indexes** Since you can only query efficiently by primary key, DynamoDB offers two index types to support alternate access patterns:

- _Global Secondary Index (GSI)_ — a different partition/sort key combination, with its own throughput.
- _Local Secondary Index (LSI)_ — same partition key but a different sort key, defined at table creation.

**Capacity Modes** You either provision read/write capacity units in advance or use on-demand mode where DynamoDB scales automatically and bills per request.

**Consistency** Reads default to _eventually consistent_ (cheaper, faster) but can be requested as _strongly consistent_. Writes are always strongly consistent within a region.

**Streams and TTL** DynamoDB Streams capture item-level changes for downstream processing (e.g., triggering Lambda functions). Time-to-Live lets items auto-expire.

## How It Differs From a Relational Database

The biggest mental shift is that DynamoDB is designed around **access patterns**, not around normalized entities. In a relational database, you model your data first (tables, foreign keys, normal forms) and then write whatever queries you need — the engine figures out joins, indexes, and execution plans. In DynamoDB, you start by listing every query your application will run, then design tables and keys specifically to serve those queries cheaply.

**Schema** is rigid in SQL (every row matches the table definition) and flexible in DynamoDB (each item can have different attributes).

**Joins** don't exist in DynamoDB. You either denormalize (duplicate data across items so a single query returns everything) or use a _single-table design_ where multiple entity types coexist in one table, distinguished by key prefixes. Relational databases instead split entities into separate tables and join them at query time.

**Query language** in SQL is declarative and expressive — you can filter, group, aggregate, and join arbitrarily. DynamoDB only supports `GetItem`, `Query` (against a key), and `Scan` (full-table read, expensive and to be avoided). There's no `GROUP BY`, no native aggregations, no ad-hoc analytics.

**Transactions** exist in both, but DynamoDB transactions are limited (up to 100 items, single region, more expensive per operation). Relational databases handle complex multi-table ACID transactions natively.

**Scaling** is where DynamoDB shines. Relational databases scale vertically (bigger machine) or with painful sharding work. DynamoDB scales horizontally and transparently — partitions are added automatically as data and traffic grow, and latency stays in the single-digit milliseconds at virtually any size.

**Cost model** also differs: SQL databases typically charge for the instance, regardless of usage. DynamoDB charges per read/write operation and storage, which is great for spiky or unpredictable workloads but can get expensive for heavy scan-style analytics.

## When Each Fits

Relational databases remain the better choice when you need flexible ad-hoc queries, complex joins, strong relational integrity, or analytical reporting. DynamoDB fits when you have well-known access patterns, need predictable low-latency at scale, or are building serverless applications where operational overhead must be minimized — think session stores, shopping carts, IoT telemetry, gaming leaderboards, or user profile lookups.

A useful way to remember it: _SQL optimizes for query flexibility, DynamoDB optimizes for predictable performance at scale._