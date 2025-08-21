# System Design Study Guide - Part 3: Distributed System Patterns

## 1. Microservices Architecture Patterns

### What are Microservices?
Microservices are an architectural style where applications are structured as a collection of loosely coupled, independently deployable services. Each service is focused on a single business capability.

### Microservices vs Monolithic Architecture

**Monolithic Architecture:**
```
┌─────────────────────────────┐
│     Monolithic Application   │
├─────────────────────────────┤
│  ┌─────┐ ┌─────┐ ┌─────┐  │
│  │ UI  │ │Logic│ │ Data │  │
│  └─────┘ └─────┘ └─────┘  │
│  ┌─────────────────────┐   │
│  │    Shared Database   │   │
│  └─────────────────────┘   │
└─────────────────────────────┘
```

**Microservices Architecture:**
```
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ User │ │Order │ │Inven-│ │Pay-  │
│ Svc  │ │ Svc  │ │tory  │ │ment  │
├──────┤ ├──────┤ ├──────┤ ├──────┤
│ DB   │ │ DB   │ │ DB   │ │ DB   │
└──────┘ └──────┘ └──────┘ └──────┘
```

### When to Use Microservices

**Good Fit:**
- Large teams (>20 developers)
- Multiple business domains
- Different scaling requirements per component
- Need for technology diversity
- Frequent independent deployments
- Complex domain requiring bounded contexts

**Not a Good Fit:**
- Small teams (<5 developers)
- Simple CRUD applications
- Tight deadlines for MVP
- Limited operational expertise
- Unclear domain boundaries

### Microservices Design Principles

**1. Single Responsibility**
Each service handles one business capability.

```python
# Good: User service handles only user-related operations
class UserService:
    def create_user(self, user_data): pass
    def update_profile(self, user_id, profile): pass
    def authenticate(self, credentials): pass

# Bad: Mixed responsibilities
class MixedService:
    def create_user(self, user_data): pass
    def process_order(self, order): pass  # Should be in OrderService
    def send_email(self, email): pass     # Should be in NotificationService
```

**2. Autonomous Teams**
Each team owns their service end-to-end.

```yaml
# Team ownership example
services:
  user-service:
    team: identity-team
    tech-stack: Python/PostgreSQL
    on-call: identity-team-rotation
    
  payment-service:
    team: payments-team
    tech-stack: Java/Oracle
    on-call: payments-team-rotation
```

**3. Decentralized Data Management**
Each service manages its own data.

```sql
-- User Service Database
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP
);

-- Order Service Database
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID,  -- Reference to user, not foreign key
    total DECIMAL(10,2),
    created_at TIMESTAMP
);
```

**4. Design for Failure**
Assume dependencies will fail.

```python
import circuitbreaker

class PaymentServiceClient:
    @circuitbreaker.circuit(failure_threshold=5, recovery_timeout=30)
    def process_payment(self, payment_data):
        try:
            response = requests.post(
                "http://payment-service/process",
                json=payment_data,
                timeout=5
            )
            return response.json()
        except requests.RequestException as e:
            # Fallback logic
            return self.queue_payment_for_retry(payment_data)
```

### Service Boundaries and Domain-Driven Design

**Bounded Contexts:**
Define clear boundaries between services based on business domains.

```
E-commerce Platform Bounded Contexts:

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Identity  │     │   Catalog   │     │   Orders    │
│   Context   │     │   Context   │     │   Context   │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ User        │     │ Product     │     │ Order       │
│ Auth        │     │ Category    │     │ Cart        │
│ Profile     │     │ Inventory   │     │ Checkout    │
└─────────────┘     └─────────────┘     └─────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Shipping   │     │   Payment   │     │Notification │
│   Context   │     │   Context   │     │   Context   │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ Shipment    │     │ Transaction │     │ Email       │
│ Tracking    │     │ Refund      │     │ SMS         │
│ Carrier     │     │ Wallet      │     │ Push        │
└─────────────┘     └─────────────┘     └─────────────┘
```

### Data Management Patterns in Microservices

**Database per Service:**
```yaml
services:
  user-service:
    database: PostgreSQL
    connection: postgres://user-db:5432/users
    
  order-service:
    database: MongoDB
    connection: mongodb://order-db:27017/orders
    
  analytics-service:
    database: ClickHouse
    connection: clickhouse://analytics-db:9000/events
```

**Shared Database Anti-Pattern:**
Avoid multiple services accessing the same database directly.

```python
# Anti-pattern: Direct database access
class OrderService:
    def create_order(self, user_id, items):
        # Bad: Directly accessing user service's database
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        
# Pattern: API calls between services
class OrderService:
    def create_order(self, user_id, items):
        # Good: Call user service API
        user = user_service_client.get_user(user_id)
```

## 2. Service-Oriented Architecture (SOA)

### SOA vs Microservices

**SOA Characteristics:**
- Enterprise-wide architecture
- Reusable services
- Often uses ESB (Enterprise Service Bus)
- Standardized contracts (SOAP/WSDL)
- Centralized governance

**Microservices Characteristics:**
- Application-specific architecture
- Services owned by teams
- Direct service-to-service communication
- Technology diversity allowed
- Decentralized governance

### ESB (Enterprise Service Bus) Pattern

```
Traditional ESB Architecture:

┌─────────┐  ┌─────────┐  ┌─────────┐
│Service A│  │Service B│  │Service C│
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
          ┌───────┴────────┐
          │      ESB        │
          │ ┌────────────┐ │
          │ │ Routing    │ │
          │ │ Transform  │ │
          │ │ Orchestra  │ │
          │ └────────────┘ │
          └───────┬────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
┌────┴────┐  ┌────┴────┐  ┌────┴────┐
│Service D│  │Service E│  │Service F│
└─────────┘  └─────────┘  └─────────┘
```

**ESB Responsibilities:**
- Message routing
- Protocol transformation
- Message transformation
- Service orchestration
- Security enforcement

### Modern SOA with API Management

```python
# API Gateway as modern ESB
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

class APIGateway:
    def __init__(self):
        self.services = {
            'user': 'http://user-service:8000',
            'order': 'http://order-service:8001',
            'inventory': 'http://inventory-service:8002'
        }
    
    def route_request(self, service, path, method='GET', data=None):
        url = f"{self.services[service]}{path}"
        
        # Add common headers
        headers = {
            'X-Request-ID': request.headers.get('X-Request-ID'),
            'X-User-ID': request.headers.get('X-User-ID')
        }
        
        # Transform request if needed
        if service == 'legacy-service':
            data = self.transform_to_soap(data)
        
        response = requests.request(
            method=method,
            url=url,
            json=data,
            headers=headers
        )
        
        return response.json()

gateway = APIGateway()

@app.route('/api/<service>/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def handle_request(service, path):
    return jsonify(
        gateway.route_request(
            service,
            f"/{path}",
            request.method,
            request.json
        )
    )
```

## 3. Event-Driven Architecture

### Core Concepts

**Event:**
A significant change in state.

```python
# Event structure
event = {
    "event_id": "evt_123",
    "event_type": "order.created",
    "timestamp": "2024-01-15T10:30:00Z",
    "data": {
        "order_id": "ord_456",
        "user_id": "usr_789",
        "total": 99.99,
        "items": [...]
    },
    "metadata": {
        "source": "order-service",
        "version": "1.0"
    }
}
```

**Event Producer:**
Service that generates events.

**Event Consumer:**
Service that processes events.

**Event Broker:**
Infrastructure that routes events from producers to consumers.

### Event-Driven Patterns

**1. Event Notification:**
Notify other services that something happened.

```python
class OrderService:
    def create_order(self, order_data):
        # Create order in database
        order = self.repository.create(order_data)
        
        # Notify other services
        self.event_bus.publish('order.created', {
            'order_id': order.id,
            'user_id': order.user_id,
            'total': order.total
        })
        
        return order

class InventoryService:
    def handle_order_created(self, event):
        # React to order creation
        order_id = event['data']['order_id']
        self.reserve_inventory(order_id)

class EmailService:
    def handle_order_created(self, event):
        # Send confirmation email
        user_id = event['data']['user_id']
        self.send_order_confirmation(user_id, event['data'])
```

**2. Event-Carried State Transfer:**
Include all necessary data in the event.

```python
# Full state in event
event = {
    "event_type": "user.updated",
    "data": {
        "user_id": "usr_123",
        "email": "new@example.com",
        "name": "John Doe",
        "address": {
            "street": "123 Main St",
            "city": "Seattle",
            "zip": "98101"
        },
        "preferences": {
            "newsletter": true,
            "notifications": "email"
        }
    }
}

# Consumer doesn't need to call back
class RecommendationService:
    def handle_user_updated(self, event):
        user_data = event['data']
        # Update local cache with full user data
        self.cache.set(f"user:{user_data['user_id']}", user_data)
        # Recalculate recommendations
        self.update_recommendations(user_data)
```

**3. Event Sourcing:**
Store all changes as events.

```python
class EventStore:
    def __init__(self):
        self.events = []
    
    def append(self, event):
        event['sequence'] = len(self.events)
        self.events.append(event)
    
    def get_events(self, aggregate_id, from_sequence=0):
        return [
            e for e in self.events
            if e['aggregate_id'] == aggregate_id
            and e['sequence'] >= from_sequence
        ]

class Order:
    def __init__(self, order_id):
        self.order_id = order_id
        self.items = []
        self.status = 'pending'
        self.total = 0
    
    def apply_event(self, event):
        if event['type'] == 'OrderCreated':
            self.status = 'created'
        elif event['type'] == 'ItemAdded':
            self.items.append(event['data']['item'])
            self.total += event['data']['price']
        elif event['type'] == 'OrderShipped':
            self.status = 'shipped'
    
    @classmethod
    def from_events(cls, order_id, events):
        order = cls(order_id)
        for event in events:
            order.apply_event(event)
        return order
```

### Choreography vs Orchestration

**Choreography (Decentralized):**
Services react to events independently.

```python
# Each service knows what to do when events occur
class PaymentService:
    def __init__(self):
        self.subscribe('order.created', self.process_payment)
    
    def process_payment(self, event):
        # Process payment
        payment_result = self.charge_card(event['data'])
        
        # Publish result
        if payment_result.success:
            self.publish('payment.succeeded', payment_result)
        else:
            self.publish('payment.failed', payment_result)

class ShippingService:
    def __init__(self):
        self.subscribe('payment.succeeded', self.ship_order)
    
    def ship_order(self, event):
        # Create shipment
        shipment = self.create_shipment(event['data'])
        self.publish('order.shipped', shipment)
```

**Orchestration (Centralized):**
Central coordinator manages the workflow.

```python
class OrderOrchestrator:
    def process_order(self, order_data):
        # Step 1: Create order
        order = self.order_service.create_order(order_data)
        
        # Step 2: Process payment
        payment_result = self.payment_service.process_payment(
            order.payment_info
        )
        
        if not payment_result.success:
            self.order_service.cancel_order(order.id)
            return {'status': 'failed', 'reason': 'payment_failed'}
        
        # Step 3: Update inventory
        inventory_result = self.inventory_service.reserve_items(
            order.items
        )
        
        if not inventory_result.success:
            self.payment_service.refund(payment_result.transaction_id)
            self.order_service.cancel_order(order.id)
            return {'status': 'failed', 'reason': 'out_of_stock'}
        
        # Step 4: Arrange shipping
        shipping = self.shipping_service.schedule_delivery(order)
        
        return {'status': 'success', 'order_id': order.id}
```

## 4. CQRS (Command Query Responsibility Segregation)

### Understanding CQRS

CQRS separates read and write operations into different models.

```
Traditional Model:
┌─────────────────┐
│   Application   │
└────────┬────────┘
         │
    ┌────┴────┐
    │  Model  │
    └────┬────┘
         │
    ┌────┴────┐
    │Database │
    └─────────┘

CQRS Model:
┌─────────────────┐
│   Application   │
└──┬──────────┬───┘
   │          │
┌──┴───┐  ┌───┴──┐
│Write │  │ Read │
│Model │  │Model │
└──┬───┘  └───┬──┘
   │          │
┌──┴───┐  ┌───┴──┐
│Write │  │ Read │
│ DB   │  │ DB   │
└──────┘  └──────┘
```

### CQRS Implementation

**Command Side (Write):**
```python
class CreateOrderCommand:
    def __init__(self, user_id, items, payment_info):
        self.user_id = user_id
        self.items = items
        self.payment_info = payment_info

class OrderCommandHandler:
    def handle_create_order(self, command: CreateOrderCommand):
        # Validate command
        if not self.validate_items(command.items):
            raise ValueError("Invalid items")
        
        # Create order aggregate
        order = Order()
        order.create(
            command.user_id,
            command.items,
            command.payment_info
        )
        
        # Save to write store
        self.repository.save(order)
        
        # Publish events
        for event in order.get_uncommitted_events():
            self.event_bus.publish(event)
        
        return order.id
```

**Query Side (Read):**
```python
class OrderReadModel:
    def __init__(self):
        self.orders_view = {}  # Denormalized view
    
    def handle_order_created(self, event):
        # Update read model
        self.orders_view[event['order_id']] = {
            'order_id': event['order_id'],
            'user_id': event['user_id'],
            'user_name': self.get_user_name(event['user_id']),
            'items': event['items'],
            'total': event['total'],
            'status': 'created',
            'created_at': event['timestamp']
        }
    
    def handle_order_shipped(self, event):
        self.orders_view[event['order_id']]['status'] = 'shipped'
        self.orders_view[event['order_id']]['shipped_at'] = event['timestamp']

class OrderQueryHandler:
    def __init__(self, read_model):
        self.read_model = read_model
    
    def get_orders_by_user(self, user_id):
        return [
            order for order in self.read_model.orders_view.values()
            if order['user_id'] == user_id
        ]
    
    def get_order_details(self, order_id):
        return self.read_model.orders_view.get(order_id)
```

### Benefits and Challenges of CQRS

**Benefits:**
- Optimized read and write models
- Independent scaling
- Simplified queries (denormalized views)
- Better performance
- Event sourcing compatibility

**Challenges:**
- Increased complexity
- Eventual consistency
- Data synchronization
- More infrastructure

**When to Use CQRS:**
- Complex domain logic
- Different read/write patterns
- High performance requirements
- Need for event sourcing
- Collaborative domains

## 5. Event Sourcing

### Core Concepts

Store the state as a sequence of events rather than current state.

```python
# Traditional: Store current state
{
    "account_id": "acc_123",
    "balance": 150.00
}

# Event Sourcing: Store events
[
    {"type": "AccountOpened", "amount": 0, "timestamp": "2024-01-01T10:00:00Z"},
    {"type": "MoneyDeposited", "amount": 100, "timestamp": "2024-01-02T10:00:00Z"},
    {"type": "MoneyDeposited", "amount": 75, "timestamp": "2024-01-03T10:00:00Z"},
    {"type": "MoneyWithdrawn", "amount": 25, "timestamp": "2024-01-04T10:00:00Z"}
]
```

### Event Store Implementation

```python
class Event:
    def __init__(self, aggregate_id, event_type, data, version):
        self.aggregate_id = aggregate_id
        self.event_type = event_type
        self.data = data
        self.version = version
        self.timestamp = datetime.utcnow()

class EventStore:
    def __init__(self):
        self.events = []
        self.snapshots = {}
    
    def save_events(self, aggregate_id, events, expected_version):
        # Optimistic concurrency control
        current_version = self.get_version(aggregate_id)
        if current_version != expected_version:
            raise ConcurrencyException()
        
        for event in events:
            event.version = current_version + 1
            self.events.append(event)
            current_version += 1
    
    def get_events(self, aggregate_id, from_version=0):
        return [
            e for e in self.events
            if e.aggregate_id == aggregate_id
            and e.version > from_version
        ]
    
    def save_snapshot(self, aggregate_id, snapshot, version):
        self.snapshots[aggregate_id] = {
            'data': snapshot,
            'version': version
        }
    
    def get_snapshot(self, aggregate_id):
        return self.snapshots.get(aggregate_id)
```

### Aggregate Reconstruction

```python
class BankAccount:
    def __init__(self, account_id):
        self.account_id = account_id
        self.balance = 0
        self.is_frozen = False
        self.version = 0
        self.uncommitted_events = []
    
    def apply_event(self, event):
        if event.event_type == 'AccountOpened':
            self.balance = event.data['initial_balance']
        elif event.event_type == 'MoneyDeposited':
            self.balance += event.data['amount']
        elif event.event_type == 'MoneyWithdrawn':
            self.balance -= event.data['amount']
        elif event.event_type == 'AccountFrozen':
            self.is_frozen = True
        
        self.version = event.version
    
    def deposit(self, amount):
        if self.is_frozen:
            raise Exception("Account is frozen")
        
        event = Event(
            self.account_id,
            'MoneyDeposited',
            {'amount': amount},
            self.version + 1
        )
        
        self.apply_event(event)
        self.uncommitted_events.append(event)
    
    @classmethod
    def from_events(cls, account_id, events):
        account = cls(account_id)
        for event in events:
            account.apply_event(event)
        return account
```

### Projections and Read Models

```python
class AccountProjection:
    def __init__(self, event_store):
        self.event_store = event_store
        self.accounts = {}
        self.last_processed_event = 0
    
    def rebuild(self):
        """Rebuild projection from all events"""
        self.accounts = {}
        all_events = self.event_store.get_all_events()
        
        for event in all_events:
            self.process_event(event)
    
    def process_event(self, event):
        account_id = event.aggregate_id
        
        if event.event_type == 'AccountOpened':
            self.accounts[account_id] = {
                'balance': event.data['initial_balance'],
                'status': 'active',
                'opened_at': event.timestamp
            }
        elif event.event_type == 'MoneyDeposited':
            self.accounts[account_id]['balance'] += event.data['amount']
        elif event.event_type == 'MoneyWithdrawn':
            self.accounts[account_id]['balance'] -= event.data['amount']
        elif event.event_type == 'AccountFrozen':
            self.accounts[account_id]['status'] = 'frozen'
    
    def get_account_balance(self, account_id):
        return self.accounts.get(account_id, {}).get('balance', 0)
```

## 6. Saga Pattern for Distributed Transactions

### Understanding Sagas

A saga is a sequence of local transactions where each transaction updates data within a single service.

### Choreography-Based Saga

Each service listens to events and decides what to do.

```python
# Order Service
class OrderService:
    def create_order(self, order_data):
        order = Order.create(order_data)
        self.repository.save(order)
        
        # Start saga
        self.publish_event('OrderCreated', {
            'order_id': order.id,
            'user_id': order.user_id,
            'items': order.items,
            'payment_info': order.payment_info
        })

    def handle_payment_failed(self, event):
        order = self.repository.get(event['order_id'])
        order.cancel()
        self.repository.save(order)
        
        self.publish_event('OrderCancelled', {
            'order_id': order.id
        })

# Payment Service
class PaymentService:
    def handle_order_created(self, event):
        try:
            payment = self.process_payment(event['payment_info'])
            self.publish_event('PaymentSucceeded', {
                'order_id': event['order_id'],
                'payment_id': payment.id
            })
        except PaymentException as e:
            self.publish_event('PaymentFailed', {
                'order_id': event['order_id'],
                'reason': str(e)
            })

# Inventory Service
class InventoryService:
    def handle_payment_succeeded(self, event):
        try:
            self.reserve_items(event['order_id'])
            self.publish_event('InventoryReserved', {
                'order_id': event['order_id']
            })
        except InsufficientInventory as e:
            self.publish_event('InventoryReservationFailed', {
                'order_id': event['order_id']
            })
    
    def handle_order_cancelled(self, event):
        self.release_items(event['order_id'])
```

### Orchestration-Based Saga

A central orchestrator coordinates the saga.

```python
class OrderSagaOrchestrator:
    def __init__(self):
        self.state_machine = self.create_state_machine()
        self.saga_instances = {}
    
    def create_state_machine(self):
        return {
            'START': {
                'success': 'PAYMENT_PENDING',
                'command': 'CreateOrder'
            },
            'PAYMENT_PENDING': {
                'success': 'INVENTORY_PENDING',
                'failure': 'CANCELLING_ORDER',
                'command': 'ProcessPayment',
                'compensation': 'RefundPayment'
            },
            'INVENTORY_PENDING': {
                'success': 'SHIPPING_PENDING',
                'failure': 'REFUNDING_PAYMENT',
                'command': 'ReserveInventory',
                'compensation': 'ReleaseInventory'
            },
            'SHIPPING_PENDING': {
                'success': 'COMPLETED',
                'failure': 'RELEASING_INVENTORY',
                'command': 'CreateShipment',
                'compensation': 'CancelShipment'
            },
            'COMPLETED': {
                'terminal': True
            },
            'CANCELLING_ORDER': {
                'success': 'FAILED',
                'command': 'CancelOrder'
            },
            'FAILED': {
                'terminal': True
            }
        }
    
    def start_saga(self, order_data):
        saga_id = str(uuid.uuid4())
        
        self.saga_instances[saga_id] = {
            'state': 'START',
            'data': order_data,
            'completed_steps': []
        }
        
        self.execute_step(saga_id)
        return saga_id
    
    def execute_step(self, saga_id):
        saga = self.saga_instances[saga_id]
        current_state = self.state_machine[saga['state']]
        
        if 'terminal' in current_state:
            return
        
        command = current_state['command']
        
        try:
            result = self.execute_command(command, saga['data'])
            saga['completed_steps'].append({
                'command': command,
                'result': result
            })
            saga['state'] = current_state['success']
        except Exception as e:
            saga['state'] = current_state['failure']
            self.compensate(saga_id)
        
        self.execute_step(saga_id)
    
    def compensate(self, saga_id):
        saga = self.saga_instances[saga_id]
        
        # Execute compensations in reverse order
        for step in reversed(saga['completed_steps']):
            state_def = self.find_state_by_command(step['command'])
            if 'compensation' in state_def:
                self.execute_command(
                    state_def['compensation'],
                    step['result']
                )
```

## 7. API Design Patterns

### REST API Design

**Resource-Oriented Design:**
```python
# Good REST design
GET    /api/users           # List users
GET    /api/users/123        # Get specific user
POST   /api/users           # Create user
PUT    /api/users/123        # Update user
PATCH  /api/users/123        # Partial update
DELETE /api/users/123        # Delete user

# Nested resources
GET    /api/users/123/orders     # User's orders
POST   /api/users/123/orders     # Create order for user

# Filtering, sorting, pagination
GET    /api/users?status=active&sort=created_at&page=2&limit=20
```

**RESTful API Implementation:**
```python
from flask import Flask, request, jsonify
from flask_restful import Api, Resource

app = Flask(__name__)
api = Api(app)

class UserResource(Resource):
    def get(self, user_id=None):
        if user_id:
            user = User.get_by_id(user_id)
            if not user:
                return {'error': 'User not found'}, 404
            return user.to_dict(), 200
        else:
            # List users with pagination
            page = request.args.get('page', 1, type=int)
            limit = request.args.get('limit', 20, type=int)
            
            users = User.paginate(page, limit)
            return {
                'users': [u.to_dict() for u in users.items],
                'total': users.total,
                'page': page,
                'pages': users.pages
            }, 200
    
    def post(self):
        data = request.get_json()
        
        # Validate input
        errors = self.validate_user_data(data)
        if errors:
            return {'errors': errors}, 400
        
        user = User.create(data)
        return user.to_dict(), 201, {'Location': f'/api/users/{user.id}'}
    
    def put(self, user_id):
        user = User.get_by_id(user_id)
        if not user:
            return {'error': 'User not found'}, 404
        
        data = request.get_json()
        user.update(data)
        return user.to_dict(), 200
    
    def delete(self, user_id):
        user = User.get_by_id(user_id)
        if not user:
            return {'error': 'User not found'}, 404
        
        user.delete()
        return '', 204

api.add_resource(UserResource, '/api/users', '/api/users/<int:user_id>')
```

### GraphQL API Design

**Schema Definition:**
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  orders(limit: Int, offset: Int): [Order!]!
  totalSpent: Float!
}

type Order {
  id: ID!
  user: User!
  items: [OrderItem!]!
  total: Float!
  status: OrderStatus!
  createdAt: DateTime!
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

type Query {
  user(id: ID!): User
  users(filter: UserFilter, limit: Int, offset: Int): [User!]!
  order(id: ID!): Order
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  createOrder(input: CreateOrderInput!): Order!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}
```

**GraphQL Resolver Implementation:**
```python
import graphene
from graphene import relay
from graphql import GraphQLError

class UserType(graphene.ObjectType):
    id = graphene.ID(required=True)
    name = graphene.String(required=True)
    email = graphene.String(required=True)
    orders = graphene.List('OrderType', limit=graphene.Int(), offset=graphene.Int())
    total_spent = graphene.Float()
    
    def resolve_orders(self, info, limit=10, offset=0):
        # N+1 query prevention with DataLoader
        return order_loader.load_many(self.id, limit, offset)
    
    def resolve_total_spent(self, info):
        # Computed field
        return sum(order.total for order in self.orders)

class Query(graphene.ObjectType):
    user = graphene.Field(UserType, id=graphene.ID(required=True))
    users = graphene.List(
        UserType,
        filter=graphene.Argument(UserFilterInput),
        limit=graphene.Int(),
        offset=graphene.Int()
    )
    
    def resolve_user(self, info, id):
        # Check permissions
        if not info.context.user.can_view_user(id):
            raise GraphQLError('Unauthorized')
        
        return User.get_by_id(id)
    
    def resolve_users(self, info, filter=None, limit=20, offset=0):
        query = User.query
        
        if filter:
            if filter.status:
                query = query.filter_by(status=filter.status)
            if filter.created_after:
                query = query.filter(User.created_at > filter.created_after)
        
        return query.limit(limit).offset(offset).all()
```

### gRPC API Design

**Protocol Buffers Definition:**
```protobuf
syntax = "proto3";

package user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (Empty);
  
  // Streaming example
  rpc WatchUserUpdates(WatchUserUpdatesRequest) returns (stream UserUpdate);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
  UserStatus status = 5;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}

message GetUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  UserFilter filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}
```

**gRPC Server Implementation:**
```python
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserServicer(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        user = User.get_by_id(request.id)
        
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details('User not found')
            return user_pb2.User()
        
        return user_pb2.User(
            id=user.id,
            name=user.name,
            email=user.email,
            created_at=int(user.created_at.timestamp()),
            status=user_pb2.UserStatus.Value(f'USER_STATUS_{user.status.upper()}')
        )
    
    def ListUsers(self, request, context):
        users = User.paginate(
            page_size=request.page_size,
            page_token=request.page_token
        )
        
        return user_pb2.ListUsersResponse(
            users=[self._user_to_proto(u) for u in users],
            next_page_token=users.next_page_token,
            total_count=users.total
        )
    
    def WatchUserUpdates(self, request, context):
        # Server streaming
        user_id = request.user_id
        
        for update in self.user_update_stream(user_id):
            if context.is_active():
                yield user_pb2.UserUpdate(
                    user=self._user_to_proto(update.user),
                    update_type=update.type,
                    timestamp=int(update.timestamp.timestamp())
                )
            else:
                break

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()
```

## 8. Synchronous vs Asynchronous Communication

### Synchronous Communication

**Characteristics:**
- Caller waits for response
- Simple mental model
- Tight coupling
- Cascading failures possible

**HTTP/REST Example:**
```python
# Synchronous service calls
class OrderService:
    def create_order(self, order_data):
        # Synchronous calls to other services
        user = self.user_service.get_user(order_data['user_id'])
        if not user:
            raise UserNotFoundException()
        
        payment = self.payment_service.process_payment(
            order_data['payment_info']
        )
        if not payment.successful:
            raise PaymentFailedException()
        
        inventory = self.inventory_service.reserve_items(
            order_data['items']
        )
        if not inventory.reserved:
            self.payment_service.refund(payment.id)
            raise InsufficientInventoryException()
        
        order = Order.create(order_data)
        return order
```

### Asynchronous Communication

**Characteristics:**
- Caller doesn't wait
- Better fault isolation
- Complex error handling
- Eventual consistency

**Message Queue Example:**
```python
# Asynchronous with message queue
class OrderService:
    def create_order(self, order_data):
        order = Order.create(order_data)
        order.status = 'pending'
        self.repository.save(order)
        
        # Send message to queue
        self.message_queue.send('process-order', {
            'order_id': order.id,
            'user_id': order.user_id,
            'items': order.items,
            'payment_info': order.payment_info
        })
        
        return order  # Return immediately

class OrderProcessor:
    def process_order_message(self, message):
        order_id = message['order_id']
        
        try:
            # Process payment asynchronously
            payment_result = self.payment_service.process(
                message['payment_info']
            )
            
            if payment_result.successful:
                self.update_order_status(order_id, 'paid')
                
                # Send next message
                self.message_queue.send('reserve-inventory', {
                    'order_id': order_id,
                    'items': message['items']
                })
            else:
                self.update_order_status(order_id, 'payment_failed')
        except Exception as e:
            # Retry logic
            self.retry_queue.send(message, delay=60)
```

### Hybrid Approaches

**Async with Callback:**
```python
class OrderService:
    def create_order(self, order_data, callback_url):
        order = Order.create(order_data)
        
        # Start async processing
        self.task_queue.enqueue('process_order', {
            'order_id': order.id,
            'callback_url': callback_url
        })
        
        # Return immediately with tracking info
        return {
            'order_id': order.id,
            'status': 'processing',
            'status_url': f'/api/orders/{order.id}/status'
        }

class OrderProcessor:
    def process_order(self, task_data):
        # Long-running processing
        result = self.process_order_steps(task_data['order_id'])
        
        # Notify caller via webhook
        requests.post(task_data['callback_url'], json={
            'order_id': task_data['order_id'],
            'status': result.status,
            'details': result.details
        })
```

## Summary and Key Takeaways

1. **Microservices require careful boundary definition** - Use Domain-Driven Design
2. **Event-driven architecture enables loose coupling** - But adds complexity
3. **CQRS optimizes for different access patterns** - Separate read and write models
4. **Event sourcing provides complete audit trail** - Store events, not state
5. **Sagas manage distributed transactions** - Choose choreography or orchestration
6. **API design impacts system evolution** - REST for resources, GraphQL for flexibility, gRPC for performance
7. **Async communication improves resilience** - But requires careful error handling
8. **Patterns are tools, not rules** - Choose based on your specific requirements

## Next Steps
- Implement a simple event-driven system
- Build a CQRS example with separate models
- Create a saga for a multi-step workflow
- Study Part 4: Storage and Databases for data management patterns