# TypeScript Classes - Complete Guide
## From Functions to Classes: Everything You Need to Know

---

## Introduction

If you're coming from a functional programming background, classes might seem foreign. This guide will teach you everything about TypeScript classes, comparing them with functions along the way.

### What is a Class?

A class is a blueprint for creating objects. It encapsulates data (properties) and behavior (methods) together. Think of it as a template that defines:
- **What data** an object holds (properties)
- **What actions** it can perform (methods)
- **How to create** new instances (constructor)

---

## Part 1: Basic Class Syntax

### Your First Class

```typescript
// Function approach (what you're familiar with)
function createUser(name: string, age: number) {
  return {
    name,
    age,
    greet: () => `Hello, I'm ${name}`
  };
}

// Class approach (equivalent)
class User {
  name: string;
  age: number;
  
  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
  
  greet(): string {
    return `Hello, I'm ${this.name}`;
  }
}

// Usage comparison
const userFunc = createUser('Alice', 30);
const userClass = new User('Alice', 30);

console.log(userFunc.greet());  // "Hello, I'm Alice"
console.log(userClass.greet()); // "Hello, I'm Alice"
```

### Class Anatomy

```typescript
class ClassName {
  // 1. Property declarations
  propertyName: type;
  
  // 2. Constructor (runs when creating new instance)
  constructor(parameters) {
    // Initialize properties
  }
  
  // 3. Methods (functions that belong to the class)
  methodName(): returnType {
    // Method implementation
  }
}
```

### Properties: Different Ways to Declare

```typescript
// Method 1: Declare then initialize in constructor
class Person1 {
  name: string;
  age: number;
  
  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}

// Method 2: Initialize directly
class Person2 {
  name: string = 'Unknown';
  age: number = 0;
  
  constructor(name?: string, age?: number) {
    if (name) this.name = name;
    if (age) this.age = age;
  }
}

// Method 3: Parameter properties (TypeScript shorthand)
class Person3 {
  constructor(
    public name: string,
    public age: number
  ) {
    // Properties are automatically created and initialized!
  }
}

// All three create the same structure
const p1 = new Person1('Bob', 25);
const p2 = new Person2('Bob', 25);
const p3 = new Person3('Bob', 25);
```

---

## Part 2: Access Modifiers

### Public, Private, and Protected

```typescript
class BankAccount {
  public accountHolder: string;      // Accessible everywhere (default)
  private balance: number;          // Only accessible inside this class
  protected accountType: string;    // Accessible in this class and subclasses
  
  constructor(holder: string, initialBalance: number) {
    this.accountHolder = holder;
    this.balance = initialBalance;
    this.accountType = 'standard';
  }
  
  // Public method (accessible everywhere)
  public deposit(amount: number): void {
    this.balance += amount;
  }
  
  // Private method (only callable inside this class)
  private validateAmount(amount: number): boolean {
    return amount > 0 && amount <= this.balance;
  }
  
  // Protected method (callable in this class and subclasses)
  protected calculateInterest(): number {
    return this.balance * 0.02;
  }
  
  // Public method using private method
  public withdraw(amount: number): boolean {
    if (this.validateAmount(amount)) {
      this.balance -= amount;
      return true;
    }
    return false;
  }
  
  // Getter to safely access private property
  public getBalance(): number {
    return this.balance;
  }
}

const account = new BankAccount('Alice', 1000);
account.deposit(500);              // ✅ OK - public method
console.log(account.accountHolder); // ✅ OK - public property
// account.balance;                // ❌ Error - private property
// account.validateAmount(100);    // ❌ Error - private method
console.log(account.getBalance()); // ✅ OK - public method
```

### Readonly Properties

```typescript
class Product {
  readonly id: string;
  readonly createdAt: Date;
  name: string;
  price: number;
  
  constructor(name: string, price: number) {
    this.id = `prod_${Date.now()}`;  // Can set in constructor
    this.createdAt = new Date();     // Can set in constructor
    this.name = name;
    this.price = price;
  }
  
  updatePrice(newPrice: number): void {
    this.price = newPrice;            // ✅ OK
    // this.id = 'new_id';           // ❌ Error - readonly
  }
}

const product = new Product('Laptop', 999);
product.price = 899;                  // ✅ OK
// product.id = 'custom_id';          // ❌ Error - readonly
```

---

## Part 3: Getters and Setters

### Computed Properties with Get/Set

```typescript
class Rectangle {
  constructor(
    private _width: number,
    private _height: number
  ) {}
  
  // Getter - accessed like a property, computed on the fly
  get area(): number {
    return this._width * this._height;
  }
  
  get perimeter(): number {
    return 2 * (this._width + this._height);
  }
  
  // Setters with validation
  set width(value: number) {
    if (value <= 0) {
      throw new Error('Width must be positive');
    }
    this._width = value;
  }
  
  get width(): number {
    return this._width;
  }
  
  set height(value: number) {
    if (value <= 0) {
      throw new Error('Height must be positive');
    }
    this._height = value;
  }
  
  get height(): number {
    return this._height;
  }
}

const rect = new Rectangle(10, 5);
console.log(rect.area);        // 50 - called like a property!
console.log(rect.perimeter);   // 30

rect.width = 20;               // Uses setter
console.log(rect.area);        // 100 - automatically updated

// rect.width = -5;            // Error: Width must be positive
```

### Real-World Example: User Class with Validation

```typescript
class User {
  private _email: string = '';
  private _age: number = 0;
  
  constructor(
    public firstName: string,
    public lastName: string
  ) {}
  
  // Computed property
  get fullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }
  
  // Allow setting full name and split it
  set fullName(value: string) {
    const parts = value.split(' ');
    this.firstName = parts[0] || '';
    this.lastName = parts[1] || '';
  }
  
  // Email with validation
  get email(): string {
    return this._email;
  }
  
  set email(value: string) {
    if (!value.includes('@')) {
      throw new Error('Invalid email');
    }
    this._email = value;
  }
  
  // Age with validation
  get age(): number {
    return this._age;
  }
  
  set age(value: number) {
    if (value < 0 || value > 150) {
      throw new Error('Invalid age');
    }
    this._age = value;
  }
  
  // Computed property based on age
  get isAdult(): boolean {
    return this._age >= 18;
  }
}

const user = new User('John', 'Doe');
console.log(user.fullName);     // "John Doe"

user.fullName = 'Jane Smith';   // Setter splits the name
console.log(user.firstName);    // "Jane"
console.log(user.lastName);     // "Smith"

user.email = 'jane@example.com';
user.age = 25;
console.log(user.isAdult);      // true
```

---

## Part 4: Methods and 'this' Context

### Instance Methods vs Arrow Functions

```typescript
class Counter {
  count: number = 0;
  
  // Regular method - 'this' depends on how it's called
  increment(): void {
    this.count++;
  }
  
  // Arrow function - 'this' is always the instance
  decrement = (): void => {
    this.count--;
  }
  
  // Method that returns a function
  getIncrementer(): () => void {
    return () => {
      this.count++;  // 'this' is captured from the method
    };
  }
}

const counter = new Counter();

// Regular method
const inc = counter.increment;
// inc();  // ❌ Error: 'this' is undefined
counter.increment();  // ✅ OK

// Arrow function property
const dec = counter.decrement;
dec();  // ✅ OK - 'this' is preserved

// Using bind to fix 'this'
const boundInc = counter.increment.bind(counter);
boundInc();  // ✅ OK
```

### Method Chaining

```typescript
class Calculator {
  private value: number = 0;
  
  add(n: number): this {
    this.value += n;
    return this;  // Return 'this' for chaining
  }
  
  subtract(n: number): this {
    this.value -= n;
    return this;
  }
  
  multiply(n: number): this {
    this.value *= n;
    return this;
  }
  
  divide(n: number): this {
    if (n !== 0) {
      this.value /= n;
    }
    return this;
  }
  
  getResult(): number {
    return this.value;
  }
  
  reset(): this {
    this.value = 0;
    return this;
  }
}

const calc = new Calculator();
const result = calc
  .add(10)
  .multiply(2)
  .subtract(5)
  .divide(3)
  .getResult();  // 5

console.log(result);
```

---

## Part 5: Static Members

### Static Properties and Methods

```typescript
class MathUtils {
  // Static property
  static readonly PI: number = 3.14159;
  static readonly E: number = 2.71828;
  
  // Static method - called on the class, not instances
  static round(value: number, decimals: number = 0): number {
    const factor = Math.pow(10, decimals);
    return Math.round(value * factor) / factor;
  }
  
  static random(min: number, max: number): number {
    return Math.random() * (max - min) + min;
  }
  
  // Static factory method
  static createFromDegrees(degrees: number): Angle {
    return new Angle(degrees * this.PI / 180);
  }
}

// Using static members - no need to create instance
console.log(MathUtils.PI);           // 3.14159
console.log(MathUtils.round(3.14159, 2)); // 3.14
console.log(MathUtils.random(1, 10));     // Random number

class Angle {
  constructor(public radians: number) {}
}

// Static method as factory
const angle = MathUtils.createFromDegrees(90);
```

### Singleton Pattern with Static

```typescript
class Database {
  private static instance: Database;
  private connection: any;
  
  // Private constructor prevents direct instantiation
  private constructor() {
    this.connection = this.connect();
  }
  
  // Static method to get the single instance
  static getInstance(): Database {
    if (!Database.instance) {
      Database.instance = new Database();
    }
    return Database.instance;
  }
  
  private connect(): any {
    console.log('Connecting to database...');
    return { connected: true };
  }
  
  query(sql: string): void {
    console.log(`Executing: ${sql}`);
  }
}

// Always get the same instance
const db1 = Database.getInstance();
const db2 = Database.getInstance();
console.log(db1 === db2);  // true - same instance

db1.query('SELECT * FROM users');
```

---

## Part 6: Inheritance

### Basic Inheritance with 'extends'

```typescript
// Base class (parent/superclass)
class Animal {
  constructor(
    public name: string,
    protected age: number
  ) {}
  
  move(distance: number): void {
    console.log(`${this.name} moved ${distance} meters`);
  }
  
  makeSound(): void {
    console.log('Some generic sound');
  }
}

// Derived class (child/subclass)
class Dog extends Animal {
  constructor(
    name: string,
    age: number,
    public breed: string
  ) {
    super(name, age);  // Call parent constructor
  }
  
  // Override parent method
  makeSound(): void {
    console.log('Woof! Woof!');
  }
  
  // Add new method
  wagTail(): void {
    console.log(`${this.name} is wagging tail`);
  }
  
  // Access protected member from parent
  getAge(): number {
    return this.age;  // Can access protected member
  }
}

class Cat extends Animal {
  constructor(name: string, age: number) {
    super(name, age);
  }
  
  makeSound(): void {
    console.log('Meow!');
  }
  
  purr(): void {
    console.log(`${this.name} is purring`);
  }
}

// Usage
const dog = new Dog('Buddy', 3, 'Golden Retriever');
dog.move(10);        // Inherited method
dog.makeSound();     // Overridden method - "Woof! Woof!"
dog.wagTail();       // Dog-specific method

const cat = new Cat('Whiskers', 2);
cat.makeSound();     // "Meow!"
cat.purr();          // Cat-specific method
```

### Method Overriding and Super Calls

```typescript
class Employee {
  constructor(
    public name: string,
    public department: string,
    protected salary: number
  ) {}
  
  getDetails(): string {
    return `${this.name} works in ${this.department}`;
  }
  
  calculateBonus(): number {
    return this.salary * 0.1;  // 10% bonus
  }
}

class Manager extends Employee {
  constructor(
    name: string,
    department: string,
    salary: number,
    public teamSize: number
  ) {
    super(name, department, salary);
  }
  
  // Override with super call
  getDetails(): string {
    const basicDetails = super.getDetails();  // Call parent method
    return `${basicDetails} and manages ${this.teamSize} people`;
  }
  
  // Override with different logic
  calculateBonus(): number {
    const baseBonus = super.calculateBonus();
    const teamBonus = this.teamSize * 1000;
    return baseBonus + teamBonus;
  }
}

const manager = new Manager('Alice', 'Engineering', 100000, 5);
console.log(manager.getDetails());
// "Alice works in Engineering and manages 5 people"
console.log(manager.calculateBonus());  // 10000 + 5000 = 15000
```

---

## Part 7: Abstract Classes

### Abstract Classes and Methods

```typescript
// Abstract class - cannot be instantiated directly
abstract class Shape {
  constructor(public color: string) {}
  
  // Abstract method - must be implemented by subclasses
  abstract getArea(): number;
  abstract getPerimeter(): number;
  
  // Concrete method - inherited by all subclasses
  describe(): string {
    return `A ${this.color} shape with area ${this.getArea()}`;
  }
}

// Concrete implementations
class Circle extends Shape {
  constructor(
    color: string,
    public radius: number
  ) {
    super(color);
  }
  
  getArea(): number {
    return Math.PI * this.radius ** 2;
  }
  
  getPerimeter(): number {
    return 2 * Math.PI * this.radius;
  }
}

class Square extends Shape {
  constructor(
    color: string,
    public side: number
  ) {
    super(color);
  }
  
  getArea(): number {
    return this.side ** 2;
  }
  
  getPerimeter(): number {
    return 4 * this.side;
  }
}

// const shape = new Shape('red');  // ❌ Error - cannot instantiate abstract class
const circle = new Circle('blue', 5);
const square = new Square('red', 10);

console.log(circle.describe());  // "A blue shape with area 78.54"
console.log(square.describe());  // "A red shape with area 100"
```

### Abstract Class as Template

```typescript
abstract class DataProcessor<T> {
  protected data: T[] = [];
  
  // Template method pattern
  process(input: T[]): void {
    this.data = input;
    this.validate();
    this.transform();
    this.save();
  }
  
  // Abstract methods that subclasses must implement
  protected abstract validate(): void;
  protected abstract transform(): void;
  
  // Concrete method with default implementation
  protected save(): void {
    console.log('Saving processed data...');
    console.log(this.data);
  }
}

class NumberProcessor extends DataProcessor<number> {
  protected validate(): void {
    this.data = this.data.filter(n => !isNaN(n) && isFinite(n));
    console.log('Validated numbers');
  }
  
  protected transform(): void {
    this.data = this.data.map(n => Math.round(n * 100) / 100);
    console.log('Rounded to 2 decimal places');
  }
}

class StringProcessor extends DataProcessor<string> {
  protected validate(): void {
    this.data = this.data.filter(s => s.length > 0);
    console.log('Removed empty strings');
  }
  
  protected transform(): void {
    this.data = this.data.map(s => s.trim().toLowerCase());
    console.log('Trimmed and lowercased');
  }
}

const numProcessor = new NumberProcessor();
numProcessor.process([1.234, 5.678, NaN, 9.012]);

const strProcessor = new StringProcessor();
strProcessor.process(['  Hello  ', '', 'WORLD  ']);
```

---

## Part 8: Interfaces with Classes

### Implementing Interfaces

```typescript
// Define contracts with interfaces
interface Drivable {
  speed: number;
  drive(): void;
}

interface Flyable {
  altitude: number;
  fly(): void;
}

// Class implementing single interface
class Car implements Drivable {
  speed: number = 0;
  
  drive(): void {
    this.speed = 60;
    console.log(`Driving at ${this.speed} km/h`);
  }
}

// Class implementing multiple interfaces
class FlyingCar implements Drivable, Flyable {
  speed: number = 0;
  altitude: number = 0;
  
  drive(): void {
    this.speed = 100;
    console.log(`Driving at ${this.speed} km/h`);
  }
  
  fly(): void {
    this.altitude = 1000;
    console.log(`Flying at ${this.altitude} meters`);
  }
  
  land(): void {
    this.altitude = 0;
    console.log('Landed safely');
  }
}

// Using interface as type
function race(vehicle: Drivable): void {
  vehicle.drive();
  console.log(`Racing at ${vehicle.speed} km/h`);
}

const car = new Car();
const flyingCar = new FlyingCar();

race(car);        // Works with Car
race(flyingCar);  // Also works with FlyingCar
```

### Interface Extending and Class Hierarchies

```typescript
interface Person {
  name: string;
  age: number;
}

interface Employee extends Person {
  employeeId: string;
  department: string;
}

interface Manager extends Employee {
  teamMembers: string[];
}

class CompanyEmployee implements Employee {
  constructor(
    public name: string,
    public age: number,
    public employeeId: string,
    public department: string
  ) {}
  
  getInfo(): string {
    return `${this.name} (${this.employeeId}) - ${this.department}`;
  }
}

class CompanyManager extends CompanyEmployee implements Manager {
  teamMembers: string[] = [];
  
  addTeamMember(name: string): void {
    this.teamMembers.push(name);
  }
  
  getTeamSize(): number {
    return this.teamMembers.length;
  }
}
```

---

## Part 9: Generics in Classes

### Generic Classes

```typescript
// Generic class for any type T
class Container<T> {
  private items: T[] = [];
  
  add(item: T): void {
    this.items.push(item);
  }
  
  get(index: number): T | undefined {
    return this.items[index];
  }
  
  remove(item: T): boolean {
    const index = this.items.indexOf(item);
    if (index > -1) {
      this.items.splice(index, 1);
      return true;
    }
    return false;
  }
  
  getAll(): T[] {
    return [...this.items];
  }
}

// Using with different types
const numberContainer = new Container<number>();
numberContainer.add(1);
numberContainer.add(2);
// numberContainer.add('three');  // ❌ Error - must be number

const stringContainer = new Container<string>();
stringContainer.add('hello');
stringContainer.add('world');

interface User {
  id: number;
  name: string;
}

const userContainer = new Container<User>();
userContainer.add({ id: 1, name: 'Alice' });
userContainer.add({ id: 2, name: 'Bob' });
```

### Generic Constraints

```typescript
interface Identifiable {
  id: string | number;
}

// T must extend Identifiable
class Repository<T extends Identifiable> {
  private items = new Map<string | number, T>();
  
  save(item: T): void {
    this.items.set(item.id, item);  // Can access 'id' because of constraint
  }
  
  findById(id: string | number): T | undefined {
    return this.items.get(id);
  }
  
  update(id: string | number, updates: Partial<T>): void {
    const item = this.items.get(id);
    if (item) {
      this.items.set(id, { ...item, ...updates });
    }
  }
  
  delete(id: string | number): boolean {
    return this.items.delete(id);
  }
}

interface Product extends Identifiable {
  id: number;
  name: string;
  price: number;
}

const productRepo = new Repository<Product>();
productRepo.save({ id: 1, name: 'Laptop', price: 999 });

// const invalidRepo = new Repository<string>();  // ❌ Error - string doesn't have 'id'
```

---

## Part 10: Decorators (Experimental)

### Class and Method Decorators

```typescript
// Note: Decorators are experimental. Enable with "experimentalDecorators": true in tsconfig.json

// Class decorator
function Component(config: { selector: string }) {
  return function(constructor: Function) {
    constructor.prototype.selector = config.selector;
  };
}

// Method decorator
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${key} with args:`, args);
    const result = original.apply(this, args);
    console.log(`${key} returned:`, result);
    return result;
  };
  
  return descriptor;
}

// Property decorator
function Required(target: any, key: string) {
  let value = target[key];
  
  const getter = () => value;
  const setter = (newVal: any) => {
    if (!newVal) {
      throw new Error(`${key} is required`);
    }
    value = newVal;
  };
  
  Object.defineProperty(target, key, {
    get: getter,
    set: setter,
    enumerable: true,
    configurable: true
  });
}

@Component({ selector: 'app-user' })
class UserComponent {
  @Required
  name: string = '';
  
  @Log
  greet(greeting: string): string {
    return `${greeting}, ${this.name}!`;
  }
}

const component = new UserComponent();
component.name = 'Alice';
console.log(component.greet('Hello'));  // Logs method call and return
```

---

## Part 11: Practical Patterns and Use Cases

### Factory Pattern

```typescript
abstract class Vehicle {
  abstract drive(): void;
}

class Car extends Vehicle {
  drive(): void {
    console.log('Driving a car');
  }
}

class Truck extends Vehicle {
  drive(): void {
    console.log('Driving a truck');
  }
}

class Motorcycle extends Vehicle {
  drive(): void {
    console.log('Riding a motorcycle');
  }
}

class VehicleFactory {
  static create(type: 'car' | 'truck' | 'motorcycle'): Vehicle {
    switch(type) {
      case 'car':
        return new Car();
      case 'truck':
        return new Truck();
      case 'motorcycle':
        return new Motorcycle();
      default:
        throw new Error(`Unknown vehicle type: ${type}`);
    }
  }
}

// Usage
const vehicle = VehicleFactory.create('car');
vehicle.drive();  // "Driving a car"
```

### Builder Pattern

```typescript
class Pizza {
  size: string = 'medium';
  cheese: boolean = true;
  pepperoni: boolean = false;
  mushrooms: boolean = false;
  olives: boolean = false;
  
  describe(): string {
    const toppings = [];
    if (this.cheese) toppings.push('cheese');
    if (this.pepperoni) toppings.push('pepperoni');
    if (this.mushrooms) toppings.push('mushrooms');
    if (this.olives) toppings.push('olives');
    
    return `${this.size} pizza with ${toppings.join(', ')}`;
  }
}

class PizzaBuilder {
  private pizza: Pizza;
  
  constructor() {
    this.pizza = new Pizza();
  }
  
  setSize(size: 'small' | 'medium' | 'large'): this {
    this.pizza.size = size;
    return this;
  }
  
  addCheese(): this {
    this.pizza.cheese = true;
    return this;
  }
  
  addPepperoni(): this {
    this.pizza.pepperoni = true;
    return this;
  }
  
  addMushrooms(): this {
    this.pizza.mushrooms = true;
    return this;
  }
  
  addOlives(): this {
    this.pizza.olives = true;
    return this;
  }
  
  build(): Pizza {
    return this.pizza;
  }
}

// Usage with method chaining
const pizza = new PizzaBuilder()
  .setSize('large')
  .addCheese()
  .addPepperoni()
  .addMushrooms()
  .build();

console.log(pizza.describe());
// "large pizza with cheese, pepperoni, mushrooms"
```

### Observer Pattern

```typescript
interface Observer {
  update(data: any): void;
}

class Subject {
  private observers: Observer[] = [];
  
  attach(observer: Observer): void {
    this.observers.push(observer);
  }
  
  detach(observer: Observer): void {
    const index = this.observers.indexOf(observer);
    if (index > -1) {
      this.observers.splice(index, 1);
    }
  }
  
  notify(data: any): void {
    this.observers.forEach(observer => observer.update(data));
  }
}

class NewsAgency extends Subject {
  private news: string = '';
  
  setNews(news: string): void {
    this.news = news;
    this.notify(news);
  }
}

class NewsChannel implements Observer {
  constructor(private name: string) {}
  
  update(news: string): void {
    console.log(`${this.name} broadcasting: ${news}`);
  }
}

// Usage
const agency = new NewsAgency();
const cnn = new NewsChannel('CNN');
const bbc = new NewsChannel('BBC');

agency.attach(cnn);
agency.attach(bbc);

agency.setNews('Breaking: TypeScript classes are awesome!');
// CNN broadcasting: Breaking: TypeScript classes are awesome!
// BBC broadcasting: Breaking: TypeScript classes are awesome!
```

---

## Part 12: Classes vs Functions - When to Use Which

### Comparison Table

| Aspect | Classes | Functions |
|--------|---------|-----------|
| **State Management** | Encapsulated with data | Passed as arguments |
| **Inheritance** | Supported with `extends` | Composition/HOF |
| **Type Safety** | Strong with properties | Parameters/return types |
| **Testing** | Mock entire objects | Mock individual functions |
| **Memory** | One instance per object | Shared functions |
| **'this' context** | Bound to instance | Depends on call |

### When to Use Classes

```typescript
// ✅ Good use cases for classes:

// 1. When you need to maintain state
class ShoppingCart {
  private items: Item[] = [];
  
  addItem(item: Item): void {
    this.items.push(item);
  }
  
  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}

// 2. When you need inheritance hierarchies
abstract class Animal {
  abstract makeSound(): void;
}

class Dog extends Animal {
  makeSound(): void {
    console.log('Woof!');
  }
}

// 3. When implementing design patterns
class Singleton {
  private static instance: Singleton;
  
  private constructor() {}
  
  static getInstance(): Singleton {
    if (!this.instance) {
      this.instance = new Singleton();
    }
    return this.instance;
  }
}

// 4. When modeling domain objects
class User {
  constructor(
    private id: string,
    private email: string
  ) {}
  
  updateEmail(newEmail: string): void {
    // validation logic
    this.email = newEmail;
  }
}
```

### When to Use Functions

```typescript
// ✅ Good use cases for functions:

// 1. Pure transformations
const double = (n: number): number => n * 2;
const capitalize = (s: string): string => s.toUpperCase();

// 2. Utilities and helpers
function formatDate(date: Date): string {
  return date.toISOString().split('T')[0];
}

// 3. Functional composition
const pipe = (...fns: Function[]) => (value: any) =>
  fns.reduce((acc, fn) => fn(acc), value);

// 4. Simple factories
function createUser(name: string, age: number) {
  return { name, age, id: Date.now().toString() };
}
```

### Converting Between Styles

```typescript
// Function-based approach
function createCounter(initial: number = 0) {
  let count = initial;
  
  return {
    increment: () => ++count,
    decrement: () => --count,
    getValue: () => count,
    reset: () => count = initial
  };
}

// Equivalent class-based approach
class Counter {
  private count: number;
  private initial: number;
  
  constructor(initial: number = 0) {
    this.count = initial;
    this.initial = initial;
  }
  
  increment(): number {
    return ++this.count;
  }
  
  decrement(): number {
    return --this.count;
  }
  
  getValue(): number {
    return this.count;
  }
  
  reset(): void {
    this.count = this.initial;
  }
}

// Both work similarly
const funcCounter = createCounter(10);
const classCounter = new Counter(10);

console.log(funcCounter.increment());  // 11
console.log(classCounter.increment()); // 11
```

---

## Common Pitfalls and Best Practices

### Common Mistakes to Avoid

```typescript
// ❌ BAD: Forgetting to use 'new'
class Person {
  constructor(public name: string) {}
}
// const person = Person('Alice');  // Error! Must use 'new'
const person = new Person('Alice');  // ✅ Correct

// ❌ BAD: Mutating state directly from outside
class BadBankAccount {
  balance: number = 0;  // Public by default!
}
const badAccount = new BadBankAccount();
badAccount.balance = 1000000;  // Anyone can change it!

// ✅ GOOD: Use private and methods
class GoodBankAccount {
  private balance: number = 0;
  
  deposit(amount: number): void {
    if (amount > 0) {
      this.balance += amount;
    }
  }
  
  getBalance(): number {
    return this.balance;
  }
}

// ❌ BAD: Forgetting to call super() in constructor
class Child extends Parent {
  constructor() {
    // super();  // Must call super before accessing 'this'
    this.property = 'value';  // Error!
  }
}

// ✅ GOOD: Call super() first
class GoodChild extends Parent {
  constructor() {
    super();  // Call parent constructor first
    this.property = 'value';  // Now OK
  }
}

// ❌ BAD: Using arrow functions for methods that will be overridden
class Parent {
  method = () => {  // Arrow function
    console.log('Parent');
  }
}

class Child extends Parent {
  method = () => {  // This doesn't override, creates new property
    console.log('Child');
  }
}

// ✅ GOOD: Use regular methods for overridable behavior
class GoodParent {
  method(): void {
    console.log('Parent');
  }
}

class GoodChild extends GoodParent {
  method(): void {
    console.log('Child');
  }
}
```

### Best Practices

1. **Use access modifiers explicitly**
2. **Initialize properties in constructor or at declaration**
3. **Use readonly for immutable properties**
4. **Prefer composition over inheritance**
5. **Keep classes focused (Single Responsibility)**
6. **Use interfaces for contracts**
7. **Make methods small and focused**
8. **Use static for utility methods**
9. **Document complex classes**
10. **Test classes thoroughly**

---

## Summary

Classes in TypeScript provide:
- **Encapsulation**: Bundle data and methods together
- **Inheritance**: Reuse code through class hierarchies
- **Polymorphism**: Use different implementations through common interfaces
- **Type Safety**: Strong typing for properties and methods
- **Access Control**: Public, private, protected modifiers
- **Abstraction**: Abstract classes and interfaces

Choose classes when you need:
- Stateful objects
- Complex hierarchies
- Design patterns
- Domain modeling

Choose functions when you need:
- Pure transformations
- Simple utilities
- Functional composition
- Stateless operations

Both paradigms have their place in TypeScript, and you can mix them as needed for your application's requirements.

---

*This guide covered everything from basic class syntax to advanced patterns. Practice creating your own classes to solidify these concepts!*