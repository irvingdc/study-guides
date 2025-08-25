# Dependency Inversion Principle (DIP) - Complete Guide
## Object-Oriented vs Functional Approaches in TypeScript

---

## Introduction

The Dependency Inversion Principle is one of the SOLID principles that states:

1. **High-level modules should not depend on low-level modules. Both should depend on abstractions.**
2. **Abstractions should not depend on details. Details should depend on abstractions.**

In simpler terms: depend on interfaces/contracts, not concrete implementations.

### Why Dependency Inversion Matters

- **Flexibility**: Easy to swap implementations
- **Testability**: Simple to mock dependencies
- **Maintainability**: Changes in low-level modules don't affect high-level logic
- **Decoupling**: Reduces tight coupling between components

---

## Part 1: Object-Oriented Approach

### Traditional OOP with Interfaces

```typescript
// ============================================
// BAD: Direct dependency on concrete classes
// ============================================

// ❌ Low-level module (concrete implementation)
class MySQLDatabase {
  save(data: any): void {
    console.log('Saving to MySQL:', data);
    // MySQL specific implementation
  }
  
  find(id: string): any {
    console.log('Finding in MySQL:', id);
    // MySQL specific query
    return { id, source: 'mysql' };
  }
}

// ❌ High-level module directly depends on low-level module
class UserService {
  private database: MySQLDatabase; // Direct dependency!
  
  constructor() {
    this.database = new MySQLDatabase(); // Tight coupling!
  }
  
  createUser(name: string): void {
    const user = { id: Date.now().toString(), name };
    this.database.save(user); // Tied to MySQL
  }
}

// Problems:
// 1. Can't switch to PostgreSQL without changing UserService
// 2. Hard to test UserService without real MySQL
// 3. UserService knows too much about database details
```

```typescript
// ============================================
// GOOD: Dependency Inversion with Interfaces
// ============================================

// ✅ Abstraction (interface)
interface Database {
  save(data: any): void;
  find(id: string): any;
  update(id: string, data: any): void;
  delete(id: string): void;
}

// ✅ High-level module depends on abstraction
class UserService {
  constructor(private database: Database) {} // Depends on interface!
  
  createUser(name: string): void {
    const user = { id: Date.now().toString(), name };
    this.database.save(user);
  }
  
  findUser(id: string): any {
    return this.database.find(id);
  }
}

// ✅ Low-level modules implement abstraction
class MySQLDatabase implements Database {
  save(data: any): void {
    console.log('MySQL: INSERT INTO users...', data);
  }
  
  find(id: string): any {
    console.log('MySQL: SELECT * FROM users WHERE id =', id);
    return { id, source: 'mysql' };
  }
  
  update(id: string, data: any): void {
    console.log('MySQL: UPDATE users SET...', id, data);
  }
  
  delete(id: string): void {
    console.log('MySQL: DELETE FROM users WHERE id =', id);
  }
}

class MongoDatabase implements Database {
  save(data: any): void {
    console.log('MongoDB: db.users.insertOne()', data);
  }
  
  find(id: string): any {
    console.log('MongoDB: db.users.findOne()', id);
    return { id, source: 'mongo' };
  }
  
  update(id: string, data: any): void {
    console.log('MongoDB: db.users.updateOne()', id, data);
  }
  
  delete(id: string): void {
    console.log('MongoDB: db.users.deleteOne()', id);
  }
}

// ✅ Easy to swap implementations
const mysqlService = new UserService(new MySQLDatabase());
const mongoService = new UserService(new MongoDatabase());

// ✅ Easy to test with mock
class MockDatabase implements Database {
  private data = new Map();
  
  save(data: any): void {
    this.data.set(data.id, data);
  }
  
  find(id: string): any {
    return this.data.get(id);
  }
  
  update(id: string, data: any): void {
    this.data.set(id, { ...this.data.get(id), ...data });
  }
  
  delete(id: string): void {
    this.data.delete(id);
  }
}

const testService = new UserService(new MockDatabase());
```

### Advanced OOP Pattern: Repository Pattern

```typescript
// ============================================
// Repository Pattern with Dependency Inversion
// ============================================

// Domain model (pure, no dependencies)
interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
}

// Repository abstraction
interface UserRepository {
  save(user: User): Promise<void>;
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  update(id: string, updates: Partial<User>): Promise<void>;
  delete(id: string): Promise<void>;
}

// Business logic depends on abstraction
class UserManagementService {
  constructor(
    private userRepo: UserRepository,
    private emailService: EmailService,
    private logger: Logger
  ) {}
  
  async registerUser(email: string, name: string): Promise<User> {
    // Check if user exists
    const existing = await this.userRepo.findByEmail(email);
    if (existing) {
      this.logger.warn(`Registration attempt for existing email: ${email}`);
      throw new Error('User already exists');
    }
    
    // Create new user
    const user: User = {
      id: this.generateId(),
      email,
      name,
      createdAt: new Date()
    };
    
    // Save user
    await this.userRepo.save(user);
    this.logger.info(`New user registered: ${user.id}`);
    
    // Send welcome email
    await this.emailService.sendWelcome(email, name);
    
    return user;
  }
  
  private generateId(): string {
    return `user_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Concrete implementations
class PostgresUserRepository implements UserRepository {
  constructor(private db: PostgresConnection) {}
  
  async save(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (id, email, name, created_at) VALUES ($1, $2, $3, $4)',
      [user.id, user.email, user.name, user.createdAt]
    );
  }
  
  async findById(id: string): Promise<User | null> {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return result.rows[0] || null;
  }
  
  async findByEmail(email: string): Promise<User | null> {
    const result = await this.db.query(
      'SELECT * FROM users WHERE email = $1',
      [email]
    );
    return result.rows[0] || null;
  }
  
  async update(id: string, updates: Partial<User>): Promise<void> {
    // Build dynamic UPDATE query
    const fields = Object.keys(updates);
    const values = Object.values(updates);
    const setClause = fields.map((f, i) => `${f} = $${i + 2}`).join(', ');
    
    await this.db.query(
      `UPDATE users SET ${setClause} WHERE id = $1`,
      [id, ...values]
    );
  }
  
  async delete(id: string): Promise<void> {
    await this.db.query('DELETE FROM users WHERE id = $1', [id]);
  }
}

// In-memory implementation for testing
class InMemoryUserRepository implements UserRepository {
  private users = new Map<string, User>();
  
  async save(user: User): Promise<void> {
    this.users.set(user.id, user);
  }
  
  async findById(id: string): Promise<User | null> {
    return this.users.get(id) || null;
  }
  
  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.users.values()) {
      if (user.email === email) return user;
    }
    return null;
  }
  
  async update(id: string, updates: Partial<User>): Promise<void> {
    const user = this.users.get(id);
    if (user) {
      this.users.set(id, { ...user, ...updates });
    }
  }
  
  async delete(id: string): Promise<void> {
    this.users.delete(id);
  }
}
```

---

## Part 2: Functional Programming Approach

### Pure Functions and Higher-Order Functions

```typescript
// ============================================
// Functional Dependency Inversion
// ============================================

// Instead of interfaces, we use function types
type SaveFunction = (data: any) => Promise<void>;
type FindFunction = (id: string) => Promise<any>;
type UpdateFunction = (id: string, data: any) => Promise<void>;
type DeleteFunction = (id: string) => Promise<void>;

// Database operations as a collection of functions
interface DatabaseOperations {
  save: SaveFunction;
  find: FindFunction;
  update: UpdateFunction;
  delete: DeleteFunction;
}

// Higher-order function that creates user service
function createUserService(db: DatabaseOperations) {
  return {
    async createUser(name: string) {
      const user = { id: Date.now().toString(), name };
      await db.save(user);
      return user;
    },
    
    async findUser(id: string) {
      return await db.find(id);
    },
    
    async updateUser(id: string, name: string) {
      await db.update(id, { name });
    },
    
    async deleteUser(id: string) {
      await db.delete(id);
    }
  };
}

// Concrete implementations as functions
const mysqlOperations: DatabaseOperations = {
  save: async (data) => {
    console.log('MySQL save:', data);
    // MySQL implementation
  },
  find: async (id) => {
    console.log('MySQL find:', id);
    return { id, source: 'mysql' };
  },
  update: async (id, data) => {
    console.log('MySQL update:', id, data);
  },
  delete: async (id) => {
    console.log('MySQL delete:', id);
  }
};

const mongoOperations: DatabaseOperations = {
  save: async (data) => {
    console.log('MongoDB save:', data);
    // MongoDB implementation
  },
  find: async (id) => {
    console.log('MongoDB find:', id);
    return { id, source: 'mongo' };
  },
  update: async (id, data) => {
    console.log('MongoDB update:', id, data);
  },
  delete: async (id) => {
    console.log('MongoDB delete:', id);
  }
};

// Usage
const mysqlUserService = createUserService(mysqlOperations);
const mongoUserService = createUserService(mongoOperations);

// Testing is easy with mock functions
const mockOperations: DatabaseOperations = {
  save: jest.fn(),
  find: jest.fn().mockResolvedValue({ id: '1', name: 'Test' }),
  update: jest.fn(),
  delete: jest.fn()
};

const testUserService = createUserService(mockOperations);
```

### Function Composition and Currying

```typescript
// ============================================
// Advanced Functional Patterns
// ============================================

// Types for our domain
type User = {
  id: string;
  email: string;
  name: string;
};

type Logger = (level: string, message: string) => void;
type Validator = (user: Partial<User>) => boolean;
type Hasher = (password: string) => string;

// Curried function for dependency injection
const createUserRegistration = 
  (log: Logger) =>
  (validate: Validator) =>
  (hash: Hasher) =>
  (saveUser: (user: User) => Promise<void>) =>
  async (email: string, name: string, password: string): Promise<User> => {
    // Use injected dependencies
    log('info', `Attempting to register user: ${email}`);
    
    const userData = { email, name };
    
    if (!validate(userData)) {
      log('error', 'Validation failed');
      throw new Error('Invalid user data');
    }
    
    const user: User = {
      id: `user_${Date.now()}`,
      email,
      name
    };
    
    // Note: In real app, you'd save hashed password separately
    const hashedPassword = hash(password);
    log('debug', `Password hashed: ${hashedPassword.substring(0, 10)}...`);
    
    await saveUser(user);
    log('info', `User registered successfully: ${user.id}`);
    
    return user;
  };

// Implementations
const consoleLogger: Logger = (level, message) => {
  console.log(`[${level.toUpperCase()}] ${message}`);
};

const emailValidator: Validator = (user) => {
  return !!(user.email && user.email.includes('@'));
};

const bcryptHasher: Hasher = (password) => {
  // Simplified - use real bcrypt in production
  return `hashed_${password}`;
};

const saveToDatabase = async (user: User) => {
  console.log('Saving to database:', user);
  // Database save logic
};

// Compose the function with dependencies
const registerUser = 
  createUserRegistration
    (consoleLogger)
    (emailValidator)
    (bcryptHasher)
    (saveToDatabase);

// Use it
registerUser('john@example.com', 'John Doe', 'secretpassword');
```

### Functional Dependency Injection with Reader Monad Pattern

```typescript
// ============================================
// Reader Monad-like Pattern
// ============================================

// Dependencies container
interface Dependencies {
  database: DatabaseOperations;
  logger: Logger;
  cache: CacheOperations;
  config: AppConfig;
}

// Reader type - a function that takes dependencies and returns a result
type Reader<D, A> = (deps: D) => A;

// Async Reader for async operations
type AsyncReader<D, A> = (deps: D) => Promise<A>;

// User service functions that return Readers
const createUser = (name: string): AsyncReader<Dependencies, User> =>
  async (deps) => {
    deps.logger('info', `Creating user: ${name}`);
    
    const user: User = {
      id: Date.now().toString(),
      email: `${name.toLowerCase()}@example.com`,
      name
    };
    
    await deps.database.save(user);
    await deps.cache.set(user.id, user);
    
    return user;
  };

const findUser = (id: string): AsyncReader<Dependencies, User | null> =>
  async (deps) => {
    // Check cache first
    const cached = await deps.cache.get(id);
    if (cached) {
      deps.logger('debug', `Cache hit for user: ${id}`);
      return cached;
    }
    
    // Fallback to database
    deps.logger('debug', `Cache miss, querying database for user: ${id}`);
    const user = await deps.database.find(id);
    
    if (user) {
      await deps.cache.set(id, user);
    }
    
    return user;
  };

// Compose operations
const getUserWithLogging = (id: string): AsyncReader<Dependencies, User | null> =>
  async (deps) => {
    deps.logger('info', `Fetching user: ${id}`);
    const user = await findUser(id)(deps);
    
    if (user) {
      deps.logger('info', `Found user: ${user.name}`);
    } else {
      deps.logger('warn', `User not found: ${id}`);
    }
    
    return user;
  };

// Create dependencies
const dependencies: Dependencies = {
  database: mysqlOperations,
  logger: consoleLogger,
  cache: {
    get: async (key) => {
      console.log(`Cache get: ${key}`);
      return null; // Simplified
    },
    set: async (key, value) => {
      console.log(`Cache set: ${key}`, value);
    }
  },
  config: {
    appName: 'MyApp',
    version: '1.0.0'
  }
};

// Run the Reader with dependencies
const user = await getUserWithLogging('123')(dependencies);
```

---

## Part 3: Comparison and Best Practices

### OOP vs Functional Approaches

| Aspect | Object-Oriented | Functional |
|--------|-----------------|------------|
| **Abstraction** | Interfaces/Abstract classes | Function types/signatures |
| **Injection** | Constructor/Setter | Function parameters/Currying |
| **Testing** | Mock objects | Mock functions |
| **Composition** | Inheritance/Composition | Function composition |
| **State** | Encapsulated in objects | Passed through functions |
| **Flexibility** | Runtime polymorphism | Higher-order functions |

### When to Use Which Approach

**Use OOP Dependency Inversion when:**
- Working with stateful services
- Need complex hierarchies
- Team is familiar with OOP patterns
- Using frameworks that expect classes (Angular, NestJS)

**Use Functional Dependency Inversion when:**
- Building pure, testable functions
- Want maximum flexibility
- Prefer composition over inheritance
- Working with functional libraries (fp-ts, Ramda)

### Hybrid Approach

```typescript
// ============================================
// Combining OOP and Functional Approaches
// ============================================

// Functional core with OOP shell
class UserController {
  constructor(
    private deps: Dependencies,
    private userOps = {
      create: createUser,
      find: findUser
    }
  ) {}
  
  async handleCreateUser(req: Request): Promise<Response> {
    const { name } = req.body;
    
    // Use functional operations with injected dependencies
    const user = await this.userOps.create(name)(this.deps);
    
    return {
      status: 200,
      body: user
    };
  }
  
  async handleGetUser(req: Request): Promise<Response> {
    const { id } = req.params;
    
    const user = await this.userOps.find(id)(this.deps);
    
    if (!user) {
      return { status: 404, body: { error: 'User not found' } };
    }
    
    return { status: 200, body: user };
  }
}
```

### Testing Strategies

```typescript
// ============================================
// Testing with Dependency Inversion
// ============================================

// OOP Testing
describe('UserService (OOP)', () => {
  it('should create user', async () => {
    const mockDb = new MockDatabase();
    const service = new UserService(mockDb);
    
    await service.createUser('John');
    
    expect(mockDb.data.size).toBe(1);
  });
});

// Functional Testing
describe('UserService (Functional)', () => {
  it('should create user', async () => {
    const mockSave = jest.fn();
    const mockOps = { ...mockOperations, save: mockSave };
    const service = createUserService(mockOps);
    
    await service.createUser('John');
    
    expect(mockSave).toHaveBeenCalledWith(
      expect.objectContaining({ name: 'John' })
    );
  });
});

// Reader Pattern Testing
describe('UserService (Reader)', () => {
  it('should create user with caching', async () => {
    const mockDeps: Dependencies = {
      database: { save: jest.fn() },
      cache: { set: jest.fn() },
      logger: jest.fn(),
      config: { appName: 'Test', version: '1.0' }
    };
    
    const user = await createUser('John')(mockDeps);
    
    expect(mockDeps.database.save).toHaveBeenCalled();
    expect(mockDeps.cache.set).toHaveBeenCalledWith(user.id, user);
    expect(mockDeps.logger).toHaveBeenCalledWith('info', expect.any(String));
  });
});
```

---

## Real-World Example: Payment Processing

```typescript
// ============================================
// Complete Example: Payment System
// ============================================

// Domain types
type Payment = {
  id: string;
  amount: number;
  currency: string;
  status: 'pending' | 'completed' | 'failed';
};

// OOP Approach
interface PaymentGateway {
  charge(amount: number, currency: string): Promise<string>;
  refund(transactionId: string): Promise<void>;
}

class PaymentService {
  constructor(
    private gateway: PaymentGateway,
    private repository: PaymentRepository,
    private notifier: NotificationService
  ) {}
  
  async processPayment(amount: number, currency: string): Promise<Payment> {
    const payment: Payment = {
      id: this.generateId(),
      amount,
      currency,
      status: 'pending'
    };
    
    await this.repository.save(payment);
    
    try {
      const transactionId = await this.gateway.charge(amount, currency);
      payment.status = 'completed';
      await this.repository.update(payment.id, { status: 'completed' });
      await this.notifier.notify('Payment successful', payment);
    } catch (error) {
      payment.status = 'failed';
      await this.repository.update(payment.id, { status: 'failed' });
      await this.notifier.notify('Payment failed', payment);
      throw error;
    }
    
    return payment;
  }
  
  private generateId(): string {
    return `pay_${Date.now()}`;
  }
}

// Functional Approach
type ProcessPayment = (
  gateway: PaymentGateway,
  save: (payment: Payment) => Promise<void>,
  notify: (message: string, payment: Payment) => Promise<void>
) => (amount: number, currency: string) => Promise<Payment>;

const processPayment: ProcessPayment = (gateway, save, notify) => async (amount, currency) => {
  const payment: Payment = {
    id: `pay_${Date.now()}`,
    amount,
    currency,
    status: 'pending'
  };
  
  await save(payment);
  
  try {
    await gateway.charge(amount, currency);
    payment.status = 'completed';
    await save({ ...payment, status: 'completed' });
    await notify('Payment successful', payment);
  } catch (error) {
    payment.status = 'failed';
    await save({ ...payment, status: 'failed' });
    await notify('Payment failed', payment);
    throw error;
  }
  
  return payment;
};

// Different gateway implementations
class StripeGateway implements PaymentGateway {
  async charge(amount: number, currency: string): Promise<string> {
    console.log(`Charging ${amount} ${currency} via Stripe`);
    return 'stripe_txn_123';
  }
  
  async refund(transactionId: string): Promise<void> {
    console.log(`Refunding Stripe transaction: ${transactionId}`);
  }
}

class PayPalGateway implements PaymentGateway {
  async charge(amount: number, currency: string): Promise<string> {
    console.log(`Charging ${amount} ${currency} via PayPal`);
    return 'paypal_txn_456';
  }
  
  async refund(transactionId: string): Promise<void> {
    console.log(`Refunding PayPal transaction: ${transactionId}`);
  }
}

// Usage - Easy to switch implementations
const stripePayments = new PaymentService(
  new StripeGateway(),
  new PostgresPaymentRepository(),
  new EmailNotificationService()
);

const paypalPayments = new PaymentService(
  new PayPalGateway(),
  new PostgresPaymentRepository(),
  new EmailNotificationService()
);
```

---

## Key Takeaways

1. **Dependency Inversion is about depending on abstractions, not concretions**
2. **OOP uses interfaces/abstract classes for abstraction**
3. **FP uses function signatures and higher-order functions**
4. **Both approaches improve testability and flexibility**
5. **Choose based on your team's expertise and project requirements**
6. **Can combine both approaches for maximum flexibility**

### Best Practices

- **Keep abstractions stable**: Don't change interfaces frequently
- **Program to interfaces**: Always depend on abstractions in high-level code
- **Single Responsibility**: Each abstraction should have one reason to change
- **Dependency Injection**: Use DI containers or functional composition
- **Test with mocks**: Leverage dependency inversion for easy testing
- **Document contracts**: Make interface expectations clear

---

## Additional Resources

- Martin, Robert C. "Clean Architecture"
- Fowler, Martin. "Inversion of Control Containers and the Dependency Injection pattern"
- Functional Programming Patterns in TypeScript
- SOLID Principles in TypeScript

---

*This guide provides a comprehensive understanding of Dependency Inversion in both OOP and Functional paradigms. Practice these patterns to write more maintainable, testable, and flexible code.*