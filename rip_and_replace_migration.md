# Migrating Redis Enterprise from 6.2 to 7.22

## Strategy: Rip-and-Replace via Replica Of

Rather than relying on the in-place cluster upgrade path (which requires intermediate version hops), this migration stands up a fresh Redis Enterprise 7.22 cluster and uses **Replica Of** to stream data from the old 6.2 cluster into it. Once everything is in sync, clients are switched over and the old cluster is retired.

> **Why this works:** Replica Of is officially supported across different Redis Enterprise clusters and versions — it is the intended tool for this kind of cross-cluster migration.

---

## Overview of Steps

1. Plan and inventory the 6.2 cluster
2. Stand up the new 7.22 cluster
3. Create matching empty databases on 7.22
4. Set up Replica Of (6.2 → 7.22) for each database
5. Wait for full sync, then verify data
6. Freeze writes, cut over clients to 7.22
7. Detach Replica Of and promote 7.22 databases to primary
8. Validate and decommission the 6.2 cluster

---

## 1. Scope

| Item             | Detail                                           |
| ---------------- | ------------------------------------------------ |
| Source           | Redis Enterprise Software 6.2.x                  |
| Destination      | Redis Enterprise Software 7.22.x                 |
| DB type          | Standard (non-Active-Active) databases           |
| Migration method | Replica Of (Active-Passive), then client cutover |

This method bypasses the constraint of needing an intermediate version when doing in-place upgrades between major RS releases.

---

## 2. Planning & Prerequisites

### 2.1 Take Inventory of the 6.2 Cluster

For each database on the source cluster, note down:

- DB name, ID, port, and endpoint
- Shard count and whether replication (HA) is enabled
- Any modules loaded (Search, JSON, TimeSeries, Bloom, etc.)
- Redis version used (6.0 or 6.2) and protocol mode (RESP2/RESP3)

Use these details to design the equivalent database on 7.22:

- Match or increase memory limits and shard counts
- Replicate the same HA and persistence settings
- Install the same modules at compatible versions

> **Tip:** On RS 7.22, each database can independently run Redis 7.2, 7.4, or 8.x. Start with 7.2 or 7.4 for better compatibility with 6.2 data, then plan version bumps later.

### 2.2 Verify and Back Up the 6.2 Cluster

Run the following on every node in the 6.2 cluster before doing anything else:

```shell
rlcheck
rladmin status extra all
```

Resolve any warnings or errors before proceeding.

For each database:

- Confirm persistence (AOF or RDB) and scheduled backups are healthy
- Trigger a fresh backup (export or "backup now") prior to enabling Replica Of

### 2.3 Network Connectivity

From the 7.22 cluster nodes to the 6.2 cluster nodes:

- TCP access must be open on the database ports (typically in the `10000–19999` range)
- Firewalls and load balancers along the path must allow this traffic
- If using TLS for Replica Of, prepare certificates in advance and follow the "Configure TLS for Replica Of" procedure (covers syncer and proxy certificates)

### 2.4 Capacity and Licensing on 7.22

- Confirm the 7.22 cluster license covers enough RAM/flash shards for all databases being migrated
- Size the new cluster nodes to accommodate all migrated databases with room to spare (CPU, RAM, storage)

---

## 3. Build the 7.22 Cluster

1. Install Redis Enterprise 7.22 on fresh hosts (Linux, supported OS)
2. Initialize the cluster using the Cluster Manager UI or:
   ```shell
   rladmin cluster create
   ```
3. Optional configuration:
   - Enable rack-zone awareness for multi-AZ setups
   - Configure IPv4/IPv6, multi-IP, or load balancer integration as needed
4. Confirm the cluster is healthy:
   ```shell
   rlcheck
   rladmin status extra all
   ```

---

## 4. Create Empty Databases on 7.22

For each database being migrated, create a new database on the 7.22 cluster:

1. In the Cluster Manager UI: **Databases → Create database**
2. Set the following:
   - **Name:** use the same name as the source DB where possible
   - **Port:** pick any available port (no need to match the 6.2 port)
   - **Sharding & replication:** same or greater capacity than the source
   - **Persistence:** RDB or AOF to satisfy your RPO/RTO requirements
   - **Modules:** enable the same modules as the source with compatible versions
   - **Redis version:** select 7.2 or 7.4 (Replica Of works across versions and clusters)
3. **Do not configure Replica Of at this point** — finish creating the database first

> **Note:** From RS 7.22, you can flag a Replica Of database as strictly read-only (`replica_read_only`) at creation. For migration purposes, leave this off — you will remove Replica Of later and promote the database to primary.

---

## 5. Wire Up Replica Of (6.2 → 7.22)

### 5.1 Get the Source URL from the 6.2 Cluster

On the 6.2 Cluster Manager UI:

- Navigate to **Databases → \<source DB\> → Configuration**
- Locate the **Replica Of source URL** field and copy the URL

The format is:

```
redis://[username]:[password]@[host]:[port]
```

> If the DNS name fails to resolve from the 7.22 nodes, use the raw IP address instead — the DNS hostname must be resolvable from the destination cluster's perspective.

### 5.2 Configure Replica Of on the 7.22 Destination DB

On the 7.22 cluster, for each destination database:

1. Go to **Databases → \<destination DB\> → Configuration → Edit**
2. Expand the **Replica Of** section
3. Click **+ Add source database**
4. Select **External (different cluster)** in the dialog
5. Paste the source URL from the 6.2 DB
6. Optionally enable **compression** if clusters are in different regions or bandwidth is limited (uses gzip; trades CPU for reduced replication traffic)
7. Click **Add source**, then **Save**

### 5.3 Wait for Sync to Complete

Monitor the status for each destination DB under **Configuration → Replica Of**:

| Status      | Meaning                                                                  |
| ----------- | ------------------------------------------------------------------------ |
| **Syncing** | Initial full sync in progress (percentage displayed)                     |
| **Synced**  | Full sync done; incremental replication ongoing with a visible Lag value |

Wait until every database reaches **Synced** with a stable, low lag before proceeding.

**Important behaviors during Replica Of:**

- When Replica Of is first configured, any existing data in the destination DB is **flushed** and replaced by source data
- While Replica Of is active, the destination DB does **not** expire or evict keys — be mindful of this for TTL-heavy workloads

---

## 6. Client Cutover

### 6.1 Pre-Cutover Checks

Before touching anything, confirm for every destination DB:

- Status is **Synced** with low lag
- Key counts and a sample of representative keys match the source (`INFO keyspace`, spot-check business keys)

### 6.2 Stop Writes to the 6.2 Cluster

1. Schedule a maintenance window
2. At the start of the window, put upstream systems into read-only mode or block write traffic to the 6.2 endpoints
3. Confirm writes have stopped:
   - Watch application logs and metrics
   - Monitor 6.2 DB keyspace and traffic metrics until stable

Wait for:

- Replica Of status to show **Synced**
- Lag to reach effectively **0**

### 6.3 Remove Replica Of from the 7.22 Databases

For each destination DB on 7.22:

1. Go to **Configuration → Edit**
2. Under **Replica Of**, delete the source entry
3. Save

The 7.22 database is now a standalone primary. Writes can be accepted.

> **Warning:** Do not allow writes into a database while Replica Of is still configured. If the replication backlog overflows and a resync is triggered, the destination DB will be flushed and reloaded from source — any writes made during that window would be lost.

### 6.4 Redirect Clients to 7.22

Update connection configuration depending on how clients connect:

- **DNS-based:** update DNS records to resolve DB names to 7.22 endpoints
- **Load balancer-based:** repoint VIPs to the new cluster nodes/endpoints
- **Direct IP:port:** update client config to use 7.22 endpoints

After redirecting, run:

- Connectivity tests
- Critical functional tests (reads and writes) for each application
- Basic performance sanity checks

---

## 7. Post-Cutover Validation

For each migrated database:

1. Run a full health check on the 7.22 cluster:
   ```shell
   rlcheck
   rladmin status extra all
   ```
2. Verify:
   - Key counts and top keys look correct
   - Module behavior (Search, JSON, etc.) is functioning as expected
   - Persistence and backups are running and succeeding on 7.22
3. Monitor through at least one full business cycle:
   - Latency and throughput
   - Memory usage and shard distribution
   - Errors related to ACLs, TLS, or unsupported commands

---

## 8. Decommission the 6.2 Cluster

Once the 7.22 cluster has been running stably and backups are confirmed:

1. Permanently disable client access to the 6.2 cluster
2. Optionally, keep the 6.2 cluster in read-only mode for a short fallback window
3. When confident:
   - Delete or archive all databases on 6.2
   - Shut down and uninstall the 6.2 cluster

---

## 9. Additional Scenarios

**Cross-region or multi-cluster migration**
Replica Of supports cross-cluster setups over WAN — enable compression to reduce bandwidth usage.

**Migrating multiple databases in parallel**
Each database migration is independent from Redis' perspective. You can run Replica Of for all databases simultaneously and cut them over in batches or all at once.

**Upgrading the Redis DB version after migration**
Once everything is stable on RS 7.22, use the standard **Upgrade Redis database** workflow to move individual databases from 7.2/7.4 to 7.4/8.x — do this after validating client compatibility with the target version.
