---
title: Getting Started — Self-Hosted / BYOC
tags: [start, self-hosted, byoc, on-prem, setup]
---
## Getting Started — Self-Hosted / BYOC / On-Prem
### Prerequisites
- FE nodes: Java 8+ runtime
- BE nodes: Linux with sufficient disk, memory
- Network: FE and BE nodes must be able to communicate
### Deployment Steps
1. Deploy FE nodes (1 Leader + 2 Followers for HA)
2. Deploy BE nodes (3+ for production)
3. Register BEs with FE: `ALTER SYSTEM ADD BACKEND "<be_host>:9050";`
4. Create database and tables
### Connect with VeloCLI (Preferred)
```bash
velo auth add local --host <fe_host> --port 9030 --http-port 8030 --user root --password <pass>
velo use local
velo auth status    # Verify: shows backends, version, latency
```
### Connect via MySQL Client
```bash
mysql -h <fe_host> -P 9030 -u root
```
### Self-Hosted Properties
```sql
PROPERTIES ("replication_num" = "3");  -- 3 replicas for HA
```
Reference: [Apache Doris Installation](https://doris.apache.org/docs/install/cluster-deployment/standard-deployment)
