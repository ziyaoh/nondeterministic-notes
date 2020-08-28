# Grokking the System Design Interview

This is my note studing [Grokking the System Design Interview](https://www.educative.io/courses/grokking-the-system-design-interview)

## Basics

### Load Balancing

A load balancer spreads the traffic across a cluster of servers to prevent single point of failure and improve responsiveness and availability. It could be place in front of basically any service, database.

A load balancer needs to perform health check on backend servers, and only route traffic to those healthy ones.

We should add redundant load balancers to prevent a LB itself to become a single point of failure. One way to achieve this is to use anycast.

### Caching at Scale

Multiple servers each with their own cache doesn't perform well as expected, since the same request might go to different servers. We could use a
- global cache
- distributed cache

Content Distribution Network (CDN) is a kind of cache that comes into play for sites serving large amounts of **static media**.

cache update policy
- write-through cache

    Update in both cache and db before replying to client.

    No data loss, consistency guaranteed, high latency for write

- write-back cache

    Sync update in cache only, async update to db in batch (or when the cached item if about to be evicted).

    Low latency for write; risk of data loss in case cache crashes

- write-around cache

    Sync update to db only, somehow forces a cache miss on this key for the next read.

    Not flooding the cache with data that may not subsequently be re-read; higher latency to read a recently updated value

### Data Partitioning

- horizontal partitioning (sharding)

    different rows in different tables

- vertical partitioning

    partition data by logic

- directory based partitioning

    decouple partitioning schema and application code with a lookup service

potential problems
- joins and denormalization
- referential integrity
- rebalancing in case data or traffic distribution not uniform

### Indexes

### Proxies

Proxies are used to filter requests, log requests, or sometimes transform requests (by adding/removing headers, encrypting/decrypting, or compressing a resource). Another advantage of a proxy server is that its cache can serve a lot of requests. If multiple clients access a particular resource, the proxy server can cache it and serve it to all the clients without going to the remote server.

### Redundancy and Replication

### SQL vs NoSQL

Examples of NoSQL
- Key-Value Store: Redis, Dynamo
- Document Database: MongoDB
- Wide-Column Database: Cassandra, HBase
- Grph Database: Neo4J

high level differences:
- Storage

    SQL stores data in tables

    NoSQL has different data models

- Schema

    SQL has schema

    NoSQL is schemaless, or dynamic to change


- Querying

    SQL databases use SQL

    some NoSQL also supports SQL querying, others use UnQL

- Scalability

    SQL databases are vertically scalable, horizontal scaling is hard

    NoSQL is easily horizontally scalable

- ACID

    vast majority of SQL databases are ACID compliant

    most of NoSQL sacrifice ACID for performance and scalability

### Long-Polling vs WebSockets vs Server-Sent Events

Polling & Long Polling works by client actively sends requests to retrive latest update from server. Polling is when server responds ASAP, and Long Polling is when server delays the response if there's no current update.

WebSockets is a peer-to-peer protocol on top of TCP, so that once established, the server and client can exchange data in both directions at any time.

Server-Sent Events (SSEs) is a persistent and long-term connection so that server can actively send data to client.

## TinyURL

### Requirements
- given a URL, service should generate a shorter and unique alias
- when user access a short link, we should redirect them to the original link
- users can pick a custom alias for their link
- links will expire after a default timespan, user can specify this expiration time

### Design

Read heavy, append only, no update, no relation - NoSQL database

A offline key generation service computing and storing candidate keys before hand

Cache some available keys in application for faster new key lookup

Cache recent read requests

Lazy / batch cleaning of expired translations

## Pastebin
Users of the service enter a piece of text and get a random URL to access it

### Requirements
- user can upload text and get a unique URL to access it
- data and link will expire after a default timespan, user can specify this expiration time
- user can pick a custom URL for their data

### Design

Pretty much the same with TinyURL.

Seperate metadata storage with content storage.

## Instagram

### Requirements
- users can upload/download/view photos
- users can search based on photo/viedo titles
- users can follow other users
- system should generate and display a user's News Feed consisting of top photos from all the people the user follows

## Dropbox

### Requirements
- user can upload and download files/photos from any device
- user can share files or folders with other users
- service should support automatic synchronization between devices
- system should support storing large files up to a GB
- ACID-ity is required on all file operations
- offline editing is required