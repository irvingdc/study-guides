# System Design Study Guide - Part 1: Fundamentals and Core Concepts

## 1. Basic Building Blocks of Distributed Systems

### What is a Distributed System?
A distributed system is a collection of independent computers that appears to its users as a single coherent system. These computers communicate through a network to coordinate their actions and share resources.

**Key Characteristics:**
- **Concurrency**: Multiple computations happening simultaneously
- **Lack of global clock**: No single source of time across all nodes
- **Independent failures**: Parts can fail while others continue operating
- **Message passing**: Communication through network messages

**Why Distributed Systems?**
1. **Scalability**: Handle growing amounts of work by adding resources
2. **Reliability**: Continue operating despite failures
3. **Performance**: Parallel processing and geographic distribution
4. **Cost-effectiveness**: Use commodity hardware instead of supercomputers
5. **Geographic distribution**: Serve users globally with low latency

### Components of a Distributed System

**Nodes (Servers)**
- Physical or virtual machines that run your application
- Can be stateless (no persistent data) or stateful (maintains data)
- Examples: Web servers, application servers, database servers

**Network**
- The communication medium connecting nodes
- Can be LAN (Local Area Network) or WAN (Wide Area Network)
- Characterized by bandwidth, latency, and reliability

**Storage**
- Persistent data storage systems
- Can be centralized (single database) or distributed (sharded databases)
- Includes file systems, databases, object stores

**Protocols**
- Rules for communication between components
- Examples: HTTP/HTTPS, TCP/IP, WebSocket, gRPC

## 2. Client-Server Architecture Evolution

### Single Server Architecture
The simplest architecture where everything runs on one machine.

**Components:**
- Web server (nginx/Apache)
- Application server (Node.js/Django/Rails)
- Database (PostgreSQL/MySQL)
- File storage (local filesystem)

**Pros:**
- Simple to develop and deploy
- No network latency between components
- Easy debugging and monitoring
- ACID transactions are straightforward

**Cons:**
- Single point of failure
- Limited scalability
- Resource contention between components
- Difficult to update without downtime

**When to use:** MVPs, prototypes, or applications with <1000 daily active users

### Two-Tier Architecture
Separation of presentation and data layers.

**Structure:**
```
[Client Tier] <---> [Data Tier]
   (UI)              (Database)
```

**Example:** A desktop application directly connecting to a database

**Limitations:**
- Fat client with business logic
- Difficult to update client software
- Database connection scaling issues
- Security concerns with direct database access

### Three-Tier Architecture
The most common architecture for web applications.

**Structure:**
```
[Presentation Tier] <---> [Logic Tier] <---> [Data Tier]
    (Web UI)            (Application)       (Database)
```

**Benefits:**
- Clear separation of concerns
- Independent scaling of tiers
- Easier maintenance and updates
- Better security through abstraction

**Real-world example:**
- **Presentation**: React SPA served from CDN
- **Logic**: Node.js API servers behind load balancer
- **Data**: PostgreSQL with read replicas

### N-Tier/Microservices Architecture
Further decomposition into specialized services.

**Structure:**
```
[Client] <---> [API Gateway] <---> [Service A]
                    |               [Service B]
                    |               [Service C]
                    v                   |
              [Service Mesh]     [Databases]
```

**When to evolve to N-Tier:**
- Team size > 20 developers
- Need for independent deployment cycles
- Different scaling requirements per component
- Technology diversity requirements

## 3. Network Fundamentals

### OSI Model (Simplified for System Design)
Understanding layers helps in debugging and designing systems.

**Layer 7 - Application (HTTP/HTTPS)**
- What users interact with
- HTTP methods: GET, POST, PUT, DELETE, PATCH
- Status codes: 2xx (success), 3xx (redirect), 4xx (client error), 5xx (server error)
- Headers: Content-Type, Authorization, Cache-Control

**Layer 4 - Transport (TCP/UDP)**
- **TCP**: Reliable, ordered, connection-oriented
  - Use for: HTTP, database connections, file transfers
  - Three-way handshake establishes connection
  - Guarantees delivery and order
  
- **UDP**: Fast, connectionless, no delivery guarantee
  - Use for: Video streaming, gaming, DNS queries
  - Lower latency than TCP
  - No connection overhead

**Layer 3 - Network (IP)**
- IP addressing and routing
- IPv4 (32-bit) vs IPv6 (128-bit)
- Public vs private IP addresses
- NAT (Network Address Translation)

### HTTP vs HTTPS
**HTTP:**
- Plain text communication
- Default port 80
- Vulnerable to man-in-the-middle attacks

**HTTPS:**
- Encrypted using TLS/SSL
- Default port 443
- Certificate-based authentication
- ~10-20ms additional handshake latency

**TLS Handshake Process:**
1. Client hello (supported cipher suites)
2. Server hello (chosen cipher, certificate)
3. Client verifies certificate
4. Key exchange
5. Encrypted communication begins

### HTTP/2 vs HTTP/1.1
**HTTP/1.1 Limitations:**
- Head-of-line blocking
- One request per TCP connection
- Text-based protocol
- No request prioritization

**HTTP/2 Improvements:**
- Multiplexing (multiple streams per connection)
- Server push capabilities
- Header compression (HPACK)
- Binary protocol
- Request prioritization

**When HTTP/2 matters:**
- High-latency connections (mobile)
- Pages with many resources
- Real-time applications

### WebSockets
Full-duplex communication over single TCP connection.

**Use cases:**
- Real-time chat applications
- Live sports updates
- Collaborative editing
- Financial trading platforms
- Multiplayer gaming

**WebSocket Handshake:**
```
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: [base64 key]
```

**Scaling WebSockets:**
- Sticky sessions for load balancing
- Redis Pub/Sub for multi-server communication
- Connection limits per server (~10k-65k)

## 4. Latency, Throughput, and Bandwidth

### Latency
Time for data to travel from source to destination.

**Types of Latency:**
- **Network Latency**: Physical distance and routing
- **Processing Latency**: Time to process request
- **Queuing Latency**: Time waiting in queue
- **Disk I/O Latency**: Reading/writing to disk

**Typical Latencies (Orders of Magnitude):**
```
L1 Cache Reference                  0.5 ns
L2 Cache Reference                  7 ns
Main Memory Reference               100 ns
SSD Random Read                     150 μs (microseconds)
HDD Seek                            10 ms (milliseconds)
Network Round Trip (same datacenter) 0.5 ms
Network Round Trip (across US)       40 ms
Network Round Trip (across Pacific)  150 ms
```

**Reducing Latency:**
1. **Caching**: Store frequently accessed data in memory
2. **CDNs**: Serve static content from edge locations
3. **Connection Pooling**: Reuse database/HTTP connections
4. **Parallel Processing**: Execute independent operations simultaneously
5. **Data Locality**: Keep data close to computation

### Throughput
Amount of data processed per unit time.

**Measurement Units:**
- Requests per second (RPS)
- Queries per second (QPS)
- Transactions per second (TPS)
- Megabytes per second (MB/s)

**Throughput vs Latency:**
- High throughput doesn't guarantee low latency
- Example: Batch processing has high throughput but high latency
- Real-time systems need both low latency and sufficient throughput

**Improving Throughput:**
1. **Vertical Scaling**: More powerful hardware
2. **Horizontal Scaling**: More servers
3. **Load Balancing**: Distribute work evenly
4. **Async Processing**: Don't block on I/O
5. **Batch Operations**: Process multiple items together

### Bandwidth
Maximum rate of data transfer across a network.

**Bandwidth Planning:**
```
Daily Active Users: 1 million
Average request size: 10 KB
Requests per user per day: 100
Peak factor: 3x average

Average bandwidth = (1M * 100 * 10KB) / (24 * 3600s) = 115 MB/s
Peak bandwidth = 115 * 3 = 345 MB/s
```

**Bandwidth Optimization:**
- Compression (gzip, Brotli)
- Image optimization (WebP, lazy loading)
- Minification (CSS, JavaScript)
- HTTP/2 multiplexing
- Caching strategies

## 5. CAP Theorem and Its Practical Implications

### Understanding CAP
You can only guarantee 2 out of 3:

**Consistency (C)**
- All nodes see the same data simultaneously
- After a write, all reads return the new value
- Example: Bank account balance

**Availability (A)**
- System remains operational
- Every request receives a response
- Example: Social media feed

**Partition Tolerance (P)**
- System continues despite network failures
- Required for distributed systems
- Cannot be sacrificed in practice

### CAP Combinations in Practice

**CP Systems (Consistency + Partition Tolerance)**
- Choose consistency over availability during partitions
- Examples: MongoDB, HBase, Redis
- Use case: Financial transactions, inventory systems

**AP Systems (Availability + Partition Tolerance)**
- Choose availability over consistency during partitions
- Examples: Cassandra, DynamoDB, CouchDB
- Use case: Social media, recommendation systems

**CA Systems (Consistency + Availability)**
- Cannot handle network partitions
- Only possible in single-node systems
- Examples: PostgreSQL (single node), MySQL (single node)

### Real-World CAP Trade-offs

**Banking System (CP)**
```
Scenario: Network partition between datacenters
Choice: Reject transactions rather than risk inconsistency
Result: Some ATMs go offline, but no incorrect balances
```

**Social Media Feed (AP)**
```
Scenario: Network partition between regions
Choice: Show potentially stale feed rather than error
Result: Users might see old posts, but service stays up
```

**E-commerce Inventory (Tunable)**
```
Normal operations: Strong consistency
Black Friday: Eventual consistency with overselling insurance
Result: Better user experience with business risk management
```

## 6. ACID vs BASE Properties

### ACID Properties (Traditional Databases)

**Atomicity**
- All or nothing execution
- Example: Transfer $100 from A to B
  - Debit from A and credit to B must both succeed or both fail
  - Implementation: Transaction logs, rollback mechanisms

**Consistency**
- Database remains in valid state
- Constraints and rules are enforced
- Example: Account balance cannot be negative
  - Implementation: Check constraints, foreign keys, triggers

**Isolation**
- Concurrent transactions don't interfere
- Levels of isolation:
  1. **Read Uncommitted**: Dirty reads possible
  2. **Read Committed**: No dirty reads, but non-repeatable reads
  3. **Repeatable Read**: No dirty or non-repeatable reads
  4. **Serializable**: Full isolation, lowest performance

**Durability**
- Committed transactions survive failures
- Data persisted to disk
- Implementation: Write-ahead logging (WAL), replication

### BASE Properties (NoSQL/Distributed Systems)

**Basically Available**
- System is available most of the time
- May return stale or approximate data
- Example: Show cached product prices during database issues

**Soft State**
- Data may change without input
- System determines data changes internally
- Example: Session data with TTL, cache expiration

**Eventual Consistency**
- System will become consistent over time
- No guarantee when consistency achieved
- Example: DNS propagation, CDN cache updates

### When to Choose ACID vs BASE

**Choose ACID when:**
- Financial transactions
- Inventory management
- User authentication
- Order processing
- Medical records

**Choose BASE when:**
- Social media feeds
- Product recommendations
- Analytics data
- User activity tracking
- Content delivery

## 7. Consistency Models

### Strong Consistency
All reads return the most recent write.

**Implementation:**
- Synchronous replication
- Distributed locks
- Consensus protocols (Raft, Paxos)

**Cost:**
- Higher latency (wait for all replicas)
- Lower availability (requires majority)
- More complex implementation

**Example:**
```
User updates profile photo
→ Write to primary database
→ Synchronously replicate to all replicas
→ Acknowledge to user
→ All subsequent reads show new photo
```

### Eventual Consistency
System will become consistent, given enough time.

**Implementation:**
- Asynchronous replication
- Conflict resolution strategies
- Vector clocks for ordering

**Benefits:**
- Lower latency
- Higher availability
- Simpler implementation

**Example:**
```
User likes a post
→ Write to nearest datacenter
→ Acknowledge to user immediately
→ Asynchronously replicate to other datacenters
→ Like count may differ temporarily across regions
```

### Weak Consistency
No guarantee when all nodes will be consistent.

**Use cases:**
- Live video streaming
- Multiplayer gaming
- Real-time collaboration

**Example:**
```
Video conference participant speaks
→ Send to nearest server
→ Best-effort delivery to other participants
→ Some may experience delay or packet loss
→ No attempt to synchronize all views
```

### Read Your Writes Consistency
User always sees their own writes.

**Implementation:**
- Sticky sessions
- Read from primary after write
- Session-based routing

**Example:**
```
User posts comment
→ Write to primary
→ Route user's reads to primary for X seconds
→ User sees their comment immediately
→ Others may see it after replication delay
```

### Monotonic Read Consistency
Once a user sees a value, they won't see an older value.

**Implementation:**
- Track read timestamps per session
- Route to replicas with sufficient replication lag

**Example:**
```
User refreshes feed
→ See post with 100 likes
→ Refresh again
→ Must see ≥100 likes (never fewer)
```

## 8. System Reliability Metrics

### SLA (Service Level Agreement)
Legal contract defining service expectations.

**Components:**
- Uptime percentage (99.9%, 99.99%)
- Response time thresholds
- Support response times
- Penalties for violations

**Uptime Calculations:**
```
99% (two nines)    = 3.65 days downtime/year
99.9% (three nines) = 8.76 hours downtime/year
99.99% (four nines) = 52.56 minutes downtime/year
99.999% (five nines) = 5.26 minutes downtime/year
```

### SLO (Service Level Objective)
Internal goals, stricter than SLA.

**Examples:**
- 99.95% uptime (internal) vs 99.9% SLA (external)
- P50 latency < 100ms
- P99 latency < 1000ms
- Error rate < 0.1%

**Error Budget:**
```
Monthly error budget = (1 - SLO) * time
99.9% SLO = 0.1% * 30 days = 43.2 minutes/month
```

### SLI (Service Level Indicator)
Metrics that measure SLO compliance.

**Common SLIs:**
1. **Availability**: Successful requests / Total requests
2. **Latency**: Response time percentiles
3. **Throughput**: Requests per second
4. **Error Rate**: Failed requests / Total requests
5. **Durability**: Data not lost / Total data

**Measuring SLIs:**
```python
# Availability SLI
availability = successful_requests / total_requests * 100

# Latency SLI (P99)
latencies.sort()
p99_index = int(len(latencies) * 0.99)
p99_latency = latencies[p99_index]

# Error Rate SLI
error_rate = (4xx_errors + 5xx_errors) / total_requests * 100
```

### Improving Reliability

**Redundancy:**
- Multiple servers (N+1 redundancy)
- Multi-region deployment
- Database replication
- Backup systems

**Fault Tolerance:**
- Graceful degradation
- Circuit breakers
- Retry with exponential backoff
- Timeout configurations

**Monitoring and Alerting:**
- Real-time metrics dashboards
- Anomaly detection
- On-call rotations
- Incident response procedures

**Testing:**
- Unit tests (>80% coverage)
- Integration tests
- Load testing
- Chaos engineering

## 9. Practical Application: Designing a Simple System

Let's apply these fundamentals to design a URL shortener.

### Requirements
- Shorten long URLs to 7-character codes
- 100M URLs per month
- <100ms latency
- 99.9% availability

### Capacity Estimation
```
Write QPS = 100M / (30 * 24 * 3600) = 40 QPS
Read QPS = 40 * 10 (read:write ratio) = 400 QPS
Storage (5 years) = 100M * 12 * 5 * 500 bytes = 300 GB
Bandwidth = 400 QPS * 500 bytes = 200 KB/s
```

### Architecture Decisions

**CAP Choice**: AP (Availability + Partition Tolerance)
- URL redirects should always work
- Slight delay in global propagation acceptable

**Consistency Model**: Eventual consistency
- New URLs can take seconds to propagate
- Read-your-writes for URL creator

**Database**: NoSQL (DynamoDB)
- Simple key-value access pattern
- Predictable performance at scale
- Built-in replication

**Caching**: Redis
- Cache popular URLs (80/20 rule)
- Reduce database load
- Sub-millisecond latency

### System Components
```
[Client] → [CDN] → [Load Balancer]
                           ↓
                    [App Servers]
                      ↙        ↘
                [Cache]     [Database]
                (Redis)     (DynamoDB)
```

### Reliability Measures
- Multi-region deployment
- Database replication
- Cache cluster with replicas
- Health checks and auto-scaling
- Circuit breakers for dependencies

## Summary and Key Takeaways

1. **Start Simple**: Begin with monolithic architecture and evolve based on requirements
2. **Understand Trade-offs**: Every design decision has pros and cons
3. **Know Your Numbers**: Latency, throughput, and storage calculations matter
4. **CAP is Fundamental**: You must choose what to sacrifice during network partitions
5. **Consistency is Expensive**: Stronger consistency means higher latency and lower availability
6. **Measure Everything**: SLIs inform SLOs which fulfill SLAs
7. **Design for Failure**: Assume components will fail and plan accordingly
8. **Cache Strategically**: Not everything needs caching, but the right things do

## Next Steps
- Practice calculating capacity requirements for different scales
- Draw architecture diagrams for systems you use daily
- Identify CAP choices in real-world systems
- Study Part 2: Components and Building Blocks for deeper technical details