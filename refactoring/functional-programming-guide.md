# Functional Programming Guide for TypeScript
## Alternative Approaches to OOP Patterns

---

## 1. SOLID Principles in Functional Programming

### 1.1 Single Responsibility Principle (SRP) - Functional Approach

In FP, we achieve SRP through **pure functions** and **function composition**.

```typescript
// ❌ OOP Approach: Class with multiple responsibilities
class User {
  save(): void { /* database logic */ }
  sendEmail(): void { /* email logic */ }
  validate(): boolean { /* validation logic */ }
}

// ✅ FP Approach: Separate pure functions
// Each function has a single responsibility

// Data types (immutable)
type User = {
  readonly id: string;
  readonly name: string;
  readonly email: string;
  readonly age: number;
};

// Pure functions for each responsibility
const createUser = (name: string, email: string, age: number): User => ({
  id: generateId(),
  name,
  email,
  age
});

// Validation functions
const validateEmail = (email: string): boolean =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

const validateAge = (age: number): boolean => 
  age >= 18 && age <= 120;

const validateUser = (user: User): Either<string[], User> => {
  const errors: string[] = [];
  
  if (!validateEmail(user.email)) errors.push('Invalid email');
  if (!validateAge(user.age)) errors.push('Invalid age');
  
  return errors.length > 0 
    ? left(errors) 
    : right(user);
};

// Database functions
const saveUser = (db: Database) => (user: User): Promise<void> =>
  db.insert('users', user);

// Email functions  
const sendWelcomeEmail = (emailService: EmailService) => (user: User): Promise<void> =>
  emailService.send(user.email, 'Welcome!', `Hello ${user.name}`);

// Composition using pipe
import { pipe } from 'fp-ts/function';

const processNewUser = (db: Database, emailService: EmailService) => 
  pipe(
    createUser,
    validateUser,
    chain(saveUser(db)),
    chain(sendWelcomeEmail(emailService))
  );
```

### 1.2 Open/Closed Principle (OCP) - Functional Approach

In FP, we achieve OCP through **higher-order functions** and **function composition**.

```typescript
// ❌ OOP Approach: Classes and inheritance
interface DiscountStrategy {
  calculateDiscount(amount: number): number;
}

// ✅ FP Approach: Functions as first-class citizens
type DiscountCalculator = (amount: number) => number;

// Base discount functions
const regularDiscount: DiscountCalculator = (amount) => amount * 0.95;
const premiumDiscount: DiscountCalculator = (amount) => amount * 0.90;
const vipDiscount: DiscountCalculator = (amount) => amount * 0.85;

// Higher-order function for combining discounts
const combineDiscounts = (...discounts: DiscountCalculator[]): DiscountCalculator =>
  (amount) => discounts.reduce((acc, discount) => discount(acc), amount);

// Conditional discount using higher-order function
const conditionalDiscount = (
  condition: (amount: number) => boolean,
  discount: DiscountCalculator
): DiscountCalculator =>
  (amount) => condition(amount) ? discount(amount) : amount;

// Volume discount
const volumeDiscount = conditionalDiscount(
  amount => amount > 1000,
  amount => amount * 0.9
);

// Create new discount types without modifying existing code
const seasonalDiscount = (season: string): DiscountCalculator => {
  const rates = { summer: 0.85, winter: 0.90, spring: 0.95, fall: 0.92 };
  return (amount) => amount * (rates[season] || 1);
};

// Compose discounts
const calculateFinalPrice = combineDiscounts(
  regularDiscount,
  volumeDiscount,
  seasonalDiscount('summer')
);
```

### 1.3 Liskov Substitution Principle (LSP) - Functional Approach

In FP, we use **algebraic data types** and **pattern matching** instead of inheritance.

```typescript
// ❌ OOP Approach: Inheritance hierarchy
class Rectangle { setWidth(); setHeight(); }
class Square extends Rectangle { /* problematic */ }

// ✅ FP Approach: Algebraic Data Types (ADTs)
type Shape = 
  | { type: 'rectangle'; width: number; height: number }
  | { type: 'square'; side: number }
  | { type: 'circle'; radius: number };

// Functions work with all shape types predictably
const area = (shape: Shape): number => {
  switch (shape.type) {
    case 'rectangle':
      return shape.width * shape.height;
    case 'square':
      return shape.side * shape.side;
    case 'circle':
      return Math.PI * shape.radius * shape.radius;
  }
};

const perimeter = (shape: Shape): number => {
  switch (shape.type) {
    case 'rectangle':
      return 2 * (shape.width + shape.height);
    case 'square':
      return 4 * shape.side;
    case 'circle':
      return 2 * Math.PI * shape.radius;
  }
};

// Shape transformations
const scale = (factor: number) => (shape: Shape): Shape => {
  switch (shape.type) {
    case 'rectangle':
      return { ...shape, width: shape.width * factor, height: shape.height * factor };
    case 'square':
      return { ...shape, side: shape.side * factor };
    case 'circle':
      return { ...shape, radius: shape.radius * factor };
  }
};
```

### 1.4 Interface Segregation Principle (ISP) - Functional Approach

In FP, we achieve ISP through **function composition** and **partial application**.

```typescript
// ❌ OOP Approach: Large interfaces
interface Employee {
  work(): void;
  code(): void;
  manage(): void;
  sell(): void;
}

// ✅ FP Approach: Composable function types
type Work = () => void;
type Code = (project: string) => void;
type Manage = (team: string[]) => void;
type Sell = (product: string) => void;

// Compose capabilities as needed
type Developer = {
  work: Work;
  code: Code;
};

type Manager = {
  work: Work;
  manage: Manage;
};

type SalesRep = {
  work: Work;
  sell: Sell;
};

// Factory functions for creating roles
const createDeveloper = (name: string): Developer => ({
  work: () => console.log(`${name} is working`),
  code: (project) => console.log(`${name} is coding ${project}`)
});

const createManager = (name: string): Manager => ({
  work: () => console.log(`${name} is working`),
  manage: (team) => console.log(`${name} is managing ${team.join(', ')}`)
});

// Using partial application for shared behavior
const createWorker = (name: string) => ({
  work: () => console.log(`${name} is working`)
});

const withCoding = (worker: { work: Work }, name: string) => ({
  ...worker,
  code: (project: string) => console.log(`${name} is coding ${project}`)
});

const withManaging = (worker: { work: Work }, name: string) => ({
  ...worker,
  manage: (team: string[]) => console.log(`${name} is managing ${team}`)
});
```

### 1.5 Dependency Inversion Principle (DIP) - Functional Approach

In FP, we achieve DIP through **higher-order functions** and **dependency injection via function parameters**.

```typescript
// ❌ OOP Approach: Classes with injected dependencies
class OrderService {
  constructor(private db: Database, private email: EmailService) {}
}

// ✅ FP Approach: Dependencies as function parameters
// Define effect types
type DatabaseOps = {
  save: <T>(table: string, data: T) => Promise<void>;
  find: <T>(table: string, id: string) => Promise<T | null>;
};

type EmailOps = {
  send: (to: string, subject: string, body: string) => Promise<void>;
};

type PaymentOps = {
  charge: (amount: number, method: string) => Promise<boolean>;
};

// Pure business logic with injected dependencies
const createOrder = (deps: {
  db: DatabaseOps;
  email: EmailOps;
  payment: PaymentOps;
}) => async (orderData: OrderData): Promise<Order> => {
  // Business logic using injected dependencies
  const order = {
    id: generateId(),
    items: orderData.items,
    total: calculateTotal(orderData.items)
  };
  
  const paid = await deps.payment.charge(order.total, orderData.paymentMethod);
  if (!paid) throw new Error('Payment failed');
  
  await deps.db.save('orders', order);
  await deps.email.send(
    orderData.customerEmail,
    'Order Confirmation',
    `Order ${order.id} confirmed`
  );
  
  return order;
};

// Different implementations for different environments
const prodDeps = {
  db: mysqlOps,
  email: sendgridOps,
  payment: stripeOps
};

const testDeps = {
  db: inMemoryOps,
  email: mockEmailOps,
  payment: mockPaymentOps
};

// Usage
const processOrder = createOrder(prodDeps);
const testProcessOrder = createOrder(testDeps);
```

---

## 2. Design Patterns in Functional Programming

### 2.1 Factory Pattern - Functional Approach

```typescript
// ✅ FP Factory: Functions that return functions
type NotificationType = 'email' | 'sms' | 'push';
type Notifier = (message: string, recipient: string) => Promise<void>;

const notifierFactory = (type: NotificationType): Notifier => {
  const notifiers: Record<NotificationType, Notifier> = {
    email: async (message, recipient) => 
      console.log(`Email to ${recipient}: ${message}`),
    sms: async (message, recipient) => 
      console.log(`SMS to ${recipient}: ${message}`),
    push: async (message, recipient) => 
      console.log(`Push to ${recipient}: ${message}`)
  };
  
  return notifiers[type] || notifiers.email;
};

// Using partial application
const createNotifier = (type: NotificationType) => 
  (recipient: string) => 
    (message: string) => 
      notifierFactory(type)(message, recipient);

const emailUser = createNotifier('email')('user@example.com');
await emailUser('Hello!');
```

### 2.2 Singleton Pattern - Functional Approach

```typescript
// ✅ FP Singleton: Module with closure
const ConfigManager = (() => {
  let config: Map<string, any> | null = null;
  
  const initialize = (): Map<string, any> => {
    if (!config) {
      config = new Map();
      config.set('api.url', 'https://api.example.com');
      config.set('api.timeout', 5000);
    }
    return config;
  };
  
  return {
    get: (key: string): any => {
      const cfg = initialize();
      return cfg.get(key);
    },
    set: (key: string, value: any): void => {
      const cfg = initialize();
      cfg.set(key, value);
    }
  };
})();

// Or using a lazy evaluation pattern
const lazyConfig = (() => {
  let instance: Config | null = null;
  return () => {
    if (!instance) {
      instance = loadConfig();
    }
    return instance;
  };
})();
```

### 2.3 Strategy Pattern - Functional Approach

```typescript
// ✅ FP Strategy: Functions as strategies
type SortStrategy<T> = (items: T[]) => T[];

const sortStrategies = {
  alphabetical: <T extends { name: string }>(items: T[]): T[] =>
    [...items].sort((a, b) => a.name.localeCompare(b.name)),
    
  numerical: <T extends { value: number }>(items: T[]): T[] =>
    [...items].sort((a, b) => a.value - b.value),
    
  byDate: <T extends { date: Date }>(items: T[]): T[] =>
    [...items].sort((a, b) => a.date.getTime() - b.date.getTime())
};

// Higher-order function to apply strategy
const sortWith = <T>(strategy: SortStrategy<T>) => (items: T[]): T[] =>
  strategy(items);

// Usage
const items = [{ name: 'Z' }, { name: 'A' }, { name: 'M' }];
const sorted = sortWith(sortStrategies.alphabetical)(items);
```

### 2.4 Observer Pattern - Functional Approach

```typescript
// ✅ FP Observer: Event emitter with functional approach
type Listener<T> = (data: T) => void;
type Unsubscribe = () => void;

const createEventEmitter = <T>() => {
  let listeners: Listener<T>[] = [];
  
  return {
    emit: (data: T): void => {
      listeners.forEach(listener => listener(data));
    },
    
    subscribe: (listener: Listener<T>): Unsubscribe => {
      listeners = [...listeners, listener];
      return () => {
        listeners = listeners.filter(l => l !== listener);
      };
    }
  };
};

// Usage with composition
const priceEmitter = createEventEmitter<number>();

const unsubscribe1 = priceEmitter.subscribe(
  price => console.log(`Logger: Price is ${price}`)
);

const unsubscribe2 = priceEmitter.subscribe(
  price => price > 100 && console.log('Alert: High price!')
);

priceEmitter.emit(150);
```

---

## 3. Code Smells - Functional Solutions

### 3.1 Long Method - Functional Refactoring

```typescript
// ❌ Long imperative function
function processOrder(orderData: any): void {
  // 100+ lines of mixed concerns
}

// ✅ FP Approach: Function composition
const processOrder = pipe(
  validateOrderData,
  checkInventory,
  calculatePricing,
  processPayment,
  createOrder,
  saveOrder,
  sendNotifications
);

// Each function is small and focused
const validateOrderData = (data: OrderData): Either<Error, OrderData> =>
  data.items.length > 0 
    ? right(data)
    : left(new Error('No items'));

const calculatePricing = (data: ValidatedOrder): PricedOrder => ({
  ...data,
  subtotal: data.items.reduce((sum, item) => sum + item.price * item.quantity, 0),
  tax: calculateTax(data),
  shipping: calculateShipping(data),
  total: calculateTotal(data)
});

// Using Result/Either monad for error handling
const processOrderSafely = (data: OrderData): TaskEither<Error, Order> =>
  pipe(
    data,
    validateOrderData,
    chain(checkInventory),
    map(calculatePricing),
    chain(processPayment),
    chain(saveOrder)
  );
```

### 3.2 Primitive Obsession - Functional Solution

```typescript
// ❌ Using primitives everywhere
function sendEmail(to: string, from: string, subject: string): void {}

// ✅ FP Approach: Branded types and validation
type Email = string & { readonly __brand: unique symbol };
type NonEmptyString = string & { readonly __brand: unique symbol };

const createEmail = (value: string): Option<Email> =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)
    ? some(value as Email)
    : none;

const createNonEmptyString = (value: string): Option<NonEmptyString> =>
  value.trim().length > 0
    ? some(value as NonEmptyString)
    : none;

// Function with validated types
const sendEmail = (to: Email, from: Email, subject: NonEmptyString): IO<void> =>
  () => console.log(`Sending email from ${from} to ${to}: ${subject}`);

// Usage requires validation
pipe(
  sequenceT(option)(
    createEmail('user@example.com'),
    createEmail('noreply@example.com'),
    createNonEmptyString('Hello')
  ),
  map(([to, from, subject]) => sendEmail(to, from, subject)())
);
```

### 3.3 Feature Envy - Functional Solution

```typescript
// ❌ Method that uses another object's data extensively
class Order {
  calculateCustomerDiscount(customer: Customer): number {
    if (customer.loyaltyPoints > 1000 && customer.yearsSince > 2) {
      return customer.totalPurchases * 0.1;
    }
    return 0;
  }
}

// ✅ FP Approach: Functions operate on their natural data
const calculateCustomerDiscount = (customer: Customer): number => {
  const isLoyal = customer.loyaltyPoints > 1000 && customer.yearsSince > 2;
  return isLoyal ? customer.totalPurchases * 0.1 : 0;
};

// Even better: Make it configurable
const createDiscountCalculator = (config: {
  minPoints: number;
  minYears: number;
  discountRate: number;
}) => (customer: Customer): number => {
  const isEligible = 
    customer.loyaltyPoints > config.minPoints && 
    customer.yearsSince > config.minYears;
  return isEligible ? customer.totalPurchases * config.discountRate : 0;
};

const calculateLoyaltyDiscount = createDiscountCalculator({
  minPoints: 1000,
  minYears: 2,
  discountRate: 0.1
});
```

---

## 4. Functional Refactoring Techniques

### 4.1 Replace Loop with Pipeline

```typescript
// ❌ Imperative loops
function processItems(items: Item[]): ProcessedItem[] {
  const result = [];
  for (const item of items) {
    if (item.isActive) {
      const processed = transformItem(item);
      if (processed.value > 100) {
        result.push(processed);
      }
    }
  }
  return result;
}

// ✅ FP Pipeline
const processItems = (items: Item[]): ProcessedItem[] =>
  pipe(
    items,
    filter(item => item.isActive),
    map(transformItem),
    filter(processed => processed.value > 100)
  );

// Or using array methods
const processItems2 = (items: Item[]): ProcessedItem[] =>
  items
    .filter(item => item.isActive)
    .map(transformItem)
    .filter(processed => processed.value > 100);
```

### 4.2 Replace Conditional with Function Map

```typescript
// ❌ Multiple if-else or switch
function getProcessor(type: string): Processor {
  if (type === 'json') return new JsonProcessor();
  if (type === 'xml') return new XmlProcessor();
  if (type === 'csv') return new CsvProcessor();
  throw new Error('Unknown type');
}

// ✅ FP Approach
const processors: Record<string, (data: string) => any> = {
  json: (data) => JSON.parse(data),
  xml: (data) => parseXml(data),
  csv: (data) => parseCsv(data)
};

const getProcessor = (type: string) => 
  processors[type] || processors.json;

// With better type safety
type ProcessorType = 'json' | 'xml' | 'csv';
const typedProcessors: Record<ProcessorType, (data: string) => any> = {
  json: JSON.parse,
  xml: parseXml,
  csv: parseCsv
};
```

### 4.3 Replace Mutation with Transformation

```typescript
// ❌ Mutating data
function addTax(order: Order): void {
  order.tax = order.subtotal * 0.1;
  order.total = order.subtotal + order.tax;
}

// ✅ FP Transformation
const addTax = (order: Order): Order => {
  const tax = order.subtotal * 0.1;
  return {
    ...order,
    tax,
    total: order.subtotal + tax
  };
};

// Using lenses for deep updates
import { Lens } from 'monocle-ts';

const taxLens = Lens.fromProp<Order>()('tax');
const totalLens = Lens.fromProp<Order>()('total');

const addTaxWithLens = (order: Order): Order => {
  const tax = order.subtotal * 0.1;
  return pipe(
    order,
    taxLens.set(tax),
    totalLens.set(order.subtotal + tax)
  );
};
```

---

## 5. Functional Testing Patterns

### 5.1 Property-Based Testing

```typescript
import * as fc from 'fast-check';

// Test mathematical properties
describe('Functional calculations', () => {
  it('should maintain associativity', () => {
    fc.assert(
      fc.property(
        fc.array(fc.integer()),
        (numbers) => {
          const sum1 = numbers.reduce((a, b) => a + b, 0);
          const sum2 = numbers.reduceRight((a, b) => a + b, 0);
          return sum1 === sum2;
        }
      )
    );
  });
});

// Test pure functions
const add = (a: number, b: number): number => a + b;

describe('Pure function properties', () => {
  it('should be commutative', () => {
    fc.assert(
      fc.property(
        fc.integer(),
        fc.integer(),
        (a, b) => add(a, b) === add(b, a)
      )
    );
  });
});
```

### 5.2 Testing with Dependency Injection

```typescript
// Pure function with injected dependencies
const createUserService = (deps: {
  validateUser: (user: User) => boolean;
  saveUser: (user: User) => Promise<void>;
  sendEmail: (email: string) => Promise<void>;
}) => async (userData: UserData): Promise<User> => {
  const user = createUser(userData);
  
  if (!deps.validateUser(user)) {
    throw new Error('Invalid user');
  }
  
  await deps.saveUser(user);
  await deps.sendEmail(user.email);
  
  return user;
};

// Easy to test with mocks
describe('User Service', () => {
  it('should create user with valid data', async () => {
    const mockDeps = {
      validateUser: jest.fn().mockReturnValue(true),
      saveUser: jest.fn().mockResolvedValue(undefined),
      sendEmail: jest.fn().mockResolvedValue(undefined)
    };
    
    const userService = createUserService(mockDeps);
    const user = await userService({ name: 'John', email: 'john@example.com' });
    
    expect(mockDeps.validateUser).toHaveBeenCalledWith(expect.objectContaining({
      name: 'John',
      email: 'john@example.com'
    }));
  });
});
```

---

## 6. Advanced Functional Patterns

### 6.1 Monads for Error Handling

```typescript
import { Either, left, right, chain, map } from 'fp-ts/Either';
import { pipe } from 'fp-ts/function';

type ValidationError = string[];
type ValidatedData<T> = Either<ValidationError, T>;

const validateAge = (age: number): ValidatedData<number> =>
  age >= 18 ? right(age) : left(['Age must be at least 18']);

const validateEmail = (email: string): ValidatedData<string> =>
  email.includes('@') ? right(email) : left(['Invalid email']);

const createValidatedUser = (
  name: string,
  email: string,
  age: number
): ValidatedData<User> =>
  pipe(
    sequenceT(either)(
      right(name),
      validateEmail(email),
      validateAge(age)
    ),
    map(([name, email, age]) => ({ name, email, age }))
  );
```

### 6.2 Functors and Applicatives

```typescript
import { Option, some, none, map as mapOption } from 'fp-ts/Option';
import { Task } from 'fp-ts/Task';
import { array } from 'fp-ts/Array';

// Functor example: Transform values inside containers
const doubleIfPositive = (n: number): Option<number> =>
  n > 0 ? some(n * 2) : none;

const result = pipe(
  some(5),
  chain(doubleIfPositive),
  mapOption(n => n + 1)
);

// Applicative: Apply functions inside containers
const addOption = (a: Option<number>) => (b: Option<number>): Option<number> =>
  pipe(
    sequenceT(option)(a, b),
    mapOption(([x, y]) => x + y)
  );

// Traverse: Transform and sequence
const fetchUser = (id: string): Task<User> => async () => 
  fetch(`/api/users/${id}`).then(r => r.json());

const fetchUsers = (ids: string[]): Task<User[]> =>
  array.sequence(task)(ids.map(fetchUser));
```

### 6.3 Kleisli Composition

```typescript
// Composing functions that return monads
type KleisliArrow<A, B> = (a: A) => Task<B>;

const fetchUserById: KleisliArrow<string, User> = 
  (id) => () => fetch(`/api/users/${id}`).then(r => r.json());

const fetchOrdersByUser: KleisliArrow<User, Order[]> = 
  (user) => () => fetch(`/api/orders?userId=${user.id}`).then(r => r.json());

const fetchOrderItems: KleisliArrow<Order[], OrderItem[]> = 
  (orders) => () => Promise.all(
    orders.map(o => fetch(`/api/items?orderId=${o.id}`).then(r => r.json()))
  ).then(results => results.flat());

// Compose them
const fetchUserOrderItems = flow(
  fetchUserById,
  chain(fetchOrdersByUser),
  chain(fetchOrderItems)
);
```

---

## 7. Performance Optimization with FP

### 7.1 Memoization

```typescript
const memoize = <Args extends unknown[], Return>(
  fn: (...args: Args) => Return
): ((...args: Args) => Return) => {
  const cache = new Map<string, Return>();
  
  return (...args: Args): Return => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
};

// Usage
const expensiveCalculation = memoize((n: number): number => {
  console.log(`Calculating for ${n}`);
  return n * n;
});
```

### 7.2 Lazy Evaluation

```typescript
// Lazy sequences
class LazySeq<T> {
  constructor(private generator: () => IterableIterator<T>) {}
  
  map<U>(fn: (value: T) => U): LazySeq<U> {
    const gen = this.generator;
    return new LazySeq(function* () {
      for (const value of gen()) {
        yield fn(value);
      }
    });
  }
  
  filter(predicate: (value: T) => boolean): LazySeq<T> {
    const gen = this.generator;
    return new LazySeq(function* () {
      for (const value of gen()) {
        if (predicate(value)) yield value;
      }
    });
  }
  
  take(n: number): T[] {
    const result: T[] = [];
    const iterator = this.generator();
    
    for (let i = 0; i < n; i++) {
      const { value, done } = iterator.next();
      if (done) break;
      result.push(value);
    }
    
    return result;
  }
}

// Usage
const infiniteNumbers = new LazySeq(function* () {
  let i = 0;
  while (true) yield i++;
});

const result = infiniteNumbers
  .map(x => x * 2)
  .filter(x => x % 3 === 0)
  .take(5); // [0, 6, 12, 18, 24]
```

### 7.3 Transducers

```typescript
// Composable transformations
type Transducer<A, B> = <R>(reducer: (acc: R, value: B) => R) => (acc: R, value: A) => R;

const map = <A, B>(fn: (a: A) => B): Transducer<A, B> =>
  reducer => (acc, value) => reducer(acc, fn(value));

const filter = <A>(predicate: (a: A) => boolean): Transducer<A, A> =>
  reducer => (acc, value) => predicate(value) ? reducer(acc, value) : acc;

const compose = <A, B, C>(
  t1: Transducer<B, C>,
  t2: Transducer<A, B>
): Transducer<A, C> =>
  reducer => t2(t1(reducer));

// Usage
const transformation = compose(
  filter<number>(x => x % 2 === 0),
  map<number, number>(x => x * 2)
);

const result = [1, 2, 3, 4, 5].reduce(
  transformation((acc, val) => [...acc, val]),
  [] as number[]
); // [4, 8]
```

---

## Summary

Functional Programming in TypeScript offers powerful alternatives to OOP patterns:

1. **Pure Functions** instead of methods with side effects
2. **Function Composition** instead of inheritance
3. **Immutability** instead of mutation
4. **Higher-Order Functions** instead of design patterns
5. **Algebraic Data Types** instead of class hierarchies
6. **Monads** for error handling instead of exceptions
7. **Lazy Evaluation** for performance optimization

The key advantages of FP approach:
- **Predictability**: Pure functions always return the same output for the same input
- **Testability**: No hidden state or dependencies make testing easier
- **Composability**: Small functions can be combined to create complex behavior
- **Concurrency**: Immutability eliminates race conditions
- **Debugging**: Data flow is explicit and traceable