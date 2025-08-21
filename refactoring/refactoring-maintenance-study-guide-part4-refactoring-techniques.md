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
    
    // Problema: Este bloque genera el header - debería ser un método
    html += '<html><head><title>Invoice</title></head><body>';
    html += '<div class="header">';
    html += `<h1>Invoice #${order.id}</h1>`;
    html += `<p>Date: ${new Date().toLocaleDateString()}</p>`;
    html += `<p>Customer: ${order.customer.name}</p>`;
    html += `<p>Email: ${order.customer.email}</p>`;
    html += '</div>';
    
    // Problema: Este bloque calcula totales - debería ser un método
    let subtotal = 0;
    for (const item of order.items) {
      subtotal += item.price * item.quantity;
    }
    const taxRate = 0.08;
    const tax = subtotal * taxRate;
    const shipping = order.items.length > 5 ? 0 : 10;
    const total = subtotal + tax + shipping;
    
    // Problema: Este bloque genera la tabla de items - debería ser un método
    html += '<table>';
    html += '<thead><tr><th>Item</th><th>Qty</th><th>Price</th><th>Total</th></tr></thead>';
    html += '<tbody>';
    for (const item of order.items) {
      html += '<tr>';
      html += `<td>${item.name}</td>`;
      html += `<td>${item.quantity}</td>`;
      html += `<td>$${item.price.toFixed(2)}</td>`;
      html += `<td>$${(item.price * item.quantity).toFixed(2)}</td>`;
      html += '</tr>';
    }
    html += '</tbody></table>';
    
    // Problema: Este bloque genera el resumen - debería ser un método
    html += '<div class="summary">';
    html += `<p>Subtotal: $${subtotal.toFixed(2)}</p>`;
    html += `<p>Tax (${(taxRate * 100).toFixed(0)}%): $${tax.toFixed(2)}</p>`;
    html += `<p>Shipping: $${shipping.toFixed(2)}</p>`;
    html += `<p class="total">Total: $${total.toFixed(2)}</p>`;
    html += '</div>';
    
    html += '</body></html>';
    
    return html;
  }
}

// ✅ DESPUÉS: Métodos extraídos con responsabilidades claras
class RefactoredInvoiceGenerator {
  private readonly TAX_RATE = 0.08;
  private readonly FREE_SHIPPING_THRESHOLD = 5;
  private readonly STANDARD_SHIPPING_COST = 10;
  
  generateInvoice(order: Order): string {
    // Ahora el método principal es claro y legible
    const totals = this.calculateTotals(order);
    
    return this.buildHTML(
      this.generateHeader(order),
      this.generateItemsTable(order.items),
      this.generateSummary(totals)
    );
  }
  
  // Método extraído: Cálculo de totales
  private calculateTotals(order: Order): InvoiceTotals {
    const subtotal = this.calculateSubtotal(order.items);
    const tax = this.calculateTax(subtotal);
    const shipping = this.calculateShipping(order.items);
    const total = subtotal + tax + shipping;
    
    return { subtotal, tax, shipping, total };
  }
  
  // Método extraído: Cálculo de subtotal
  private calculateSubtotal(items: OrderItem[]): number {
    return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }
  
  // Método extraído: Cálculo de impuestos
  private calculateTax(amount: number): number {
    return amount * this.TAX_RATE;
  }
  
  // Método extraído: Cálculo de envío
  private calculateShipping(items: OrderItem[]): number {
    return items.length > this.FREE_SHIPPING_THRESHOLD 
      ? 0 
      : this.STANDARD_SHIPPING_COST;
  }
  
  // Método extraído: Generación de header
  private generateHeader(order: Order): string {
    return `
      <div class="header">
        <h1>Invoice #${order.id}</h1>
        <p>Date: ${this.formatDate(new Date())}</p>
        <p>Customer: ${order.customer.name}</p>
        <p>Email: ${order.customer.email}</p>
      </div>
    `;
  }
  
  // Método extraído: Generación de tabla de items
  private generateItemsTable(items: OrderItem[]): string {
    const rows = items.map(item => this.generateItemRow(item)).join('');
    
    return `
      <table>
        <thead>
          <tr>
            <th>Item</th>
            <th>Qty</th>
            <th>Price</th>
            <th>Total</th>
          </tr>
        </thead>
        <tbody>
          ${rows}
        </tbody>
      </table>
    `;
  }
  
  // Método extraído: Generación de fila de item
  private generateItemRow(item: OrderItem): string {
    const lineTotal = item.price * item.quantity;
    
    return `
      <tr>
        <td>${item.name}</td>
        <td>${item.quantity}</td>
        <td>${this.formatCurrency(item.price)}</td>
        <td>${this.formatCurrency(lineTotal)}</td>
      </tr>
    `;
  }
  
  // Método extraído: Generación de resumen
  private generateSummary(totals: InvoiceTotals): string {
    return `
      <div class="summary">
        <p>Subtotal: ${this.formatCurrency(totals.subtotal)}</p>
        <p>Tax (${this.formatPercentage(this.TAX_RATE)}): ${this.formatCurrency(totals.tax)}</p>
        <p>Shipping: ${this.formatCurrency(totals.shipping)}</p>
        <p class="total">Total: ${this.formatCurrency(totals.total)}</p>
      </div>
    `;
  }
  
  // Método extraído: Construcción de HTML completo
  private buildHTML(header: string, itemsTable: string, summary: string): string {
    return `
      <html>
        <head>
          <title>Invoice</title>
          ${this.generateStyles()}
        </head>
        <body>
          ${header}
          ${itemsTable}
          ${summary}
        </body>
      </html>
    `;
  }
  
  // Métodos auxiliares de formateo
  private formatCurrency(amount: number): string {
    return `$${amount.toFixed(2)}`;
  }
  
  private formatPercentage(rate: number): string {
    return `${(rate * 100).toFixed(0)}%`;
  }
  
  private formatDate(date: Date): string {
    return date.toLocaleDateString('en-US', {
      year: 'numeric',
      month: 'long',
      day: 'numeric'
    });
  }
  
  private generateStyles(): string {
    return `
      <style>
        .header { margin-bottom: 20px; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 8px; text-align: left; border: 1px solid #ddd; }
        .summary { margin-top: 20px; }
        .total { font-weight: bold; font-size: 1.2em; }
      </style>
    `;
  }
}

// ============================================
// EJEMPLO 2: Extract Method con Variables Locales
// ============================================

// ❌ ANTES: Lógica compleja con muchas variables locales
class PriceCalculator {
  calculatePrice(product: Product, customer: Customer, quantity: number): number {
    // Problema: Lógica de descuento por cantidad mezclada
    let quantityDiscount = 0;
    if (quantity > 100) {
      quantityDiscount = 0.10;
    } else if (quantity > 50) {
      quantityDiscount = 0.07;
    } else if (quantity > 20) {
      quantityDiscount = 0.05;
    } else if (quantity > 10) {
      quantityDiscount = 0.02;
    }
    
    // Problema: Lógica de descuento por cliente mezclada
    let customerDiscount = 0;
    if (customer.type === 'vip') {
      customerDiscount = 0.15;
    } else if (customer.type === 'premium') {
      customerDiscount = 0.10;
    } else if (customer.type === 'regular' && customer.yearsAsCustomer > 5) {
      customerDiscount = 0.05;
    }
    
    // Problema: Lógica de descuento estacional mezclada
    let seasonalDiscount = 0;
    const month = new Date().getMonth();
    if (month === 11) { // December
      seasonalDiscount = 0.15; // Christmas discount
    } else if (month === 10) { // November
      seasonalDiscount = 0.20; // Black Friday
    } else if (month === 6 || month === 7) { // Summer
      seasonalDiscount = 0.10;
    }
    
    // Problema: Cálculo final complejo
    const basePrice = product.price * quantity;
    const maxDiscount = Math.max(quantityDiscount, customerDiscount, seasonalDiscount);
    const discountAmount = basePrice * maxDiscount;
    const priceAfterDiscount = basePrice - discountAmount;
    
    // Problema: Cálculo de impuestos y fees mezclado
    const taxRate = customer.location === 'CA' ? 0.0725 : 0.05;
    const tax = priceAfterDiscount * taxRate;
    const processingFee = priceAfterDiscount < 100 ? 5 : 0;
    
    return priceAfterDiscount + tax + processingFee;
  }
}

// ✅ DESPUÉS: Métodos extraídos con parámetros apropiados
class RefactoredPriceCalculator {
  calculatePrice(product: Product, customer: Customer, quantity: number): PriceBreakdown {
    const basePrice = this.calculateBasePrice(product.price, quantity);
    const discount = this.calculateBestDiscount(quantity, customer);
    const priceAfterDiscount = this.applyDiscount(basePrice, discount);
    const tax = this.calculateTax(priceAfterDiscount, customer.location);
    const fees = this.calculateFees(priceAfterDiscount);
    const finalPrice = priceAfterDiscount + tax + fees;
    
    return {
      basePrice,
      discount,
      discountAmount: basePrice * discount,
      priceAfterDiscount,
      tax,
      fees,
      finalPrice
    };
  }
  
  // Método extraído: Precio base
  private calculateBasePrice(unitPrice: number, quantity: number): number {
    return unitPrice * quantity;
  }
  
  // Método extraído: Mejor descuento disponible
  private calculateBestDiscount(quantity: number, customer: Customer): number {
    const discounts = [
      this.getQuantityDiscount(quantity),
      this.getCustomerDiscount(customer),
      this.getSeasonalDiscount(new Date())
    ];
    
    return Math.max(...discounts);
  }
  
  // Método extraído: Descuento por cantidad
  private getQuantityDiscount(quantity: number): number {
    const discountTiers: Array<{ minQuantity: number; discount: number }> = [
      { minQuantity: 100, discount: 0.10 },
      { minQuantity: 50, discount: 0.07 },
      { minQuantity: 20, discount: 0.05 },
      { minQuantity: 10, discount: 0.02 }
    ];
    
    for (const tier of discountTiers) {
      if (quantity > tier.minQuantity) {
        return tier.discount;
      }
    }
    
    return 0;
  }
  
  // Método extraído: Descuento por tipo de cliente
  private getCustomerDiscount(customer: Customer): number {
    const discountRules: Record<string, (c: Customer) => number> = {
      'vip': () => 0.15,
      'premium': () => 0.10,
      'regular': (c) => c.yearsAsCustomer > 5 ? 0.05 : 0
    };
    
    const rule = discountRules[customer.type];
    return rule ? rule(customer) : 0;
  }
  
  // Método extraído: Descuento estacional
  private getSeasonalDiscount(date: Date): number {
    const month = date.getMonth();
    const seasonalDiscounts: Record<number, { name: string; discount: number }> = {
      11: { name: 'Christmas', discount: 0.15 },
      10: { name: 'Black Friday', discount: 0.20 },
      6: { name: 'Summer Sale', discount: 0.10 },
      7: { name: 'Summer Sale', discount: 0.10 }
    };
    
    const seasonal = seasonalDiscounts[month];
    return seasonal ? seasonal.discount : 0;
  }
  
  // Método extraído: Aplicar descuento
  private applyDiscount(basePrice: number, discountRate: number): number {
    return basePrice * (1 - discountRate);
  }
  
  // Método extraído: Cálculo de impuestos
  private calculateTax(amount: number, location: string): number {
    const taxRate = this.getTaxRate(location);
    return amount * taxRate;
  }
  
  // Método extraído: Obtener tasa de impuesto
  private getTaxRate(location: string): number {
    const taxRates: Record<string, number> = {
      'CA': 0.0725,
      'NY': 0.08,
      'TX': 0.0625,
      'FL': 0.06
    };
    
    return taxRates[location] || 0.05; // Default 5%
  }
  
  // Método extraído: Cálculo de fees
  private calculateFees(amount: number): number {
    const PROCESSING_FEE_THRESHOLD = 100;
    const SMALL_ORDER_FEE = 5;
    
    return amount < PROCESSING_FEE_THRESHOLD ? SMALL_ORDER_FEE : 0;
  }
}

// ============================================
// EJEMPLO 3: Extract Method para Mejorar Testabilidad
// ============================================

// ❌ ANTES: Difícil de testear porque todo está mezclado
class UserAuthenticator {
  async authenticate(username: string, password: string): Promise<AuthResult> {
    // Problema: Validación, hash, búsqueda y verificación todo junto
    if (!username || username.length < 3) {
      return { success: false, error: 'Invalid username' };
    }
    
    if (!password || password.length < 8) {
      return { success: false, error: 'Invalid password' };
    }
    
    // Hash del password
    const crypto = require('crypto');
    const hash = crypto.createHash('sha256');
    hash.update(password + 'salt123'); // Problema: Salt hardcodeado
    const hashedPassword = hash.digest('hex');
    
    // Buscar usuario en DB
    const user = await database.query(
      'SELECT * FROM users WHERE username = ?',
      [username]
    );
    
    if (!user) {
      // Log intento fallido
      console.log(`Failed login attempt for username: ${username}`);
      await database.query(
        'INSERT INTO login_attempts (username, success, timestamp) VALUES (?, ?, ?)',
        [username, false, new Date()]
      );
      return { success: false, error: 'User not found' };
    }
    
    // Verificar password
    if (user.password !== hashedPassword) {
      // Log intento fallido
      console.log(`Invalid password for user: ${username}`);
      await database.query(
        'INSERT INTO login_attempts (username, success, timestamp) VALUES (?, ?, ?)',
        [username, false, new Date()]
      );
      
      // Incrementar contador de intentos fallidos
      const attempts = user.failed_attempts + 1;
      await database.query(
        'UPDATE users SET failed_attempts = ? WHERE id = ?',
        [attempts, user.id]
      );
      
      // Bloquear si hay muchos intentos
      if (attempts >= 5) {
        await database.query(
          'UPDATE users SET locked = true WHERE id = ?',
          [user.id]
        );
        return { success: false, error: 'Account locked' };
      }
      
      return { success: false, error: 'Invalid password' };
    }
    
    // Verificar si la cuenta está bloqueada
    if (user.locked) {
      return { success: false, error: 'Account is locked' };
    }
    
    // Login exitoso
    await database.query(
      'UPDATE users SET last_login = ?, failed_attempts = 0 WHERE id = ?',
      [new Date(), user.id]
    );
    
    await database.query(
      'INSERT INTO login_attempts (username, success, timestamp) VALUES (?, ?, ?)',
      [username, true, new Date()]
    );
    
    // Generar token
    const token = crypto.randomBytes(32).toString('hex');
    await database.query(
      'INSERT INTO sessions (user_id, token, created_at) VALUES (?, ?, ?)',
      [user.id, token, new Date()]
    );
    
    return {
      success: true,
      user: {
        id: user.id,
        username: user.username,
        email: user.email
      },
      token: token
    };
  }
}

// ✅ DESPUÉS: Métodos extraídos que son fáciles de testear individualmente
class RefactoredUserAuthenticator {
  private readonly MAX_LOGIN_ATTEMPTS = 5;
  private readonly MIN_USERNAME_LENGTH = 3;
  private readonly MIN_PASSWORD_LENGTH = 8;
  
  constructor(
    private userRepository: UserRepository,
    private sessionService: SessionService,
    private cryptoService: CryptoService,
    private auditService: AuditService
  ) {}
  
  async authenticate(username: string, password: string): Promise<AuthResult> {
    // Validación
    const validationError = this.validateCredentials(username, password);
    if (validationError) {
      return { success: false, error: validationError };
    }
    
    // Buscar usuario
    const user = await this.findUser(username);
    if (!user) {
      await this.handleFailedLogin(username, 'User not found');
      return { success: false, error: 'Invalid credentials' };
    }
    
    // Verificar estado de la cuenta
    const accountError = this.checkAccountStatus(user);
    if (accountError) {
      return { success: false, error: accountError };
    }
    
    // Verificar password
    const isValidPassword = await this.verifyPassword(password, user.passwordHash);
    if (!isValidPassword) {
      await this.handleInvalidPassword(user);
      return { success: false, error: 'Invalid credentials' };
    }
    
    // Login exitoso
    return await this.handleSuccessfulLogin(user);
  }
  
  // Método extraído: Validación de credenciales (puro, fácil de testear)
  private validateCredentials(username: string, password: string): string | null {
    if (!this.isValidUsername(username)) {
      return 'Invalid username format';
    }
    
    if (!this.isValidPassword(password)) {
      return 'Invalid password format';
    }
    
    return null;
  }
  
  // Método extraído: Validación de username (puro)
  private isValidUsername(username: string): boolean {
    return !!username && username.length >= this.MIN_USERNAME_LENGTH;
  }
  
  // Método extraído: Validación de password (puro)
  private isValidPassword(password: string): boolean {
    return !!password && password.length >= this.MIN_PASSWORD_LENGTH;
  }
  
  // Método extraído: Búsqueda de usuario (mockeable)
  private async findUser(username: string): Promise<User | null> {
    return await this.userRepository.findByUsername(username);
  }
  
  // Método extraído: Verificación de estado de cuenta (puro)
  private checkAccountStatus(user: User): string | null {
    if (user.locked) {
      return 'Account is locked';
    }
    
    if (user.suspended) {
      return 'Account is suspended';
    }
    
    if (!user.emailVerified) {
      return 'Email not verified';
    }
    
    return null;
  }
  
  // Método extraído: Verificación de password (mockeable)
  private async verifyPassword(plainPassword: string, hashedPassword: string): Promise<boolean> {
    const hash = await this.cryptoService.hashPassword(plainPassword);
    return hash === hashedPassword;
  }
  
  // Método extraído: Manejo de login fallido
  private async handleFailedLogin(username: string, reason: string): Promise<void> {
    await this.auditService.logFailedLogin(username, reason);
  }
  
  // Método extraído: Manejo de password inválido
  private async handleInvalidPassword(user: User): Promise<void> {
    const newAttempts = user.failedAttempts + 1;
    
    await this.userRepository.incrementFailedAttempts(user.id);
    await this.auditService.logFailedLogin(user.username, 'Invalid password');
    
    if (newAttempts >= this.MAX_LOGIN_ATTEMPTS) {
      await this.lockAccount(user);
    }
  }
  
  // Método extraído: Bloqueo de cuenta
  private async lockAccount(user: User): Promise<void> {
    await this.userRepository.lockAccount(user.id);
    await this.auditService.logAccountLocked(user.username);
  }
  
  // Método extraído: Manejo de login exitoso
  private async handleSuccessfulLogin(user: User): Promise<AuthResult> {
    await this.userRepository.resetFailedAttempts(user.id);
    await this.userRepository.updateLastLogin(user.id);
    await this.auditService.logSuccessfulLogin(user.username);
    
    const session = await this.sessionService.createSession(user.id);
    
    return {
      success: true,
      user: this.sanitizeUser(user),
      token: session.token
    };
  }
  
  // Método extraído: Sanitización de datos de usuario
  private sanitizeUser(user: User): SafeUser {
    return {
      id: user.id,
      username: user.username,
      email: user.email,
      role: user.role
    };
  }
}

// ============================================
// EJEMPLO 4: Extract Method para Condiciones Complejas
// ============================================

// ❌ ANTES: Condiciones complejas difíciles de entender
class LoanApprovalService {
  approveLoan(application: LoanApplication): LoanDecision {
    // Problema: Condición súper compleja e ilegible
    if ((application.creditScore > 750 && application.income > 100000 && application.debtToIncomeRatio < 0.3) ||
        (application.creditScore > 700 && application.income > 80000 && application.debtToIncomeRatio < 0.25 && application.yearsEmployed > 2) ||
        (application.creditScore > 650 && application.income > 60000 && application.debtToIncomeRatio < 0.2 && application.yearsEmployed > 5 && application.hasCollateral) ||
        (application.creditScore > 600 && application.isExistingCustomer && application.previousLoansRepaid >= 3 && application.debtToIncomeRatio < 0.35)) {
      
      // Problema: Más lógica compleja para determinar el monto
      let maxLoanAmount: number;
      if (application.creditScore > 750) {
        maxLoanAmount = application.income * 5;
      } else if (application.creditScore > 700) {
        maxLoanAmount = application.income * 4;
      } else if (application.creditScore > 650) {
        maxLoanAmount = application.income * 3;
      } else {
        maxLoanAmount = application.income * 2;
      }
      
      // Problema: Cálculo de tasa de interés complejo
      let interestRate: number;
      if (application.creditScore > 750 && application.debtToIncomeRatio < 0.2) {
        interestRate = 3.5;
      } else if (application.creditScore > 700 && application.debtToIncomeRatio < 0.3) {
        interestRate = 4.5;
      } else if (application.creditScore > 650) {
        interestRate = 5.5;
      } else {
        interestRate = 6.5;
      }
      
      if (application.isExistingCustomer) {
        interestRate -= 0.5;
      }
      
      return {
        approved: true,
        maxAmount: Math.min(maxLoanAmount, application.requestedAmount),
        interestRate: interestRate,
        term: 360 // 30 years in months
      };
    }
    
    return {
      approved: false,
      reason: 'Does not meet minimum requirements'
    };
  }
}

// ✅ DESPUÉS: Condiciones extraídas en métodos con nombres descriptivos
class RefactoredLoanApprovalService {
  approveLoan(application: LoanApplication): LoanDecision {
    if (!this.meetsApprovalCriteria(application)) {
      return this.createRejection(application);
    }
    
    return this.createApproval(application);
  }
  
  // Método extraído: Criterio principal de aprobación
  private meetsApprovalCriteria(application: LoanApplication): boolean {
    return this.isExcellentCandidate(application) ||
           this.isGoodCandidate(application) ||
           this.isAcceptableWithCollateral(application) ||
           this.isQualifiedExistingCustomer(application);
  }
  
  // Métodos extraídos: Cada tipo de candidato
  private isExcellentCandidate(app: LoanApplication): boolean {
    return app.creditScore > 750 && 
           app.income > 100000 && 
           app.debtToIncomeRatio < 0.3;
  }
  
  private isGoodCandidate(app: LoanApplication): boolean {
    return app.creditScore > 700 && 
           app.income > 80000 && 
           app.debtToIncomeRatio < 0.25 && 
           app.yearsEmployed > 2;
  }
  
  private isAcceptableWithCollateral(app: LoanApplication): boolean {
    return app.creditScore > 650 && 
           app.income > 60000 && 
           app.debtToIncomeRatio < 0.2 && 
           app.yearsEmployed > 5 && 
           app.hasCollateral;
  }
  
  private isQualifiedExistingCustomer(app: LoanApplication): boolean {
    return app.creditScore > 600 && 
           app.isExistingCustomer && 
           app.previousLoansRepaid >= 3 && 
           app.debtToIncomeRatio < 0.35;
  }
  
  // Método extraído: Crear aprobación
  private createApproval(application: LoanApplication): LoanDecision {
    const maxAmount = this.calculateMaxLoanAmount(application);
    const interestRate = this.calculateInterestRate(application);
    const term = this.determineLoanTerm(application);
    
    return {
      approved: true,
      maxAmount: Math.min(maxAmount, application.requestedAmount),
      interestRate,
      term,
      conditions: this.getApprovalConditions(application)
    };
  }
  
  // Método extraído: Crear rechazo
  private createRejection(application: LoanApplication): LoanDecision {
    return {
      approved: false,
      reason: this.getRejectionReason(application),
      suggestions: this.getImprovementSuggestions(application)
    };
  }
  
  // Método extraído: Cálculo de monto máximo
  private calculateMaxLoanAmount(application: LoanApplication): number {
    const multipliers = [
      { minScore: 750, multiplier: 5 },
      { minScore: 700, multiplier: 4 },
      { minScore: 650, multiplier: 3 },
      { minScore: 0, multiplier: 2 }
    ];
    
    const multiplier = multipliers.find(m => application.creditScore > m.minScore)!;
    return application.income * multiplier.multiplier;
  }
  
  // Método extraído: Cálculo de tasa de interés
  private calculateInterestRate(application: LoanApplication): number {
    let baseRate = this.getBaseInterestRate(application);
    baseRate = this.applyCustomerDiscount(baseRate, application);
    baseRate = this.applyCollateralDiscount(baseRate, application);
    
    return Math.max(baseRate, 3.0); // Minimum rate
  }
  
  private getBaseInterestRate(application: LoanApplication): number {
    const rateTable = [
      { minScore: 750, maxDTI: 0.2, rate: 3.5 },
      { minScore: 750, maxDTI: 1.0, rate: 4.0 },
      { minScore: 700, maxDTI: 0.3, rate: 4.5 },
      { minScore: 700, maxDTI: 1.0, rate: 5.0 },
      { minScore: 650, maxDTI: 1.0, rate: 5.5 },
      { minScore: 0, maxDTI: 1.0, rate: 6.5 }
    ];
    
    const rate = rateTable.find(r => 
      application.creditScore > r.minScore && 
      application.debtToIncomeRatio < r.maxDTI
    );
    
    return rate ? rate.rate : 7.0;
  }
  
  private applyCustomerDiscount(rate: number, application: LoanApplication): number {
    if (application.isExistingCustomer) {
      return rate - 0.5;
    }
    return rate;
  }
  
  private applyCollateralDiscount(rate: number, application: LoanApplication): number {
    if (application.hasCollateral) {
      return rate - 0.25;
    }
    return rate;
  }
  
  private determineLoanTerm(application: LoanApplication): number {
    // Lógica para determinar el plazo del préstamo
    if (application.requestedAmount < 100000) {
      return 180; // 15 years
    }
    return 360; // 30 years
  }
  
  private getApprovalConditions(application: LoanApplication): string[] {
    const conditions: string[] = [];
    
    if (application.creditScore < 700) {
      conditions.push('Provide proof of income for last 3 years');
    }
    
    if (application.debtToIncomeRatio > 0.3) {
      conditions.push('Provide explanation for high debt ratio');
    }
    
    if (!application.hasCollateral) {
      conditions.push('Consider providing collateral for better terms');
    }
    
    return conditions;
  }
  
  private getRejectionReason(application: LoanApplication): string {
    if (application.creditScore < 600) {
      return 'Credit score below minimum requirement';
    }
    
    if (application.debtToIncomeRatio > 0.5) {
      return 'Debt-to-income ratio too high';
    }
    
    if (application.income < 30000) {
      return 'Income below minimum requirement';
    }
    
    return 'Does not meet minimum requirements';
  }
  
  private getImprovementSuggestions(application: LoanApplication): string[] {
    const suggestions: string[] = [];
    
    if (application.creditScore < 650) {
      suggestions.push('Improve credit score to at least 650');
    }
    
    if (application.debtToIncomeRatio > 0.35) {
      suggestions.push('Reduce debt-to-income ratio below 35%');
    }
    
    if (application.yearsEmployed < 2) {
      suggestions.push('Maintain stable employment for at least 2 years');
    }
    
    return suggestions;
  }
}

// Interfaces auxiliares
interface Order {
  id: string;
  customer: {
    name: string;
    email: string;
  };
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
  shipping: number;
  total: number;
}

interface Product {
  id: string;
  name: string;
  price: number;
}

interface Customer {
  id: string;
  name: string;
  email: string;
  type: 'regular' | 'premium' | 'vip';
  location: string;
  yearsAsCustomer: number;
}

interface PriceBreakdown {
  basePrice: number;
  discount: number;
  discountAmount: number;
  priceAfterDiscount: number;
  tax: number;
  fees: number;
  finalPrice: number;
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
  role: string;
  locked: boolean;
  suspended: boolean;
  emailVerified: boolean;
  failedAttempts: number;
}

interface SafeUser {
  id: string;
  username: string;
  email: string;
  role: string;
}

interface LoanApplication {
  creditScore: number;
  income: number;
  debtToIncomeRatio: number;
  yearsEmployed: number;
  hasCollateral: boolean;
  isExistingCustomer: boolean;
  previousLoansRepaid: number;
  requestedAmount: number;
}

interface LoanDecision {
  approved: boolean;
  maxAmount?: number;
  interestRate?: number;
  term?: number;
  reason?: string;
  conditions?: string[];
  suggestions?: string[];
}

// Servicios auxiliares (para el ejemplo de autenticación)
interface UserRepository {
  findByUsername(username: string): Promise<User | null>;
  incrementFailedAttempts(userId: string): Promise<void>;
  resetFailedAttempts(userId: string): Promise<void>;
  updateLastLogin(userId: string): Promise<void>;
  lockAccount(userId: string): Promise<void>;
}

interface SessionService {
  createSession(userId: string): Promise<{ token: string }>;
}

interface CryptoService {
  hashPassword(password: string): Promise<string>;
}

interface AuditService {
  logFailedLogin(username: string, reason: string): Promise<void>;
  logSuccessfulLogin(username: string): Promise<void>;
  logAccountLocked(username: string): Promise<void>;
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