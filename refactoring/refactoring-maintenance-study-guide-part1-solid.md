# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 1: Principios SOLID
### Para Entrevista Senior Software Engineer - Zillow

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
// Esta clase viola SRP porque tiene 4 razones diferentes para cambiar:
// 1. Cambios en la estructura de datos del usuario
// 2. Cambios en la lógica de persistencia
// 3. Cambios en la lógica de envío de emails
// 4. Cambios en las reglas de validación

class User {
  constructor(
    private name: string, 
    private email: string,
    private age: number
  ) {}
  
  // Responsabilidad 1: Gestión de datos del usuario
  getName(): string {
    return this.name;
  }
  
  setName(name: string): void {
    this.name = name;
  }
  
  // Responsabilidad 2: Persistencia en base de datos
  save(): void {
    // Problema: Si cambia el mecanismo de persistencia (de SQL a NoSQL),
    // necesitamos modificar la clase User
    const db = new Database();
    db.query(`INSERT INTO users (name, email, age) VALUES (?, ?, ?)`, 
      [this.name, this.email, this.age]);
  }
  
  // Responsabilidad 3: Envío de emails
  sendWelcomeEmail(): void {
    // Problema: Si cambia el servicio de email o el template,
    // necesitamos modificar la clase User
    const emailService = new EmailService();
    emailService.send(this.email, 'Welcome!', `Hello ${this.name}...`);
  }
  
  // Responsabilidad 4: Validación
  validateEmail(): boolean {
    // Problema: Si cambian las reglas de validación de email,
    // necesitamos modificar la clase User
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(this.email);
  }
  
  validateAge(): boolean {
    // Problema: Si cambian las reglas de negocio sobre la edad,
    // necesitamos modificar la clase User
    return this.age >= 18 && this.age <= 120;
  }
}

// ✅ SOLUCIÓN: Separar responsabilidades en clases especializadas
// Cada clase tiene una única razón para cambiar

// 1. Clase User: Solo gestiona los datos del usuario
// Razón para cambiar: Cambios en la estructura de datos del usuario
class User {
  constructor(
    private id: string,
    private name: string,
    private email: string,
    private age: number
  ) {}
  
  // Solo métodos relacionados con acceso a datos del usuario
  getId(): string { return this.id; }
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  getAge(): number { return this.age; }
  
  setName(name: string): void { this.name = name; }
  setEmail(email: string): void { this.email = email; }
  setAge(age: number): void { this.age = age; }
}

// 2. UserRepository: Solo maneja persistencia
// Razón para cambiar: Cambios en el mecanismo de persistencia
class UserRepository {
  constructor(private database: Database) {}
  
  async save(user: User): Promise<void> {
    await this.database.query(
      `INSERT INTO users (id, name, email, age) VALUES (?, ?, ?, ?)`,
      [user.getId(), user.getName(), user.getEmail(), user.getAge()]
    );
  }
  
  async findById(id: string): Promise<User | null> {
    const result = await this.database.query(
      `SELECT * FROM users WHERE id = ?`,
      [id]
    );
    
    if (!result) return null;
    
    return new User(result.id, result.name, result.email, result.age);
  }
  
  async update(user: User): Promise<void> {
    await this.database.query(
      `UPDATE users SET name = ?, email = ?, age = ? WHERE id = ?`,
      [user.getName(), user.getEmail(), user.getAge(), user.getId()]
    );
  }
}

// 3. EmailService: Solo maneja envío de emails
// Razón para cambiar: Cambios en el servicio o templates de email
class EmailService {
  constructor(private emailProvider: EmailProvider) {}
  
  async sendWelcomeEmail(user: User): Promise<void> {
    const template = this.loadTemplate('welcome');
    const content = this.renderTemplate(template, {
      name: user.getName(),
      email: user.getEmail()
    });
    
    await this.emailProvider.send({
      to: user.getEmail(),
      subject: 'Welcome to our platform!',
      body: content
    });
  }
  
  private loadTemplate(name: string): string {
    // Cargar template de email
    return `<h1>Welcome {{name}}!</h1>`;
  }
  
  private renderTemplate(template: string, data: any): string {
    // Renderizar template con datos
    return template.replace('{{name}}', data.name);
  }
}

// 4. UserValidator: Solo maneja validación
// Razón para cambiar: Cambios en las reglas de validación
class UserValidator {
  private readonly MIN_AGE = 18;
  private readonly MAX_AGE = 120;
  private readonly EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  
  validate(user: User): ValidationResult {
    const errors: string[] = [];
    
    if (!this.isValidEmail(user.getEmail())) {
      errors.push('Invalid email format');
    }
    
    if (!this.isValidAge(user.getAge())) {
      errors.push(`Age must be between ${this.MIN_AGE} and ${this.MAX_AGE}`);
    }
    
    if (!this.isValidName(user.getName())) {
      errors.push('Name cannot be empty');
    }
    
    return {
      isValid: errors.length === 0,
      errors
    };
  }
  
  private isValidEmail(email: string): boolean {
    return this.EMAIL_REGEX.test(email);
  }
  
  private isValidAge(age: number): boolean {
    return age >= this.MIN_AGE && age <= this.MAX_AGE;
  }
  
  private isValidName(name: string): boolean {
    return name.trim().length > 0;
  }
}

// 5. UserService: Orquesta las operaciones (Facade pattern)
// Razón para cambiar: Cambios en el flujo de negocio
class UserService {
  constructor(
    private userRepository: UserRepository,
    private emailService: EmailService,
    private userValidator: UserValidator
  ) {}
  
  async createUser(userData: CreateUserDto): Promise<User> {
    // Crear usuario
    const user = new User(
      generateId(),
      userData.name,
      userData.email,
      userData.age
    );
    
    // Validar
    const validation = this.userValidator.validate(user);
    if (!validation.isValid) {
      throw new ValidationError(validation.errors);
    }
    
    // Guardar
    await this.userRepository.save(user);
    
    // Enviar email de bienvenida
    await this.emailService.sendWelcomeEmail(user);
    
    return user;
  }
}
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
// Cada vez que agregamos un nuevo tipo de descuento, debemos modificar esta clase
// Esto viola OCP porque está cerrada para extensión y abierta para modificación

class DiscountCalculator {
  calculate(customerType: string, amount: number): number {
    // Problema 1: Agregar un nuevo tipo requiere modificar este método
    // Problema 2: La lógica de cada descuento está acoplada a esta clase
    // Problema 3: Si hay un bug en un descuento, podríamos romper otros
    
    if (customerType === 'regular') {
      // 5% de descuento para clientes regulares
      return amount * 0.95;
    } else if (customerType === 'premium') {
      // 10% de descuento para clientes premium
      return amount * 0.90;
    } else if (customerType === 'vip') {
      // 15% de descuento para clientes VIP
      return amount * 0.85;
    } else if (customerType === 'employee') {
      // Problema: Tuvimos que MODIFICAR el código para agregar este caso
      return amount * 0.70;
    }
    // Problema: ¿Qué pasa si olvidamos un else if?
    return amount;
  }
}

// Imagina que necesitas agregar:
// - Descuentos por temporada
// - Descuentos por cantidad
// - Descuentos por primera compra
// - Descuentos combinados
// Cada uno requeriría modificar esta clase!

// ✅ SOLUCIÓN: Aplicar OCP usando abstracción y polimorfismo

// 1. Definir una abstracción (interface) que representa el contrato
interface DiscountStrategy {
  // Cada estrategia debe implementar este método
  calculateDiscount(amount: number): number;
  
  // Método opcional para describir el descuento
  getDescription?(): string;
}

// 2. Implementaciones concretas - CERRADAS para modificación
// Cada tipo de descuento es una clase separada que no cambiará

class RegularCustomerDiscount implements DiscountStrategy {
  private readonly DISCOUNT_RATE = 0.05;
  
  calculateDiscount(amount: number): number {
    return amount * (1 - this.DISCOUNT_RATE);
  }
  
  getDescription(): string {
    return `Regular customer discount: ${this.DISCOUNT_RATE * 100}%`;
  }
}

class PremiumCustomerDiscount implements DiscountStrategy {
  private readonly DISCOUNT_RATE = 0.10;
  
  calculateDiscount(amount: number): number {
    return amount * (1 - this.DISCOUNT_RATE);
  }
  
  getDescription(): string {
    return `Premium customer discount: ${this.DISCOUNT_RATE * 100}%`;
  }
}

class VIPCustomerDiscount implements DiscountStrategy {
  private readonly DISCOUNT_RATE = 0.15;
  
  calculateDiscount(amount: number): number {
    return amount * (1 - this.DISCOUNT_RATE);
  }
  
  getDescription(): string {
    return `VIP customer discount: ${this.DISCOUNT_RATE * 100}%`;
  }
}

// 3. EXTENSIÓN sin modificación - Agregar nuevos tipos es fácil
// No tocamos ninguna clase existente!

class SeasonalDiscount implements DiscountStrategy {
  constructor(
    private season: 'summer' | 'winter' | 'spring' | 'fall',
    private discountRate: number
  ) {}
  
  calculateDiscount(amount: number): number {
    return amount * (1 - this.discountRate);
  }
  
  getDescription(): string {
    return `${this.season} seasonal discount: ${this.discountRate * 100}%`;
  }
}

class QuantityDiscount implements DiscountStrategy {
  constructor(
    private quantity: number,
    private threshold: number,
    private discountRate: number
  ) {}
  
  calculateDiscount(amount: number): number {
    if (this.quantity >= this.threshold) {
      return amount * (1 - this.discountRate);
    }
    return amount;
  }
  
  getDescription(): string {
    return `Quantity discount: ${this.discountRate * 100}% for ${this.threshold}+ items`;
  }
}

class FirstPurchaseDiscount implements DiscountStrategy {
  private readonly DISCOUNT_RATE = 0.20;
  
  calculateDiscount(amount: number): number {
    return amount * (1 - this.DISCOUNT_RATE);
  }
  
  getDescription(): string {
    return `First purchase discount: ${this.DISCOUNT_RATE * 100}%`;
  }
}

// 4. Estrategia compuesta para combinar descuentos
class CompositeDiscount implements DiscountStrategy {
  constructor(private strategies: DiscountStrategy[]) {}
  
  calculateDiscount(amount: number): number {
    // Aplica todos los descuentos en secuencia
    return this.strategies.reduce(
      (currentAmount, strategy) => strategy.calculateDiscount(currentAmount),
      amount
    );
  }
  
  getDescription(): string {
    return this.strategies
      .map(s => s.getDescription?.() || 'Unknown discount')
      .join(' + ');
  }
}

// 5. Calculadora que usa las estrategias - CERRADA para modificación
class DiscountCalculator {
  constructor(private strategy: DiscountStrategy) {}
  
  calculate(amount: number): number {
    // Este método NUNCA cambia, sin importar cuántos tipos de descuento agreguemos
    return this.strategy.calculateDiscount(amount);
  }
  
  setStrategy(strategy: DiscountStrategy): void {
    // Permite cambiar la estrategia en runtime
    this.strategy = strategy;
  }
  
  getDescription(): string {
    return this.strategy.getDescription?.() || 'No discount description available';
  }
}

// 6. Factory para crear estrategias basadas en reglas de negocio
class DiscountStrategyFactory {
  static createForCustomer(customer: Customer): DiscountStrategy {
    // La lógica de selección está centralizada aquí
    const strategies: DiscountStrategy[] = [];
    
    // Agregar descuento por tipo de cliente
    switch (customer.type) {
      case 'regular':
        strategies.push(new RegularCustomerDiscount());
        break;
      case 'premium':
        strategies.push(new PremiumCustomerDiscount());
        break;
      case 'vip':
        strategies.push(new VIPCustomerDiscount());
        break;
    }
    
    // Agregar descuento por primera compra si aplica
    if (customer.isFirstPurchase) {
      strategies.push(new FirstPurchaseDiscount());
    }
    
    // Agregar descuento de temporada si hay uno activo
    const currentSeason = this.getCurrentSeason();
    if (this.hasSeasonalPromotion(currentSeason)) {
      strategies.push(new SeasonalDiscount(currentSeason, 0.1));
    }
    
    // Retornar estrategia única o compuesta
    if (strategies.length === 0) {
      return new NoDiscount();
    } else if (strategies.length === 1) {
      return strategies[0];
    } else {
      return new CompositeDiscount(strategies);
    }
  }
  
  private static getCurrentSeason(): 'summer' | 'winter' | 'spring' | 'fall' {
    const month = new Date().getMonth();
    if (month >= 2 && month <= 4) return 'spring';
    if (month >= 5 && month <= 7) return 'summer';
    if (month >= 8 && month <= 10) return 'fall';
    return 'winter';
  }
  
  private static hasSeasonalPromotion(season: string): boolean {
    // Lógica para determinar si hay promoción de temporada
    return season === 'summer' || season === 'winter';
  }
}

// Clase para cuando no hay descuento
class NoDiscount implements DiscountStrategy {
  calculateDiscount(amount: number): number {
    return amount;
  }
  
  getDescription(): string {
    return 'No discount applied';
  }
}

// USO DEL SISTEMA
interface Customer {
  id: string;
  type: 'regular' | 'premium' | 'vip';
  isFirstPurchase: boolean;
}

// Ejemplo de uso
const customer: Customer = {
  id: '123',
  type: 'premium',
  isFirstPurchase: true
};

const strategy = DiscountStrategyFactory.createForCustomer(customer);
const calculator = new DiscountCalculator(strategy);

const originalAmount = 100;
const discountedAmount = calculator.calculate(originalAmount);

console.log(`Original: $${originalAmount}`);
console.log(`Final: $${discountedAmount}`);
console.log(`Discount applied: ${calculator.getDescription()}`);
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
// ❌ PROBLEMA: Violación clásica de LSP con Rectangle y Square

// Clase base Rectangle
class Rectangle {
  constructor(
    protected width: number, 
    protected height: number
  ) {}
  
  setWidth(width: number): void {
    this.width = width;
  }
  
  setHeight(height: number): void {
    this.height = height;
  }
  
  getWidth(): number {
    return this.width;
  }
  
  getHeight(): number {
    return this.height;
  }
  
  getArea(): number {
    return this.width * this.height;
  }
}

// Square hereda de Rectangle - PROBLEMA!
class Square extends Rectangle {
  constructor(side: number) {
    super(side, side);
  }
  
  // VIOLACIÓN DE LSP: Cambia el comportamiento esperado de setWidth
  setWidth(width: number): void {
    // Un cuadrado debe mantener width === height
    this.width = width;
    this.height = width; // Efecto secundario no esperado!
  }
  
  // VIOLACIÓN DE LSP: Cambia el comportamiento esperado de setHeight
  setHeight(height: number): void {
    this.width = height; // Efecto secundario no esperado!
    this.height = height;
  }
}

// Demostración de por qué esto viola LSP
function demonstrateLSPViolation() {
  // Esta función debería funcionar con cualquier Rectangle
  function testRectangle(rectangle: Rectangle) {
    rectangle.setWidth(5);
    rectangle.setHeight(4);
    
    // Expectativa: width = 5, height = 4, area = 20
    console.log(`Width: ${rectangle.getWidth()}`); 
    console.log(`Height: ${rectangle.getHeight()}`);
    console.log(`Area: ${rectangle.getArea()}`);
    
    // Para Rectangle: width = 5, height = 4, area = 20 ✅
    // Para Square: width = 4, height = 4, area = 16 ❌ (Violación!)
    
    const expectedArea = 5 * 4;
    const actualArea = rectangle.getArea();
    
    if (actualArea !== expectedArea) {
      throw new Error(`LSP Violation! Expected area ${expectedArea}, got ${actualArea}`);
    }
  }
  
  const rectangle = new Rectangle(3, 4);
  testRectangle(rectangle); // Funciona correctamente
  
  const square = new Square(5);
  testRectangle(square); // Lanza error! Viola LSP
}

// ✅ SOLUCIÓN 1: Usar composición en lugar de herencia

interface Shape {
  getArea(): number;
  getPerimeter(): number;
  describe(): string;
}

// Rectangle como una implementación independiente
class Rectangle implements Shape {
  constructor(
    private width: number,
    private height: number
  ) {}
  
  setDimensions(width: number, height: number): void {
    this.width = width;
    this.height = height;
  }
  
  getWidth(): number {
    return this.width;
  }
  
  getHeight(): number {
    return this.height;
  }
  
  getArea(): number {
    return this.width * this.height;
  }
  
  getPerimeter(): number {
    return 2 * (this.width + this.height);
  }
  
  describe(): string {
    return `Rectangle: ${this.width}x${this.height}`;
  }
}

// Square como una implementación independiente
class Square implements Shape {
  constructor(private side: number) {}
  
  setSide(side: number): void {
    this.side = side;
  }
  
  getSide(): number {
    return this.side;
  }
  
  getArea(): number {
    return this.side * this.side;
  }
  
  getPerimeter(): number {
    return 4 * this.side;
  }
  
  describe(): string {
    return `Square: ${this.side}x${this.side}`;
  }
}

// ✅ SOLUCIÓN 2: Hacer Rectangle inmutable y usar factory methods

abstract class ImmutableShape {
  abstract getArea(): number;
  abstract getPerimeter(): number;
  abstract scale(factor: number): ImmutableShape;
}

class ImmutableRectangle extends ImmutableShape {
  constructor(
    private readonly width: number,
    private readonly height: number
  ) {
    super();
  }
  
  getWidth(): number {
    return this.width;
  }
  
  getHeight(): number {
    return this.height;
  }
  
  getArea(): number {
    return this.width * this.height;
  }
  
  getPerimeter(): number {
    return 2 * (this.width + this.height);
  }
  
  // Retorna una nueva instancia en lugar de modificar
  scale(factor: number): ImmutableRectangle {
    return new ImmutableRectangle(
      this.width * factor,
      this.height * factor
    );
  }
  
  withWidth(width: number): ImmutableRectangle {
    return new ImmutableRectangle(width, this.height);
  }
  
  withHeight(height: number): ImmutableRectangle {
    return new ImmutableRectangle(this.width, height);
  }
}

class ImmutableSquare extends ImmutableShape {
  constructor(private readonly side: number) {
    super();
  }
  
  getSide(): number {
    return this.side;
  }
  
  getArea(): number {
    return this.side * this.side;
  }
  
  getPerimeter(): number {
    return 4 * this.side;
  }
  
  scale(factor: number): ImmutableSquare {
    return new ImmutableSquare(this.side * factor);
  }
  
  withSide(side: number): ImmutableSquare {
    return new ImmutableSquare(side);
  }
}

// ✅ SOLUCIÓN 3: Ejemplo más complejo - Sistema de Notificaciones

// Mal ejemplo que viola LSP
abstract class Notification {
  abstract send(message: string): void;
}

class EmailNotification extends Notification {
  send(message: string): void {
    console.log(`Sending email: ${message}`);
  }
}

class SMSNotification extends Notification {
  send(message: string): void {
    // VIOLACIÓN: SMS tiene límite de caracteres
    if (message.length > 160) {
      throw new Error('SMS message too long'); // Violación de LSP!
    }
    console.log(`Sending SMS: ${message}`);
  }
}

// Solución correcta que respeta LSP
interface NotificationSender {
  send(message: string): NotificationResult;
  canSend(message: string): boolean;
  getMaxMessageLength(): number | null;
}

interface NotificationResult {
  success: boolean;
  error?: string;
  messageId?: string;
}

class EmailNotificationSender implements NotificationSender {
  send(message: string): NotificationResult {
    if (!this.canSend(message)) {
      return {
        success: false,
        error: 'Message exceeds maximum length'
      };
    }
    
    // Enviar email
    console.log(`Sending email: ${message}`);
    return {
      success: true,
      messageId: `email-${Date.now()}`
    };
  }
  
  canSend(message: string): boolean {
    const maxLength = this.getMaxMessageLength();
    return maxLength === null || message.length <= maxLength;
  }
  
  getMaxMessageLength(): number | null {
    return 100000; // Límite muy alto para emails
  }
}

class SMSNotificationSender implements NotificationSender {
  private readonly MAX_SMS_LENGTH = 160;
  
  send(message: string): NotificationResult {
    if (!this.canSend(message)) {
      // Intentar dividir el mensaje
      const parts = this.splitMessage(message);
      console.log(`Sending ${parts.length} SMS parts`);
      
      return {
        success: true,
        messageId: `sms-multipart-${Date.now()}`
      };
    }
    
    console.log(`Sending SMS: ${message}`);
    return {
      success: true,
      messageId: `sms-${Date.now()}`
    };
  }
  
  canSend(message: string): boolean {
    // SMS siempre puede enviar (dividiendo si es necesario)
    return true;
  }
  
  getMaxMessageLength(): number {
    return this.MAX_SMS_LENGTH;
  }
  
  private splitMessage(message: string): string[] {
    const parts: string[] = [];
    for (let i = 0; i < message.length; i += this.MAX_SMS_LENGTH) {
      parts.push(message.substring(i, i + this.MAX_SMS_LENGTH));
    }
    return parts;
  }
}

// Cliente que usa las notificaciones sin preocuparse por los detalles
class NotificationService {
  constructor(private senders: NotificationSender[]) {}
  
  broadcast(message: string): void {
    for (const sender of this.senders) {
      const result = sender.send(message);
      
      if (result.success) {
        console.log(`✅ Notification sent: ${result.messageId}`);
      } else {
        console.log(`❌ Failed to send: ${result.error}`);
      }
    }
  }
}

// Uso correcto que respeta LSP
const notificationService = new NotificationService([
  new EmailNotificationSender(),
  new SMSNotificationSender()
]);

// Funciona correctamente con cualquier tipo de notificación
notificationService.broadcast('Hello World!');
notificationService.broadcast('A'.repeat(200)); // Mensaje largo
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
  // Métodos de información básica
  getId(): string;
  getName(): string;
  getEmail(): string;
  
  // Métodos de trabajo
  work(): void;
  takeBreak(): void;
  attendMeeting(): void;
  
  // Métodos de desarrollo (no todos los empleados programan!)
  writeCode(): void;
  reviewCode(): void;
  deployCode(): void;
  
  // Métodos de gestión (no todos los empleados gestionan!)
  approveTimeOff(): void;
  conductPerformanceReview(): void;
  manageTeam(): void;
  
  // Métodos de ventas (no todos venden!)
  makeSale(): void;
  callClient(): void;
  prepareQuote(): void;
  
  // Métodos biológicos (¿qué pasa con robots o IA?)
  eat(): void;
  sleep(): void;
}

// Problemas al implementar esta interface:

class Developer implements Employee {
  getId(): string { return 'dev-001'; }
  getName(): string { return 'John Developer'; }
  getEmail(): string { return 'john@company.com'; }
  
  work(): void { console.log('Writing code...'); }
  takeBreak(): void { console.log('Taking a coffee break...'); }
  attendMeeting(): void { console.log('In standup meeting...'); }
  
  writeCode(): void { console.log('Writing clean code...'); }
  reviewCode(): void { console.log('Reviewing PR...'); }
  deployCode(): void { console.log('Deploying to production...'); }
  
  // ❌ PROBLEMA: Developer no gestiona equipos!
  approveTimeOff(): void { 
    throw new Error('Developers cannot approve time off');
  }
  conductPerformanceReview(): void { 
    throw new Error('Developers cannot conduct reviews');
  }
  manageTeam(): void { 
    throw new Error('Developers do not manage teams');
  }
  
  // ❌ PROBLEMA: Developer no hace ventas!
  makeSale(): void { 
    throw new Error('Developers do not make sales');
  }
  callClient(): void { 
    throw new Error('Developers do not call clients');
  }
  prepareQuote(): void { 
    throw new Error('Developers do not prepare quotes');
  }
  
  eat(): void { console.log('Eating lunch...'); }
  sleep(): void { console.log('Power napping...'); }
}

class Manager implements Employee {
  // ... debe implementar TODOS los métodos incluso los de desarrollo y ventas!
  writeCode(): void { 
    throw new Error('Managers do not write code');
  }
  // ... más métodos no aplicables
}

class RobotWorker implements Employee {
  // ... debe implementar métodos biológicos!
  eat(): void { 
    throw new Error('Robots do not eat'); // ❌ Violación de LSP también!
  }
  sleep(): void { 
    throw new Error('Robots do not sleep');
  }
  // ...
}

// ✅ SOLUCIÓN: Aplicar ISP segregando interfaces

// 1. Interface base con solo lo esencial
interface Identifiable {
  getId(): string;
}

interface Person {
  getName(): string;
  getEmail(): string;
}

// 2. Interfaces segregadas por capacidad/responsabilidad
interface Workable {
  work(): void;
  takeBreak(): void;
}

interface MeetingParticipant {
  attendMeeting(): void;
  scheduleMeeting?(time: Date): void;
}

interface Programmer {
  writeCode(): void;
  reviewCode(pullRequestId: string): void;
  debugCode(): void;
}

interface Deployable {
  deployCode(environment: 'dev' | 'staging' | 'prod'): void;
  rollbackDeployment(): void;
}

interface TeamManager {
  manageTeam(teamId: string): void;
  approveTimeOff(requestId: string): boolean;
  conductPerformanceReview(employeeId: string): void;
  assignTasks(): void;
}

interface SalesPerson {
  makeSale(product: string, amount: number): void;
  callClient(clientId: string): void;
  prepareQuote(requirements: string[]): Quote;
  followUpLead(leadId: string): void;
}

interface BiologicalNeeds {
  eat(): void;
  sleep(): void;
  rest(): void;
}

// 3. Interfaces compuestas para roles específicos
interface DeveloperRole extends 
  Identifiable, 
  Person, 
  Workable, 
  MeetingParticipant, 
  Programmer {}

interface SeniorDeveloperRole extends 
  DeveloperRole, 
  Deployable {}

interface TechLeadRole extends 
  SeniorDeveloperRole, 
  TeamManager {}

interface SalesRole extends 
  Identifiable, 
  Person, 
  Workable, 
  MeetingParticipant, 
  SalesPerson {}

interface HumanEmployee extends 
  Identifiable, 
  Person, 
  BiologicalNeeds {}

interface RobotEmployee extends 
  Identifiable, 
  Workable {}

// 4. Implementaciones limpias y enfocadas

class JuniorDeveloper implements DeveloperRole, BiologicalNeeds {
  constructor(
    private id: string,
    private name: string,
    private email: string
  ) {}
  
  // Identifiable
  getId(): string { return this.id; }
  
  // Person
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  
  // Workable
  work(): void {
    console.log(`${this.name} is coding...`);
  }
  
  takeBreak(): void {
    console.log(`${this.name} is taking a break...`);
  }
  
  // MeetingParticipant
  attendMeeting(): void {
    console.log(`${this.name} is in a meeting...`);
  }
  
  // Programmer
  writeCode(): void {
    console.log(`${this.name} is writing code...`);
  }
  
  reviewCode(pullRequestId: string): void {
    console.log(`${this.name} is reviewing PR #${pullRequestId}...`);
  }
  
  debugCode(): void {
    console.log(`${this.name} is debugging...`);
  }
  
  // BiologicalNeeds
  eat(): void {
    console.log(`${this.name} is having lunch...`);
  }
  
  sleep(): void {
    console.log(`${this.name} is sleeping...`);
  }
  
  rest(): void {
    console.log(`${this.name} is resting...`);
  }
}

class SeniorDeveloper implements SeniorDeveloperRole, BiologicalNeeds {
  constructor(
    private id: string,
    private name: string,
    private email: string
  ) {}
  
  // ... implementa métodos de DeveloperRole ...
  
  // Deployable (adicional para Senior)
  deployCode(environment: 'dev' | 'staging' | 'prod'): void {
    console.log(`${this.name} is deploying to ${environment}...`);
  }
  
  rollbackDeployment(): void {
    console.log(`${this.name} is rolling back deployment...`);
  }
  
  // ... resto de implementaciones ...
  getId(): string { return this.id; }
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  work(): void { console.log('Working...'); }
  takeBreak(): void { console.log('Taking break...'); }
  attendMeeting(): void { console.log('In meeting...'); }
  writeCode(): void { console.log('Writing code...'); }
  reviewCode(prId: string): void { console.log(`Reviewing ${prId}...`); }
  debugCode(): void { console.log('Debugging...'); }
  eat(): void { console.log('Eating...'); }
  sleep(): void { console.log('Sleeping...'); }
  rest(): void { console.log('Resting...'); }
}

class TechLead implements TechLeadRole, BiologicalNeeds {
  constructor(
    private id: string,
    private name: string,
    private email: string,
    private teamMembers: string[]
  ) {}
  
  // ... implementa métodos de SeniorDeveloperRole ...
  
  // TeamManager (adicional para Tech Lead)
  manageTeam(teamId: string): void {
    console.log(`${this.name} is managing team ${teamId}...`);
  }
  
  approveTimeOff(requestId: string): boolean {
    console.log(`${this.name} is reviewing time off request ${requestId}...`);
    return true;
  }
  
  conductPerformanceReview(employeeId: string): void {
    console.log(`${this.name} is conducting review for ${employeeId}...`);
  }
  
  assignTasks(): void {
    console.log(`${this.name} is assigning tasks to team...`);
  }
  
  // ... resto de implementaciones ...
  getId(): string { return this.id; }
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  work(): void { console.log('Working...'); }
  takeBreak(): void { console.log('Taking break...'); }
  attendMeeting(): void { console.log('In meeting...'); }
  writeCode(): void { console.log('Writing code...'); }
  reviewCode(prId: string): void { console.log(`Reviewing ${prId}...`); }
  debugCode(): void { console.log('Debugging...'); }
  deployCode(env: 'dev' | 'staging' | 'prod'): void { 
    console.log(`Deploying to ${env}...`); 
  }
  rollbackDeployment(): void { console.log('Rolling back...'); }
  eat(): void { console.log('Eating...'); }
  sleep(): void { console.log('Sleeping...'); }
  rest(): void { console.log('Resting...'); }
}

class SalesRepresentative implements SalesRole, BiologicalNeeds {
  constructor(
    private id: string,
    private name: string,
    private email: string
  ) {}
  
  // ... Solo implementa métodos relevantes para ventas
  // No necesita implementar métodos de programación!
  
  getId(): string { return this.id; }
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  
  work(): void {
    console.log(`${this.name} is working on sales...`);
  }
  
  takeBreak(): void {
    console.log(`${this.name} is taking a break...`);
  }
  
  attendMeeting(): void {
    console.log(`${this.name} is in a sales meeting...`);
  }
  
  makeSale(product: string, amount: number): void {
    console.log(`${this.name} sold ${product} for $${amount}`);
  }
  
  callClient(clientId: string): void {
    console.log(`${this.name} is calling client ${clientId}...`);
  }
  
  prepareQuote(requirements: string[]): Quote {
    console.log(`${this.name} is preparing a quote...`);
    return { id: 'Q001', amount: 10000, requirements };
  }
  
  followUpLead(leadId: string): void {
    console.log(`${this.name} is following up on lead ${leadId}...`);
  }
  
  eat(): void { console.log('Having lunch...'); }
  sleep(): void { console.log('Sleeping...'); }
  rest(): void { console.log('Resting...'); }
}

class AIAssistant implements RobotEmployee, Programmer {
  constructor(private id: string) {}
  
  // RobotEmployee - No necesita métodos biológicos!
  getId(): string { return this.id; }
  
  work(): void {
    console.log('AI is processing tasks...');
  }
  
  // Programmer
  writeCode(): void {
    console.log('AI is generating code...');
  }
  
  reviewCode(pullRequestId: string): void {
    console.log(`AI is analyzing PR #${pullRequestId}...`);
  }
  
  debugCode(): void {
    console.log('AI is identifying bugs...');
  }
  
  // No necesita eat(), sleep(), etc.!
}

interface Quote {
  id: string;
  amount: number;
  requirements: string[];
}

// 5. Uso del sistema con interfaces segregadas

class WorkScheduler {
  // Solo necesita saber sobre Workable
  scheduleWork(workers: Workable[]): void {
    workers.forEach(worker => {
      worker.work();
      setTimeout(() => worker.takeBreak(), 3600000);
    });
  }
}

class MeetingOrganizer {
  // Solo necesita saber sobre MeetingParticipant
  organizeMeeting(participants: MeetingParticipant[]): void {
    participants.forEach(p => p.attendMeeting());
  }
}

class CodeReviewSystem {
  // Solo necesita saber sobre Programmer
  assignReview(reviewer: Programmer, pullRequestId: string): void {
    reviewer.reviewCode(pullRequestId);
  }
}

class DeploymentPipeline {
  // Solo necesita saber sobre Deployable
  deploy(deployer: Deployable, environment: 'dev' | 'staging' | 'prod'): void {
    try {
      deployer.deployCode(environment);
    } catch (error) {
      console.error('Deployment failed, rolling back...');
      deployer.rollbackDeployment();
    }
  }
}

// Ejemplo de uso
const junior = new JuniorDeveloper('j001', 'Alice', 'alice@company.com');
const senior = new SeniorDeveloper('s001', 'Bob', 'bob@company.com');
const techLead = new TechLead('tl001', 'Carol', 'carol@company.com', ['j001', 's001']);
const sales = new SalesRepresentative('sr001', 'Dave', 'dave@company.com');
const ai = new AIAssistant('ai001');

// El scheduler solo conoce la interface Workable
const scheduler = new WorkScheduler();
scheduler.scheduleWork([junior, senior, techLead, sales, ai]);

// El sistema de review solo conoce Programmer
const reviewSystem = new CodeReviewSystem();
reviewSystem.assignReview(junior, 'PR-123');
reviewSystem.assignReview(ai, 'PR-124');
// reviewSystem.assignReview(sales, 'PR-125'); // ❌ Error de compilación!

// El pipeline solo conoce Deployable
const pipeline = new DeploymentPipeline();
// pipeline.deploy(junior, 'prod'); // ❌ Error de compilación!
pipeline.deploy(senior, 'staging'); // ✅ OK
pipeline.deploy(techLead, 'prod'); // ✅ OK
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

// Módulo de bajo nivel - Implementación específica de base de datos
class MySQLDatabase {
  connect(): void {
    console.log('Connecting to MySQL...');
  }
  
  query(sql: string): any[] {
    console.log(`Executing MySQL query: ${sql}`);
    return [];
  }
  
  close(): void {
    console.log('Closing MySQL connection');
  }
}

// Módulo de bajo nivel - Implementación específica de email
class GmailService {
  authenticate(): void {
    console.log('Authenticating with Gmail API...');
  }
  
  sendEmail(to: string, subject: string, body: string): void {
    console.log(`Sending email via Gmail to ${to}`);
  }
}

// Módulo de bajo nivel - Implementación específica de pagos
class StripePaymentGateway {
  processPayment(amount: number, currency: string): boolean {
    console.log(`Processing ${currency} ${amount} via Stripe`);
    return true;
  }
  
  refund(transactionId: string): boolean {
    console.log(`Refunding transaction ${transactionId} via Stripe`);
    return true;
  }
}

// ❌ PROBLEMA: Módulo de alto nivel con dependencias directas
// Esta clase está fuertemente acoplada a implementaciones específicas
class OrderService {
  private database: MySQLDatabase;      // ❌ Dependencia directa
  private emailService: GmailService;   // ❌ Dependencia directa
  private paymentGateway: StripePaymentGateway; // ❌ Dependencia directa
  
  constructor() {
    // ❌ PROBLEMA: Creación directa de dependencias
    // No podemos cambiar las implementaciones sin modificar esta clase
    this.database = new MySQLDatabase();
    this.emailService = new GmailService();
    this.paymentGateway = new StripePaymentGateway();
    
    this.database.connect();
    this.emailService.authenticate();
  }
  
  createOrder(orderData: any): void {
    // ❌ PROBLEMA: Lógica de negocio mezclada con detalles de implementación
    
    // Guardar orden - acoplado a MySQL
    const sql = `INSERT INTO orders (customer_id, total) VALUES (?, ?)`;
    this.database.query(sql);
    
    // Procesar pago - acoplado a Stripe
    const paymentSuccess = this.paymentGateway.processPayment(
      orderData.total, 
      'USD'
    );
    
    if (paymentSuccess) {
      // Enviar confirmación - acoplado a Gmail
      this.emailService.sendEmail(
        orderData.customerEmail,
        'Order Confirmation',
        `Your order has been confirmed!`
      );
    }
    
    // ❌ PROBLEMAS:
    // 1. No podemos cambiar de MySQL a MongoDB sin modificar esta clase
    // 2. No podemos cambiar de Gmail a SendGrid sin modificar esta clase
    // 3. No podemos cambiar de Stripe a PayPal sin modificar esta clase
    // 4. No podemos testear sin conectarnos a servicios reales
    // 5. Si cambia la API de cualquier servicio, debemos modificar esta clase
  }
}

// ✅ SOLUCIÓN: Aplicar DIP usando abstracciones

// 1. Definir abstracciones (interfaces) para cada dependencia

// Abstracción para persistencia
interface Repository<T> {
  save(entity: T): Promise<void>;
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  update(id: string, entity: Partial<T>): Promise<void>;
  delete(id: string): Promise<void>;
}

// Abstracción para notificaciones
interface NotificationService {
  sendNotification(notification: Notification): Promise<NotificationResult>;
  sendBulkNotifications(notifications: Notification[]): Promise<NotificationResult[]>;
}

interface Notification {
  recipient: string;
  subject: string;
  message: string;
  type: 'email' | 'sms' | 'push';
  metadata?: Record<string, any>;
}

interface NotificationResult {
  success: boolean;
  messageId?: string;
  error?: string;
}

// Abstracción para procesamiento de pagos
interface PaymentProcessor {
  processPayment(payment: PaymentRequest): Promise<PaymentResult>;
  refundPayment(transactionId: string, amount?: number): Promise<RefundResult>;
  getTransactionStatus(transactionId: string): Promise<TransactionStatus>;
}

interface PaymentRequest {
  amount: number;
  currency: string;
  customerId: string;
  description?: string;
  metadata?: Record<string, any>;
}

interface PaymentResult {
  success: boolean;
  transactionId?: string;
  error?: string;
}

interface RefundResult {
  success: boolean;
  refundId?: string;
  error?: string;
}

interface TransactionStatus {
  status: 'pending' | 'completed' | 'failed' | 'refunded';
  amount: number;
  currency: string;
}

// 2. Implementaciones concretas que dependen de las abstracciones

// Implementación de Repository para MySQL
class MySQLOrderRepository implements Repository<Order> {
  constructor(private connection: MySQLConnection) {}
  
  async save(order: Order): Promise<void> {
    const query = `
      INSERT INTO orders (id, customer_id, total, status, created_at)
      VALUES (?, ?, ?, ?, ?)
    `;
    
    await this.connection.execute(query, [
      order.id,
      order.customerId,
      order.total,
      order.status,
      order.createdAt
    ]);
    
    // Guardar items de la orden
    for (const item of order.items) {
      await this.saveOrderItem(order.id, item);
    }
  }
  
  async findById(id: string): Promise<Order | null> {
    const query = `SELECT * FROM orders WHERE id = ?`;
    const result = await this.connection.execute(query, [id]);
    
    if (!result || result.length === 0) {
      return null;
    }
    
    const orderData = result[0];
    const items = await this.getOrderItems(id);
    
    return new Order(
      orderData.id,
      orderData.customer_id,
      items,
      orderData.total,
      orderData.status,
      orderData.created_at
    );
  }
  
  async findAll(): Promise<Order[]> {
    const query = `SELECT * FROM orders`;
    const results = await this.connection.execute(query);
    
    const orders: Order[] = [];
    for (const row of results) {
      const items = await this.getOrderItems(row.id);
      orders.push(new Order(
        row.id,
        row.customer_id,
        items,
        row.total,
        row.status,
        row.created_at
      ));
    }
    
    return orders;
  }
  
  async update(id: string, updates: Partial<Order>): Promise<void> {
    const fields = Object.keys(updates)
      .map(key => `${key} = ?`)
      .join(', ');
    
    const query = `UPDATE orders SET ${fields} WHERE id = ?`;
    const values = [...Object.values(updates), id];
    
    await this.connection.execute(query, values);
  }
  
  async delete(id: string): Promise<void> {
    await this.connection.execute(`DELETE FROM order_items WHERE order_id = ?`, [id]);
    await this.connection.execute(`DELETE FROM orders WHERE id = ?`, [id]);
  }
  
  private async saveOrderItem(orderId: string, item: OrderItem): Promise<void> {
    const query = `
      INSERT INTO order_items (order_id, product_id, quantity, price)
      VALUES (?, ?, ?, ?)
    `;
    
    await this.connection.execute(query, [
      orderId,
      item.productId,
      item.quantity,
      item.price
    ]);
  }
  
  private async getOrderItems(orderId: string): Promise<OrderItem[]> {
    const query = `SELECT * FROM order_items WHERE order_id = ?`;
    const results = await this.connection.execute(query, [orderId]);
    
    return results.map((row: any) => ({
      productId: row.product_id,
      quantity: row.quantity,
      price: row.price
    }));
  }
}

// Implementación alternativa para MongoDB
class MongoOrderRepository implements Repository<Order> {
  constructor(private collection: MongoCollection) {}
  
  async save(order: Order): Promise<void> {
    await this.collection.insertOne({
      _id: order.id,
      customerId: order.customerId,
      items: order.items,
      total: order.total,
      status: order.status,
      createdAt: order.createdAt
    });
  }
  
  async findById(id: string): Promise<Order | null> {
    const doc = await this.collection.findOne({ _id: id });
    
    if (!doc) return null;
    
    return new Order(
      doc._id,
      doc.customerId,
      doc.items,
      doc.total,
      doc.status,
      doc.createdAt
    );
  }
  
  async findAll(): Promise<Order[]> {
    const docs = await this.collection.find({}).toArray();
    return docs.map(doc => new Order(
      doc._id,
      doc.customerId,
      doc.items,
      doc.total,
      doc.status,
      doc.createdAt
    ));
  }
  
  async update(id: string, updates: Partial<Order>): Promise<void> {
    await this.collection.updateOne(
      { _id: id },
      { $set: updates }
    );
  }
  
  async delete(id: string): Promise<void> {
    await this.collection.deleteOne({ _id: id });
  }
}

// Implementación de NotificationService para Email
class EmailNotificationService implements NotificationService {
  constructor(private emailProvider: EmailProvider) {}
  
  async sendNotification(notification: Notification): Promise<NotificationResult> {
    if (notification.type !== 'email') {
      return {
        success: false,
        error: 'This service only handles email notifications'
      };
    }
    
    try {
      const messageId = await this.emailProvider.send({
        to: notification.recipient,
        subject: notification.subject,
        body: notification.message,
        ...notification.metadata
      });
      
      return {
        success: true,
        messageId
      };
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  async sendBulkNotifications(
    notifications: Notification[]
  ): Promise<NotificationResult[]> {
    const results: NotificationResult[] = [];
    
    for (const notification of notifications) {
      const result = await this.sendNotification(notification);
      results.push(result);
    }
    
    return results;
  }
}

// Implementación de PaymentProcessor para Stripe
class StripePaymentProcessor implements PaymentProcessor {
  constructor(private stripeClient: StripeClient) {}
  
  async processPayment(payment: PaymentRequest): Promise<PaymentResult> {
    try {
      const charge = await this.stripeClient.charges.create({
        amount: Math.round(payment.amount * 100), // Stripe usa centavos
        currency: payment.currency.toLowerCase(),
        customer: payment.customerId,
        description: payment.description,
        metadata: payment.metadata
      });
      
      return {
        success: true,
        transactionId: charge.id
      };
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  async refundPayment(
    transactionId: string, 
    amount?: number
  ): Promise<RefundResult> {
    try {
      const refund = await this.stripeClient.refunds.create({
        charge: transactionId,
        amount: amount ? Math.round(amount * 100) : undefined
      });
      
      return {
        success: true,
        refundId: refund.id
      };
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  async getTransactionStatus(transactionId: string): Promise<TransactionStatus> {
    const charge = await this.stripeClient.charges.retrieve(transactionId);
    
    let status: 'pending' | 'completed' | 'failed' | 'refunded';
    if (charge.refunded) {
      status = 'refunded';
    } else if (charge.paid) {
      status = 'completed';
    } else if (charge.status === 'failed') {
      status = 'failed';
    } else {
      status = 'pending';
    }
    
    return {
      status,
      amount: charge.amount / 100,
      currency: charge.currency.toUpperCase()
    };
  }
}

// 3. Módulo de alto nivel que depende solo de abstracciones
class OrderService {
  constructor(
    private orderRepository: Repository<Order>,      // ✅ Abstracción
    private notificationService: NotificationService, // ✅ Abstracción
    private paymentProcessor: PaymentProcessor,      // ✅ Abstracción
    private logger: Logger                           // ✅ Abstracción
  ) {
    // Las dependencias se inyectan, no se crean internamente
    // OrderService no sabe ni le importa qué implementaciones concretas se usan
  }
  
  async createOrder(orderData: CreateOrderRequest): Promise<OrderResult> {
    try {
      // 1. Validar datos de la orden
      const validation = this.validateOrderData(orderData);
      if (!validation.isValid) {
        return {
          success: false,
          error: validation.error
        };
      }
      
      // 2. Crear la orden
      const order = new Order(
        generateId(),
        orderData.customerId,
        orderData.items,
        this.calculateTotal(orderData.items),
        'pending',
        new Date()
      );
      
      // 3. Procesar el pago
      const paymentResult = await this.paymentProcessor.processPayment({
        amount: order.total,
        currency: orderData.currency || 'USD',
        customerId: order.customerId,
        description: `Order ${order.id}`,
        metadata: { orderId: order.id }
      });
      
      if (!paymentResult.success) {
        this.logger.error(`Payment failed for order ${order.id}: ${paymentResult.error}`);
        return {
          success: false,
          error: 'Payment processing failed'
        };
      }
      
      // 4. Actualizar estado de la orden
      order.status = 'confirmed';
      order.transactionId = paymentResult.transactionId;
      
      // 5. Guardar la orden
      await this.orderRepository.save(order);
      
      // 6. Enviar notificación
      await this.notificationService.sendNotification({
        recipient: orderData.customerEmail,
        subject: 'Order Confirmation',
        message: this.generateOrderConfirmationMessage(order),
        type: 'email',
        metadata: {
          orderId: order.id,
          template: 'order-confirmation'
        }
      });
      
      this.logger.info(`Order ${order.id} created successfully`);
      
      return {
        success: true,
        orderId: order.id,
        order
      };
      
    } catch (error) {
      this.logger.error(`Error creating order: ${error.message}`, error);
      return {
        success: false,
        error: 'An unexpected error occurred'
      };
    }
  }
  
  async cancelOrder(orderId: string): Promise<CancelResult> {
    try {
      // 1. Buscar la orden
      const order = await this.orderRepository.findById(orderId);
      
      if (!order) {
        return {
          success: false,
          error: 'Order not found'
        };
      }
      
      if (order.status === 'cancelled') {
        return {
          success: false,
          error: 'Order is already cancelled'
        };
      }
      
      // 2. Procesar reembolso si hay transacción
      if (order.transactionId) {
        const refundResult = await this.paymentProcessor.refundPayment(
          order.transactionId
        );
        
        if (!refundResult.success) {
          this.logger.error(`Refund failed for order ${orderId}: ${refundResult.error}`);
          return {
            success: false,
            error: 'Refund processing failed'
          };
        }
        
        order.refundId = refundResult.refundId;
      }
      
      // 3. Actualizar estado de la orden
      order.status = 'cancelled';
      order.cancelledAt = new Date();
      
      await this.orderRepository.update(orderId, {
        status: order.status,
        refundId: order.refundId,
        cancelledAt: order.cancelledAt
      });
      
      // 4. Notificar al cliente
      await this.notificationService.sendNotification({
        recipient: order.customerEmail,
        subject: 'Order Cancellation',
        message: `Your order ${orderId} has been cancelled and refunded.`,
        type: 'email',
        metadata: {
          orderId: order.id,
          template: 'order-cancellation'
        }
      });
      
      this.logger.info(`Order ${orderId} cancelled successfully`);
      
      return {
        success: true,
        refundId: order.refundId
      };
      
    } catch (error) {
      this.logger.error(`Error cancelling order ${orderId}: ${error.message}`, error);
      return {
        success: false,
        error: 'An unexpected error occurred'
      };
    }
  }
  
  private validateOrderData(data: CreateOrderRequest): ValidationResult {
    if (!data.customerId) {
      return { isValid: false, error: 'Customer ID is required' };
    }
    
    if (!data.items || data.items.length === 0) {
      return { isValid: false, error: 'Order must have at least one item' };
    }
    
    if (!data.customerEmail) {
      return { isValid: false, error: 'Customer email is required' };
    }
    
    return { isValid: true };
  }
  
  private calculateTotal(items: OrderItem[]): number {
    return items.reduce((total, item) => total + (item.price * item.quantity), 0);
  }
  
  private generateOrderConfirmationMessage(order: Order): string {
    return `
      Thank you for your order!
      
      Order ID: ${order.id}
      Total: $${order.total}
      Status: ${order.status}
      
      Items:
      ${order.items.map(item => 
        `- Product ${item.productId}: ${item.quantity} x $${item.price}`
      ).join('\n')}
      
      We'll send you tracking information once your order ships.
    `;
  }
}

// 4. Configuración e inyección de dependencias

// Contenedor de inversión de control (IoC Container)
class DIContainer {
  private services = new Map<string, any>();
  private factories = new Map<string, () => any>();
  
  register<T>(name: string, factory: () => T): void {
    this.factories.set(name, factory);
  }
  
  registerSingleton<T>(name: string, instance: T): void {
    this.services.set(name, instance);
  }
  
  resolve<T>(name: string): T {
    if (this.services.has(name)) {
      return this.services.get(name);
    }
    
    const factory = this.factories.get(name);
    if (!factory) {
      throw new Error(`Service ${name} not registered`);
    }
    
    const instance = factory();
    return instance;
  }
}

// Configuración de producción
function configureProductionContainer(): DIContainer {
  const container = new DIContainer();
  
  // Registrar implementaciones de producción
  container.registerSingleton('mysqlConnection', new MySQLConnection(config.mysql));
  container.registerSingleton('stripeClient', new StripeClient(config.stripe.apiKey));
  container.registerSingleton('emailProvider', new SendGridProvider(config.sendgrid));
  container.registerSingleton('logger', new ProductionLogger());
  
  container.register('orderRepository', () => 
    new MySQLOrderRepository(container.resolve('mysqlConnection'))
  );
  
  container.register('notificationService', () =>
    new EmailNotificationService(container.resolve('emailProvider'))
  );
  
  container.register('paymentProcessor', () =>
    new StripePaymentProcessor(container.resolve('stripeClient'))
  );
  
  container.register('orderService', () =>
    new OrderService(
      container.resolve('orderRepository'),
      container.resolve('notificationService'),
      container.resolve('paymentProcessor'),
      container.resolve('logger')
    )
  );
  
  return container;
}

// Configuración de testing
function configureTestContainer(): DIContainer {
  const container = new DIContainer();
  
  // Registrar mocks para testing
  container.registerSingleton('orderRepository', new InMemoryOrderRepository());
  container.registerSingleton('notificationService', new MockNotificationService());
  container.registerSingleton('paymentProcessor', new MockPaymentProcessor());
  container.registerSingleton('logger', new ConsoleLogger());
  
  container.register('orderService', () =>
    new OrderService(
      container.resolve('orderRepository'),
      container.resolve('notificationService'),
      container.resolve('paymentProcessor'),
      container.resolve('logger')
    )
  );
  
  return container;
}

// Uso del sistema
const container = process.env.NODE_ENV === 'test' 
  ? configureTestContainer() 
  : configureProductionContainer();

const orderService = container.resolve<OrderService>('orderService');

// El código de alto nivel no cambia, independiente de las implementaciones
await orderService.createOrder({
  customerId: 'CUST-123',
  customerEmail: 'customer@example.com',
  items: [
    { productId: 'PROD-1', quantity: 2, price: 29.99 },
    { productId: 'PROD-2', quantity: 1, price: 49.99 }
  ],
  currency: 'USD'
});

// Tipos auxiliares
interface Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  total: number;
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  createdAt: Date;
  transactionId?: string;
  refundId?: string;
  cancelledAt?: Date;
  customerEmail?: string;
}

interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

interface CreateOrderRequest {
  customerId: string;
  customerEmail: string;
  items: OrderItem[];
  currency?: string;
}

interface OrderResult {
  success: boolean;
  orderId?: string;
  order?: Order;
  error?: string;
}

interface CancelResult {
  success: boolean;
  refundId?: string;
  error?: string;
}

interface ValidationResult {
  isValid: boolean;
  error?: string;
}

interface Logger {
  debug(message: string, data?: any): void;
  info(message: string, data?: any): void;
  warn(message: string, data?: any): void;
  error(message: string, error?: any): void;
}

// Interfaces auxiliares para las implementaciones concretas
interface MySQLConnection {
  execute(query: string, params?: any[]): Promise<any>;
}

interface MongoCollection {
  insertOne(doc: any): Promise<void>;
  findOne(filter: any): Promise<any>;
  find(filter: any): { toArray(): Promise<any[]> };
  updateOne(filter: any, update: any): Promise<void>;
  deleteOne(filter: any): Promise<void>;
}

interface StripeClient {
  charges: {
    create(params: any): Promise<any>;
    retrieve(id: string): Promise<any>;
  };
  refunds: {
    create(params: any): Promise<any>;
  };
}

interface EmailProvider {
  send(email: any): Promise<string>;
}

// Implementaciones mock para testing
class InMemoryOrderRepository implements Repository<Order> {
  private orders = new Map<string, Order>();
  
  async save(order: Order): Promise<void> {
    this.orders.set(order.id, { ...order });
  }
  
  async findById(id: string): Promise<Order | null> {
    return this.orders.get(id) || null;
  }
  
  async findAll(): Promise<Order[]> {
    return Array.from(this.orders.values());
  }
  
  async update(id: string, updates: Partial<Order>): Promise<void> {
    const order = this.orders.get(id);
    if (order) {
      this.orders.set(id, { ...order, ...updates });
    }
  }
  
  async delete(id: string): Promise<void> {
    this.orders.delete(id);
  }
}

class MockNotificationService implements NotificationService {
  private sentNotifications: Notification[] = [];
  
  async sendNotification(notification: Notification): Promise<NotificationResult> {
    this.sentNotifications.push(notification);
    return { success: true, messageId: `mock-${Date.now()}` };
  }
  
  async sendBulkNotifications(
    notifications: Notification[]
  ): Promise<NotificationResult[]> {
    return Promise.all(notifications.map(n => this.sendNotification(n)));
  }
  
  getSentNotifications(): Notification[] {
    return this.sentNotifications;
  }
}

class MockPaymentProcessor implements PaymentProcessor {
  private shouldFail = false;
  
  setShouldFail(fail: boolean): void {
    this.shouldFail = fail;
  }
  
  async processPayment(payment: PaymentRequest): Promise<PaymentResult> {
    if (this.shouldFail) {
      return { success: false, error: 'Mock payment failed' };
    }
    return { success: true, transactionId: `mock-txn-${Date.now()}` };
  }
  
  async refundPayment(transactionId: string): Promise<RefundResult> {
    if (this.shouldFail) {
      return { success: false, error: 'Mock refund failed' };
    }
    return { success: true, refundId: `mock-refund-${Date.now()}` };
  }
  
  async getTransactionStatus(transactionId: string): Promise<TransactionStatus> {
    return { status: 'completed', amount: 100, currency: 'USD' };
  }
}

class ConsoleLogger implements Logger {
  debug(message: string, data?: any): void {
    console.debug(`[DEBUG] ${message}`, data);
  }
  
  info(message: string, data?: any): void {
    console.info(`[INFO] ${message}`, data);
  }
  
  warn(message: string, data?: any): void {
    console.warn(`[WARN] ${message}`, data);
  }
  
  error(message: string, error?: any): void {
    console.error(`[ERROR] ${message}`, error);
  }
}

class ProductionLogger implements Logger {
  debug(message: string, data?: any): void {
    // Enviar a servicio de logging
  }
  
  info(message: string, data?: any): void {
    // Enviar a servicio de logging
  }
  
  warn(message: string, data?: any): void {
    // Enviar a servicio de logging
  }
  
  error(message: string, error?: any): void {
    // Enviar a servicio de logging y alertas
  }
}

// Helpers
function generateId(): string {
  return `ORDER-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

declare const config: any;
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

Aplicar estos principios requiere práctica, pero los beneficios en mantenibilidad, escalabilidad y calidad del código son enormes. En tu entrevista con Zillow, poder explicar estos principios con ejemplos prácticos y demostrar cuándo y cómo aplicarlos será fundamental para mostrar tu nivel como Senior Software Engineer.