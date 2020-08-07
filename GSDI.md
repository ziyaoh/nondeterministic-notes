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

## TinyURL

### Requirements
- given a URL, service should generate a shorter and unique alias
- when user access a short link, we should redirect them to the original link
- users can pick a custom alias for their link
- links will expire after a default timespan, user can specify this expiration time

## Pastebin
Users of the service enter a piece of text and get a random URL to access it

### Requirements
- user can upload text and get a unique URL to access it
- data and link will expire after a default timespan, user can specify this expiration time
- user can pick a custom URL for their data