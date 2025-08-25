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