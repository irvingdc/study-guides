# System Design Study Guide - Part 6: Scalability and Performance

## 1. Horizontal vs Vertical Scaling

### Understanding Scaling Approaches

**Vertical Scaling (Scale Up):**
- Add more power (CPU, RAM, Storage) to existing servers
- Single machine handles more load
- Simpler architecture, no distributed system complexity
- Hardware limits and diminishing returns
- Single point of failure

**Horizontal Scaling (Scale Out):**
- Add more servers to the resource pool
- Distribute load across multiple machines
- Better fault tolerance and availability
- More complex architecture
- Theoretically unlimited scaling

### Vertical Scaling Implementation

```python
class VerticalScalingMonitor:
    def __init__(self):
        self.cpu_threshold = 80  # percent
        self.memory_threshold = 85  # percent
        self.current_instance_type = 't3.medium'
        
        self.instance_hierarchy = [
            't3.micro',    # 2 vCPU, 1 GB RAM
            't3.small',    # 2 vCPU, 2 GB RAM
            't3.medium',   # 2 vCPU, 4 GB RAM
            't3.large',    # 2 vCPU, 8 GB RAM
            't3.xlarge',   # 4 vCPU, 16 GB RAM
            't3.2xlarge',  # 8 vCPU, 32 GB RAM
            'm5.4xlarge',  # 16 vCPU, 64 GB RAM
            'm5.8xlarge',  # 32 vCPU, 128 GB RAM
            'm5.16xlarge', # 64 vCPU, 256 GB RAM
        ]
    
    def check_scaling_need(self, metrics):
        """Determine if vertical scaling is needed"""
        
        cpu_usage = metrics['cpu_usage']
        memory_usage = metrics['memory_usage']
        
        if cpu_usage > self.cpu_threshold or memory_usage > self.memory_threshold:
            return self.recommend_upgrade()
        elif cpu_usage < 30 and memory_usage < 30:
            return self.recommend_downgrade()
        
        return None
    
    def recommend_upgrade(self):
        current_index = self.instance_hierarchy.index(self.current_instance_type)
        
        if current_index < len(self.instance_hierarchy) - 1:
            return {
                'action': 'upgrade',
                'from': self.current_instance_type,
                'to': self.instance_hierarchy[current_index + 1],
                'estimated_downtime': '5-10 minutes'
            }
        else:
            return {
                'action': 'consider_horizontal_scaling',
                'reason': 'Reached maximum instance size'
            }
    
    def execute_vertical_scaling(self, target_instance_type):
        """
        Steps for vertical scaling:
        1. Create snapshot/backup
        2. Stop instance
        3. Change instance type
        4. Start instance
        5. Verify application health
        """
        steps = [
            self.create_snapshot(),
            self.stop_instance(),
            self.change_instance_type(target_instance_type),
            self.start_instance(),
            self.health_check()
        ]
        
        for step in steps:
            if not step['success']:
                self.rollback()
                return False
        
        return True
```

### Horizontal Scaling Implementation

```python
class HorizontalScalingManager:
    def __init__(self, min_instances=2, max_instances=10):
        self.min_instances = min_instances
        self.max_instances = max_instances
        self.current_instances = min_instances
        self.scale_up_threshold = 70  # CPU %
        self.scale_down_threshold = 30  # CPU %
        self.cooldown_period = 300  # seconds
        self.last_scaling_time = 0
    
    def auto_scale(self, metrics):
        """Auto-scaling based on metrics"""
        
        # Check cooldown period
        if time.time() - self.last_scaling_time < self.cooldown_period:
            return None
        
        avg_cpu = self.calculate_average_cpu(metrics)
        
        if avg_cpu > self.scale_up_threshold:
            return self.scale_up()
        elif avg_cpu < self.scale_down_threshold:
            return self.scale_down()
        
        return None
    
    def scale_up(self):
        """Add more instances"""
        if self.current_instances >= self.max_instances:
            return {'status': 'max_capacity_reached'}
        
        new_instances_count = min(
            self.current_instances + 2,  # Add 2 instances
            self.max_instances
        )
        
        new_instances = []
        for i in range(new_instances_count - self.current_instances):
            instance = self.launch_instance()
            new_instances.append(instance)
        
        # Update load balancer
        self.register_with_load_balancer(new_instances)
        
        self.current_instances = new_instances_count
        self.last_scaling_time = time.time()
        
        return {
            'action': 'scaled_up',
            'new_instance_count': self.current_instances,
            'added_instances': len(new_instances)
        }
    
    def scale_down(self):
        """Remove instances"""
        if self.current_instances <= self.min_instances:
            return {'status': 'min_capacity_reached'}
        
        instances_to_remove = min(
            2,  # Remove 2 instances
            self.current_instances - self.min_instances
        )
        
        # Gracefully drain connections
        removed_instances = self.drain_and_terminate(instances_to_remove)
        
        self.current_instances -= instances_to_remove
        self.last_scaling_time = time.time()
        
        return {
            'action': 'scaled_down',
            'new_instance_count': self.current_instances,
            'removed_instances': instances_to_remove
        }
    
    def predictive_scaling(self, historical_data):
        """Predict future load and scale proactively"""
        
        # Analyze patterns
        daily_pattern = self.analyze_daily_pattern(historical_data)
        weekly_pattern = self.analyze_weekly_pattern(historical_data)
        
        # Predict next hour's load
        current_hour = datetime.now().hour
        current_day = datetime.now().weekday()
        
        predicted_load = (
            daily_pattern[current_hour] * 0.7 +
            weekly_pattern[current_day][current_hour] * 0.3
        )
        
        # Scale based on prediction
        required_instances = self.calculate_required_instances(predicted_load)
        
        if required_instances > self.current_instances:
            return self.scale_to(required_instances)
        
        return None
```

## 2. Database Scaling Strategies

### Read Replica Scaling

```python
class ReadReplicaManager:
    def __init__(self, master_db):
        self.master = master_db
        self.read_replicas = []
        self.read_replica_lag_threshold = 1000  # milliseconds
    
    def add_read_replica(self, region=None):
        """Add a new read replica"""
        
        replica_config = {
            'source': self.master,
            'region': region or self.master['region'],
            'instance_type': self.master['instance_type'],
            'replication_type': 'async'
        }
        
        # Create replica
        replica = self.create_replica(replica_config)
        
        # Wait for initial sync
        self.wait_for_sync(replica)
        
        self.read_replicas.append(replica)
        
        return replica
    
    def route_query(self, query, consistency_level='eventual'):
        """Route queries based on consistency requirements"""
        
        if self.is_write_query(query):
            return self.execute_on_master(query)
        
        if consistency_level == 'strong':
            return self.execute_on_master(query)
        
        if consistency_level == 'bounded':
            # Use replica with acceptable lag
            replica = self.get_replica_with_max_lag(self.read_replica_lag_threshold)
            if replica:
                return self.execute_on_replica(replica, query)
            else:
                return self.execute_on_master(query)
        
        # Eventual consistency - use any available replica
        replica = self.get_least_loaded_replica()
        return self.execute_on_replica(replica, query)
    
    def get_replica_with_max_lag(self, max_lag_ms):
        """Get replica with lag below threshold"""
        
        eligible_replicas = []
        
        for replica in self.read_replicas:
            lag = self.get_replication_lag(replica)
            if lag < max_lag_ms:
                eligible_replicas.append({
                    'replica': replica,
                    'lag': lag,
                    'load': self.get_replica_load(replica)
                })
        
        if not eligible_replicas:
            return None
        
        # Return replica with lowest load
        return min(eligible_replicas, key=lambda x: x['load'])['replica']
    
    def handle_replica_failure(self, failed_replica):
        """Handle read replica failure"""
        
        # Remove from pool
        self.read_replicas.remove(failed_replica)
        
        # Check if we need more replicas
        if len(self.read_replicas) < 2:
            # Create new replica
            self.add_read_replica()
        
        # Redistribute load
        self.rebalance_connections()
```

### Database Sharding Implementation

```python
class DatabaseShardManager:
    def __init__(self, num_shards=4):
        self.num_shards = num_shards
        self.shards = self.initialize_shards()
        self.shard_key_function = self.hash_based_sharding
    
    def initialize_shards(self):
        """Initialize database shards"""
        shards = {}
        
        for i in range(self.num_shards):
            shards[i] = {
                'id': i,
                'host': f'shard{i}.db.example.com',
                'connection_pool': self.create_connection_pool(f'shard{i}'),
                'weight': 1,
                'status': 'active'
            }
        
        return shards
    
    def hash_based_sharding(self, key):
        """Determine shard using consistent hashing"""
        hash_value = hashlib.md5(str(key).encode()).hexdigest()
        shard_id = int(hash_value, 16) % self.num_shards
        return shard_id
    
    def range_based_sharding(self, key):
        """Determine shard based on key range"""
        ranges = [
            (0, 1000000),
            (1000001, 2000000),
            (2000001, 3000000),
            (3000001, float('inf'))
        ]
        
        for i, (start, end) in enumerate(ranges):
            if start <= key <= end:
                return i
        
        return 0  # Default shard
    
    def geo_sharding(self, location):
        """Shard based on geographic location"""
        geo_mapping = {
            'US': 0,
            'EU': 1,
            'ASIA': 2,
            'OTHER': 3
        }
        
        region = self.get_region(location)
        return geo_mapping.get(region, 3)
    
    def execute_query(self, query, shard_key):
        """Execute query on appropriate shard"""
        
        shard_id = self.shard_key_function(shard_key)
        shard = self.shards[shard_id]
        
        if shard['status'] != 'active':
            # Failover to replica shard
            shard = self.get_replica_shard(shard_id)
        
        return self.execute_on_shard(shard, query)
    
    def execute_cross_shard_query(self, query):
        """Execute query across multiple shards"""
        
        results = []
        threads = []
        
        # Execute on all shards in parallel
        for shard_id, shard in self.shards.items():
            if shard['status'] == 'active':
                thread = threading.Thread(
                    target=lambda s, q, r: r.append(self.execute_on_shard(s, q)),
                    args=(shard, query, results)
                )
                thread.start()
                threads.append(thread)
        
        # Wait for all queries to complete
        for thread in threads:
            thread.join()
        
        # Merge results
        return self.merge_shard_results(results)
    
    def rebalance_shards(self):
        """Rebalance data across shards"""
        
        # Calculate ideal distribution
        total_data = sum(self.get_shard_size(s) for s in self.shards.values())
        ideal_size = total_data / self.num_shards
        
        migrations = []
        
        for shard in self.shards.values():
            current_size = self.get_shard_size(shard)
            
            if current_size > ideal_size * 1.2:  # 20% over ideal
                # Need to migrate data out
                excess = current_size - ideal_size
                migrations.append({
                    'from': shard,
                    'amount': excess,
                    'direction': 'out'
                })
            elif current_size < ideal_size * 0.8:  # 20% under ideal
                # Need to migrate data in
                deficit = ideal_size - current_size
                migrations.append({
                    'to': shard,
                    'amount': deficit,
                    'direction': 'in'
                })
        
        # Execute migrations
        self.execute_migrations(migrations)
```

## 3. Caching at Different Layers

### Multi-Layer Caching Strategy

```python
class MultiLayerCache:
    def __init__(self):
        self.browser_cache = BrowserCache()
        self.cdn_cache = CDNCache()
        self.app_cache = ApplicationCache()
        self.db_cache = DatabaseCache()
    
    def get_with_cache(self, key, fetch_function):
        """Get value using cache hierarchy"""
        
        # L1: Browser cache (client-side)
        cache_headers = self.get_cache_headers(key)
        
        # L2: CDN cache (edge)
        cdn_value = self.cdn_cache.get(key)
        if cdn_value:
            return {
                'value': cdn_value,
                'cache_hit': 'CDN',
                'headers': cache_headers
            }
        
        # L3: Application cache (server-side memory)
        app_value = self.app_cache.get(key)
        if app_value:
            # Populate CDN cache
            self.cdn_cache.set(key, app_value, ttl=3600)
            return {
                'value': app_value,
                'cache_hit': 'Application',
                'headers': cache_headers
            }
        
        # L4: Database cache (query result cache)
        db_value = self.db_cache.get(key)
        if db_value:
            # Populate higher level caches
            self.app_cache.set(key, db_value, ttl=300)
            self.cdn_cache.set(key, db_value, ttl=3600)
            return {
                'value': db_value,
                'cache_hit': 'Database',
                'headers': cache_headers
            }
        
        # Cache miss - fetch from source
        value = fetch_function()
        
        # Populate all cache layers
        self.populate_caches(key, value)
        
        return {
            'value': value,
            'cache_hit': None,
            'headers': cache_headers
        }
    
    def get_cache_headers(self, key):
        """Generate appropriate cache headers"""
        
        content_type = self.determine_content_type(key)
        
        if content_type in ['image', 'css', 'js']:
            # Long cache for static assets
            return {
                'Cache-Control': 'public, max-age=31536000, immutable',
                'ETag': self.generate_etag(key)
            }
        elif content_type == 'api':
            # Short cache for API responses
            return {
                'Cache-Control': 'private, max-age=60',
                'Vary': 'Accept-Encoding, Authorization'
            }
        else:
            # No cache for dynamic content
            return {
                'Cache-Control': 'no-cache, no-store, must-revalidate',
                'Pragma': 'no-cache',
                'Expires': '0'
            }

class ApplicationCache:
    def __init__(self, max_memory_mb=512):
        self.cache = {}
        self.max_memory = max_memory_mb * 1024 * 1024  # Convert to bytes
        self.current_memory = 0
        self.access_counts = {}
        self.last_access = {}
    
    def get(self, key):
        """Get from application cache with LRU eviction"""
        
        if key in self.cache:
            self.access_counts[key] = self.access_counts.get(key, 0) + 1
            self.last_access[key] = time.time()
            return self.cache[key]
        
        return None
    
    def set(self, key, value, ttl=300):
        """Set in application cache with memory management"""
        
        value_size = self.get_size(value)
        
        # Check if we need to evict
        while self.current_memory + value_size > self.max_memory:
            self.evict_lru()
        
        self.cache[key] = {
            'value': value,
            'expires_at': time.time() + ttl,
            'size': value_size
        }
        
        self.current_memory += value_size
        self.access_counts[key] = 1
        self.last_access[key] = time.time()
    
    def evict_lru(self):
        """Evict least recently used item"""
        
        if not self.cache:
            return
        
        # Find LRU item
        lru_key = min(self.last_access.keys(), key=self.last_access.get)
        
        # Remove from cache
        item = self.cache.pop(lru_key)
        self.current_memory -= item['size']
        del self.access_counts[lru_key]
        del self.last_access[lru_key]
```

### Cache Warming and Preloading

```python
class CacheWarmer:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def warm_cache_on_startup(self):
        """Preload frequently accessed data on startup"""
        
        # Load popular items
        popular_items = self.database.query("""
            SELECT item_id, data
            FROM items
            WHERE access_count > 1000
            ORDER BY access_count DESC
            LIMIT 100
        """)
        
        for item in popular_items:
            self.cache.set(f"item:{item['item_id']}", item['data'])
        
        # Load configuration data
        configs = self.database.query("SELECT * FROM configurations")
        for config in configs:
            self.cache.set(f"config:{config['key']}", config['value'])
        
        return len(popular_items) + len(configs)
    
    def predictive_cache_warming(self, user_id):
        """Predictively warm cache based on user behavior"""
        
        # Get user's historical access pattern
        history = self.database.query("""
            SELECT item_id, COUNT(*) as frequency
            FROM user_access_log
            WHERE user_id = %s
              AND timestamp > NOW() - INTERVAL '7 days'
            GROUP BY item_id
            ORDER BY frequency DESC
            LIMIT 20
        """, [user_id])
        
        # Preload likely items
        for record in history:
            item = self.database.get(f"SELECT * FROM items WHERE id = %s", 
                                    [record['item_id']])
            self.cache.set(f"user:{user_id}:item:{record['item_id']}", 
                          item, ttl=1800)
    
    def scheduled_cache_refresh(self):
        """Refresh cache entries before they expire"""
        
        while True:
            # Get items expiring soon
            expiring_keys = self.cache.get_expiring_keys(threshold=60)
            
            for key in expiring_keys:
                # Check if still frequently accessed
                if self.is_frequently_accessed(key):
                    # Refresh from database
                    fresh_data = self.fetch_from_database(key)
                    self.cache.set(key, fresh_data)
            
            time.sleep(30)  # Check every 30 seconds
```

## 4. Performance Optimization Techniques

### Query Optimization

```python
class QueryOptimizer:
    def __init__(self, database):
        self.database = database
        self.query_cache = {}
        self.slow_query_threshold = 1000  # milliseconds
    
    def optimize_query(self, query):
        """Analyze and optimize SQL query"""
        
        # Get query execution plan
        explain_result = self.database.execute(f"EXPLAIN ANALYZE {query}")
        
        optimizations = []
        
        # Check for missing indexes
        if self.has_sequential_scan(explain_result):
            optimizations.append(self.suggest_index(query))
        
        # Check for N+1 queries
        if self.detect_n_plus_one(query):
            optimizations.append(self.suggest_join_or_batch(query))
        
        # Check for SELECT *
        if "SELECT *" in query:
            optimizations.append({
                'type': 'select_specific_columns',
                'suggestion': 'Select only needed columns instead of *'
            })
        
        # Check for missing LIMIT
        if self.is_unbounded_query(query):
            optimizations.append({
                'type': 'add_limit',
                'suggestion': 'Add LIMIT clause to prevent fetching too many rows'
            })
        
        return optimizations
    
    def create_covering_index(self, table, columns):
        """Create covering index for query optimization"""
        
        index_name = f"idx_{table}_{'_'.join(columns)}"
        
        # Check if index already exists
        existing_indexes = self.database.query(f"""
            SELECT indexname 
            FROM pg_indexes 
            WHERE tablename = '{table}'
        """)
        
        if index_name not in [idx['indexname'] for idx in existing_indexes]:
            # Create index
            columns_str = ', '.join(columns)
            self.database.execute(f"""
                CREATE INDEX CONCURRENTLY {index_name}
                ON {table} ({columns_str})
            """)
            
            return f"Created index: {index_name}"
        
        return f"Index already exists: {index_name}"
    
    def batch_queries(self, queries):
        """Batch multiple queries for efficiency"""
        
        if len(queries) == 1:
            return self.database.execute(queries[0])
        
        # Use transaction for batching
        results = []
        
        with self.database.transaction():
            for query in queries:
                result = self.database.execute(query)
                results.append(result)
        
        return results
    
    def implement_query_result_cache(self, query, cache_key=None):
        """Cache query results"""
        
        if not cache_key:
            cache_key = hashlib.md5(query.encode()).hexdigest()
        
        # Check cache
        if cache_key in self.query_cache:
            cache_entry = self.query_cache[cache_key]
            if time.time() < cache_entry['expires_at']:
                return cache_entry['result']
        
        # Execute query
        start_time = time.time()
        result = self.database.execute(query)
        execution_time = (time.time() - start_time) * 1000
        
        # Log slow queries
        if execution_time > self.slow_query_threshold:
            self.log_slow_query(query, execution_time)
        
        # Cache result
        self.query_cache[cache_key] = {
            'result': result,
            'expires_at': time.time() + 300  # 5 minutes
        }
        
        return result
```

### Code-Level Optimizations

```python
class PerformanceOptimizer:
    def __init__(self):
        self.profiler = cProfile.Profile()
    
    def optimize_loops(self, data_processor):
        """Optimize loop performance"""
        
        # Bad: Multiple passes through data
        def inefficient_processing(data):
            filtered = []
            for item in data:
                if item['status'] == 'active':
                    filtered.append(item)
            
            transformed = []
            for item in filtered:
                transformed.append({
                    'id': item['id'],
                    'name': item['name'].upper()
                })
            
            return transformed
        
        # Good: Single pass with generator
        def efficient_processing(data):
            return [
                {'id': item['id'], 'name': item['name'].upper()}
                for item in data
                if item['status'] == 'active'
            ]
        
        # Best: Generator for memory efficiency
        def memory_efficient_processing(data):
            for item in data:
                if item['status'] == 'active':
                    yield {'id': item['id'], 'name': item['name'].upper()}
    
    def optimize_string_operations(self):
        """Optimize string concatenation"""
        
        # Bad: String concatenation in loop
        def bad_concat(items):
            result = ""
            for item in items:
                result += str(item) + ","
            return result
        
        # Good: Join
        def good_concat(items):
            return ",".join(str(item) for item in items)
        
        # Best: StringIO for large strings
        def best_concat(items):
            from io import StringIO
            buffer = StringIO()
            for item in items:
                buffer.write(str(item))
                buffer.write(",")
            return buffer.getvalue()
    
    def implement_lazy_loading(self):
        """Implement lazy loading pattern"""
        
        class LazyProperty:
            def __init__(self, function):
                self.function = function
                self.name = function.__name__
            
            def __get__(self, obj, type=None):
                if obj is None:
                    return self
                
                # Compute value on first access
                value = self.function(obj)
                # Store for future access
                setattr(obj, self.name, value)
                return value
        
        class DataModel:
            def __init__(self, id):
                self.id = id
            
            @LazyProperty
            def expensive_computation(self):
                # Only computed when accessed
                print(f"Computing expensive value for {self.id}")
                time.sleep(1)  # Simulate expensive operation
                return f"Computed value for {self.id}"
    
    def use_connection_pooling(self):
        """Implement connection pooling"""
        
        from queue import Queue
        
        class ConnectionPool:
            def __init__(self, create_connection, max_connections=10):
                self.create_connection = create_connection
                self.pool = Queue(maxsize=max_connections)
                self.size = 0
                self.max_connections = max_connections
            
            def get_connection(self):
                if self.pool.empty() and self.size < self.max_connections:
                    # Create new connection
                    self.size += 1
                    return self.create_connection()
                
                # Get from pool (blocks if empty)
                return self.pool.get()
            
            def return_connection(self, conn):
                if not conn.is_closed():
                    self.pool.put(conn)
                else:
                    self.size -= 1
            
            @contextmanager
            def connection(self):
                conn = self.get_connection()
                try:
                    yield conn
                finally:
                    self.return_connection(conn)
```

## 5. Auto-Scaling Strategies

### Metric-Based Auto-Scaling

```python
class MetricBasedAutoScaler:
    def __init__(self):
        self.metrics = {
            'cpu': {'threshold': 70, 'weight': 0.4},
            'memory': {'threshold': 80, 'weight': 0.3},
            'request_rate': {'threshold': 1000, 'weight': 0.2},
            'response_time': {'threshold': 500, 'weight': 0.1}
        }
        
        self.scaling_policies = {
            'scale_up': {
                'threshold': 0.7,
                'cooldown': 300,
                'increment': 2
            },
            'scale_down': {
                'threshold': 0.3,
                'cooldown': 600,
                'increment': 1
            }
        }
    
    def calculate_scaling_score(self, current_metrics):
        """Calculate composite scaling score"""
        
        score = 0
        
        for metric_name, config in self.metrics.items():
            if metric_name in current_metrics:
                metric_value = current_metrics[metric_name]
                threshold = config['threshold']
                weight = config['weight']
                
                # Normalize metric to 0-1 scale
                normalized = min(metric_value / threshold, 1.0)
                score += normalized * weight
        
        return score
    
    def make_scaling_decision(self, current_metrics, current_instances):
        """Decide whether to scale based on metrics"""
        
        score = self.calculate_scaling_score(current_metrics)
        
        if score > self.scaling_policies['scale_up']['threshold']:
            return {
                'action': 'scale_up',
                'reason': f'Scaling score {score:.2f} exceeds threshold',
                'new_instances': current_instances + self.scaling_policies['scale_up']['increment']
            }
        elif score < self.scaling_policies['scale_down']['threshold']:
            return {
                'action': 'scale_down',
                'reason': f'Scaling score {score:.2f} below threshold',
                'new_instances': max(2, current_instances - self.scaling_policies['scale_down']['increment'])
            }
        else:
            return {
                'action': 'no_change',
                'reason': f'Scaling score {score:.2f} within thresholds'
            }

class PredictiveAutoScaler:
    def __init__(self):
        self.historical_data = []
        self.prediction_window = 3600  # 1 hour ahead
    
    def train_model(self, historical_metrics):
        """Train ML model for load prediction"""
        
        from sklearn.ensemble import RandomForestRegressor
        from sklearn.preprocessing import StandardScaler
        
        # Prepare features
        features = []
        targets = []
        
        for i in range(len(historical_metrics) - 1):
            current = historical_metrics[i]
            future = historical_metrics[i + 1]
            
            features.append([
                current['hour'],
                current['day_of_week'],
                current['cpu'],
                current['memory'],
                current['request_rate']
            ])
            
            targets.append(future['request_rate'])
        
        # Train model
        scaler = StandardScaler()
        X = scaler.fit_transform(features)
        y = targets
        
        model = RandomForestRegressor(n_estimators=100)
        model.fit(X, y)
        
        return model, scaler
    
    def predict_load(self, model, scaler, current_metrics):
        """Predict future load"""
        
        features = [[
            current_metrics['hour'],
            current_metrics['day_of_week'],
            current_metrics['cpu'],
            current_metrics['memory'],
            current_metrics['request_rate']
        ]]
        
        X = scaler.transform(features)
        predicted_load = model.predict(X)[0]
        
        return predicted_load
    
    def schedule_scaling(self, predicted_load, current_capacity):
        """Schedule scaling based on predictions"""
        
        required_capacity = self.calculate_required_capacity(predicted_load)
        
        if required_capacity > current_capacity:
            # Schedule scale up
            return {
                'action': 'schedule_scale_up',
                'when': 'in_30_minutes',
                'target_capacity': required_capacity,
                'reason': f'Predicted load: {predicted_load:.0f} requests/sec'
            }
        elif required_capacity < current_capacity * 0.7:
            # Schedule scale down
            return {
                'action': 'schedule_scale_down',
                'when': 'in_45_minutes',
                'target_capacity': required_capacity,
                'reason': f'Predicted load: {predicted_load:.0f} requests/sec'
            }
        
        return {'action': 'no_change'}
```

## 6. Global Distribution and Geo-Replication

### Multi-Region Architecture

```python
class MultiRegionDeployment:
    def __init__(self):
        self.regions = {
            'us-east-1': {'primary': True, 'latency': 0},
            'us-west-2': {'primary': False, 'latency': 40},
            'eu-west-1': {'primary': False, 'latency': 80},
            'ap-southeast-1': {'primary': False, 'latency': 150}
        }
        
        self.data_sync_strategy = 'async'  # async or sync
    
    def setup_geo_replication(self):
        """Setup cross-region replication"""
        
        primary_region = self.get_primary_region()
        secondary_regions = self.get_secondary_regions()
        
        replication_config = {
            'source': primary_region,
            'targets': secondary_regions,
            'replication_type': self.data_sync_strategy,
            'conflict_resolution': 'last_write_wins'
        }
        
        # Setup database replication
        self.setup_database_replication(replication_config)
        
        # Setup storage replication
        self.setup_storage_replication(replication_config)
        
        # Setup CDN distribution
        self.setup_cdn_distribution()
        
        return replication_config
    
    def implement_geo_routing(self):
        """Route users to nearest region"""
        
        class GeoRouter:
            def __init__(self, regions):
                self.regions = regions
            
            def get_nearest_region(self, user_location):
                """Find nearest region based on user location"""
                
                user_lat, user_lon = user_location
                min_distance = float('inf')
                nearest_region = None
                
                for region, config in self.regions.items():
                    region_lat, region_lon = config['coordinates']
                    distance = self.calculate_distance(
                        user_lat, user_lon,
                        region_lat, region_lon
                    )
                    
                    if distance < min_distance:
                        min_distance = distance
                        nearest_region = region
                
                return nearest_region
            
            def calculate_distance(self, lat1, lon1, lat2, lon2):
                """Calculate distance using Haversine formula"""
                from math import radians, sin, cos, sqrt, atan2
                
                R = 6371  # Earth's radius in km
                
                lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
                
                dlat = lat2 - lat1
                dlon = lon2 - lon1
                
                a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
                c = 2 * atan2(sqrt(a), sqrt(1-a))
                
                return R * c
    
    def handle_region_failover(self, failed_region):
        """Handle region failure"""
        
        # Promote secondary to primary if needed
        if self.regions[failed_region]['primary']:
            # Find best secondary to promote
            best_secondary = self.select_new_primary()
            self.promote_to_primary(best_secondary)
        
        # Update DNS to route away from failed region
        self.update_dns_routing(exclude_region=failed_region)
        
        # Notify monitoring
        self.alert_on_region_failure(failed_region)
        
        return {
            'failed_region': failed_region,
            'new_primary': best_secondary if self.regions[failed_region]['primary'] else None,
            'status': 'failover_complete'
        }
```

### Cross-Region Data Consistency

```python
class CrossRegionConsistency:
    def __init__(self):
        self.vector_clock = {}
        self.conflict_resolution = 'last_write_wins'
    
    def write_with_consistency(self, key, value, consistency_level='eventual'):
        """Write data with specified consistency level"""
        
        if consistency_level == 'strong':
            # Synchronous replication to all regions
            return self.strong_consistency_write(key, value)
        elif consistency_level == 'bounded':
            # Synchronous to nearby regions, async to far
            return self.bounded_consistency_write(key, value)
        else:
            # Asynchronous replication
            return self.eventual_consistency_write(key, value)
    
    def strong_consistency_write(self, key, value):
        """Write with strong consistency across regions"""
        
        regions = self.get_all_regions()
        write_quorum = len(regions) // 2 + 1
        
        # Phase 1: Prepare
        prepare_results = []
        for region in regions:
            result = self.prepare_write(region, key, value)
            prepare_results.append(result)
        
        successful_prepares = sum(1 for r in prepare_results if r['success'])
        
        if successful_prepares >= write_quorum:
            # Phase 2: Commit
            for region in regions:
                self.commit_write(region, key, value)
            
            return {'success': True, 'consistency': 'strong'}
        else:
            # Rollback
            for region in regions:
                self.rollback_write(region, key)
            
            return {'success': False, 'reason': 'Insufficient quorum'}
    
    def resolve_conflicts(self, conflicts):
        """Resolve data conflicts between regions"""
        
        if self.conflict_resolution == 'last_write_wins':
            # Choose the write with latest timestamp
            return max(conflicts, key=lambda x: x['timestamp'])
        
        elif self.conflict_resolution == 'vector_clock':
            # Use vector clocks for causality
            return self.vector_clock_resolution(conflicts)
        
        elif self.conflict_resolution == 'custom':
            # Application-specific resolution
            return self.custom_conflict_resolution(conflicts)
    
    def vector_clock_resolution(self, conflicts):
        """Resolve conflicts using vector clocks"""
        
        # Find concurrent updates
        concurrent = []
        
        for i, update1 in enumerate(conflicts):
            is_concurrent = True
            
            for j, update2 in enumerate(conflicts):
                if i != j:
                    if self.happens_before(update1['vector_clock'], 
                                          update2['vector_clock']):
                        is_concurrent = False
                        break
            
            if is_concurrent:
                concurrent.append(update1)
        
        if len(concurrent) == 1:
            return concurrent[0]
        else:
            # Multiple concurrent updates - merge or pick one
            return self.merge_concurrent_updates(concurrent)
```

## 7. Edge Computing

### Edge Server Implementation

```python
class EdgeComputing:
    def __init__(self):
        self.edge_locations = {}
        self.compute_functions = {}
    
    def deploy_edge_function(self, function_code, locations=None):
        """Deploy compute function to edge locations"""
        
        function_id = str(uuid.uuid4())
        
        # Compile/prepare function
        compiled_function = self.compile_function(function_code)
        
        # Deploy to specified locations or all
        target_locations = locations or self.edge_locations.keys()
        
        for location in target_locations:
            self.edge_locations[location]['functions'][function_id] = {
                'code': compiled_function,
                'created_at': time.time(),
                'invocations': 0,
                'avg_latency': 0
            }
        
        return function_id
    
    def execute_at_edge(self, function_id, request, user_location):
        """Execute function at nearest edge location"""
        
        # Find nearest edge location
        nearest_edge = self.find_nearest_edge(user_location)
        
        # Check if function is deployed there
        if function_id not in self.edge_locations[nearest_edge]['functions']:
            # Deploy on-demand
            self.deploy_edge_function(function_id, [nearest_edge])
        
        # Execute function
        start_time = time.time()
        result = self.execute_function(
            self.edge_locations[nearest_edge]['functions'][function_id],
            request
        )
        latency = time.time() - start_time
        
        # Update metrics
        self.update_edge_metrics(nearest_edge, function_id, latency)
        
        return {
            'result': result,
            'edge_location': nearest_edge,
            'latency_ms': latency * 1000
        }
    
    def implement_edge_caching(self):
        """Implement intelligent edge caching"""
        
        class EdgeCache:
            def __init__(self, size_limit_mb=100):
                self.cache = {}
                self.size_limit = size_limit_mb * 1024 * 1024
                self.current_size = 0
                self.access_patterns = {}
            
            def should_cache_at_edge(self, content_key):
                """Determine if content should be cached at edge"""
                
                # Check access frequency
                access_count = self.access_patterns.get(content_key, {}).get('count', 0)
                
                # Check content size
                content_size = self.get_content_size(content_key)
                
                # Decision factors
                factors = {
                    'high_frequency': access_count > 100,
                    'reasonable_size': content_size < 10 * 1024 * 1024,  # 10MB
                    'geographic_clustering': self.has_geographic_clustering(content_key),
                    'latency_sensitive': self.is_latency_sensitive(content_key)
                }
                
                # Cache if multiple factors are true
                return sum(factors.values()) >= 2
            
            def eviction_policy(self):
                """LFU with aging for edge cache eviction"""
                
                if self.current_size < self.size_limit:
                    return None
                
                # Calculate scores
                scores = {}
                current_time = time.time()
                
                for key, metadata in self.cache.items():
                    age = current_time - metadata['last_access']
                    frequency = metadata['access_count']
                    size = metadata['size']
                    
                    # Lower score = more likely to evict
                    score = frequency / (1 + age/3600) / (1 + size/1024/1024)
                    scores[key] = score
                
                # Evict lowest scoring item
                evict_key = min(scores, key=scores.get)
                self.evict(evict_key)
                
                return evict_key
```

## 8. Capacity Planning and Load Testing

### Capacity Planning

```python
class CapacityPlanner:
    def __init__(self):
        self.growth_rate = 0.2  # 20% monthly growth
        self.peak_factor = 3  # Peak traffic is 3x average
        self.redundancy_factor = 1.5  # 50% redundancy
    
    def calculate_capacity_requirements(self, current_metrics, months_ahead=12):
        """Calculate future capacity requirements"""
        
        projections = []
        
        for month in range(1, months_ahead + 1):
            # Project growth
            projected_users = current_metrics['users'] * (1 + self.growth_rate) ** month
            projected_rps = projected_users * current_metrics['rps_per_user']
            
            # Calculate peak
            peak_rps = projected_rps * self.peak_factor
            
            # Calculate infrastructure needs
            servers_needed = math.ceil(peak_rps / current_metrics['rps_per_server'])
            servers_with_redundancy = math.ceil(servers_needed * self.redundancy_factor)
            
            # Storage requirements
            storage_per_user = current_metrics['storage_per_user']
            total_storage = projected_users * storage_per_user
            
            # Bandwidth requirements
            bandwidth_per_request = current_metrics['bandwidth_per_request']
            peak_bandwidth = peak_rps * bandwidth_per_request
            
            projections.append({
                'month': month,
                'users': int(projected_users),
                'peak_rps': int(peak_rps),
                'servers': servers_with_redundancy,
                'storage_tb': total_storage / 1024 / 1024 / 1024 / 1024,
                'bandwidth_gbps': peak_bandwidth * 8 / 1024 / 1024 / 1024,
                'estimated_cost': self.calculate_cost(
                    servers_with_redundancy,
                    total_storage,
                    peak_bandwidth
                )
            })
        
        return projections
    
    def calculate_cost(self, servers, storage_bytes, bandwidth_bytes_per_sec):
        """Estimate infrastructure costs"""
        
        # AWS pricing example
        server_cost_per_month = 100  # per server
        storage_cost_per_gb_month = 0.023
        bandwidth_cost_per_gb = 0.09
        
        monthly_cost = (
            servers * server_cost_per_month +
            (storage_bytes / 1024 / 1024 / 1024) * storage_cost_per_gb_month +
            (bandwidth_bytes_per_sec * 86400 * 30 / 1024 / 1024 / 1024) * bandwidth_cost_per_gb
        )
        
        return monthly_cost

class LoadTester:
    def __init__(self, target_url):
        self.target_url = target_url
        self.results = []
    
    async def run_load_test(self, concurrent_users=100, duration_seconds=60):
        """Run load test with specified parameters"""
        
        print(f"Starting load test: {concurrent_users} users for {duration_seconds}s")
        
        start_time = time.time()
        tasks = []
        
        for user_id in range(concurrent_users):
            task = asyncio.create_task(
                self.simulate_user(user_id, start_time + duration_seconds)
            )
            tasks.append(task)
        
        # Wait for all users to complete
        results = await asyncio.gather(*tasks)
        
        # Analyze results
        return self.analyze_results(results)
    
    async def simulate_user(self, user_id, end_time):
        """Simulate a single user's behavior"""
        
        user_results = {
            'user_id': user_id,
            'requests': [],
            'errors': 0
        }
        
        async with aiohttp.ClientSession() as session:
            while time.time() < end_time:
                # Simulate user think time
                await asyncio.sleep(random.uniform(1, 3))
                
                # Make request
                start = time.time()
                try:
                    async with session.get(self.target_url) as response:
                        latency = time.time() - start
                        user_results['requests'].append({
                            'timestamp': start,
                            'latency': latency,
                            'status': response.status
                        })
                except Exception as e:
                    user_results['errors'] += 1
        
        return user_results
    
    def analyze_results(self, all_results):
        """Analyze load test results"""
        
        all_requests = []
        total_errors = 0
        
        for user_result in all_results:
            all_requests.extend(user_result['requests'])
            total_errors += user_result['errors']
        
        latencies = [r['latency'] for r in all_requests]
        latencies.sort()
        
        return {
            'total_requests': len(all_requests),
            'total_errors': total_errors,
            'error_rate': total_errors / (len(all_requests) + total_errors),
            'latency_p50': latencies[len(latencies) // 2] if latencies else 0,
            'latency_p95': latencies[int(len(latencies) * 0.95)] if latencies else 0,
            'latency_p99': latencies[int(len(latencies) * 0.99)] if latencies else 0,
            'rps': len(all_requests) / 60  # Requests per second
        }
```

## Summary and Key Takeaways

1. **Start with vertical scaling, move to horizontal** - Vertical is simpler but has limits
2. **Cache aggressively at multiple layers** - Browser, CDN, application, database
3. **Use read replicas for read-heavy workloads** - Separate reads from writes
4. **Implement auto-scaling based on multiple metrics** - Not just CPU
5. **Plan for peak traffic, not average** - Systems fail at peak
6. **Geographic distribution reduces latency** - Deploy close to users
7. **Edge computing for latency-sensitive operations** - Process data near source
8. **Load test before production** - Find bottlenecks early
9. **Monitor and optimize continuously** - Performance degrades over time
10. **Design for failure** - Everything fails, plan accordingly

## Next Steps
- Implement auto-scaling policies
- Set up multi-region deployment
- Conduct load testing on your system
- Study Part 7: Real-World Case Studies for practical applications