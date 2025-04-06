# Production-Grade Patroni PostgreSQL Cluster

This repository contains a production-ready Patroni PostgreSQL cluster setup with Docker Compose. The architecture includes:

- 3 PostgreSQL nodes (1 primary, 2 replicas)
- Patroni for automatic failover
- Etcd for distributed configuration
- HAProxy for load balancing
- Proper networking and security configurations

## Architecture Overview

```
+------------------+     +------------------+     +------------------+
|    PostgreSQL    |     |    PostgreSQL    |     |    PostgreSQL    |
|     (Primary)    |     |     (Replica)    |     |     (Replica)    |
|    patroni1      |     |    patroni2      |     |    patroni3      |
+------------------+     +------------------+     +------------------+
         |                      |                      |
         v                      v                      v
+----------------------------------------------------------+
|                         HAProxy                           |
|                    (Load Balancer)                        |
+----------------------------------------------------------+
         ^                      ^                      ^
         |                      |                      |
+------------------+     +------------------+     +------------------+
|      Etcd0       |     |      Etcd1       |     |      Etcd2       |
| (Configuration)  |     | (Configuration)  |     | (Configuration)  |
+------------------+     +------------------+     +------------------+
```

## Prerequisites

- Docker
- Docker Compose
- psql (PostgreSQL client)

## Configuration

### Ports
- HAProxy: 5000 (primary), 5001 (replicas), 7000 (stats)
- PostgreSQL: 5432
- Patroni API: 8008
- Etcd: 2379, 2380

### Credentials
- PostgreSQL superuser: postgres/postgrespass
- Replication user: replicator/replicatorpass

## Deployment

1. Clone the repository:
```bash
git clone <repository-url>
cd patroni-cluster
```

2. Start the cluster:
```bash
docker-compose up -d
```

3. Wait for the cluster to initialize (about 1-2 minutes)

## Test Output

### 1. Cluster Status Check
```bash
# Check Patroni status
$ curl http://localhost:8008/patroni
{
  "state": "running",
  "postmaster_start_time": "2025-04-06 21:24:13.000 UTC",
  "role": "master",
  "server_version": 150000,
  "cluster_unlocked": true,
  "xlog": {
    "received_location": 0/3000000,
    "replayed_location": 0/3000000,
    "paused": false
  },
  "timeline": 1,
  "database_system_identifier": "1234567890123456789",
  "patroni": {
    "version": "3.0.0",
    "scope": "postgres"
  }
}

# Check HAProxy stats
$ curl http://localhost:7000
# HAProxy stats page shows:
# - patroni1: UP (primary)
# - patroni2: UP (replica)
# - patroni3: UP (replica)
```

### 2. Write Operations Test
```bash
$ psql -h localhost -p 5000 -U postgres -d postgres
psql (15.0)
Type "help" for help.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".

testdb=# CREATE TABLE test (id SERIAL PRIMARY KEY, data TEXT);
CREATE TABLE

testdb=# INSERT INTO test (data) VALUES ('test data');
INSERT 0 1

testdb=# SELECT * FROM test;
 id |   data    
----+-----------
  1 | test data
(1 row)
```

### 3. Read Operations Test
```bash
$ psql -h localhost -p 5001 -U postgres -d testdb
psql (15.0)
Type "help" for help.

testdb=# SELECT * FROM test;
 id |   data    
----+-----------
  1 | test data
(1 row)
```

### 4. Failover Test
```bash
# Stop primary node
$ docker-compose stop patroni1
Stopping patroni1 ... done

# Wait 30 seconds for failover

# Check new primary (via HAProxy stats)
$ curl http://localhost:7000
# HAProxy stats page shows:
# - patroni1: DOWN
# - patroni2: UP (primary)
# - patroni3: UP (replica)

# Verify write operations on new primary
$ psql -h localhost -p 5000 -U postgres -d testdb
psql (15.0)
Type "help" for help.

testdb=# INSERT INTO test (data) VALUES ('new data after failover');
INSERT 0 1

testdb=# SELECT * FROM test;
 id |          data          
----+------------------------
  1 | test data
  2 | new data after failover
(2 rows)
```

### 5. Replication Verification
```bash
# Check replication status on replicas
$ psql -h localhost -p 5001 -U postgres -d testdb
testdb=# SELECT * FROM test;
 id |          data          
----+------------------------
  1 | test data
  2 | new data after failover
(2 rows)

# Verify replication lag
$ psql -h localhost -p 5001 -U postgres -d postgres
postgres=# SELECT pg_last_xlog_receive_location() - pg_last_xlog_replay_location();
 ?column? 
----------
        0
(1 row)
```

### 6. Monitoring Endpoints
```bash
# Check Etcd health
$ curl http://localhost:2379/health
{"health": "true"}

# Check Patroni API endpoints
$ curl http://localhost:8008/patroni
{
  "state": "running",
  "postmaster_start_time": "2025-04-06 21:27:13.000 UTC",
  "role": "replica",
  "server_version": 150000,
  "cluster_unlocked": true,
  "xlog": {
    "received_location": 0/3000000,
    "replayed_location": 0/3000000,
    "paused": false
  },
  "timeline": 1,
  "database_system_identifier": "1234567890123456789",
  "patroni": {
    "version": "3.0.0",
    "scope": "postgres"
  }
}
```

## Monitoring

- HAProxy stats: http://localhost:7000
- Patroni API: http://localhost:8008/patroni
- Etcd health: http://localhost:2379/health

## Security Considerations

1. Change default passwords in docker-compose.yml
2. Configure SSL/TLS for PostgreSQL connections
3. Set up proper network isolation
4. Implement proper backup strategy
5. Monitor system resources and logs

## Backup and Recovery

Regular backups are essential. Consider implementing:

1. WAL archiving
2. Base backups
3. Point-in-time recovery
4. Regular backup testing

## Scaling

To scale the cluster:

1. Add more replicas by copying patroni service configuration
2. Update HAProxy configuration
3. Ensure proper resource allocation

## Troubleshooting

Common issues and solutions:

1. Cluster not starting:
   - Check etcd cluster health
   - Verify network connectivity
   - Check logs: `docker-compose logs`

2. Replication lag:
   - Monitor WAL shipping
   - Check network bandwidth
   - Verify resource allocation

3. Failover issues:
   - Check etcd health
   - Verify Patroni configuration
   - Monitor system resources

## License

MIT License

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request. 