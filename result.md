### Scenario

We want to upgrade from 6.2.8 to 7.22. According to the Redis [upgrade](https://redis.io/docs/latest/operate/rs/installing-upgrading/upgrading/upgrade-cluster/#supported-upgrade-paths) docs, if we are going to use [inplace upgrade](https://redis.io/docs/latest/operate/rs/installing-upgrading/upgrading/upgrade-cluster/#in-place-upgrade) or [rolling upgrade](https://redis.io/docs/latest/operate/rs/installing-upgrading/upgrading/upgrade-cluster/#rolling-upgrade) within that specific version, we must first upgrade to a supported intermediate version, which in this case is 7.4.x.

So we want to test it without using either in-place upgrade or rolling upgrade first. Instead, we are going to use Rip‑and‑Replace Migration with this scenario:

### Two Redis Cluster

1. First cluster with 6.2.8 Version that contain 3 databases
2. Second cluster with 7.22

Tutorial for that can be found in [rip-and-replace](./rip_and_replace_migration.md)

- We are using 3 nodes for 6.2.8 version:
  1. 10.100.14.117
  2. 10.100.13.231
  3. 10.100.14.194
- On 6.2.8 we have three databases available as you can see on this screenshot:

![6-2-8-databases](./assets/6-2-8-databases.png)

- run `rlcheck` and `rladmin status extra all`

![rlcheck-6-2-8](./assets/rlcheck-6-2-8.png)

![rladmin-status-6-2-8](./assets/rladmin-status-6-2-8.png)

- Run backup for every databases

![export-backup-6-2-8](./assets/export-backup-6-2-8.png)

- For the second cluster 7.22 is installed on these 3 nodes

  1. 10.100.13.13
  2. 10.100.13.68
  3. 10.100.13.69

- Create empty database similar to 6.2.8

![7-2-2-databases](./assets/7-2-2-databases.png)

- Run Replica of in destination databases (7.22 Redis version)
  We can follow this format for URI source `redis://[username]:[password]@[host]:[port]`
  To make it easier just go to Databases, select database and click on Endpoint and Get Replica of source URL.

![get-replica-source-url-6-2-8](./assets/get-replica-source-url-6-2-8.png)

_Note_ If you failed to connect using for example private endpoint dns like this: `redis://admin:ASKRZo5A9mkUCmwQWbgDLtrOKSh10yvWsxZiVgvxm1Qxx0Y8@redis-19588.redisoss1.alldataint.com:19588` then you should try using IP addresses because every domain must be registered in a DNS server.

We can see the replication being successful if they are Synced

![synced-replica](./assets/synced-replica.png)

And we can compare between these two databases from source and destination.

![source db](./assets/source_db_6-2.png)
![destination db](./assets/result-destination.png)
