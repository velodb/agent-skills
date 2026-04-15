---
title: Getting Started — VeloDB Cloud
tags: [start, cloud, connection, setup]
---
## Getting Started — VeloDB Cloud
### Connection Info
You'll need: **Host**, **Port** (usually 9030 for MySQL protocol), **HTTP Port** (usually 8080), **User**, **Password**, **Warehouse name**.
### Connect with VeloCLI (Preferred)
```bash
velo auth add cloud --host <host> --port 9030 --http-port 8080 --user <user> --password <pass>
velo use cloud
velo auth status    # Verify: shows backends, version, latency
```
### Connect via MySQL Client
```bash
mysql -h <host> -P 9030 -u <user> -p<password>
```
### Connect via JDBC
```
jdbc:mysql://<host>:9030/<database>?useSSL=false
```
### First Steps
1. Create a database: `CREATE DATABASE IF NOT EXISTS my_db;`
2. Use the database: `USE my_db;`
3. Create your first table (see use case templates for guidance)
4. Load data via Stream Load, INSERT, or external connectors
### Cloud-Specific Properties
Always set these for cloud mode:
```sql
PROPERTIES ("replication_num" = "1");
```
Reference: [VeloDB Cloud Docs](https://docs.velodb.io)
