# System Design Study Guide - Part 2: Components and Building Blocks

## 1. Load Balancers

### What is a Load Balancer?
A load balancer distributes incoming network traffic across multiple servers to ensure no single server becomes overwhelmed, improving responsiveness and availability.

### Layer 4 vs Layer 7 Load Balancing

**Layer 4 (Transport Layer) Load Balancing**
- Operates at TCP/UDP level
- Makes routing decisions based on IP addresses and ports
- Cannot inspect application data
- Faster performance (less processing overhead)
- Single TCP connection from client to server

**Example L4 Configuration:**
```
Client (192.168.1.100:45678) → LB → Server1 (10.0.0.1:80)
Client (192.168.1.101:45679) → LB → Server2 (10.0.0.2:80)
```

**When to use L4:**
- Simple load distribution
- Non-HTTP protocols (database connections, game servers)
- Minimal latency requirements
- End-to-end encryption requirements

**Layer 7 (Application Layer) Load Balancing**
- Operates at HTTP/HTTPS level
- Can inspect headers, URLs, cookies
- Content-based routing possible
- SSL termination capabilities
- Two connections: client-to-LB and LB-to-server

**Example L7 Routing Rules:**
```
/api/* → API servers
/static/* → Static content servers
/ws/* → WebSocket servers
Cookie: session=premium → Premium server pool
```

**When to use L7:**
- Content-based routing needed
- HTTP header manipulation
- SSL termination at load balancer
- Advanced features (compression, caching)

### Load Balancing Algorithms

**Round Robin**
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (cycle repeats)
```
- Simple and predictable
- Works well with equal-capacity servers
- Doesn't consider server load

**Weighted Round Robin**
```
Server A (weight=3): Gets 3 requests
Server B (weight=2): Gets 2 requests
Server C (weight=1): Gets 1 request
Cycle: A,A,A,B,B,C
```
- Accounts for different server capacities
- Static weights need manual adjustment
- Good for heterogeneous server pools

**Least Connections**
```
Server A: 5 active connections
Server B: 3 active connections → Route here
Server C: 8 active connections
```
- Routes to server with fewest active connections
- Better for long-lived connections
- Considers current load dynamically

**Least Response Time**
```
Server A: Avg response time 50ms
Server B: Avg response time 30ms → Route here
Server C: Avg response time 45ms
```
- Combines response time and active connections
- Optimal for user experience
- Requires continuous monitoring

**IP Hash**
```
hash(client_ip) % num_servers = server_index
hash(192.168.1.100) % 3 = 2 → Server C
```
- Same client always goes to same server
- Useful for session persistence
- Can cause uneven distribution

**Consistent Hashing**
```
Hash ring positions:
Server A: 100, Server B: 200, Server C: 300
Client hash: 150 → Routes to Server B
```
- Minimal disruption when adding/removing servers
- Used in distributed caches and databases
- More complex implementation

### Health Checks and Failure Handling

**Active Health Checks**
```python
# Example health check endpoint
@app.route('/health')
def health_check():
    checks = {
        'database': check_database_connection(),
        'cache': check_redis_connection(),
        'disk_space': check_disk_space() > 20,  # GB
        'memory': check_memory_usage() < 80     # Percent
    }
    
    if all(checks.values()):
        return {'status': 'healthy', 'checks': checks}, 200
    else:
        return {'status': 'unhealthy', 'checks': checks}, 503
```

**Health Check Parameters:**
- **Interval**: How often to check (e.g., every 5 seconds)
- **Timeout**: Maximum wait time (e.g., 3 seconds)
- **Unhealthy Threshold**: Failures before marking unhealthy (e.g., 2)
- **Healthy Threshold**: Successes before marking healthy (e.g., 3)

**Passive Health Checks**
- Monitor real traffic for errors
- Mark server unhealthy after X consecutive 5xx errors
- Faster detection than active checks
- No additional load on servers

### Load Balancer High Availability

**Active-Passive Setup**
```
        VIP (Virtual IP)
         ↙          ↘
   Active LB    Passive LB
    (Primary)    (Standby)
         ↓
    Server Pool
```
- Passive takes over if active fails
- Simple configuration
- Wastes standby resources

**Active-Active Setup**
```
        DNS Round Robin
         ↙          ↘
      LB-1        LB-2
    (Active)    (Active)
         ↘        ↙
       Server Pool
```
- Both handle traffic simultaneously
- Better resource utilization
- More complex state synchronization

### Popular Load Balancers

**Hardware Load Balancers**
- F5 BIG-IP
- Citrix ADC (NetScaler)
- Extremely high performance
- Expensive ($10k-$100k+)

**Software Load Balancers**
- **HAProxy**: High-performance, open-source
- **NGINX**: Web server with LB capabilities
- **Envoy**: Modern, cloud-native proxy
- **Traefik**: Docker/Kubernetes native

**Cloud Load Balancers**
- **AWS**: ALB (L7), NLB (L4), CLB (Classic)
- **Google Cloud**: Cloud Load Balancing
- **Azure**: Azure Load Balancer, Application Gateway

## 2. Reverse Proxies and API Gateways

### Reverse Proxy
A server that forwards client requests to backend servers and returns the response to clients.

**Key Functions:**
- **Security**: Hide backend server details
- **SSL Termination**: Handle encryption/decryption
- **Compression**: Reduce bandwidth usage
- **Caching**: Store frequently requested content
- **Request Routing**: Direct traffic based on rules

**NGINX Configuration Example:**
```nginx
upstream backend {
    server backend1.example.com:8080 weight=3;
    server backend2.example.com:8080 weight=2;
    server backend3.example.com:8080 backup;
}

server {
    listen 443 ssl;
    server_name api.example.com;
    
    # SSL Configuration
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # Compression
    gzip on;
    gzip_types text/plain application/json;
    
    # Caching
    location /static/ {
        proxy_cache my_cache;
        proxy_cache_valid 200 1h;
        proxy_pass http://backend;
    }
    
    # Rate Limiting
    limit_req zone=api burst=20 nodelay;
    
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### API Gateway
An API gateway is a reverse proxy with additional features specifically for APIs.

**Core Features:**
1. **Request Routing**: Route to different services based on path
2. **Authentication/Authorization**: Verify API keys, JWT tokens
3. **Rate Limiting**: Prevent abuse and ensure fair usage
4. **Request/Response Transformation**: Modify headers, body
5. **API Versioning**: Manage multiple API versions
6. **Analytics**: Track usage, performance metrics
7. **Circuit Breaking**: Prevent cascade failures

**API Gateway Architecture:**
```
Mobile App → 
Web App    →  API Gateway → Auth Service
IoT Device →      ↓       → User Service
                  ↓       → Order Service
              Analytics   → Payment Service
```

**Popular API Gateways:**
- **Kong**: Open-source, plugin ecosystem
- **AWS API Gateway**: Managed service, serverless integration
- **Apigee**: Google's enterprise solution
- **Zuul**: Netflix's gateway, Java-based
- **Tyk**: Open-source, Go-based

**Example API Gateway Configuration (Kong):**
```yaml
services:
  - name: user-service
    url: http://users.internal:8000
    routes:
      - paths: ["/api/users"]
        methods: ["GET", "POST"]
        
plugins:
  - name: rate-limiting
    config:
      second: 5
      minute: 100
      policy: local
      
  - name: jwt
    config:
      secret_is_base64: false
      claims_to_verify: ["exp", "nbf"]
      
  - name: request-transformer
    config:
      add:
        headers:
          - X-Gateway-Version:v1.0
```

## 3. Content Delivery Networks (CDNs)

### How CDNs Work
CDNs cache content at edge locations closer to users, reducing latency and origin server load.

**CDN Request Flow:**
1. User requests `www.example.com/image.jpg`
2. DNS resolves to nearest CDN edge server
3. Edge server checks cache
4. If hit: Serve from cache
5. If miss: Fetch from origin, cache, and serve

### CDN Architecture

**Points of Presence (PoPs):**
```
Origin Server (US-East)
         ↓
    CDN Network
    /    |    \
PoP-NYC PoP-LON PoP-TOK
   ↓      ↓      ↓
US Users UK Users Asian Users
```

### Types of CDN Content

**Static Content:**
- Images, videos, CSS, JavaScript
- Long cache TTL (days to months)
- Invalidation rarely needed

**Dynamic Content:**
- API responses, personalized content
- Short cache TTL (seconds to minutes)
- Cache key includes user context

**Streaming Content:**
- Live video/audio streams
- Adaptive bitrate streaming
- Segment-based caching

### CDN Caching Strategies

**Cache Headers:**
```http
# Client-side caching
Cache-Control: public, max-age=31536000

# CDN caching
Cache-Control: s-maxage=86400

# No caching
Cache-Control: no-cache, no-store, must-revalidate

# Conditional caching
ETag: "33a64df551425fcc55e4d42a148795d9"
If-None-Match: "33a64df551425fcc55e4d42a148795d9"
```

**Cache Invalidation:**
```python
# Programmatic cache purge
import boto3

cloudfront = boto3.client('cloudfront')

response = cloudfront.create_invalidation(
    DistributionId='E1QXXX',
    InvalidationBatch={
        'Paths': {
            'Quantity': 2,
            'Items': ['/images/*', '/api/v1/*']
        },
        'CallerReference': str(time.time())
    }
)
```

### CDN Benefits and Considerations

**Benefits:**
- Reduced latency (serve from edge)
- Decreased origin load
- DDoS protection
- Improved availability
- Bandwidth cost savings

**Considerations:**
- Cache invalidation complexity
- Costs for traffic and storage
- HTTPS certificate management
- Debugging cached responses
- Compliance (data residency)

**Popular CDN Providers:**
- **CloudFlare**: Free tier, DDoS protection
- **AWS CloudFront**: AWS integration
- **Akamai**: Enterprise, largest network
- **Fastly**: Real-time purging, edge compute

## 4. Web Servers vs Application Servers

### Web Servers
Handle HTTP requests, serve static content, and proxy to application servers.

**Primary Functions:**
- Serve static files (HTML, CSS, JS, images)
- Handle HTTP/HTTPS connections
- URL rewriting and redirects
- Access logging
- Basic authentication
- Virtual hosting

**Popular Web Servers:**
- **Apache**: Mature, extensive modules
- **NGINX**: High performance, event-driven
- **IIS**: Windows-native
- **Caddy**: Automatic HTTPS

**NGINX Static File Serving:**
```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;
    
    # Enable sendfile for performance
    sendfile on;
    tcp_nopush on;
    
    # Caching headers
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 365d;
        add_header Cache-Control "public, immutable";
    }
    
    # Gzip compression
    gzip on;
    gzip_types text/css application/javascript;
}
```

### Application Servers
Execute business logic and generate dynamic content.

**Primary Functions:**
- Execute application code
- Manage application lifecycle
- Database connections pooling
- Session management
- Background job processing
- WebSocket handling

**Technology-Specific Servers:**

**Java:**
- **Tomcat**: Lightweight, servlet container
- **Jetty**: Embedded-friendly
- **WildFly**: Full Java EE support

**Python:**
- **Gunicorn**: WSGI server
- **uWSGI**: Feature-rich
- **Uvicorn**: ASGI, async support

**Node.js:**
- **Express**: Minimal framework
- **Koa**: Modern, async/await
- **Fastify**: High performance

**Ruby:**
- **Puma**: Multi-threaded
- **Unicorn**: Process-based
- **Passenger**: Integrated with web servers

### Web Server + Application Server Architecture

**Typical Setup:**
```
[Client] → [NGINX] → [Gunicorn] → [Django App]
           (Web)     (App Server)  (Framework)
             ↓
        Static Files
```

**NGINX as Reverse Proxy to App Server:**
```nginx
upstream app_server {
    server 127.0.0.1:8000 fail_timeout=0;
}

server {
    listen 80;
    server_name www.example.com;
    
    # Serve static files directly
    location /static/ {
        alias /var/www/app/static/;
    }
    
    # Proxy dynamic requests to app server
    location / {
        proxy_pass http://app_server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_redirect off;
        proxy_buffering off;
    }
}
```

## 5. Service Discovery and Service Mesh

### Service Discovery
Mechanism for services to find and communicate with each other dynamically.

### Service Discovery Patterns

**Client-Side Discovery:**
```
Service A → Query Registry → Get Service B locations
         → Load balance → Call Service B directly
```

**Example with Consul:**
```python
import consul

c = consul.Consul()

# Register service
c.agent.service.register(
    name='payment-service',
    service_id='payment-1',
    address='10.0.0.5',
    port=8080,
    check=consul.Check.http('http://10.0.0.5:8080/health',
                            interval='10s')
)

# Discover service
_, services = c.health.service('payment-service', passing=True)
for service in services:
    address = service['Service']['Address']
    port = service['Service']['Port']
    # Make request to service
```

**Server-Side Discovery:**
```
Service A → Load Balancer → Service Registry
                         → Route to Service B
```

### Service Registry Options

**Consul:**
- Multi-datacenter support
- Health checking
- Key-value store
- DNS interface

**Eureka (Netflix):**
- Self-preservation mode
- Client-side caching
- REST API

**etcd:**
- Distributed key-value store
- Strong consistency (Raft)
- Watch functionality

**Kubernetes Service Discovery:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
---
# Other services can access via:
# payment-service.default.svc.cluster.local
```

### Service Mesh
Infrastructure layer handling service-to-service communication.

**Components:**
- **Data Plane**: Proxies handling actual traffic (Envoy)
- **Control Plane**: Management and configuration (Istio, Linkerd)

**Service Mesh Features:**
1. **Traffic Management**: Load balancing, retries, timeouts
2. **Security**: mTLS, authorization policies
3. **Observability**: Metrics, tracing, logs
4. **Resilience**: Circuit breaking, fault injection

**Istio Configuration Example:**
```yaml
# Virtual Service for canary deployment
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: payment-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90
    - destination:
        host: payment-service
        subset: v2
      weight: 10  # 10% canary traffic
```

## 6. Rate Limiting and Throttling

### Why Rate Limiting?
- Prevent API abuse
- Ensure fair resource usage
- Protect against DDoS attacks
- Control costs
- Maintain service quality

### Rate Limiting Algorithms

**Token Bucket**
```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate
        self.last_refill = time.time()
    
    def consume(self, tokens=1):
        self.refill()
        
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False
    
    def refill(self):
        now = time.time()
        tokens_to_add = (now - self.last_refill) * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
```

**Characteristics:**
- Allows burst traffic
- Smooth rate limiting
- Memory efficient

**Leaky Bucket**
```python
class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        self.capacity = capacity
        self.queue = []
        self.leak_rate = leak_rate
        self.last_leak = time.time()
    
    def add_request(self, request):
        self.leak()
        
        if len(self.queue) < self.capacity:
            self.queue.append(request)
            return True
        return False
    
    def leak(self):
        now = time.time()
        leaked = int((now - self.last_leak) * self.leak_rate)
        self.queue = self.queue[leaked:]
        self.last_leak = now
```

**Characteristics:**
- Smooths out bursts
- Constant output rate
- Good for network traffic shaping

**Fixed Window Counter**
```python
class FixedWindowCounter:
    def __init__(self, window_size, limit):
        self.window_size = window_size  # seconds
        self.limit = limit
        self.windows = {}
    
    def is_allowed(self, key):
        now = time.time()
        window = int(now / self.window_size)
        
        if window not in self.windows:
            self.windows[window] = {}
        
        count = self.windows[window].get(key, 0)
        
        if count < self.limit:
            self.windows[window][key] = count + 1
            return True
        return False
```

**Issue:** Race condition at window boundaries

**Sliding Window Log**
```python
class SlidingWindowLog:
    def __init__(self, window_size, limit):
        self.window_size = window_size
        self.limit = limit
        self.requests = {}
    
    def is_allowed(self, key):
        now = time.time()
        window_start = now - self.window_size
        
        if key not in self.requests:
            self.requests[key] = []
        
        # Remove old entries
        self.requests[key] = [
            t for t in self.requests[key] 
            if t > window_start
        ]
        
        if len(self.requests[key]) < self.limit:
            self.requests[key].append(now)
            return True
        return False
```

**Characteristics:**
- Most accurate
- Higher memory usage
- No boundary issues

### Distributed Rate Limiting

**Centralized Counter (Redis):**
```python
import redis

class DistributedRateLimiter:
    def __init__(self, redis_client, limit, window):
        self.redis = redis_client
        self.limit = limit
        self.window = window
    
    def is_allowed(self, key):
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - self.window
        
        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        # Add current request
        pipe.zadd(key, {str(now): now})
        # Count requests in window
        pipe.zcard(key)
        # Set expiry
        pipe.expire(key, self.window)
        
        results = pipe.execute()
        
        return results[2] <= self.limit
```

**Distributed Token Bucket:**
```lua
-- Lua script for Redis (atomic operation)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens
local elapsed = now - last_refill
local tokens_to_add = elapsed * refill_rate
tokens = math.min(capacity, tokens + tokens_to_add)

if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, capacity / refill_rate)
    return 1
else
    return 0
end
```

### Rate Limiting Strategies

**By User:**
```python
rate_limit_key = f"user:{user_id}"
```

**By IP Address:**
```python
rate_limit_key = f"ip:{request.remote_addr}"
```

**By API Endpoint:**
```python
rate_limit_key = f"endpoint:{request.path}:{user_id}"
```

**Tiered Limits:**
```python
limits = {
    'free': {'requests': 100, 'window': 3600},
    'basic': {'requests': 1000, 'window': 3600},
    'premium': {'requests': 10000, 'window': 3600},
}
```

## 7. Circuit Breakers and Bulkheads

### Circuit Breaker Pattern
Prevents cascading failures by stopping calls to failing services.

**States:**
1. **Closed**: Normal operation, requests pass through
2. **Open**: Failures exceeded threshold, requests fail immediately
3. **Half-Open**: Test if service recovered

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'
    
    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if self._should_attempt_reset():
                self.state = 'HALF_OPEN'
            else:
                raise Exception('Circuit breaker is OPEN')
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
    
    def _should_attempt_reset(self):
        return (time.time() - self.last_failure_time) >= self.recovery_timeout
```

**Configuration Considerations:**
- **Failure Threshold**: Too low causes premature opening
- **Recovery Timeout**: Balance between recovery time and retry frequency
- **Half-Open Tests**: Limited requests to verify recovery

### Bulkhead Pattern
Isolate resources to prevent total system failure.

**Thread Pool Bulkheads:**
```java
// Java example with Hystrix
@HystrixCommand(
    threadPoolKey = "paymentServicePool",
    threadPoolProperties = {
        @HystrixProperty(name = "coreSize", value = "10"),
        @HystrixProperty(name = "maxQueueSize", value = "100")
    }
)
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentService.process(request);
}
```

**Connection Pool Bulkheads:**
```python
# Database connection pools
import psycopg2.pool

# Separate pools for different operations
read_pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=1,
    maxconn=20,
    host="read-replica.db.com"
)

write_pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=1,
    maxconn=10,
    host="primary.db.com"
)

analytics_pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=1,
    maxconn=5,
    host="analytics.db.com"
)
```

**Resource Isolation Benefits:**
- Failure containment
- Predictable performance
- Easier capacity planning
- Independent scaling

## 8. Distributed Tracing and Monitoring

### Distributed Tracing
Track requests across multiple services to identify bottlenecks and failures.

**Key Concepts:**
- **Trace**: Complete request path
- **Span**: Single operation within a trace
- **Context Propagation**: Passing trace information between services

**OpenTelemetry Example:**
```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Setup tracing
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Create Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)

# Add span processor
span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Use in application
@app.route('/api/order')
def create_order():
    with tracer.start_as_current_span("create_order") as span:
        # Validate request
        with tracer.start_as_current_span("validate_request"):
            validate_order_request(request)
        
        # Check inventory
        with tracer.start_as_current_span("check_inventory") as inv_span:
            inv_span.set_attribute("item_count", len(items))
            inventory_available = check_inventory(items)
        
        # Process payment
        with tracer.start_as_current_span("process_payment"):
            payment_result = process_payment(payment_info)
        
        span.set_attribute("order_id", order_id)
        return {"order_id": order_id}
```

**Trace Visualization:**
```
[Create Order - 250ms]
  ├─[Validate Request - 10ms]
  ├─[Check Inventory - 50ms]
  │  └─[Database Query - 35ms]
  └─[Process Payment - 180ms]
     ├─[Validate Card - 20ms]
     └─[Charge Card - 150ms]
```

### Monitoring Strategies

**The Four Golden Signals (Google SRE):**

1. **Latency**
```python
# Prometheus metrics
from prometheus_client import Histogram

request_latency = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint']
)

@request_latency.time()
def handle_request():
    # Process request
    pass
```

2. **Traffic**
```python
from prometheus_client import Counter

request_count = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

@app.after_request
def track_request(response):
    request_count.labels(
        method=request.method,
        endpoint=request.endpoint,
        status=response.status_code
    ).inc()
    return response
```

3. **Errors**
```python
error_count = Counter(
    'http_errors_total',
    'Total HTTP errors',
    ['method', 'endpoint', 'error_type']
)

@app.errorhandler(Exception)
def handle_error(error):
    error_count.labels(
        method=request.method,
        endpoint=request.endpoint,
        error_type=type(error).__name__
    ).inc()
    return {"error": str(error)}, 500
```

4. **Saturation**
```python
from prometheus_client import Gauge

connection_pool_usage = Gauge(
    'db_connection_pool_usage',
    'Database connection pool utilization'
)

def update_pool_metrics():
    connection_pool_usage.set(
        pool.size() / pool.max_size()
    )
```

### Logging Best Practices

**Structured Logging:**
```python
import structlog

logger = structlog.get_logger()

logger.info(
    "order_processed",
    order_id=order_id,
    user_id=user_id,
    amount=total_amount,
    items_count=len(items),
    processing_time=processing_time,
    payment_method=payment_method
)
```

**Log Aggregation Pipeline:**
```
Application → Fluentd/Logstash → Elasticsearch → Kibana
           ↓
     Filebeat
           ↓
        S3 (Archive)
```

**Correlation IDs:**
```python
import uuid
from flask import g

@app.before_request
def before_request():
    g.correlation_id = request.headers.get(
        'X-Correlation-ID',
        str(uuid.uuid4())
    )
    
    logger.bind(correlation_id=g.correlation_id)

@app.after_request
def after_request(response):
    response.headers['X-Correlation-ID'] = g.correlation_id
    return response
```

### Alerting Strategy

**Alert Design Principles:**
1. Alert on symptoms, not causes
2. Page only for user-facing issues
3. Include playbook links
4. Avoid alert fatigue

**Example Alert Configuration (Prometheus):**
```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_errors_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
          team: api
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }}% for {{ $labels.endpoint }}"
          playbook: "https://wiki.internal/playbooks/high-error-rate"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency is high"
          description: "P99 latency is {{ $value }}s"
```

## 9. Practical Example: Building a Scalable API

Let's combine these components to design a scalable e-commerce API.

### Architecture Overview
```
[Mobile/Web Clients]
         ↓
    [CloudFlare CDN]
         ↓
    [AWS ALB (L7)]
         ↓
    [API Gateway]
    (Kong/AWS API GW)
         ↓
   [Service Mesh]
   (Istio/Envoy)
    ↙    ↓    ↘
[User]  [Order] [Payment]
[Service][Service][Service]
   ↓       ↓        ↓
[Cache] [Database] [Queue]
(Redis) (PostgreSQL)(SQS)
```

### Component Configuration

**Load Balancer (AWS ALB):**
- Health checks every 5 seconds
- Least connections algorithm
- SSL termination
- WAF rules for security

**API Gateway (Kong):**
- JWT authentication
- Rate limiting: 1000 req/hour for free, 10000 for premium
- Request/response transformation
- API versioning through headers

**Service Mesh (Istio):**
- mTLS between services
- Circuit breakers (5 failures = 30s timeout)
- Retry logic (3 attempts with exponential backoff)
- Distributed tracing with Jaeger

**Monitoring Stack:**
- Prometheus for metrics
- Grafana for visualization
- ELK stack for logs
- PagerDuty for alerting

### Handling 100K Requests/Second

**Capacity Planning:**
```
100K RPS
Each server handles 1K RPS
Need 100 servers + 20% buffer = 120 servers
3 Availability Zones = 40 servers per AZ
```

**Caching Strategy:**
- CDN for static assets (80% cache hit rate)
- Redis for session data (sub-ms latency)
- Application-level caching for database queries

**Database Optimization:**
- Read replicas for queries
- Connection pooling (100 connections per server)
- Query optimization and indexing
- Partitioning for large tables

## Summary and Key Takeaways

1. **Load Balancers are Critical**: Choose between L4/L7 based on needs
2. **API Gateways Centralize Concerns**: Authentication, rate limiting, versioning
3. **CDNs Reduce Load**: Cache at the edge for global performance
4. **Service Discovery Enables Scale**: Dynamic service registration and discovery
5. **Rate Limiting Protects Services**: Choose algorithm based on use case
6. **Circuit Breakers Prevent Cascades**: Fail fast when dependencies are down
7. **Monitoring is Non-Negotiable**: You can't fix what you can't see
8. **Distributed Tracing Shows the Full Picture**: Essential for microservices

## Next Steps
- Implement a rate limiter using Redis
- Set up distributed tracing in a sample application
- Configure a load balancer with health checks
- Study Part 3: Distributed System Patterns for architectural patterns