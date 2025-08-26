# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 4: Técnicas de Refactorización
### Para Entrevista Senior Software Engineer

---

## 4. Técnicas de Refactorización - Mejorando el Código Sistemáticamente

La refactorización es el proceso de cambiar la estructura interna del código sin modificar su comportamiento externo. Es una disciplina fundamental para mantener el código limpio, mantenible y extensible a lo largo del tiempo.

### ¿Por qué refactorizar?

1. **Mejorar el diseño del código**: El diseño se deteriora con el tiempo si no se mantiene
2. **Hacer el código más fácil de entender**: Código claro ahorra tiempo futuro
3. **Encontrar bugs**: El proceso de refactorización ayuda a descubrir problemas
4. **Programar más rápido**: Código limpio permite agregar features más rápidamente
5. **Reducir la deuda técnica**: Prevenir que los problemas se acumulen

### Cuándo refactorizar:

- **Regla de tres**: Cuando haces algo similar por tercera vez, refactoriza
- **Al agregar una feature**: Refactoriza para hacer el cambio más fácil
- **Al corregir un bug**: Refactoriza para hacer el código más claro
- **Durante code review**: Refactoriza para mejorar el código antes de merge

### Cuándo NO refactorizar:

- **Cuando necesitas reescribir desde cero**: A veces es mejor empezar de nuevo
- **Cuando hay deadlines críticos**: La refactorización puede esperar
- **Cuando el código no se tocará más**: No refactorices código que será eliminado
- **Sin tests**: Es peligroso refactorizar sin una red de seguridad

---

## 4.1 Extract Method

### Explicación Detallada

Extract Method es una de las refactorizaciones más fundamentales y útiles. Consiste en tomar un fragmento de código y convertirlo en un método separado con un nombre descriptivo. Es la base de muchas otras refactorizaciones.

#### Cuándo usar Extract Method:

1. **Código que se puede agrupar**: Fragmentos que trabajan juntos para un propósito
2. **Código que necesita un comentario**: Si necesitas explicar qué hace, extráelo
3. **Código duplicado**: Para eliminar duplicación
4. **Métodos muy largos**: Para dividir en piezas manejables
5. **Condiciones complejas**: Para dar nombres descriptivos a las condiciones

#### Beneficios:

- **Mejora la legibilidad**: Nombres descriptivos explican la intención
- **Promueve la reutilización**: Métodos pequeños pueden reutilizarse
- **Facilita el testing**: Métodos pequeños son más fáciles de testear
- **Simplifica el debugging**: Es más fácil encontrar problemas
- **Reduce la complejidad**: Cada método tiene una sola responsabilidad

#### Mecánica:

1. Crear un nuevo método con nombre descriptivo
2. Copiar el código a extraer al nuevo método
3. Identificar variables locales y parámetros necesarios
4. Reemplazar el código original con una llamada al método
5. Compilar y testear

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Extract Method Básico
// ============================================

// ❌ ANTES: Método con múltiples responsabilidades mezcladas
class InvoiceGenerator {
  generateInvoice(order: Order): string {
    let html = '';
    
    // Problema: Header generation mezclado
    html += '<div class="header">';
    html += `<h1>Invoice #${order.id}</h1>`;
    html += `<p>Customer: ${order.customer.name}</p>`;
    html += '</div>';
    
    // Problema: Cálculo de totales mezclado
    let subtotal = 0;
    for (const item of order.items) {
      subtotal += item.price * item.quantity;
    }
    const tax = subtotal * 0.08;
    const total = subtotal + tax;
    
    // Problema: Generación de items mezclada
    html += '<table>';
    for (const item of order.items) {
      html += `<tr><td>${item.name}</td><td>$${item.price}</td></tr>`;
    }
    html += '</table>';
    
    html += `<p>Total: $${total.toFixed(2)}</p>`;
    return html;
  }
}

// ✅ DESPUÉS: Métodos extraídos con responsabilidades claras
class RefactoredInvoiceGenerator {
  private readonly TAX_RATE = 0.08;
  
  generateInvoice(order: Order): string {
    const totals = this.calculateTotals(order);
    
    return [
      this.generateHeader(order),
      this.generateItemsTable(order.items),
      this.generateSummary(totals)
    ].join('');
  }
  
  private calculateTotals(order: Order): InvoiceTotals {
    const subtotal = order.items.reduce(
      (sum, item) => sum + (item.price * item.quantity), 0
    );
    const tax = subtotal * this.TAX_RATE;
    return { subtotal, tax, total: subtotal + tax };
  }
  
  private generateHeader(order: Order): string {
    return `<div class="header">
      <h1>Invoice #${order.id}</h1>
      <p>Customer: ${order.customer.name}</p>
    </div>`;
  }
  
  private generateItemsTable(items: OrderItem[]): string {
    const rows = items.map(item => 
      `<tr><td>${item.name}</td><td>$${item.price}</td></tr>`
    ).join('');
    return `<table>${rows}</table>`;
  }
  
  private generateSummary(totals: InvoiceTotals): string {
    return `<div class="summary">
      <p>Subtotal: $${totals.subtotal.toFixed(2)}</p>
      <p>Tax: $${totals.tax.toFixed(2)}</p>
      <p>Total: $${totals.total.toFixed(2)}</p>
    </div>`;
  }
}

// ============================================
// EJEMPLO 2: Extract Method con Variables Locales
// ============================================

// ❌ ANTES: Lógica compleja con muchas variables locales
class PriceCalculator {
  calculatePrice(product: Product, customer: Customer, quantity: number): number {
    // Problema: Múltiples cálculos de descuento mezclados
    let quantityDiscount = 0;
    if (quantity > 50) quantityDiscount = 0.10;
    else if (quantity > 20) quantityDiscount = 0.05;
    
    let customerDiscount = 0;
    if (customer.type === 'vip') customerDiscount = 0.15;
    else if (customer.type === 'premium') customerDiscount = 0.10;
    
    // Problema: Cálculo final complejo
    const basePrice = product.price * quantity;
    const discount = Math.max(quantityDiscount, customerDiscount);
    const finalPrice = basePrice * (1 - discount);
    const tax = finalPrice * 0.08;
    
    return finalPrice + tax;
  }
}

// ✅ DESPUÉS: Métodos extraídos con parámetros apropiados
class RefactoredPriceCalculator {
  calculatePrice(product: Product, customer: Customer, quantity: number): number {
    const basePrice = product.price * quantity;
    const discount = this.getBestDiscount(quantity, customer);
    const finalPrice = basePrice * (1 - discount);
    const tax = finalPrice * 0.08;
    return finalPrice + tax;
  }
  
  private getBestDiscount(quantity: number, customer: Customer): number {
    return Math.max(
      this.getQuantityDiscount(quantity),
      this.getCustomerDiscount(customer)
    );
  }
  
  private getQuantityDiscount(quantity: number): number {
    if (quantity > 50) return 0.10;
    if (quantity > 20) return 0.05;
    return 0;
  }
  
  private getCustomerDiscount(customer: Customer): number {
    const discounts: Record<string, number> = {
      'vip': 0.15,
      'premium': 0.10,
      'regular': 0.05
    };
    return discounts[customer.type] || 0;
  }
}

// ============================================
// EJEMPLO 3: Extract Method para Mejorar Testabilidad
// ============================================

// ❌ ANTES: Difícil de testear porque todo está mezclado
class UserAuthenticator {
  async authenticate(username: string, password: string): Promise<AuthResult> {
    // Problema: Validación, hash, búsqueda todo junto
    if (!username || !password) {
      return { success: false, error: 'Invalid credentials' };
    }
    
    // Hash del password (salt hardcodeado)
    const hashedPassword = this.hashPassword(password + 'salt123');
    
    // Buscar usuario y verificar
    const user = await database.query('SELECT * FROM users WHERE username = ?', [username]);
    
    if (!user || user.password !== hashedPassword) {
      return { success: false, error: 'Invalid credentials' };
    }
    
    if (user.locked) {
      return { success: false, error: 'Account locked' };
    }
    
    // Generar token
    const token = this.generateToken();
    return { success: true, user, token };
  }
  
  private hashPassword(pass: string): string {
    // Implementación simple
    return pass; 
  }
  
  private generateToken(): string {
    return Math.random().toString(36);
  }
}

// ✅ DESPUÉS: Métodos extraídos que son fáciles de testear individualmente
class RefactoredUserAuthenticator {
  constructor(
    private userRepository: UserRepository,
    private cryptoService: CryptoService
  ) {}
  
  async authenticate(username: string, password: string): Promise<AuthResult> {
    // Validar credenciales
    if (!this.isValidCredentials(username, password)) {
      return { success: false, error: 'Invalid credentials' };
    }
    
    // Buscar y verificar usuario
    const user = await this.userRepository.findByUsername(username);
    if (!user || !await this.verifyPassword(password, user.passwordHash)) {
      return { success: false, error: 'Invalid credentials' };
    }
    
    if (user.locked) {
      return { success: false, error: 'Account locked' };
    }
    
    // Crear sesión
    const token = await this.createSession(user.id);
    return { success: true, user: this.sanitizeUser(user), token };
  }
  
  // Métodos puros, fáciles de testear
  private isValidCredentials(username: string, password: string): boolean {
    return username?.length >= 3 && password?.length >= 8;
  }
  
  private async verifyPassword(plain: string, hashed: string): Promise<boolean> {
    return await this.cryptoService.hash(plain) === hashed;
  }
  
  private async createSession(userId: string): Promise<string> {
    return this.cryptoService.generateToken();
  }
  
  private sanitizeUser(user: User): SafeUser {
    return { id: user.id, username: user.username, email: user.email };
  }
}

// ============================================
// EJEMPLO 4: Extract Method para Condiciones Complejas
// ============================================

// ❌ ANTES: Condiciones complejas difíciles de entender
class LoanApprovalService {
  approveLoan(application: LoanApplication): LoanDecision {
    // Problema: Condición compleja ilegible
    if ((application.creditScore > 750 && application.income > 100000) ||
        (application.creditScore > 700 && application.income > 80000 && application.yearsEmployed > 2) ||
        (application.creditScore > 600 && application.isExistingCustomer)) {
      
      // Problema: Lógica de cálculo mezclada
      const maxAmount = application.income * (application.creditScore > 750 ? 5 : 3);
      const rate = application.creditScore > 750 ? 3.5 : 5.5;
      
      return {
        approved: true,
        maxAmount,
        interestRate: rate
      };
    }
    
    return { approved: false, reason: 'Requirements not met' };
  }
}

// ✅ DESPUÉS: Condiciones extraídas en métodos con nombres descriptivos
class RefactoredLoanApprovalService {
  approveLoan(application: LoanApplication): LoanDecision {
    if (!this.isEligible(application)) {
      return { approved: false, reason: 'Requirements not met' };
    }
    
    return {
      approved: true,
      maxAmount: this.calculateMaxAmount(application),
      interestRate: this.calculateRate(application)
    };
  }
  
  private isEligible(app: LoanApplication): boolean {
    return this.hasExcellentCredit(app) ||
           this.hasGoodCredit(app) ||
           this.isExistingCustomer(app);
  }
  
  private hasExcellentCredit(app: LoanApplication): boolean {
    return app.creditScore > 750 && app.income > 100000;
  }
  
  private hasGoodCredit(app: LoanApplication): boolean {
    return app.creditScore > 700 && app.income > 80000 && app.yearsEmployed > 2;
  }
  
  private isExistingCustomer(app: LoanApplication): boolean {
    return app.creditScore > 600 && app.isExistingCustomer;
  }
  
  private calculateMaxAmount(app: LoanApplication): number {
    const multiplier = app.creditScore > 750 ? 5 : 3;
    return app.income * multiplier;
  }
  
  private calculateRate(app: LoanApplication): number {
    if (app.creditScore > 750) return 3.5;
    if (app.creditScore > 700) return 4.5;
    return 5.5;
  }
}

// Interfaces auxiliares
interface Order {
  id: string;
  customer: { name: string; email: string };
  items: OrderItem[];
}

interface OrderItem {
  name: string;
  price: number;
  quantity: number;
}

interface InvoiceTotals {
  subtotal: number;
  tax: number;
  total: number;
}

interface Customer {
  type: 'regular' | 'premium' | 'vip';
  yearsAsCustomer: number;
}

interface Product {
  price: number;
}

interface AuthResult {
  success: boolean;
  error?: string;
  user?: SafeUser;
  token?: string;
}

interface User {
  id: string;
  username: string;
  email: string;
  passwordHash: string;
  locked: boolean;
}

interface SafeUser {
  id: string;
  username: string;
  email: string;
}

interface LoanApplication {
  creditScore: number;
  income: number;
  yearsEmployed: number;
  isExistingCustomer: boolean;
}

interface LoanDecision {
  approved: boolean;
  maxAmount?: number;
  interestRate?: number;
  reason?: string;
}

// Servicios auxiliares mínimos
interface UserRepository {
  findByUsername(username: string): Promise<User | null>;
}

interface CryptoService {
  hash(password: string): Promise<string>;
  generateToken(): string;
}

declare const database: any;
```

### Mejores prácticas para Extract Method:

1. **Nombres descriptivos**: El nombre del método debe explicar QUÉ hace, no CÓMO
2. **Métodos pequeños**: Idealmente 5-10 líneas, máximo 20
3. **Un nivel de abstracción**: Cada método debe trabajar en un solo nivel
4. **Minimizar parámetros**: Si necesitas muchos parámetros, considera un objeto
5. **Evitar efectos secundarios**: Los métodos deben hacer lo que su nombre indica
6. **Testabilidad**: Los métodos extraídos deben ser fáciles de testear

Extract Method es la base de muchas otras refactorizaciones y es esencial para mantener el código limpio y mantenible.

---

## Técnicas de Refactorización Funcional

### Replace Loop with Pipeline

```typescript
// ❌ Bucles imperativos
function processOrders(orders: Order[]): ProcessedOrder[] {
  const result = [];
  for (const order of orders) {
    if (order.status === 'active') {
      const processed = {
        ...order,
        total: order.items.reduce((sum, item) => sum + item.price, 0),
        tax: order.items.reduce((sum, item) => sum + item.price, 0) * 0.08
      };
      if (processed.total > 100) {
        result.push(processed);
      }
    }
  }
  return result;
}

// ✅ Pipeline funcional
import { pipe, filter, map } from 'fp-ts/Array';

const processOrders = (orders: Order[]): ProcessedOrder[] =>
  pipe(
    orders,
    filter(order => order.status === 'active'),
    map(order => ({
      ...order,
      total: calculateTotal(order.items),
      tax: calculateTax(order.items)
    })),
    filter(order => order.total > 100)
  );

const calculateTotal = (items: Item[]): number =>
  items.reduce((sum, item) => sum + item.price, 0);

const calculateTax = (items: Item[]): number =>
  calculateTotal(items) * 0.08;
```

### Replace Conditional with Function Map

```typescript
// ❌ Condicionales múltiples
function getDiscount(customerType: string): number {
  if (customerType === 'vip') return 0.20;
  if (customerType === 'premium') return 0.10;
  if (customerType === 'regular') return 0.05;
  return 0;
}

// ✅ Map de funciones
const discountStrategies: Record<string, () => number> = {
  vip: () => 0.20,
  premium: () => 0.10,
  regular: () => 0.05,
  default: () => 0
};

const getDiscount = (customerType: string): number =>
  (discountStrategies[customerType] || discountStrategies.default)();
```

### Replace Mutation with Transformation

```typescript
// ❌ Mutación de datos
function applyDiscounts(order: Order): void {
  order.subtotal = calculateSubtotal(order.items);
  order.discount = order.subtotal * 0.1;
  order.tax = (order.subtotal - order.discount) * 0.08;
  order.total = order.subtotal - order.discount + order.tax;
}

// ✅ Transformación inmutable
const applyDiscounts = (order: Order): Order => {
  const subtotal = calculateSubtotal(order.items);
  const discount = subtotal * 0.1;
  const tax = (subtotal - discount) * 0.08;
  
  return {
    ...order,
    subtotal,
    discount,
    tax,
    total: subtotal - discount + tax
  };
};
```

### Replace Class with Function Module

```typescript
// ❌ Clase con estado mutable
class ShoppingCart {
  private items: Item[] = [];
  
  addItem(item: Item): void {
    this.items.push(item);
  }
  
  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}

// ✅ Módulo funcional inmutable
type Cart = {
  readonly items: readonly Item[];
};

const createCart = (): Cart => ({ items: [] });

const addItem = (cart: Cart, item: Item): Cart => ({
  ...cart,
  items: [...cart.items, item]
});

const getTotal = (cart: Cart): number =>
  cart.items.reduce((sum, item) => sum + item.price, 0);

// Uso
let cart = createCart();
cart = addItem(cart, { id: '1', name: 'Book', price: 20 });
const total = getTotal(cart);
```

### Extract Pure Function from Method

```typescript
// ❌ Método con efectos secundarios
class OrderService {
  processOrder(order: Order): void {
    const total = order.items.reduce((sum, item) => sum + item.price, 0);
    const tax = total * 0.08;
    const finalTotal = total + tax;
    
    this.database.save({ ...order, total: finalTotal });
    this.emailService.send(order.customerEmail, 'Order processed');
  }
}

// ✅ Funciones puras extraídas
const calculateOrderTotal = (items: Item[]): number =>
  items.reduce((sum, item) => sum + item.price, 0);

const calculateTax = (amount: number, rate: number = 0.08): number =>
  amount * rate;

const processOrder = (deps: Dependencies) => async (order: Order): Promise<void> => {
  const total = calculateOrderTotal(order.items);
  const tax = calculateTax(total);
  const finalTotal = total + tax;
  
  await deps.database.save({ ...order, total: finalTotal });
  await deps.emailService.send(order.customerEmail, 'Order processed');
};
```

---

## 4.2 Inline Method

### Explicación Detallada

Inline Method es lo opuesto a Extract Method. Consiste en reemplazar una llamada a un método con el cuerpo del método mismo. Se usa cuando el método no aporta claridad adicional o cuando reorganizamos métodos.

#### Cuándo usar Inline Method:

1. **El cuerpo del método es tan claro como su nombre**
2. **Método delegador innecesario**
3. **Grupo de métodos mal factorizados** (inline todos y re-extract)
4. **Simplificar jerarquías de llamadas**

### Ejemplo Práctico

```typescript
// ❌ ANTES: Método que no añade claridad
class Order {
  private basePrice: number;
  
  getTotal(): number {
    return this.applyShipping(this.basePrice);
  }
  
  private applyShipping(price: number): number {
    return price + 5;  // Solo suma shipping fijo
  }
}

// ✅ DESPUÉS: Código inline más claro
class Order {
  private basePrice: number;
  private readonly SHIPPING_COST = 5;
  
  getTotal(): number {
    return this.basePrice + this.SHIPPING_COST;
  }
}

// Ejemplo más complejo
// ❌ ANTES: Múltiples niveles de delegación
class UserValidator {
  validate(user: User): boolean {
    return this.isValidName(user.name) && 
           this.isValidEmail(user.email) &&
           this.isValidAge(user.age);
  }
  
  private isValidName(name: string): boolean {
    return this.hasContent(name);
  }
  
  private hasContent(str: string): boolean {
    return str.length > 0;
  }
  
  private isValidEmail(email: string): boolean {
    return this.containsAt(email);
  }
  
  private containsAt(email: string): boolean {
    return email.includes('@');
  }
  
  private isValidAge(age: number): boolean {
    return this.isAdult(age);
  }
  
  private isAdult(age: number): boolean {
    return age >= 18;
  }
}

// ✅ DESPUÉS: Inline de métodos triviales
class UserValidator {
  validate(user: User): boolean {
    return user.name.length > 0 && 
           user.email.includes('@') &&
           user.age >= 18;
  }
}
```

---

## 4.3 Extract Variable

### Explicación Detallada

Extract Variable (también conocido como Introduce Explaining Variable) toma una expresión compleja y la asigna a una variable con un nombre descriptivo que explica el propósito de la expresión.

#### Cuándo usar Extract Variable:

1. **Expresiones complejas difíciles de entender**
2. **Expresiones repetidas en el código**
3. **Condiciones complicadas**
4. **Cálculos intermedios que necesitan explicación**

### Ejemplo Práctico

```typescript
// ❌ ANTES: Expresiones complejas difíciles de entender
class PriceCalculator {
  calculateTotal(order: Order): number {
    // ¿Qué significa cada parte de esta expresión?
    return order.quantity * order.itemPrice * 
           (1 - (order.quantity > 100 ? 0.1 : order.quantity > 50 ? 0.05 : 0)) *
           (1 + (order.isRush ? 0.2 : 0)) +
           (order.quantity * order.itemPrice > 1000 ? 0 : 15);
  }
}

// ✅ DESPUÉS: Variables explicativas
class PriceCalculator {
  calculateTotal(order: Order): number {
    const baseAmount = order.quantity * order.itemPrice;
    const quantityDiscount = this.getQuantityDiscount(order.quantity);
    const rushMultiplier = order.isRush ? 1.2 : 1.0;
    const isFreeShipping = baseAmount > 1000;
    const shippingCost = isFreeShipping ? 0 : 15;
    
    const discountedAmount = baseAmount * (1 - quantityDiscount);
    const amountWithRush = discountedAmount * rushMultiplier;
    
    return amountWithRush + shippingCost;
  }
  
  private getQuantityDiscount(quantity: number): number {
    if (quantity > 100) return 0.1;
    if (quantity > 50) return 0.05;
    return 0;
  }
}

// Ejemplo con condiciones complejas
// ❌ ANTES
if (platform.toUpperCase().indexOf('MAC') > -1 &&
    browser.toUpperCase().indexOf('IE') > -1 &&
    wasInitialized() && resize > 0) {
  // hacer algo
}

// ✅ DESPUÉS
const isMacOS = platform.toUpperCase().includes('MAC');
const isIEBrowser = browser.toUpperCase().includes('IE');
const isResized = resize > 0;
const canRender = isMacOS && isIEBrowser && wasInitialized() && isResized;

if (canRender) {
  // hacer algo
}
```

---

## 4.4 Inline Variable

### Explicación Detallada

Inline Variable es lo opuesto a Extract Variable. Reemplaza referencias a una variable con su expresión. Útil cuando la variable no añade claridad.

### Ejemplo Práctico

```typescript
// ❌ ANTES: Variable innecesaria
function getShipping(order: Order): number {
  const baseShipping = 5;
  const shippingCost = baseShipping;
  return shippingCost;
}

// ✅ DESPUÉS: Directo
function getShipping(order: Order): number {
  return 5;
}

// ❌ ANTES: Variable que no clarifica
function isExpensive(price: number): boolean {
  const expensiveThreshold = 100;
  const isExpensive = price > expensiveThreshold;
  return isExpensive;
}

// ✅ DESPUÉS: Más directo
function isExpensive(price: number): boolean {
  return price > 100;
}
```

---

## 4.5 Rename (Variable, Method, Class)

### Explicación Detallada

Rename es una de las refactorizaciones más importantes. Los nombres buenos son fundamentales para código legible. Si encuentras un mejor nombre, cámbialo.

#### Principios de buenos nombres:

1. **Revelación de intención**: El nombre debe explicar por qué existe
2. **Evitar desinformación**: No usar nombres engañosos
3. **Distinguible**: Nombres diferentes deben significar cosas diferentes
4. **Pronunciable**: Facilita la comunicación
5. **Buscable**: Fácil de encontrar en el código

### Ejemplo Práctico

```typescript
// ❌ ANTES: Nombres poco descriptivos
class DtaRcrd102 {
  private genymdhms: Date;
  private modymdhms: Date;
  private pszqint = 102;
}

// ✅ DESPUÉS: Nombres descriptivos
class Customer {
  private generationTimestamp: Date;
  private modificationTimestamp: Date;
  private recordId = 102;
}

// ❌ ANTES: Método con nombre confuso
class Account {
  // ¿Qué hace realmente?
  process(): void {
    // ...
  }
}

// ✅ DESPUÉS: Nombre específico
class Account {
  calculateMonthlyInterest(): void {
    // ...
  }
}

// Renombrar variables para claridad
// ❌ ANTES
const d = new Date();
const yrs = calcYrs(d);

// ✅ DESPUÉS
const currentDate = new Date();
const yearsSinceFoundation = calculateYearsSince(currentDate);
```

---

## 4.6 Move Method

### Explicación Detallada

Move Method mueve un método de una clase a otra. Se usa cuando un método usa más características de otra clase que de la propia.

#### Cuándo mover un método:

1. **Método usa más datos de otra clase**
2. **Reducir acoplamiento entre clases**
3. **Mejorar cohesión de las clases**
4. **Eliminar feature envy**

### Ejemplo Práctico

```typescript
// ❌ ANTES: Método en la clase incorrecta
class Account {
  private balance: number;
  private type: AccountType;
  
  // Este método usa más AccountType que Account
  calculateInterestRate(): number {
    if (this.type.isPremium()) {
      return this.type.getBaseRate() * 1.5;
    }
    if (this.type.isBasic()) {
      return this.type.getBaseRate();
    }
    return 0;
  }
}

class AccountType {
  constructor(
    private name: string,
    private baseRate: number
  ) {}
  
  isPremium(): boolean {
    return this.name === 'premium';
  }
  
  isBasic(): boolean {
    return this.name === 'basic';
  }
  
  getBaseRate(): number {
    return this.baseRate;
  }
}

// ✅ DESPUÉS: Método movido a la clase correcta
class Account {
  private balance: number;
  private type: AccountType;
  
  getInterestRate(): number {
    return this.type.calculateInterestRate();
  }
}

class AccountType {
  constructor(
    private name: string,
    private baseRate: number
  ) {}
  
  calculateInterestRate(): number {
    if (this.isPremium()) {
      return this.baseRate * 1.5;
    }
    if (this.isBasic()) {
      return this.baseRate;
    }
    return 0;
  }
  
  private isPremium(): boolean {
    return this.name === 'premium';
  }
  
  private isBasic(): boolean {
    return this.name === 'basic';
  }
}
```

---

## 4.7 Extract Class

### Explicación Detallada

Extract Class se usa cuando una clase está haciendo el trabajo de dos clases. Crea una nueva clase y mueve los campos y métodos relevantes.

#### Señales para Extract Class:

1. **Clase con muchas responsabilidades**
2. **Subconjunto de datos que siempre cambian juntos**
3. **Subconjunto de métodos que trabajan sobre los mismos datos**
4. **Clase difícil de entender**

### Ejemplo Práctico

```typescript
// ❌ ANTES: Clase con múltiples responsabilidades
class Person {
  private name: string;
  private email: string;
  
  // Datos de dirección mezclados
  private street: string;
  private city: string;
  private state: string;
  private zipCode: string;
  private country: string;
  
  // Datos de teléfono mezclados
  private homePhone: string;
  private workPhone: string;
  private mobilePhone: string;
  
  getAddress(): string {
    return `${this.street}, ${this.city}, ${this.state} ${this.zipCode}`;
  }
  
  isInternational(): boolean {
    return this.country !== 'USA';
  }
  
  getPreferredPhone(): string {
    return this.mobilePhone || this.homePhone || this.workPhone;
  }
}

// ✅ DESPUÉS: Clases separadas con responsabilidades claras
class Person {
  constructor(
    private name: string,
    private email: string,
    private address: Address,
    private phoneNumbers: PhoneNumbers
  ) {}
  
  getName(): string {
    return this.name;
  }
  
  getAddress(): string {
    return this.address.getFullAddress();
  }
  
  getPreferredPhone(): string {
    return this.phoneNumbers.getPreferred();
  }
}

class Address {
  constructor(
    private street: string,
    private city: string,
    private state: string,
    private zipCode: string,
    private country: string = 'USA'
  ) {}
  
  getFullAddress(): string {
    return `${this.street}, ${this.city}, ${this.state} ${this.zipCode}`;
  }
  
  isInternational(): boolean {
    return this.country !== 'USA';
  }
}

class PhoneNumbers {
  constructor(
    private home?: string,
    private work?: string,
    private mobile?: string
  ) {}
  
  getPreferred(): string {
    return this.mobile || this.home || this.work || '';
  }
  
  getAllNumbers(): string[] {
    return [this.home, this.work, this.mobile].filter(Boolean) as string[];
  }
}
```

---

## 4.8 Inline Class

### Explicación Detallada

Inline Class es lo opuesto a Extract Class. Se usa cuando una clase no está haciendo suficiente para justificar su existencia.

### Ejemplo Práctico

```typescript
// ❌ ANTES: Clase innecesaria
class PhoneNumber {
  constructor(private number: string) {}
  
  getNumber(): string {
    return this.number;
  }
}

class Person {
  constructor(
    private name: string,
    private phone: PhoneNumber
  ) {}
  
  getPhoneNumber(): string {
    return this.phone.getNumber();
  }
}

// ✅ DESPUÉS: Clase inline
class Person {
  constructor(
    private name: string,
    private phoneNumber: string
  ) {}
  
  getPhoneNumber(): string {
    return this.phoneNumber;
  }
}
```

---

## 4.9 Replace Temp with Query

### Explicación Detallada

Replace Temp with Query extrae la expresión que calcula una variable temporal en un método separado. Permite que el valor sea accesible desde otros métodos.

### Ejemplo Práctico

```typescript
// ❌ ANTES: Variables temporales
class Order {
  private quantity: number;
  private itemPrice: number;
  
  getPrice(): number {
    const basePrice = this.quantity * this.itemPrice;
    const discountFactor = basePrice > 1000 ? 0.95 : 0.98;
    return basePrice * discountFactor;
  }
  
  // No puede reusar basePrice o discountFactor
  printDetails(): void {
    const basePrice = this.quantity * this.itemPrice;
    console.log(`Base price: ${basePrice}`);
  }
}

// ✅ DESPUÉS: Métodos query
class Order {
  private quantity: number;
  private itemPrice: number;
  
  getPrice(): number {
    return this.getBasePrice() * this.getDiscountFactor();
  }
  
  printDetails(): void {
    console.log(`Base price: ${this.getBasePrice()}`);
    console.log(`Discount: ${(1 - this.getDiscountFactor()) * 100}%`);
  }
  
  private getBasePrice(): number {
    return this.quantity * this.itemPrice;
  }
  
  private getDiscountFactor(): number {
    return this.getBasePrice() > 1000 ? 0.95 : 0.98;
  }
}
```

---

## 4.10 Replace Magic Numbers with Constants

### Explicación Detallada

Los "números mágicos" son valores literales con significado especial. Reemplazarlos con constantes nombradas mejora la legibilidad y mantenibilidad.

### Ejemplo Práctico

```typescript
// ❌ ANTES: Números mágicos sin contexto
class PasswordValidator {
  validate(password: string): boolean {
    if (password.length < 8) return false;
    if (password.length > 128) return false;
    
    // ¿Qué significa 0.3?
    const specialChars = password.match(/[!@#$%^&*]/g) || [];
    if (specialChars.length / password.length < 0.3) return false;
    
    return true;
  }
  
  calculateStrength(password: string): number {
    let strength = 0;
    if (password.length > 12) strength += 25;
    if (password.length > 16) strength += 25;
    if (/[A-Z]/.test(password)) strength += 25;
    if (/[!@#$%^&*]/.test(password)) strength += 25;
    return strength;
  }
}

// ✅ DESPUÉS: Constantes con nombres descriptivos
class PasswordValidator {
  private static readonly MIN_LENGTH = 8;
  private static readonly MAX_LENGTH = 128;
  private static readonly MIN_SPECIAL_CHAR_RATIO = 0.3;
  private static readonly SPECIAL_CHAR_PATTERN = /[!@#$%^&*]/g;
  
  private static readonly STRENGTH_INCREMENT = 25;
  private static readonly STRONG_LENGTH = 12;
  private static readonly VERY_STRONG_LENGTH = 16;
  
  validate(password: string): boolean {
    if (password.length < PasswordValidator.MIN_LENGTH) return false;
    if (password.length > PasswordValidator.MAX_LENGTH) return false;
    
    const specialCharRatio = this.getSpecialCharRatio(password);
    if (specialCharRatio < PasswordValidator.MIN_SPECIAL_CHAR_RATIO) return false;
    
    return true;
  }
  
  private getSpecialCharRatio(password: string): number {
    const specialChars = password.match(PasswordValidator.SPECIAL_CHAR_PATTERN) || [];
    return specialChars.length / password.length;
  }
  
  calculateStrength(password: string): number {
    let strength = 0;
    
    if (password.length > PasswordValidator.STRONG_LENGTH) {
      strength += PasswordValidator.STRENGTH_INCREMENT;
    }
    if (password.length > PasswordValidator.VERY_STRONG_LENGTH) {
      strength += PasswordValidator.STRENGTH_INCREMENT;
    }
    if (/[A-Z]/.test(password)) {
      strength += PasswordValidator.STRENGTH_INCREMENT;
    }
    if (PasswordValidator.SPECIAL_CHAR_PATTERN.test(password)) {
      strength += PasswordValidator.STRENGTH_INCREMENT;
    }
    
    return strength;
  }
}
```

---

## 4.11 Replace Conditional with Polymorphism

### Explicación Detallada

Replace Conditional with Polymorphism elimina condicionales complejos usando herencia o composición. Cada rama del condicional se convierte en una clase.

### Ejemplo Práctico

```typescript
// ❌ ANTES: Switch/if complejo
class Employee {
  constructor(
    private type: 'engineer' | 'salesman' | 'manager',
    private monthlySalary: number,
    private commission: number = 0,
    private bonus: number = 0
  ) {}
  
  calculatePay(): number {
    switch (this.type) {
      case 'engineer':
        return this.monthlySalary;
      case 'salesman':
        return this.monthlySalary + this.commission;
      case 'manager':
        return this.monthlySalary + this.bonus;
      default:
        throw new Error('Invalid employee type');
    }
  }
  
  getVacationDays(): number {
    switch (this.type) {
      case 'engineer':
        return 15;
      case 'salesman':
        return 12;
      case 'manager':
        return 20;
      default:
        return 10;
    }
  }
}

// ✅ DESPUÉS: Polimorfismo con herencia
abstract class Employee {
  constructor(protected monthlySalary: number) {}
  
  abstract calculatePay(): number;
  abstract getVacationDays(): number;
}

class Engineer extends Employee {
  calculatePay(): number {
    return this.monthlySalary;
  }
  
  getVacationDays(): number {
    return 15;
  }
}

class Salesman extends Employee {
  constructor(
    monthlySalary: number,
    private commission: number
  ) {
    super(monthlySalary);
  }
  
  calculatePay(): number {
    return this.monthlySalary + this.commission;
  }
  
  getVacationDays(): number {
    return 12;
  }
}

class Manager extends Employee {
  constructor(
    monthlySalary: number,
    private bonus: number
  ) {
    super(monthlySalary);
  }
  
  calculatePay(): number {
    return this.monthlySalary + this.bonus;
  }
  
  getVacationDays(): number {
    return 20;
  }
}

// Alternativa con Strategy Pattern (composición)
interface PaymentStrategy {
  calculatePay(baseSalary: number): number;
  getVacationDays(): number;
}

class EngineerStrategy implements PaymentStrategy {
  calculatePay(baseSalary: number): number {
    return baseSalary;
  }
  
  getVacationDays(): number {
    return 15;
  }
}

class SalesmanStrategy implements PaymentStrategy {
  constructor(private commission: number) {}
  
  calculatePay(baseSalary: number): number {
    return baseSalary + this.commission;
  }
  
  getVacationDays(): number {
    return 12;
  }
}

class Employee {
  constructor(
    private baseSalary: number,
    private paymentStrategy: PaymentStrategy
  ) {}
  
  calculatePay(): number {
    return this.paymentStrategy.calculatePay(this.baseSalary);
  }
  
  getVacationDays(): number {
    return this.paymentStrategy.getVacationDays();
  }
}
```

---

## 4.12 Compose Method

### Explicación Detallada

Compose Method transforma un método complejo en una secuencia de pasos donde cada paso está al mismo nivel de abstracción.

### Ejemplo Práctico

```typescript
// ❌ ANTES: Método con múltiples niveles de abstracción
class ReportGenerator {
  generateReport(data: Data): string {
    let report = '';
    
    // Header - alto nivel
    report += '<header>';
    report += `<h1>${data.title}</h1>`;
    report += `<date>${new Date().toISOString()}</date>`;
    report += '</header>';
    
    // Processing - bajo nivel
    const processedData = [];
    for (const item of data.items) {
      if (item.value > 0) {
        const processed = {
          name: item.name.toUpperCase(),
          value: Math.round(item.value * 100) / 100,
          percentage: (item.value / data.total) * 100
        };
        processedData.push(processed);
      }
    }
    
    // Body - mezcla de niveles
    report += '<body><table>';
    for (const item of processedData) {
      report += '<tr>';
      report += `<td>${item.name}</td>`;
      report += `<td>${item.value}</td>`;
      report += `<td>${item.percentage.toFixed(2)}%</td>`;
      report += '</tr>';
    }
    report += '</table></body>';
    
    // Footer
    report += `<footer>Total: ${data.total}</footer>`;
    
    return report;
  }
}

// ✅ DESPUÉS: Método compuesto de pasos claros
class ReportGenerator {
  generateReport(data: Data): string {
    const processedData = this.processData(data);
    
    return this.composeReport(
      this.generateHeader(data),
      this.generateBody(processedData),
      this.generateFooter(data)
    );
  }
  
  private composeReport(...sections: string[]): string {
    return sections.join('');
  }
  
  private generateHeader(data: Data): string {
    return `
      <header>
        <h1>${data.title}</h1>
        <date>${new Date().toISOString()}</date>
      </header>
    `;
  }
  
  private processData(data: Data): ProcessedItem[] {
    return data.items
      .filter(item => item.value > 0)
      .map(item => this.processItem(item, data.total));
  }
  
  private processItem(item: Item, total: number): ProcessedItem {
    return {
      name: item.name.toUpperCase(),
      value: this.roundToTwoDecimals(item.value),
      percentage: this.calculatePercentage(item.value, total)
    };
  }
  
  private generateBody(items: ProcessedItem[]): string {
    const rows = items.map(item => this.generateRow(item)).join('');
    return `<body><table>${rows}</table></body>`;
  }
  
  private generateRow(item: ProcessedItem): string {
    return `
      <tr>
        <td>${item.name}</td>
        <td>${item.value}</td>
        <td>${item.percentage.toFixed(2)}%</td>
      </tr>
    `;
  }
  
  private generateFooter(data: Data): string {
    return `<footer>Total: ${data.total}</footer>`;
  }
  
  private roundToTwoDecimals(value: number): number {
    return Math.round(value * 100) / 100;
  }
  
  private calculatePercentage(value: number, total: number): number {
    return (value / total) * 100;
  }
}
```

---

## 4.13 Identificación de Code Smells

### Code Smells Comunes y sus Refactorizaciones

```typescript
// 1. LONG METHOD
// Smell: Método con más de 10-20 líneas
// Solución: Extract Method, Compose Method

// 2. LARGE CLASS
// Smell: Clase con muchas responsabilidades
// Solución: Extract Class, Extract Subclass

// 3. LONG PARAMETER LIST
// ❌ ANTES
function createUser(
  firstName: string,
  lastName: string,
  email: string,
  phone: string,
  address: string,
  city: string,
  state: string,
  zip: string
) { }

// ✅ DESPUÉS: Introduce Parameter Object
interface UserData {
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  address: Address;
}

function createUser(userData: UserData) { }

// 4. FEATURE ENVY
// Smell: Método que usa más otra clase que la propia
// Solución: Move Method, Extract Method

// 5. DATA CLUMPS
// Smell: Grupos de datos que siempre aparecen juntos
// Solución: Extract Class, Introduce Parameter Object

// 6. PRIMITIVE OBSESSION
// ❌ ANTES
class Order {
  constructor(
    private amount: number,  // ¿En qué moneda?
    private status: string,  // ¿Qué valores son válidos?
    private priority: number // ¿Qué significa cada número?
  ) {}
}

// ✅ DESPUÉS: Replace Primitive with Object
class Money {
  constructor(
    private amount: number,
    private currency: Currency
  ) {}
}

enum OrderStatus {
  PENDING = 'pending',
  PROCESSING = 'processing',
  COMPLETED = 'completed'
}

enum Priority {
  LOW = 1,
  MEDIUM = 2,
  HIGH = 3
}

class Order {
  constructor(
    private amount: Money,
    private status: OrderStatus,
    private priority: Priority
  ) {}
}

// 7. SWITCH STATEMENTS
// Smell: Switch/if repetidos con los mismos casos
// Solución: Replace Conditional with Polymorphism

// 8. LAZY CLASS
// Smell: Clase que no hace suficiente
// Solución: Inline Class, Collapse Hierarchy

// 9. SPECULATIVE GENERALITY
// Smell: Código "por si acaso" que no se usa
// Solución: Remove Dead Code, Inline Class/Method

// 10. TEMPORARY FIELD
// Smell: Campo que solo se usa en ciertas circunstancias
// Solución: Extract Class, Introduce Null Object
```

---

## 4.14 Estrategias de Refactorización

### Refactorización Preparatoria

```typescript
// Antes de añadir una nueva feature, refactoriza para hacerlo fácil

// Ejemplo: Necesitas añadir descuento por volumen
// ❌ ANTES: Código rígido
class OrderService {
  calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price, 0);
  }
}

// ✅ REFACTORIZACIÓN PREPARATORIA
class OrderService {
  calculateTotal(items: Item[]): number {
    const subtotal = this.calculateSubtotal(items);
    const discount = this.calculateDiscount(subtotal, items);
    return subtotal - discount;
  }
  
  private calculateSubtotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price, 0);
  }
  
  private calculateDiscount(subtotal: number, items: Item[]): number {
    // Ahora es fácil añadir lógica de descuento
    return 0;
  }
}
```

### Refactorización Comprehensiva

```typescript
// Cuando refactorizas para entender mejor el código

// ❌ ANTES: Código críptico
function calc(x: number[], f: boolean): number {
  let r = 0;
  for (let i = 0; i < x.length; i++) {
    if (f) {
      r += x[i] * 1.1;
    } else {
      r += x[i];
    }
  }
  return r;
}

// ✅ DESPUÉS: Código autodocumentado
function calculateTotalPrice(
  prices: number[],
  includeTax: boolean
): number {
  const TAX_RATE = 0.1;
  
  return prices.reduce((total, price) => {
    const priceWithTax = includeTax ? price * (1 + TAX_RATE) : price;
    return total + priceWithTax;
  }, 0);
}
```

### Refactorización de Limpieza

```typescript
// Después de hacer funcionar el código, límpialo

// ❌ CÓDIGO QUE FUNCIONA PERO ESTÁ SUCIO
class DataProcessor {
  process(data: any): any {
    // TODO: limpiar esto
    let result = [];
    for (let i = 0; i < data.length; i++) {
      if (data[i].type === 'A') {
        // procesar tipo A
        const temp = data[i].value * 2;
        result.push({ ...data[i], processed: temp });
      } else if (data[i].type === 'B') {
        // procesar tipo B
        const temp = data[i].value * 3;
        result.push({ ...data[i], processed: temp });
      }
    }
    return result;
  }
}

// ✅ DESPUÉS DE LIMPIEZA
interface DataItem {
  type: 'A' | 'B';
  value: number;
}

interface ProcessedItem extends DataItem {
  processed: number;
}

class DataProcessor {
  private readonly processors: Record<string, (value: number) => number> = {
    'A': value => value * 2,
    'B': value => value * 3
  };
  
  process(items: DataItem[]): ProcessedItem[] {
    return items.map(item => this.processItem(item));
  }
  
  private processItem(item: DataItem): ProcessedItem {
    const processor = this.processors[item.type];
    if (!processor) {
      throw new Error(`Unknown item type: ${item.type}`);
    }
    
    return {
      ...item,
      processed: processor(item.value)
    };
  }
}
```

---

## Resumen de Técnicas de Refactorización

### Técnicas de Composición de Métodos
- **Extract Method**: Convierte fragmento en método con nombre descriptivo
- **Inline Method**: Reemplaza llamada con el cuerpo del método
- **Extract Variable**: Asigna expresión a variable explicativa
- **Inline Variable**: Reemplaza variable con su expresión
- **Replace Temp with Query**: Extrae expresión a método
- **Split Temporary Variable**: Usa variables diferentes para diferentes propósitos

### Técnicas de Movimiento entre Objetos
- **Move Method**: Mueve método a clase más apropiada
- **Move Field**: Mueve campo a clase más apropiada
- **Extract Class**: Crea nueva clase con parte de responsabilidad
- **Inline Class**: Fusiona clase con poca responsabilidad
- **Hide Delegate**: Oculta delegación del cliente
- **Remove Middle Man**: Cliente habla directamente con delegado

### Técnicas de Organización de Datos
- **Replace Magic Number with Constant**: Usa constantes nombradas
- **Encapsulate Field**: Hacer campo privado y proveer accesors
- **Replace Type Code with Class**: Convierte tipo primitivo en clase
- **Replace Type Code with Subclasses**: Usa herencia para tipos
- **Replace Type Code with State/Strategy**: Usa patrones para comportamiento

### Técnicas de Simplificación Condicional
- **Decompose Conditional**: Extrae condición y ramas a métodos
- **Consolidate Conditional Expression**: Combina condicionales idénticos
- **Replace Conditional with Polymorphism**: Usa herencia/composición
- **Introduce Null Object**: Usa objeto especial para null
- **Introduce Assertion**: Hace explícitas las asunciones

### Técnicas de Simplificación de Llamadas
- **Rename Method**: Usa nombre que revele intención
- **Add Parameter**: Añade parámetro necesario
- **Remove Parameter**: Elimina parámetro no usado
- **Introduce Parameter Object**: Agrupa parámetros relacionados
- **Preserve Whole Object**: Pasa objeto completo en vez de valores
- **Replace Parameter with Method Call**: Deja que receptor invoque método

Esta guía cubre las técnicas de refactorización más importantes para mantener código limpio, mantenible y extensible.