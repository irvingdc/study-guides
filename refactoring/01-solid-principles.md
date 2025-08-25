# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 1: Principios SOLID
### Para Entrevista Senior Software Engineer

---

## 1. Principios SOLID - Fundamentos Esenciales

Los principios SOLID son cinco principios de diseño orientado a objetos que fueron introducidos por Robert C. Martin (Uncle Bob) a principios de los 2000s. Estos principios son fundamentales para escribir código mantenible, escalable y robusto. El acrónimo SOLID representa:

- **S**: Single Responsibility Principle (SRP)
- **O**: Open/Closed Principle (OCP)
- **L**: Liskov Substitution Principle (LSP)
- **I**: Interface Segregation Principle (ISP)
- **D**: Dependency Inversion Principle (DIP)

### ¿Por qué son importantes los principios SOLID?

1. **Reducen la complejidad**: Al dividir responsabilidades, el código se vuelve más fácil de entender
2. **Facilitan el testing**: Clases con una sola responsabilidad son más fáciles de testear
3. **Mejoran la mantenibilidad**: Los cambios afectan menos partes del sistema
4. **Promueven la reutilización**: Componentes bien diseñados pueden reutilizarse en diferentes contextos
5. **Reducen el acoplamiento**: Las dependencias entre módulos son más claras y manejables

---

## 1.1 Single Responsibility Principle (SRP)

### Explicación Detallada

El Principio de Responsabilidad Única establece que **una clase debe tener una, y solo una, razón para cambiar**. Esto significa que una clase debe tener una única responsabilidad o propósito en el sistema.

#### ¿Qué es una "razón para cambiar"?
Una razón para cambiar es un aspecto del negocio o requerimiento que podría evolucionar. Por ejemplo:
- Cambios en la lógica de negocio
- Cambios en la forma de persistir datos
- Cambios en la forma de presentar información
- Cambios en reglas de validación
- Cambios en mecanismos de notificación

#### Señales de violación del SRP:
- La clase tiene múltiples razones para ser modificada
- La clase tiene dependencias de muchas otras clases
- Es difícil nombrar la clase con un nombre conciso
- La clase tiene métodos que operan en diferentes niveles de abstracción
- Los tests de la clase requieren muchos mocks diferentes

#### Beneficios de aplicar SRP:
- **Cohesión alta**: Todo el código relacionado está junto
- **Testing más simple**: Solo necesitas testear una responsabilidad
- **Cambios localizados**: Los cambios afectan solo una clase
- **Reutilización mejorada**: Clases especializadas son más reutilizables
- **Nombres más claros**: Es fácil nombrar clases con una sola responsabilidad

### Ejemplo Práctico con Explicación

```typescript
// ❌ PROBLEMA: Clase con múltiples responsabilidades
class User {
  constructor(
    private name: string, 
    private email: string,
    private age: number
  ) {}
  
  // Responsabilidad 1: Datos del usuario
  getName(): string { return this.name; }
  setName(name: string): void { this.name = name; }
  
  // Responsabilidad 2: Persistencia (no debería estar aquí)
  save(): void {
    const db = new Database();
    db.query(`INSERT INTO users VALUES (?, ?, ?)`, 
      [this.name, this.email, this.age]);
  }
  
  // Responsabilidad 3: Emails (no debería estar aquí)
  sendWelcomeEmail(): void {
    const emailService = new EmailService();
    emailService.send(this.email, 'Welcome!', `Hello ${this.name}`);
  }
  
  // Responsabilidad 4: Validación (no debería estar aquí)
  validateEmail(): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(this.email);
  }
}

// ✅ SOLUCIÓN: Separar responsabilidades

// 1. User: Solo datos
class User {
  constructor(
    private id: string,
    private name: string,
    private email: string,
    private age: number
  ) {}
  
  getId(): string { return this.id; }
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  setName(name: string): void { this.name = name; }
}

// 2. UserRepository: Solo persistencia
class UserRepository {
  constructor(private database: Database) {}
  
  async save(user: User): Promise<void> {
    await this.database.query(
      `INSERT INTO users VALUES (?, ?, ?, ?)`,
      [user.getId(), user.getName(), user.getEmail(), user.getAge()]
    );
  }
}

// 3. EmailService: Solo emails
class EmailService {
  async sendWelcomeEmail(user: User): Promise<void> {
    await this.emailProvider.send({
      to: user.getEmail(),
      subject: 'Welcome!',
      body: `Hello ${user.getName()}`
    });
  }
}

// 4. UserValidator: Solo validación
class UserValidator {
  validate(user: User): ValidationResult {
    const errors: string[] = [];
    
    if (!user.getEmail().includes('@')) {
      errors.push('Invalid email');
    }
    if (user.getAge() < 18) {
      errors.push('Must be 18+');
    }
    
    return { isValid: errors.length === 0, errors };
  }
}

// 5. UserService: Orquesta todo
class UserService {
  constructor(
    private repo: UserRepository,
    private email: EmailService,
    private validator: UserValidator
  ) {}
  
  async createUser(data: CreateUserDto): Promise<User> {
    const user = new User(generateId(), data.name, data.email, data.age);
    
    const validation = this.validator.validate(user);
    if (!validation.isValid) {
      throw new Error(validation.errors.join(', '));
    }
    
    await this.repo.save(user);
    await this.email.sendWelcomeEmail(user);
    return user;
  }
}

// ============================================
// ENFOQUE FUNCIONAL: SRP con funciones puras
// ============================================

// Tipos inmutables
type User = {
  readonly id: string;
  readonly name: string;
  readonly email: string;
  readonly age: number;
};

// Funciones puras con responsabilidad única
const createUser = (name: string, email: string, age: number): User => ({
  id: generateId(),
  name,
  email,
  age
});

// Validación como función pura
const validateEmail = (email: string): boolean =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

const validateAge = (age: number): boolean => 
  age >= 18 && age <= 120;

const validateUser = (user: User): Either<string[], User> => {
  const errors: string[] = [];
  if (!validateEmail(user.email)) errors.push('Invalid email');
  if (!validateAge(user.age)) errors.push('Invalid age');
  
  return errors.length > 0 ? left(errors) : right(user);
};

// Persistencia como función
const saveUser = (db: Database) => (user: User): Promise<void> =>
  db.insert('users', user);

// Email como función
const sendWelcomeEmail = (emailService: EmailService) => (user: User): Promise<void> =>
  emailService.send(user.email, 'Welcome!', `Hello ${user.name}`);

// Composición funcional
import { pipe } from 'fp-ts/function';
import { chain } from 'fp-ts/TaskEither';

const processNewUser = (deps: { db: Database; email: EmailService }) =>
  (name: string, email: string, age: number) =>
    pipe(
      createUser(name, email, age),
      validateUser,
      chain(saveUser(deps.db)),
      chain(sendWelcomeEmail(deps.email))
    );
```

### Cómo identificar y refactorizar violaciones de SRP

1. **Identificar las responsabilidades**: Lista todas las cosas que hace la clase
2. **Agrupar por razón de cambio**: Agrupa funcionalidades que cambiarían juntas
3. **Extraer clases**: Crea una nueva clase para cada grupo
4. **Establecer relaciones**: Define cómo interactúan las nuevas clases
5. **Testear**: Asegúrate de que cada clase puede testearse independientemente

---

## 1.2 Open/Closed Principle (OCP)

### Explicación Detallada

El Principio Abierto/Cerrado establece que **las entidades de software (clases, módulos, funciones) deben estar abiertas para extensión pero cerradas para modificación**. Esto significa que deberías poder agregar nueva funcionalidad sin cambiar el código existente.

#### ¿Qué significa "abierto" y "cerrado"?

- **Abierto para extensión**: Puedes agregar nuevo comportamiento o funcionalidad
- **Cerrado para modificación**: No necesitas cambiar el código existente que ya funciona

#### ¿Por qué es importante?

1. **Reduce el riesgo de bugs**: No tocas código que ya funciona
2. **Facilita el mantenimiento**: Nuevas features se agregan sin modificar lo existente
3. **Mejora la escalabilidad**: El sistema crece por adición, no por modificación
4. **Simplifica el testing**: No necesitas re-testear código no modificado

#### Técnicas para lograr OCP:

- **Abstracción mediante interfaces**: Define contratos que las implementaciones deben seguir
- **Herencia**: Extiende clases base sin modificarlas
- **Composición**: Combina objetos para crear nuevo comportamiento
- **Inyección de dependencias**: Permite cambiar comportamiento mediante configuración
- **Strategy Pattern**: Encapsula algoritmos intercambiables
- **Template Method Pattern**: Define estructura con pasos customizables

### Ejemplo Práctico con Explicación

```typescript
// ❌ PROBLEMA: Código que viola OCP
class DiscountCalculator {
  calculate(customerType: string, amount: number): number {
    // Problema: Agregar nuevos tipos requiere modificar este método
    if (customerType === 'regular') {
      return amount * 0.95;  // 5% descuento
    } else if (customerType === 'premium') {
      return amount * 0.90;  // 10% descuento
    } else if (customerType === 'vip') {
      return amount * 0.85;  // 15% descuento
    }
    // Para agregar 'employee' hay que modificar el código existente!
    return amount;
  }
}

// ✅ SOLUCIÓN: Aplicar OCP con Strategy Pattern

// 1. Interface para el contrato
interface DiscountStrategy {
  calculateDiscount(amount: number): number;
}

// 2. Implementaciones concretas - CERRADAS para modificación
class RegularDiscount implements DiscountStrategy {
  calculateDiscount(amount: number): number {
    return amount * 0.95;  // 5% descuento
  }
}

class PremiumDiscount implements DiscountStrategy {
  calculateDiscount(amount: number): number {
    return amount * 0.90;  // 10% descuento
  }
}

class VIPDiscount implements DiscountStrategy {
  calculateDiscount(amount: number): number {
    return amount * 0.85;  // 15% descuento
  }
}

// 3. EXTENSIÓN sin modificación - Agregar nuevos tipos
class EmployeeDiscount implements DiscountStrategy {
  calculateDiscount(amount: number): number {
    return amount * 0.70;  // 30% descuento
  }
}

// Podemos agregar más sin tocar código existente!
class FirstPurchaseDiscount implements DiscountStrategy {
  calculateDiscount(amount: number): number {
    return amount * 0.80;  // 20% descuento
  }
}

// 4. Calculadora refactorizada - CERRADA para modificación
class DiscountCalculator {
  constructor(private strategy: DiscountStrategy) {}
  
  calculate(amount: number): number {
    // Este método NUNCA cambia
    return this.strategy.calculateDiscount(amount);
  }
  
  setStrategy(strategy: DiscountStrategy): void {
    this.strategy = strategy;
  }
}

// 5. Uso del sistema
const calculator = new DiscountCalculator(new RegularDiscount());
console.log(calculator.calculate(100));  // 95

// Cambiar estrategia en runtime
calculator.setStrategy(new VIPDiscount());
console.log(calculator.calculate(100));  // 85

// Agregar nuevo tipo sin modificar código existente
calculator.setStrategy(new EmployeeDiscount());
console.log(calculator.calculate(100));  // 70

// ============================================
// ENFOQUE FUNCIONAL: OCP con funciones de orden superior
// ============================================

// Funciones como ciudadanos de primera clase
type DiscountFunction = (amount: number) => number;

// Funciones de descuento base
const regularDiscount: DiscountFunction = (amount) => amount * 0.95;
const premiumDiscount: DiscountFunction = (amount) => amount * 0.90;
const vipDiscount: DiscountFunction = (amount) => amount * 0.85;

// Función de orden superior para combinar descuentos
const combineDiscounts = (...discounts: DiscountFunction[]): DiscountFunction =>
  (amount) => discounts.reduce((acc, discount) => discount(acc), amount);

// Descuento condicional
const conditionalDiscount = (
  condition: (amount: number) => boolean,
  discount: DiscountFunction
): DiscountFunction =>
  (amount) => condition(amount) ? discount(amount) : amount;

// Crear nuevos descuentos sin modificar código existente
const volumeDiscount = conditionalDiscount(
  amount => amount > 1000,
  amount => amount * 0.9
);

const seasonalDiscount = (season: string): DiscountFunction => {
  const rates = { summer: 0.85, winter: 0.90, spring: 0.95, fall: 0.92 };
  return (amount) => amount * (rates[season] || 1);
};

// Composición de descuentos
const calculatePrice = combineDiscounts(
  regularDiscount,
  volumeDiscount,
  seasonalDiscount('summer')
);

console.log(calculatePrice(1500)); // Aplica todos los descuentos
```

### Beneficios del OCP en este ejemplo:

1. **Agregar nuevos descuentos no requiere modificar código existente**
2. **Cada tipo de descuento está aislado y puede testearse independientemente**
3. **Los bugs en un tipo de descuento no afectan a otros**
4. **El código es más modular y reutilizable**
5. **Es fácil combinar diferentes tipos de descuentos**

---

## 1.3 Liskov Substitution Principle (LSP)

### Explicación Detallada

El Principio de Sustitución de Liskov establece que **los objetos de una subclase deben poder reemplazar a objetos de la superclase sin alterar las propiedades deseables del programa (corrección, tareas realizadas, etc.)**.

#### ¿Qué significa en la práctica?

Si S es un subtipo de T, entonces los objetos de tipo T pueden ser reemplazados con objetos de tipo S sin alterar ninguna de las propiedades deseables del programa. En otras palabras:
- Las subclases deben cumplir el contrato establecido por la clase base
- No deben fortalecer las precondiciones
- No deben debilitar las postcondiciones
- Deben preservar los invariantes

#### Violaciones comunes de LSP:

1. **Subclases que lanzan excepciones no esperadas**
2. **Subclases que no implementan todos los métodos de la clase base**
3. **Subclases que cambian el comportamiento esperado**
4. **Subclases que tienen precondiciones más estrictas**
5. **Subclases que retornan tipos incompatibles**

#### ¿Por qué es importante LSP?

- **Confiabilidad**: El código cliente puede confiar en el comportamiento esperado
- **Reutilización**: Las abstracciones son verdaderamente intercambiables
- **Mantenibilidad**: Los cambios en subclases no rompen el código cliente
- **Testing**: Los tests de la clase base aplican a todas las subclases

### Ejemplo Práctico con Explicación

```typescript
// ❌ PROBLEMA: Violación clásica de LSP
class Rectangle {
  constructor(
    protected width: number, 
    protected height: number
  ) {}
  
  setWidth(width: number): void { this.width = width; }
  setHeight(height: number): void { this.height = height; }
  getArea(): number { return this.width * this.height; }
}

// Square hereda de Rectangle - PROBLEMA!
class Square extends Rectangle {
  setWidth(width: number): void {
    this.width = width;
    this.height = width; // Efecto secundario!
  }
  
  setHeight(height: number): void {
    this.width = height; // Efecto secundario!
    this.height = height;
  }
}

// Test que demuestra la violación
function testRectangle(rect: Rectangle) {
  rect.setWidth(5);
  rect.setHeight(4);
  // Espera area = 20, pero Square dará 16!
  console.log(rect.getArea());
}

// ✅ SOLUCIÓN: Usar composición en lugar de herencia
interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(
    private width: number,
    private height: number
  ) {}
  
  setDimensions(width: number, height: number): void {
    this.width = width;
    this.height = height;
  }
  
  getArea(): number {
    return this.width * this.height;
  }
}

class Square implements Shape {
  constructor(private side: number) {}
  
  setSide(side: number): void {
    this.side = side;
  }
  
  getArea(): number {
    return this.side * this.side;
  }
}


// Otro ejemplo: Sistema de Notificaciones
// ❌ PROBLEMA que viola LSP
abstract class Notification {
  abstract send(message: string): void;
}

class SMSNotification extends Notification {
  send(message: string): void {
    if (message.length > 160) {
      throw new Error('Too long'); // Violación!
    }
    console.log(`SMS: ${message}`);
  }
}

// ✅ SOLUCIÓN que respeta LSP
interface NotificationSender {
  send(message: string): NotificationResult;
  canSend(message: string): boolean;
}

class SMSNotificationSender implements NotificationSender {
  send(message: string): NotificationResult {
    if (message.length > 160) {
      // En lugar de lanzar error, dividimos el mensaje
      const parts = Math.ceil(message.length / 160);
      return {
        success: true,
        messageId: `sms-${parts}-parts`
      };
    }
    return { success: true, messageId: 'sms-single' };
  }
  
  canSend(message: string): boolean {
    return true; // Siempre puede enviar (dividiendo)
  }
}

// ============================================
// ENFOQUE FUNCIONAL: LSP con tipos algebraicos
// ============================================

// Tipos de datos algebraicos (ADTs) en lugar de herencia
type Shape = 
  | { type: 'rectangle'; width: number; height: number }
  | { type: 'square'; side: number }
  | { type: 'circle'; radius: number };

// Funciones que funcionan con todos los tipos de forma predecible
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

// Transformaciones sin violar expectativas
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

// Uso seguro sin sorpresas
const shapes: Shape[] = [
  { type: 'rectangle', width: 5, height: 4 },
  { type: 'square', side: 3 },
  { type: 'circle', radius: 2 }
];

shapes.map(area); // Funciona correctamente para todas
shapes.map(scale(2)); // Escala correctamente todas
```

### Reglas para cumplir LSP:

1. **Preservar la firma del método**: Los tipos de parámetros y retorno deben ser compatibles
2. **No fortalecer precondiciones**: No requieras más de lo que requiere la clase base
3. **No debilitar postcondiciones**: Cumple al menos lo que promete la clase base
4. **Preservar invariantes**: Mantén las propiedades que siempre deben ser verdaderas
5. **No lanzar excepciones inesperadas**: Solo lanza excepciones que la clase base declara

---

## 1.4 Interface Segregation Principle (ISP)

### Explicación Detallada

El Principio de Segregación de Interfaces establece que **los clientes no deberían verse forzados a depender de interfaces que no usan**. Es mejor tener muchas interfaces específicas que una interfaz general.

#### Conceptos clave:

1. **Interfaces "gordas" vs "delgadas"**: Las interfaces gordas tienen muchos métodos que no todos los clientes necesitan
2. **Cohesión de interfaces**: Los métodos en una interfaz deben estar relacionados
3. **Segregación por cliente**: Diferentes clientes pueden necesitar diferentes vistas del mismo objeto

#### Problemas de violar ISP:

- **Acoplamiento innecesario**: Los clientes dependen de métodos que no usan
- **Cambios en cascada**: Cambios en la interfaz afectan a clientes que no usan esos métodos
- **Implementaciones falsas**: Clases que deben implementar métodos que no tienen sentido para ellas
- **Complejidad aumentada**: Interfaces difíciles de entender y mantener

#### Beneficios de aplicar ISP:

- **Bajo acoplamiento**: Los clientes solo dependen de lo que necesitan
- **Alta cohesión**: Interfaces enfocadas y con propósito claro
- **Flexibilidad**: Fácil agregar nuevas implementaciones
- **Mantenibilidad**: Cambios localizados y predecibles

### Ejemplo Práctico con Explicación

```typescript
// ❌ PROBLEMA: Interface "gorda" que viola ISP
interface Employee {
  getId(): string;
  getName(): string;
  getEmail(): string;
  
  work(): void;
  attendMeeting(): void;
  
  writeCode(): void;        // No todos programan!
  approveTimeOff(): void;   // No todos gestionan!
  makeSale(): void;         // No todos venden!
  eat(): void;              // Los robots no comen!
}

class Developer implements Employee {
  // Debe implementar métodos que no usa
  approveTimeOff(): void { 
    throw new Error('Developers cannot approve');
  }
  makeSale(): void { 
    throw new Error('Developers do not sell');
  }
  // ...
}

// ✅ SOLUCIÓN: Aplicar ISP segregando interfaces

// Interfaces pequeñas y enfocadas
interface Identifiable {
  getId(): string;
}

interface Person {
  getName(): string;
  getEmail(): string;
}

interface Workable {
  work(): void;
}

interface Programmer {
  writeCode(): void;
  reviewCode(): void;
}

interface Manager {
  approveTimeOff(): void;
  manageTeam(): void;
}

interface SalesPerson {
  makeSale(): void;
  callClient(): void;
}

interface BiologicalNeeds {
  eat(): void;
  sleep(): void;
}

// Composición de interfaces para roles
interface DeveloperRole extends 
  Identifiable, 
  Person, 
  Workable, 
  Programmer {}

interface ManagerRole extends 
  Identifiable,
  Person,
  Workable,
  Manager {}

interface HumanEmployee extends 
  Identifiable, 
  Person, 
  BiologicalNeeds {}

interface RobotEmployee extends 
  Identifiable, 
  Workable {}

// Implementaciones limpias
class Developer implements DeveloperRole, BiologicalNeeds {
  constructor(
    private id: string,
    private name: string,
    private email: string
  ) {}
  
  // Solo implementa lo que necesita
  getId(): string { return this.id; }
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  
  work(): void { console.log('Coding...'); }
  writeCode(): void { console.log('Writing code...'); }
  reviewCode(): void { console.log('Reviewing...'); }
  
  eat(): void { console.log('Eating...'); }
  sleep(): void { console.log('Sleeping...'); }
}

class SalesRepresentative implements SalesRole, BiologicalNeeds {
  // Solo implementa métodos de ventas
  makeSale(): void { console.log('Making sale...'); }
  callClient(): void { console.log('Calling client...'); }
  
  // Y necesidades biológicas porque es humano
  eat(): void { console.log('Eating...'); }
  sleep(): void { console.log('Sleeping...'); }
  // ...
}

class AIAssistant implements RobotEmployee, Programmer {
  // Robot: no necesita métodos biológicos!
  getId(): string { return this.id; }
  work(): void { console.log('Processing...'); }
  
  // Puede programar
  writeCode(): void { console.log('Generating code...'); }
  reviewCode(): void { console.log('Analyzing...'); }
  
  // NO implementa eat(), sleep(), etc.!
}

// Uso: Cada sistema solo conoce las interfaces que necesita
class WorkScheduler {
  scheduleWork(workers: Workable[]): void {
    workers.forEach(w => w.work());
  }
}

class CodeReviewSystem {
  assignReview(reviewer: Programmer): void {
    reviewer.reviewCode();
  }
}

// Ejemplo de uso
const dev = new Developer('1', 'Alice', 'alice@co.com');
const ai = new AIAssistant('ai-1');
const sales = new SalesRepresentative('2', 'Bob', 'bob@co.com');

const scheduler = new WorkScheduler();
scheduler.scheduleWork([dev, ai, sales]); // Todos pueden trabajar

const reviewSystem = new CodeReviewSystem();
reviewSystem.assignReview(dev);  // ✅ OK
reviewSystem.assignReview(ai);   // ✅ OK
// reviewSystem.assignReview(sales); // ❌ Error: sales no es Programmer

// ============================================
// ENFOQUE FUNCIONAL: ISP con composición de funciones
// ============================================

// Funciones pequeñas y enfocadas
type Work = () => void;
type Code = (project: string) => void;
type Manage = (team: string[]) => void;
type Sell = (product: string) => void;

// Composición de capacidades
type DeveloperCapabilities = {
  work: Work;
  code: Code;
};

type ManagerCapabilities = {
  work: Work;
  manage: Manage;
};

// Factory functions con aplicación parcial
const createWorker = (name: string): { work: Work } => ({
  work: () => console.log(`${name} is working`)
});

const withCoding = (name: string) => (worker: { work: Work }): DeveloperCapabilities => ({
  ...worker,
  code: (project) => console.log(`${name} coding ${project}`)
});

const withManaging = (name: string) => (worker: { work: Work }): ManagerCapabilities => ({
  ...worker,
  manage: (team) => console.log(`${name} managing ${team.join(', ')}`)
});

// Composición usando pipe
import { pipe } from 'fp-ts/function';

const createDeveloper = (name: string) => 
  pipe(
    createWorker(name),
    withCoding(name)
  );

const createTechLead = (name: string) =>
  pipe(
    createWorker(name),
    withCoding(name),
    withManaging(name)
  );

// Uso
const alice = createDeveloper('Alice');
alice.work();
alice.code('frontend');

const bob = createTechLead('Bob');
bob.work();
bob.code('backend');
bob.manage(['Alice', 'Charlie']);
```

### Técnicas para aplicar ISP:

1. **Identificar los diferentes clientes de una interface**
2. **Agrupar métodos por responsabilidad o capacidad**
3. **Crear interfaces pequeñas y cohesivas**
4. **Usar composición de interfaces cuando sea necesario**
5. **Permitir que las clases implementen múltiples interfaces**

---

## 1.5 Dependency Inversion Principle (DIP)

### Explicación Detallada

El Principio de Inversión de Dependencias establece que:
1. **Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones.**
2. **Las abstracciones no deben depender de detalles. Los detalles deben depender de abstracciones.**

#### ¿Qué significa "inversión"?

El término "inversión" se refiere a que invertimos la dirección tradicional de las dependencias:
- **Tradicional**: Alto nivel → Bajo nivel → Detalles de implementación
- **Con DIP**: Alto nivel → Abstracción ← Bajo nivel

#### Conceptos clave:

1. **Módulos de alto nivel**: Contienen la lógica de negocio importante
2. **Módulos de bajo nivel**: Implementan operaciones detalladas (DB, API, archivos)
3. **Abstracciones**: Interfaces o clases abstractas que definen contratos
4. **Inyección de dependencias**: Técnica para proveer dependencias desde afuera

#### Beneficios de DIP:

- **Flexibilidad**: Fácil cambiar implementaciones sin tocar lógica de negocio
- **Testabilidad**: Puedes usar mocks y stubs fácilmente
- **Desacoplamiento**: Los módulos no conocen detalles de implementación
- **Reutilización**: La lógica de negocio es independiente de la infraestructura
- **Mantenibilidad**: Cambios en bajo nivel no afectan alto nivel

### Ejemplo Práctico con Explicación

```typescript
// ❌ PROBLEMA: Violación de DIP - Alto nivel depende de bajo nivel

class OrderService {
  private database: MySQLDatabase;      // ❌ Dependencia directa
  private emailService: GmailService;   // ❌ Dependencia directa  
  private paymentGateway: StripePayment; // ❌ Dependencia directa
  
  constructor() {
    // Creación directa = fuerte acoplamiento
    this.database = new MySQLDatabase();
    this.emailService = new GmailService();
    this.paymentGateway = new StripePayment();
  }
  
  createOrder(orderData: any): void {
    // Lógica acoplada a implementaciones específicas
    this.database.query(`INSERT INTO orders...`);
    
    const paid = this.paymentGateway.processPayment(orderData.total);
    
    if (paid) {
      this.emailService.sendEmail(orderData.email, 'Confirmed!');
    }
    
    // Problemas:
    // - No podemos cambiar de MySQL a MongoDB
    // - No podemos cambiar de Gmail a SendGrid
    // - No podemos cambiar de Stripe a PayPal
    // - No podemos testear sin servicios reales
  }
}

// ✅ SOLUCIÓN: Aplicar DIP usando abstracciones

// 1. Definir abstracciones (interfaces)
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

interface NotificationService {
  send(to: string, message: string): Promise<void>;
}

interface PaymentProcessor {
  process(amount: number): Promise<boolean>;
}

// 2. Implementaciones concretas
class MySQLOrderRepository implements OrderRepository {
  async save(order: Order): Promise<void> {
    // Implementación MySQL
    await this.db.query(`INSERT INTO orders...`);
  }
  
  async findById(id: string): Promise<Order | null> {
    const result = await this.db.query(`SELECT * FROM orders WHERE id = ?`, [id]);
    return result ? new Order(result) : null;
  }
}

class MongoOrderRepository implements OrderRepository {
  async save(order: Order): Promise<void> {
    // Implementación MongoDB
    await this.collection.insertOne(order);
  }
  
  async findById(id: string): Promise<Order | null> {
    return await this.collection.findOne({ _id: id });
  }
}

class EmailService implements NotificationService {
  async send(to: string, message: string): Promise<void> {
    await this.provider.sendEmail(to, message);
  }
}

class StripePayment implements PaymentProcessor {
  async process(amount: number): Promise<boolean> {
    const result = await this.stripe.charge(amount);
    return result.success;
  }
}

// 3. Módulo de alto nivel con DIP aplicado
class OrderService {
  constructor(
    private repository: OrderRepository,      // ✅ Abstracción
    private notifications: NotificationService, // ✅ Abstracción
    private payment: PaymentProcessor         // ✅ Abstracción
  ) {
    // Dependencias inyectadas, no creadas
  }
  
  async createOrder(orderData: any): Promise<void> {
    const order = new Order(orderData);
    
    // Usa abstracciones, no implementaciones concretas
    const paid = await this.payment.process(order.total);
    
    if (paid) {
      await this.repository.save(order);
      await this.notifications.send(order.email, 'Order confirmed!');
    }
  }
}

// 4. Configuración con inyección de dependencias

// Para producción
const orderServiceProd = new OrderService(
  new MySQLOrderRepository(),
  new EmailService(),
  new StripePayment()
);

// Para testing (con mocks)
const orderServiceTest = new OrderService(
  new InMemoryOrderRepository(),
  new MockNotificationService(),
  new MockPaymentProcessor()
);

// El código funciona igual con cualquier implementación
await orderServiceProd.createOrder(orderData);
await orderServiceTest.createOrder(orderData); // Para tests

// Ventajas de DIP:
// - Cambiar de MySQL a MongoDB: solo cambiar la configuración
// - Cambiar de Stripe a PayPal: solo cambiar la configuración
// - Testing fácil con mocks
// - OrderService no cambia nunca

// ============================================
// ENFOQUE FUNCIONAL: DIP con funciones de orden superior
// ============================================

// Tipos de efectos como parámetros
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

// Lógica de negocio pura con dependencias inyectadas
const createOrder = (deps: {
  db: DatabaseOps;
  email: EmailOps;
  payment: PaymentOps;
}) => async (orderData: OrderData): Promise<Order> => {
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

// Implementaciones para diferentes entornos
const prodDeps = {
  db: {
    save: async (table, data) => mysqlClient.insert(table, data),
    find: async (table, id) => mysqlClient.select(table, id)
  },
  email: {
    send: async (to, subject, body) => sendgrid.send({ to, subject, body })
  },
  payment: {
    charge: async (amount, method) => stripe.charge(amount, method)
  }
};

const testDeps = {
  db: {
    save: async () => {},
    find: async () => null
  },
  email: {
    send: async () => console.log('Mock email sent')
  },
  payment: {
    charge: async () => true
  }
};

// Uso
const processOrderProd = createOrder(prodDeps);
const processOrderTest = createOrder(testDeps);

await processOrderProd(orderData);
await processOrderTest(orderData);
```

### Resumen de DIP:

1. **Define abstracciones claras** (interfaces) para las dependencias
2. **Los módulos de alto nivel dependen solo de abstracciones**, no de implementaciones
3. **Las implementaciones concretas implementan las abstracciones**
4. **Usa inyección de dependencias** para proveer las implementaciones
5. **Configura las dependencias externamente** (IoC Container, Factory, etc.)

### Beneficios alcanzados:

- **Flexibilidad total**: Puedes cambiar de MySQL a MongoDB sin tocar OrderService
- **Testing fácil**: Usas mocks sin modificar el código de producción
- **Mantenimiento simple**: Los cambios están localizados
- **Código reutilizable**: OrderService funciona con cualquier implementación que cumpla las interfaces
- **Desacoplamiento**: Los módulos no se conocen entre sí directamente

---

## Conclusión de SOLID

Los principios SOLID trabajan juntos para crear código mantenible:

- **SRP** mantiene las clases enfocadas y cohesivas
- **OCP** permite extensión sin modificación
- **LSP** garantiza que las abstracciones sean confiables
- **ISP** evita dependencias innecesarias
- **DIP** desacopla los módulos y facilita el testing

Aplicar estos principios requiere práctica, pero los beneficios en mantenibilidad, escalabilidad y calidad del código son enormes. En tu entrevista técnica, poder explicar estos principios con ejemplos prácticos y demostrar cuándo y cómo aplicarlos será fundamental para mostrar tu nivel como Senior Software Engineer.