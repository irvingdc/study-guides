# System Design Study Guide - Part 4: Storage and Databases

## 1. SQL vs NoSQL Decision Matrix

### Understanding the Trade-offs

**SQL Databases (Relational)**
- Structured data with defined schema
- ACID compliance
- Strong consistency
- Complex queries with JOINs
- Vertical scaling primarily

**NoSQL Databases**
- Flexible or no schema
- BASE properties (Usually)
- Eventual consistency (Usually)
- Simple queries, denormalized data
- Horizontal scaling

### Decision Framework

```python
def choose_database_type(requirements):
    """
    Decision tree for database selection
    """
    
    # Strong indicators for SQL
    if requirements['needs_acid_transactions']:
        return "SQL (PostgreSQL, MySQL)"
    
    if requirements['complex_relationships'] and requirements['data_integrity']:
        return "SQL with proper indexing"
    
    if requirements['financial_data'] or requirements['inventory_management']:
        return "SQL for consistency guarantees"
    
    # Strong indicators for NoSQL
    if requirements['unstructured_data']:
        if requirements['full_text_search']:
            return "Document Store (Elasticsearch)"
        else:
            return "Document Store (MongoDB, DynamoDB)"
    
    if requirements['graph_traversal']:
        return "Graph Database (Neo4j, Neptune)"
    
    if requirements['time_series_data']:
        return "Time Series DB (InfluxDB, TimescaleDB)"
    
    if requirements['extreme_scale'] and requirements['simple_access_patterns']:
        return "Key-Value Store (Redis, DynamoDB)"
    
    if requirements['write_heavy'] and requirements['high_availability']:
        return "Wide Column Store (Cassandra, HBase)"
    
    # Default to SQL for general purpose
    return "SQL (start with PostgreSQL)"
```

### Detailed Comparison Matrix

| Aspect | SQL | Document Store | Key-Value | Wide Column | Graph |
|--------|-----|---------------|-----------|-------------|-------|
| **Schema** | Fixed, predefined | Flexible | None | Flexible columns | Flexible |
| **Query Language** | SQL | Database-specific | Simple get/put | CQL (Cassandra) | Cypher, Gremlin |
| **Relationships** | Foreign keys, JOINs | Embedded/References | None | None | First-class edges |
| **Transactions** | ACID | Document-level ACID | Limited | Row-level | ACID available |
| **Scaling** | Vertical (mainly) | Horizontal | Horizontal | Horizontal | Varies |
| **Use Cases** | ERP, CRM, Banking | Content management | Caching, Sessions | IoT, Logs | Social networks |
| **Examples** | PostgreSQL, MySQL | MongoDB, CouchDB | Redis, DynamoDB | Cassandra, HBase | Neo4j, Neptune |

### Real-World Examples

**E-commerce Platform Database Choices:**
```yaml
databases:
  primary:
    type: PostgreSQL
    purpose: Orders, Users, Products
    reason: ACID transactions for orders, complex queries
  
  cache:
    type: Redis
    purpose: Session storage, Product cache
    reason: Fast access, TTL support
  
  search:
    type: Elasticsearch
    purpose: Product search, Autocomplete
    reason: Full-text search, faceted search
  
  analytics:
    type: ClickHouse
    purpose: User behavior, Sales analytics
    reason: Fast analytical queries, columnar storage
  
  recommendations:
    type: Neo4j
    purpose: Product recommendations
    reason: Graph traversal for relationships
```

## 2. Database Replication Strategies

### Master-Slave Replication

**Architecture:**
```
        Write
          ↓
    [Master DB]
      ↙    ↘
   Sync    Sync
    ↓        ↓
[Slave 1] [Slave 2]
    ↑        ↑
   Read     Read
```

**Implementation Example (MySQL):**
```sql
-- On Master
CREATE USER 'replication_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
FLUSH PRIVILEGES;

-- Get master status
SHOW MASTER STATUS;

-- On Slave
CHANGE MASTER TO
  MASTER_HOST='master.db.example.com',
  MASTER_USER='replication_user',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;
```

**Handling Replication Lag:**
```python
class DatabaseRouter:
    def __init__(self, master, slaves, max_lag_seconds=1):
        self.master = master
        self.slaves = slaves
        self.max_lag_seconds = max_lag_seconds
    
    def get_read_connection(self, consistency_requirement='eventual'):
        if consistency_requirement == 'strong':
            return self.master
        
        # Check slave lag
        available_slaves = []
        for slave in self.slaves:
            lag = self.get_replication_lag(slave)
            if lag < self.max_lag_seconds:
                available_slaves.append(slave)
        
        if available_slaves:
            return random.choice(available_slaves)
        else:
            # Fallback to master if all slaves are lagging
            return self.master
    
    def get_replication_lag(self, slave):
        # Query slave status
        result = slave.execute("SHOW SLAVE STATUS")
        return result['Seconds_Behind_Master']
```

### Master-Master Replication

**Architecture:**
```
    Write ←→ Write
      ↓        ↓
[Master 1] ←→ [Master 2]
      ↓        ↓
    Read      Read
```

**Conflict Resolution Strategies:**
```python
class ConflictResolver:
    def resolve_write_conflict(self, record1, record2):
        # Last Write Wins (LWW)
        if record1['updated_at'] > record2['updated_at']:
            return record1
        elif record2['updated_at'] > record1['updated_at']:
            return record2
        else:
            # Tie-breaker using node ID
            return record1 if record1['node_id'] > record2['node_id'] else record2
    
    def resolve_increment_conflict(self, value1, value2, base_value):
        # CRDT - Convergent Replicated Data Type
        delta1 = value1 - base_value
        delta2 = value2 - base_value
        return base_value + delta1 + delta2
```

### Multi-Master with Conflict-Free Replicated Data Types (CRDTs)

```python
class GCounter:
    """Grow-only counter CRDT"""
    def __init__(self, node_id):
        self.node_id = node_id
        self.counts = {}  # {node_id: count}
    
    def increment(self, amount=1):
        if self.node_id not in self.counts:
            self.counts[self.node_id] = 0
        self.counts[self.node_id] += amount
    
    def value(self):
        return sum(self.counts.values())
    
    def merge(self, other_counter):
        for node_id, count in other_counter.counts.items():
            self.counts[node_id] = max(
                self.counts.get(node_id, 0),
                count
            )

class LWWRegister:
    """Last-Write-Wins Register CRDT"""
    def __init__(self, node_id):
        self.node_id = node_id
        self.value = None
        self.timestamp = 0
    
    def set(self, value):
        self.value = value
        self.timestamp = time.time()
    
    def merge(self, other_register):
        if other_register.timestamp > self.timestamp:
            self.value = other_register.value
            self.timestamp = other_register.timestamp
        elif other_register.timestamp == self.timestamp:
            # Tie-breaker using node ID
            if other_register.node_id > self.node_id:
                self.value = other_register.value
```

## 3. Database Sharding Techniques

### Horizontal Partitioning (Sharding)

**Sharding Strategies:**

**1. Range-Based Sharding:**
```python
class RangeShardRouter:
    def __init__(self):
        self.shards = [
            {'range': (0, 1000000), 'host': 'shard1.db.com'},
            {'range': (1000001, 2000000), 'host': 'shard2.db.com'},
            {'range': (2000001, 3000000), 'host': 'shard3.db.com'},
            {'range': (3000001, float('inf')), 'host': 'shard4.db.com'}
        ]
    
    def get_shard(self, user_id):
        for shard in self.shards:
            if shard['range'][0] <= user_id <= shard['range'][1]:
                return shard['host']
        raise ValueError(f"No shard found for user_id: {user_id}")
```

**2. Hash-Based Sharding:**
```python
class HashShardRouter:
    def __init__(self, num_shards):
        self.num_shards = num_shards
        self.shards = [f'shard{i}.db.com' for i in range(num_shards)]
    
    def get_shard(self, key):
        # Use consistent hashing for better distribution
        hash_value = hashlib.md5(str(key).encode()).hexdigest()
        shard_index = int(hash_value, 16) % self.num_shards
        return self.shards[shard_index]
```

**3. Geographic Sharding:**
```python
class GeographicShardRouter:
    def __init__(self):
        self.regions = {
            'us-east': {'host': 'us-east.db.com', 'countries': ['US', 'CA']},
            'eu-west': {'host': 'eu-west.db.com', 'countries': ['GB', 'FR', 'DE']},
            'asia-pac': {'host': 'asia-pac.db.com', 'countries': ['JP', 'CN', 'AU']}
        }
    
    def get_shard(self, user_country):
        for region, config in self.regions.items():
            if user_country in config['countries']:
                return config['host']
        return 'default.db.com'  # Fallback
```

### Consistent Hashing for Dynamic Sharding

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.nodes = nodes or []
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        self._build_ring()
    
    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def _build_ring(self):
        self.ring = {}
        self.sorted_keys = []
        
        for node in self.nodes:
            for i in range(self.virtual_nodes):
                virtual_key = f"{node}:{i}"
                hash_value = self._hash(virtual_key)
                self.ring[hash_value] = node
                self.sorted_keys.append(hash_value)
        
        self.sorted_keys.sort()
    
    def add_node(self, node):
        self.nodes.append(node)
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = node
            bisect.insort(self.sorted_keys, hash_value)
    
    def remove_node(self, node):
        self.nodes.remove(node)
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
            self.sorted_keys.remove(hash_value)
    
    def get_node(self, key):
        if not self.ring:
            return None
        
        hash_value = self._hash(key)
        index = bisect.bisect(self.sorted_keys, hash_value)
        
        if index == len(self.sorted_keys):
            index = 0
        
        return self.ring[self.sorted_keys[index]]
```

### Cross-Shard Queries and Transactions

**Scatter-Gather Pattern:**
```python
class ShardedQueryExecutor:
    def __init__(self, shard_router):
        self.shard_router = shard_router
    
    async def execute_cross_shard_query(self, query, shard_keys=None):
        if shard_keys:
            # Query specific shards
            target_shards = set(
                self.shard_router.get_shard(key) for key in shard_keys
            )
        else:
            # Query all shards (scatter)
            target_shards = self.shard_router.get_all_shards()
        
        # Execute queries in parallel
        tasks = []
        for shard in target_shards:
            task = self.execute_on_shard(shard, query)
            tasks.append(task)
        
        # Gather results
        results = await asyncio.gather(*tasks)
        
        # Merge results
        return self.merge_results(results)
    
    def merge_results(self, results):
        # Example: Merge sorted results
        merged = []
        for result_set in results:
            merged.extend(result_set)
        
        # Sort merged results (if needed)
        merged.sort(key=lambda x: x['timestamp'], reverse=True)
        return merged
```

**Two-Phase Commit for Distributed Transactions:**
```python
class TwoPhaseCommitCoordinator:
    def __init__(self, participants):
        self.participants = participants
    
    def execute_transaction(self, transaction):
        transaction_id = str(uuid.uuid4())
        
        # Phase 1: Prepare
        prepare_results = []
        for participant in self.participants:
            result = participant.prepare(transaction_id, transaction)
            prepare_results.append(result)
        
        # Check if all participants are ready
        if all(r.status == 'READY' for r in prepare_results):
            # Phase 2: Commit
            for participant in self.participants:
                participant.commit(transaction_id)
            return {'status': 'COMMITTED', 'transaction_id': transaction_id}
        else:
            # Abort transaction
            for participant in self.participants:
                participant.abort(transaction_id)
            return {'status': 'ABORTED', 'transaction_id': transaction_id}
```

## 4. Data Partitioning Strategies

### Vertical Partitioning

Split tables by columns.

```sql
-- Original table
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    password_hash VARCHAR(255),
    name VARCHAR(100),
    bio TEXT,
    profile_image BYTEA,
    created_at TIMESTAMP,
    last_login TIMESTAMP
);

-- Vertical partitioning
-- Frequently accessed data
CREATE TABLE users_core (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    password_hash VARCHAR(255),
    name VARCHAR(100),
    last_login TIMESTAMP
);

-- Rarely accessed data
CREATE TABLE users_profile (
    user_id BIGINT PRIMARY KEY REFERENCES users_core(id),
    bio TEXT,
    profile_image BYTEA,
    created_at TIMESTAMP
);
```

### Functional Partitioning

Split by feature or domain.

```python
class FunctionalPartitionRouter:
    def __init__(self):
        self.databases = {
            'users': 'postgresql://users-db:5432/users',
            'orders': 'postgresql://orders-db:5432/orders',
            'products': 'mongodb://products-db:27017/products',
            'analytics': 'clickhouse://analytics-db:9000/events',
            'sessions': 'redis://sessions-cache:6379/0'
        }
    
    def get_connection(self, domain):
        if domain not in self.databases:
            raise ValueError(f"Unknown domain: {domain}")
        return self.databases[domain]
```

## 5. Caching Strategies and Patterns

### Cache Patterns

**1. Cache-Aside (Lazy Loading):**
```python
class CacheAsidePattern:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def get(self, key):
        # Try cache first
        value = self.cache.get(key)
        
        if value is None:
            # Cache miss - load from database
            value = self.database.get(key)
            
            if value is not None:
                # Update cache for next time
                self.cache.set(key, value, ttl=3600)
        
        return value
    
    def update(self, key, value):
        # Update database
        self.database.update(key, value)
        
        # Invalidate cache
        self.cache.delete(key)
```

**2. Write-Through Cache:**
```python
class WriteThroughCache:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def get(self, key):
        value = self.cache.get(key)
        
        if value is None:
            value = self.database.get(key)
            if value is not None:
                self.cache.set(key, value)
        
        return value
    
    def set(self, key, value):
        # Update cache first
        self.cache.set(key, value)
        
        # Then update database
        self.database.set(key, value)
```

**3. Write-Behind (Write-Back) Cache:**
```python
class WriteBehindCache:
    def __init__(self, cache, database, flush_interval=60):
        self.cache = cache
        self.database = database
        self.write_queue = []
        self.flush_interval = flush_interval
        self._start_flush_timer()
    
    def set(self, key, value):
        # Update cache immediately
        self.cache.set(key, value)
        
        # Queue write to database
        self.write_queue.append({'key': key, 'value': value, 'timestamp': time.time()})
    
    def _flush_writes(self):
        if not self.write_queue:
            return
        
        # Batch write to database
        batch = self.write_queue[:100]  # Process 100 at a time
        
        try:
            self.database.batch_write(batch)
            # Remove successful writes from queue
            self.write_queue = self.write_queue[100:]
        except Exception as e:
            # Retry failed writes
            logger.error(f"Failed to flush writes: {e}")
```

**4. Refresh-Ahead Cache:**
```python
class RefreshAheadCache:
    def __init__(self, cache, database, refresh_threshold=0.5):
        self.cache = cache
        self.database = database
        self.refresh_threshold = refresh_threshold  # Refresh when 50% TTL remains
    
    def get(self, key):
        value, ttl_remaining = self.cache.get_with_ttl(key)
        
        if value is None:
            # Cache miss
            value = self.database.get(key)
            if value:
                self.cache.set(key, value, ttl=3600)
        elif ttl_remaining < (3600 * self.refresh_threshold):
            # Proactively refresh before expiration
            self._async_refresh(key)
        
        return value
    
    def _async_refresh(self, key):
        # Refresh cache asynchronously
        threading.Thread(target=self._refresh_cache, args=(key,)).start()
    
    def _refresh_cache(self, key):
        value = self.database.get(key)
        if value:
            self.cache.set(key, value, ttl=3600)
```

### Multi-Level Caching

```python
class MultiLevelCache:
    def __init__(self):
        self.l1_cache = {}  # Application memory (fastest)
        self.l2_cache = Redis()  # Redis (fast)
        self.l3_cache = CDN()  # CDN (distributed)
        self.database = PostgreSQL()  # Source of truth
    
    def get(self, key):
        # Check L1 (memory)
        if key in self.l1_cache:
            return self.l1_cache[key]
        
        # Check L2 (Redis)
        value = self.l2_cache.get(key)
        if value:
            self.l1_cache[key] = value  # Promote to L1
            return value
        
        # Check L3 (CDN)
        value = self.l3_cache.get(key)
        if value:
            self.l2_cache.set(key, value, ttl=300)  # Promote to L2
            self.l1_cache[key] = value  # Promote to L1
            return value
        
        # Load from database
        value = self.database.get(key)
        if value:
            # Populate all cache levels
            self.l3_cache.set(key, value, ttl=3600)
            self.l2_cache.set(key, value, ttl=300)
            self.l1_cache[key] = value
        
        return value
```

## 6. Redis vs Memcached

### Feature Comparison

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data Structures** | Strings, Lists, Sets, Hashes, Sorted Sets, Streams, HyperLogLog | Only key-value strings |
| **Persistence** | RDB snapshots, AOF logs | No persistence |
| **Replication** | Master-slave replication | No built-in replication |
| **Clustering** | Redis Cluster for sharding | Consistent hashing on client |
| **Pub/Sub** | Yes | No |
| **Transactions** | MULTI/EXEC commands | No |
| **Lua Scripting** | Yes | No |
| **Memory Management** | More overhead, configurable eviction | Slab allocation, LRU only |
| **Multi-threading** | Single-threaded (Redis 6+ has I/O threads) | Multi-threaded |

### Redis Advanced Features

**1. Pub/Sub Messaging:**
```python
import redis

# Publisher
def publish_message(channel, message):
    r = redis.Redis()
    r.publish(channel, json.dumps(message))

# Subscriber
def subscribe_to_channel(channel):
    r = redis.Redis()
    pubsub = r.pubsub()
    pubsub.subscribe(channel)
    
    for message in pubsub.listen():
        if message['type'] == 'message':
            data = json.loads(message['data'])
            process_message(data)
```

**2. Distributed Locks:**
```python
class RedisLock:
    def __init__(self, redis_client, key, timeout=10):
        self.redis = redis_client
        self.key = key
        self.timeout = timeout
        self.identifier = str(uuid.uuid4())
    
    def acquire(self):
        end = time.time() + self.timeout
        
        while time.time() < end:
            if self.redis.set(self.key, self.identifier, nx=True, ex=self.timeout):
                return True
            time.sleep(0.001)
        
        return False
    
    def release(self):
        # Lua script for atomic check-and-delete
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        self.redis.eval(lua_script, 1, self.key, self.identifier)
```

**3. Rate Limiting with Redis:**
```python
class RedisRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def is_allowed(self, key, limit, window_seconds):
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - window_seconds
        
        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        # Add current request
        pipe.zadd(key, {str(now): now})
        # Count requests in window
        pipe.zcard(key)
        # Set expiry
        pipe.expire(key, window_seconds + 1)
        
        results = pipe.execute()
        request_count = results[2]
        
        return request_count <= limit
```

## 7. Time-Series Databases

### When to Use Time-Series Databases

**Use Cases:**
- IoT sensor data
- Application metrics
- Financial market data
- Server monitoring
- User activity tracking

### InfluxDB Example

```python
from influxdb_client import InfluxDBClient, Point

class MetricsCollector:
    def __init__(self):
        self.client = InfluxDBClient(
            url="http://localhost:8086",
            token="my-token",
            org="my-org"
        )
        self.write_api = self.client.write_api()
    
    def write_metric(self, measurement, tags, fields):
        point = Point(measurement)
        
        for key, value in tags.items():
            point.tag(key, value)
        
        for key, value in fields.items():
            point.field(key, value)
        
        self.write_api.write(bucket="metrics", record=point)
    
    def query_metrics(self, query):
        query_api = self.client.query_api()
        tables = query_api.query(query)
        
        results = []
        for table in tables:
            for record in table.records:
                results.append({
                    'time': record.get_time(),
                    'value': record.get_value(),
                    'field': record.get_field(),
                    'measurement': record.get_measurement()
                })
        
        return results

# Usage
collector = MetricsCollector()

# Write CPU metrics
collector.write_metric(
    measurement="cpu_usage",
    tags={"host": "server01", "region": "us-east"},
    fields={"usage_percent": 65.5, "load_average": 2.1}
)

# Query metrics
query = '''
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "cpu_usage")
  |> filter(fn: (r) => r["host"] == "server01")
  |> aggregateWindow(every: 5m, fn: mean)
'''
results = collector.query_metrics(query)
```

### TimescaleDB (PostgreSQL Extension)

```sql
-- Create hypertable for time-series data
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id TEXT,
    temperature DOUBLE PRECISION,
    humidity DOUBLE PRECISION,
    pressure DOUBLE PRECISION
);

-- Convert to hypertable
SELECT create_hypertable('metrics', 'time');

-- Create indexes for common queries
CREATE INDEX ON metrics (device_id, time DESC);

-- Continuous aggregates for performance
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    device_id,
    AVG(temperature) as avg_temp,
    AVG(humidity) as avg_humidity,
    AVG(pressure) as avg_pressure,
    COUNT(*) as sample_count
FROM metrics
GROUP BY hour, device_id;

-- Retention policy
SELECT add_retention_policy('metrics', INTERVAL '30 days');

-- Compression policy
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id'
);

SELECT add_compression_policy('metrics', INTERVAL '7 days');
```

## 8. Graph Databases

### Graph Database Concepts

**Nodes, Edges, and Properties:**
```cypher
-- Neo4j Cypher example
-- Create nodes
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 35})
CREATE (post:Post {title: 'Graph Databases', content: '...'})

-- Create relationships
CREATE (alice)-[:FRIENDS_WITH {since: 2020}]->(bob)
CREATE (alice)-[:WROTE]->(post)
CREATE (bob)-[:LIKED {timestamp: '2024-01-15'}]->(post)
```

### Common Graph Queries

```cypher
-- Find friends of friends
MATCH (person:Person {name: 'Alice'})-[:FRIENDS_WITH*2]->(fof:Person)
WHERE NOT (person)-[:FRIENDS_WITH]->(fof)
RETURN DISTINCT fof.name

-- Recommendation engine
MATCH (user:User {id: $userId})-[:PURCHASED]->(product:Product)
      <-[:PURCHASED]-(other:User)-[:PURCHASED]->(rec:Product)
WHERE NOT (user)-[:PURCHASED]->(rec)
RETURN rec, COUNT(DISTINCT other) as frequency
ORDER BY frequency DESC
LIMIT 10

-- Shortest path
MATCH path = shortestPath(
    (start:Person {name: 'Alice'})-[:FRIENDS_WITH*]-(end:Person {name: 'Charlie'})
)
RETURN path

-- Social network analysis
MATCH (person:Person)
RETURN person.name,
       size((person)-[:FRIENDS_WITH]-()) as degree_centrality
ORDER BY degree_centrality DESC
```

### Graph Database Use Cases

```python
class SocialNetworkGraph:
    def __init__(self, neo4j_driver):
        self.driver = neo4j_driver
    
    def find_influencers(self, min_followers=1000):
        query = """
        MATCH (influencer:User)<-[:FOLLOWS]-(follower:User)
        WITH influencer, COUNT(follower) as follower_count
        WHERE follower_count >= $min_followers
        RETURN influencer.username, follower_count
        ORDER BY follower_count DESC
        """
        
        with self.driver.session() as session:
            result = session.run(query, min_followers=min_followers)
            return [dict(record) for record in result]
    
    def recommend_connections(self, user_id, limit=10):
        query = """
        MATCH (user:User {id: $user_id})-[:FRIENDS_WITH]->(friend:User)
              -[:FRIENDS_WITH]->(potential:User)
        WHERE NOT (user)-[:FRIENDS_WITH]-(potential)
          AND user <> potential
        WITH potential, COUNT(DISTINCT friend) as mutual_friends
        RETURN potential.id, potential.username, mutual_friends
        ORDER BY mutual_friends DESC
        LIMIT $limit
        """
        
        with self.driver.session() as session:
            result = session.run(query, user_id=user_id, limit=limit)
            return [dict(record) for record in result]
```

## 9. Data Warehousing and Data Lakes

### Data Warehouse Architecture

**Star Schema:**
```sql
-- Fact table
CREATE TABLE fact_sales (
    sale_id BIGINT PRIMARY KEY,
    date_id INT REFERENCES dim_date(date_id),
    product_id INT REFERENCES dim_product(product_id),
    customer_id INT REFERENCES dim_customer(customer_id),
    store_id INT REFERENCES dim_store(store_id),
    quantity INT,
    amount DECIMAL(10,2),
    discount DECIMAL(10,2)
);

-- Dimension tables
CREATE TABLE dim_date (
    date_id INT PRIMARY KEY,
    date DATE,
    year INT,
    quarter INT,
    month INT,
    week INT,
    day_of_week INT,
    is_holiday BOOLEAN
);

CREATE TABLE dim_product (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(255),
    category VARCHAR(100),
    brand VARCHAR(100),
    unit_price DECIMAL(10,2)
);

CREATE TABLE dim_customer (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(255),
    segment VARCHAR(50),
    city VARCHAR(100),
    state VARCHAR(50),
    country VARCHAR(50)
);
```

### Data Lake Architecture

```python
class DataLakeOrganizer:
    def __init__(self, s3_client):
        self.s3 = s3_client
        self.bucket = 'company-data-lake'
    
    def get_path(self, data_type, date, format='parquet'):
        """
        Organize data in hierarchical structure
        Format: /raw|processed|curated/domain/table/year/month/day/
        """
        year = date.year
        month = f"{date.month:02d}"
        day = f"{date.day:02d}"
        
        return f"{data_type}/{domain}/{table}/year={year}/month={month}/day={day}/"
    
    def write_raw_data(self, data, domain, table, date):
        path = self.get_path('raw', date, 'json')
        key = f"{path}data_{datetime.now().timestamp()}.json"
        
        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=json.dumps(data)
        )
        
        return key
    
    def write_processed_data(self, dataframe, domain, table, date):
        path = self.get_path('processed', date, 'parquet')
        key = f"{path}data.parquet"
        
        # Write as Parquet for better performance
        buffer = BytesIO()
        dataframe.to_parquet(buffer)
        
        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=buffer.getvalue()
        )
        
        return key
```

## 10. ETL vs ELT Pipelines

### ETL (Extract, Transform, Load)

```python
class ETLPipeline:
    def __init__(self, source_db, target_warehouse):
        self.source = source_db
        self.warehouse = target_warehouse
    
    def extract(self, query):
        """Extract data from source"""
        return pd.read_sql(query, self.source)
    
    def transform(self, df):
        """Transform data before loading"""
        # Clean data
        df = df.dropna()
        
        # Standardize formats
        df['date'] = pd.to_datetime(df['date'])
        df['amount'] = df['amount'].astype(float)
        
        # Add derived columns
        df['year'] = df['date'].dt.year
        df['month'] = df['date'].dt.month
        df['quarter'] = df['date'].dt.quarter
        
        # Aggregate if needed
        aggregated = df.groupby(['year', 'month', 'product_id']).agg({
            'amount': 'sum',
            'quantity': 'sum',
            'customer_id': 'nunique'
        }).reset_index()
        
        return aggregated
    
    def load(self, df, table_name):
        """Load into data warehouse"""
        df.to_sql(
            table_name,
            self.warehouse,
            if_exists='append',
            index=False,
            method='multi'
        )
    
    def run(self):
        # Extract
        raw_data = self.extract("SELECT * FROM sales WHERE date >= '2024-01-01'")
        
        # Transform
        transformed_data = self.transform(raw_data)
        
        # Load
        self.load(transformed_data, 'fact_sales_monthly')
```

### ELT (Extract, Load, Transform)

```python
class ELTPipeline:
    def __init__(self, source_db, data_lake, compute_engine):
        self.source = source_db
        self.lake = data_lake
        self.compute = compute_engine  # e.g., Spark, Snowflake
    
    def extract_and_load(self, table_name):
        """Extract from source and load to data lake as-is"""
        query = f"SELECT * FROM {table_name}"
        
        # Stream data in chunks
        for chunk in pd.read_sql(query, self.source, chunksize=10000):
            # Write raw data to data lake
            self.lake.write_raw(chunk, table_name)
    
    def transform_in_place(self, source_table, target_table):
        """Transform data using compute engine"""
        transformation_sql = f"""
        CREATE TABLE {target_table} AS
        SELECT
            DATE_TRUNC('month', sale_date) as month,
            product_id,
            SUM(amount) as total_amount,
            COUNT(DISTINCT customer_id) as unique_customers,
            AVG(amount) as avg_sale_amount
        FROM {source_table}
        WHERE sale_date >= '2024-01-01'
        GROUP BY 1, 2
        """
        
        self.compute.execute(transformation_sql)
    
    def run(self):
        # Extract and Load
        self.extract_and_load('sales')
        
        # Transform in the data warehouse/lake
        self.transform_in_place('raw_sales', 'processed_sales_monthly')
```

## 11. Database Optimization Techniques

### Query Optimization

```sql
-- Use EXPLAIN to understand query execution
EXPLAIN (ANALYZE, BUFFERS) 
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at >= '2024-01-01'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 5;

-- Create appropriate indexes
CREATE INDEX idx_users_created_at ON users(created_at);
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Use covering indexes
CREATE INDEX idx_orders_covering 
ON orders(user_id) 
INCLUDE (id, created_at, total);

-- Optimize JOIN order
-- Bad: Large table first
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'US';

-- Good: Filter first, then join
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'US';
```

### Connection Pooling

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

class DatabaseConnectionPool:
    def __init__(self):
        self.engine = create_engine(
            'postgresql://user:pass@localhost/db',
            poolclass=QueuePool,
            pool_size=20,           # Number of persistent connections
            max_overflow=40,        # Maximum overflow connections
            pool_timeout=30,        # Timeout for getting connection
            pool_recycle=3600,      # Recycle connections after 1 hour
            pool_pre_ping=True      # Test connections before using
        )
    
    def execute_query(self, query):
        with self.engine.connect() as conn:
            return conn.execute(query)
```

## Summary and Key Takeaways

1. **Choose databases based on access patterns** - Not all data fits in relational databases
2. **Replication provides availability and scale** - But introduces consistency challenges
3. **Sharding enables horizontal scaling** - But complicates queries and transactions
4. **Caching reduces database load** - Choose the right caching pattern for your use case
5. **Specialized databases excel at specific tasks** - Time-series for metrics, graphs for relationships
6. **Data warehouses optimize for analytics** - Different from operational databases
7. **ELT leverages modern compute power** - Transform after loading to data lake
8. **Optimization requires measurement** - Use EXPLAIN, monitor slow queries
9. **Connection pooling is essential** - Reuse connections for better performance
10. **Plan for data growth** - Design with scaling in mind from the start

## Next Steps
- Set up database replication in a test environment
- Implement different caching strategies
- Practice sharding with consistent hashing
- Study Part 5: Communication and Messaging for async patterns