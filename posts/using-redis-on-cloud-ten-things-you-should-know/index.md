---
title: Using Redis on Cloud? Here are Ten things You Should Know
description: This blog covers a range of Redis related best practices, tips and tricks including cluster scalability, client side configuration, integration, metrics and more.
tags:
  - redis
  - databases
  - nosql
  - best-practices
  - memorydb
  - elasticache
spaces:
  - databases
authorGithubAlias: abhirockzz
authorName: Abhishek Gupta
externalCanonicalUrl: https://acloudguru.com/blog/engineering/how-to-use-redis-on-cloud
date: 2023-06-12
---

It's hard to operate stateful distributed systems at scale and Redis is no exception. Managed databases make life easier by taking on much of the heavy lifting. But you still need a sound architecture and apply best practices both on the server (Redis) and client (application).

This blog covers a range of Redis related best practices, tips and tricks including cluster scalability, client side configuration, integration, metrics etc. Although I will be citing [Amazon MemoryDB](https://docs.aws.amazon.com/memorydb/latest/devguide/what-is-memorydb-for-redis.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) and [Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) for Redis from time to time, most (if not all) will be applicable to Redis clusters in general.

> This is not meant to be an exhaustive list by any means. I simply chose ten since it's a nice, wholesome number!

Let's dive right in and start off with what options you have in terms of scaling your Redis cluster.

## 1. Scalability options

You can either scale *up* or *out*:

- Scaling Up (Vertical) - You can increase the capacity of individual nodes/instances for e.g. upgrade from [Amazon EC2](https://aws.amazon.com/ec2/instance-types/?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) `db.r6g.xlarge` type to `db.r6g.2xlarge`
- Scaling Out (Horizontal) - You can add more nodes to the cluster

The requirement to scale *out* might be driven by few reasons.

If you need to tackle a *read heavy* workload, you can choose to add more replica nodes. This applies both for a Redis clustered setup (like `MemoryDB`) or a non-clustered  primary-replica mode as in the case of [ElastiCache with cluster mode disabled](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Replication.Redis-RedisCluster.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq).

If you want to increase *write* capacity, you will find yourself limited by the primary-replica mode and should opt for a Redis Cluster based setup. You can increase the number of shards in your cluster - this is because only primary nodes can accept writes and each shard can only have one primary.

> This has the added benefit of increasing the overall high availability as well.

![Amazon ElastiCache Cluster Mode](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/images/ElastiCache-NodeGroups.png?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq)

## 2. After scaling your cluster, you better use those replicas!

The default behavior in most Redis Cluster clients (including `redis-cli`) is to redirect all *reads* to the *primary* node. If you've have added read replicas to scale read traffic, they are going to sit idle!

You need to switch to [READONLY](https://redis.io/commands/readonly/) mode to ensure that the replicas handle all the read requests are not just passive participants. Make sure to configure your Redis client appropriately - this will vary with the client and programming language.

For example, in the [Go Redis client](https://github.com/go-redis/redis), you can set `ReadOnly` to `true`:

```go
client := redis.NewClusterClient(
	&redis.ClusterOptions{
		Addrs:     []string{clusterEndpoint},
		ReadOnly:  true,
		//..other options
	})
```

To optimize further, you can also use `RouteByLatency` or `RouteRandomly`, both of which automatically turn on `ReadOnly` mode.

> You can refer to how this works for [Java clients such as Lettuce](https://lettuce.io/core/release/reference/index.html#readfrom.read-from-settings)

## 3. Be mindful of consistency characteristics when using read replicas

There is a chance that your application might read stale data from replicas - this is *Eventual Consistency* in action. Since the primary to replica node replication is *asynchronous*, there is a chance that the write you sent to a primary node has not yet reflected in the read replica. This is likely when you have a high number of read replicas specially across multiple availability zones. If this is unacceptable for your use-case, you will have to resort to using primary nodes for reads as well.

The [ReplicationLag metric](https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.memorydb.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) in MemoryDB or ElastiCache for Redis can be used to check how far behind (in seconds) the replica is in applying changes from the primary node.

**What about Strong Consistency?**

In case of `MemoryDB`, [the reads from primary nodes are strongly consistent](https://docs.aws.amazon.com/memorydb/latest/devguide/consistency.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq). This is because the client application receives a successful write acknowledgement only after a write (to the primary node) is written to a *durable Multi-AZ Transaction Log*.

## 4. Remember, you can influence how your keys are distributed across a Redis cluster

Instead of using consistent hashing (like lot of other distributed databases), Redis uses the concept of hash slots. There are `16384` slots in total, a range of hash slots is assigned to each primary node in the cluster and each key belongs to a specific hash slot (thereby assigned to a particular node). Multi-key operations executed on a Redis cluster cannot work if keys belong to different hash slots.

But, you are not completely at the mercy of the cluster! It's possible to influence the key placement by using *hashtags*. Thus, you can ensure that specific keys have the same hash slot. For example, if you are storing orders for customer ID `42` in a `HASH` named `customer:42:orders` and the customer profile info in `customer:42:profile`, you can use curly braces `{}` to define the specific substring which will be hashed. In this case, our keys are `{customer:42}:orders` and `{customer:42}:profile` - `{customer:42}` now drives the hash slot placement. Now we can be confident that both these keys will in the *same* hash slot (hence same node).

## 5. Did you think about scaling (back) in?

Your application was successful, it has a lot of users and traffic. You scaled out the  cluster and things are still going great. Awesome!

**But what if you need to scale back in?**

You need to be careful about a few things before you do that:

- Is there enough free memory on each of the nodes?
- Can this be done during non-peak hours?
- How will it affect your client applications?
- Which metrics can you monitor during this phase? (e.g. `CPUUtilization`, `CurrConnections` etc.)

Refer to some of the [best practices in the MemoryDb for Redis documentation](https://docs.aws.amazon.com/memorydb/latest/devguide/best-practices-online-resharding.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) to better plan for scaling in.

## 6. When things go wrong....

Let's face it, failures are enviable. What's important is whether you are prepared for them? In case of your Redis cluster, here are some things to think about:

- **Have you tested how your application/service behavior in face of failures?** If not, please do! With MemoryDB and ElastiCache for Redis, you can leverage the [Failover API](https://docs.aws.amazon.com/memorydb/latest/devguide/autofailover.html#auto-failover-test?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) to simulate a primary node failure and trigger a failover.
- **Do you have replica nodes?** If all you have is one shard with a single primary node, you are certainly going to have downtime if that node fails.
- **Do you have multiple shards?** If all you have is one shard (with primary and replica), in case of primary node failure of that shard, the cluster cannot accept any writes.
- **Do your shards span multiple availability zones?** If you have shards across multiple AZs, you will be better prepared to tackle AZ failure.

> In all cases, `MemoryDB` ensures that no data is lost during node replacements or failover

## 7. Unable to connect to Redis, help!

> Tl;DR: It's probably the networking/security configuration

This is something which trips up folks all the time! With `MemoryDB` and `ElastiCache`, your [Redis nodes are in a VPC](https://docs.aws.amazon.com/memorydb/latest/devguide/vpcs.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq). If you have a client application deployed to a compute service such as [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq), [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html), [ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq), [Amazon App Runner](https://docs.aws.amazon.com/apprunner/latest/dg/what-is-apprunner.htm?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) etc., you need to ensure you have the right configuration - specifically in terms of VPC and Security Group(s).

This might vary depending on the compute platform you are using. For example, how you [configure a Lambda function to access resources in a VPC](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html) is slightly different compared to how App Runner does it (via a [VPC Connector](https://docs.aws.amazon.com/apprunner/latest/dg/network-vpc.html)), or even EKS (although conceptually, they are the same).

## 8. Redis 6 comes with Access Control Lists - use them!

There is no excuse to not apply authentication (username/password) and authorization (ACL based permission) to your Redis cluster. `MemoryDB` is Redis 6 compliant and [supports ACL](https://docs.aws.amazon.com/memorydb/latest/devguide/clusters.acls.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq). However, to comply with older Redis versions, it configures a *default* user per account (with username **default**) and an immutable ACL called `open-access`. If you create a `MemoryDB` cluster and associate it with this ACL:

- Clients can connect *without* authentication
- Clients can execute *any* command on any key (no permission or authorization either)

As a best practice:

- Define an explicit ACL
- Add users (along with passwords), and
- Configure access strings as per your security requirements. 

You should monitor authentication failures. For example, the [AuthenticationFailures](https://docs.aws.amazon.com/memorydb/latest/devguide/metrics.memorydb.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) metric in `MemoryDB` gives you the total number of failed authenticate attempts - set an alarm on this to detect unauthorized access attempts.

> **Don't forget perimeter security**

If you've configured `TLS` on the server, don't forget to use that in your client as well! For example, using Go Redis:

```go
client := redis.NewClusterClient(
	&redis.ClusterOptions{
		Addrs:     []string{clusterEndpoint},
		TLSConfig: &tls.Config{MaxVersion: tls.VersionTLS12},
		//..other options
	})
```

> Not using it can give your errors that's not obvious enough (e.g. a generic `i/o timeout`) and make things hard to debug - this is something you need to be careful about.

## 9. There are things you cannot do (and that's ok)

As a managed database service, `MemoryDB` or `ElastiCache` [restrict access to some of the Redis commands](https://docs.aws.amazon.com/memorydb/latest/devguide/restrictedcommands.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq). For example, you *cannot* use a subset of the [CLUSTER](https://redis.io/commands/cluster/) related commands since the  cluster management (scale, sharding etc.) is taken of by the service itself.

But, in some cases, you might be able to find alternatives. Think of monitoring slow running queries as an example. Although you *cannot* configure `latency-monitor-threshold` using [CONFIG SET](https://redis.io/commands/config-set/), you can set the `slowlog-log-slower-than` setting in the [parameter group](https://docs.aws.amazon.com/memorydb/latest/devguide/components.html#whatis.components.parametergroups?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) and then use `slowlog get` to compare against it.

## 10. Use connection pooling

Your Redis server nodes (even powerful ones) have finite resources. One of them is ability to support a certain number of concurrent connections. Most Redis clients offer connection pooling as a way to efficiently manage connections to the Redis server. Re-using connections not only benefits your Redis server, but client side performance is improved due to less overhead - this is critical in high volume scenarios.

ElastiCache provides [a few metrics](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/CacheMetrics.Redis.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq) you can track:

- `CurrConnections`: the number of client connections (excluding ones from read replicas)
- `NewConnections`: the total number of connections that have been accepted by the server during a specific period.

## 11. (bonus) Use the appropriate connection mode

This one is kind of obvious, but I am going to call it out anyway since this is one of the most common "getting started" mistake that I witness folks make.

The connection mode that you use in your client application will depend on whether you're using a standalone Redis setup, a Redis Cluster (most likely). Most Redis clients draw a clear distinction between them. For example, if you are using the [Go Redis client](https://github.com/go-redis/redis) with `MemoryDB` or `Elasticache` cluster mode enabled), you need to use [NewClusterClient](https://pkg.go.dev/github.com/go-redis/redis#NewClusterClient) (not [NewClient](https://pkg.go.dev/github.com/go-redis/redis#NewClient)):

```go
redis.NewClusterClient(&redis.ClusterOptions{//....})
```

> Interestingly enough, there is [UniversalClient](https://pkg.go.dev/github.com/go-redis/redis#NewUniversalClient) option which is a bit more flexible (at the time of writing, this is in Go Redis v9)

If you don't use the right mode of connection, you will get an error. But sometimes, the root cause will be hidden behind a generic error message - so you need to be watchful.

## Conclusion

The architectural choices you make will ultimately be driven by your [specific requirements](https://docs.aws.amazon.com/memorydb/latest/devguide/cluster-create-determine-requirements.html?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq). I would encourage you to explore the following blog posts for a deeper dive into performance characteristics of MemoryDB and ElastiCache for Redis and how they might impact the way design your solutions:

- [Optimize Redis Client Performance for Amazon ElastiCache and MemoryDB](https://aws.amazon.com/blogs/database/optimize-redis-client-performance-for-amazon-elasticache/?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq)  
- [Best practices: Redis clients and Amazon ElastiCache for Redis](https://aws.amazon.com/blogs/database/best-practices-redis-clients-and-amazon-elasticache-for-redis/?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq)
- [Measuring database performance of Amazon MemoryDB for Redis](https://aws.amazon.com/blogs/database/measuring-database-performance-of-amazon-memorydb-for-redis/?sc_channel=el&sc_campaign=datamlwave&sc_content=using-redis-on-cloud-ten-things-you-should-know&sc_geo=mult&sc_country=mult&sc_outcome=acq)

Feel free to share your Redis tips, tricks and suggestions. Until then, Happy Building!
