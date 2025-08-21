# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 5: Testing y Mantenibilidad
### Para Entrevista Senior Software Engineer

---

## 5. Testing y Mantenibilidad - Asegurando Calidad a Largo Plazo

El testing no es solo sobre encontrar bugs; es sobre diseñar software mantenible, documentar comportamiento esperado, y crear una red de seguridad para refactorización. Un buen suite de tests es la base para mantener software de alta calidad a largo plazo.

### La Pirámide de Testing

La pirámide de testing es un concepto que ayuda a balancear diferentes tipos de tests:

1. **Unit Tests (Base - 70%)**: Muchos tests pequeños y rápidos
2. **Integration Tests (Medio - 20%)**: Tests que verifican interacciones entre componentes
3. **E2E Tests (Punta - 10%)**: Pocos tests que verifican flujos completos

### ¿Por qué es importante el testing?

1. **Confianza para refactorizar**: Los tests te permiten cambiar código sin miedo
2. **Documentación viva**: Los tests muestran cómo usar el código
3. **Diseño mejorado**: TDD lleva a mejor diseño de código
4. **Prevención de regresiones**: Los bugs corregidos no vuelven
5. **Desarrollo más rápido**: Menos tiempo debugging, más tiempo desarrollando

### Principios de buenos tests:

- **F.I.R.S.T**: Fast, Independent, Repeatable, Self-validating, Timely
- **AAA**: Arrange, Act, Assert
- **Un concepto por test**: Cada test verifica una sola cosa
- **Tests como documentación**: Los nombres de tests describen comportamiento
- **Aislamiento**: Los tests no dependen unos de otros

---

## 5.1 Unit Testing Patterns

### Explicación Detallada

Los unit tests verifican unidades individuales de código (funciones, métodos, clases) en aislamiento. Son la base de una estrategia de testing sólida porque son rápidos, confiables y precisos para localizar problemas.

#### Características de buenos unit tests:

1. **Rápidos**: Deben ejecutarse en milisegundos
2. **Aislados**: No dependen de recursos externos (DB, API, filesystem)
3. **Determinísticos**: Siempre dan el mismo resultado
4. **Autocontenidos**: No requieren setup externo
5. **Enfocados**: Prueban una sola unidad de comportamiento

#### Patrones comunes:

- **Test Doubles**: Mocks, Stubs, Spies, Fakes
- **Test Data Builders**: Crear datos de prueba fácilmente
- **Object Mother**: Objetos pre-configurados para tests
- **Assertion Patterns**: Custom matchers y assertions
- **Setup/Teardown**: Preparación y limpieza consistente

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Testing Básico con AAA Pattern
// ============================================

// Código a testear
class ShoppingCart {
  private items: CartItem[] = [];
  private discountCode?: DiscountCode;
  
  addItem(product: Product, quantity: number): void {
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    
    const existingItem = this.items.find(item => item.product.id === product.id);
    
    if (existingItem) {
      existingItem.quantity += quantity;
    } else {
      this.items.push({ product, quantity });
    }
  }
  
  removeItem(productId: string): void {
    const index = this.items.findIndex(item => item.product.id === productId);
    
    if (index === -1) {
      throw new Error('Product not in cart');
    }
    
    this.items.splice(index, 1);
  }
  
  applyDiscount(code: DiscountCode): void {
    if (code.expiryDate < new Date()) {
      throw new Error('Discount code expired');
    }
    
    this.discountCode = code;
  }
  
  getTotal(): number {
    const subtotal = this.items.reduce(
      (sum, item) => sum + (item.product.price * item.quantity),
      0
    );
    
    if (!this.discountCode) {
      return subtotal;
    }
    
    return subtotal * (1 - this.discountCode.percentage / 100);
  }
  
  getItemCount(): number {
    return this.items.reduce((sum, item) => sum + item.quantity, 0);
  }
  
  isEmpty(): boolean {
    return this.items.length === 0;
  }
  
  clear(): void {
    this.items = [];
    this.discountCode = undefined;
  }
}

// ✅ Tests bien estructurados usando AAA pattern
describe('ShoppingCart', () => {
  let cart: ShoppingCart;
  
  // Setup común para todos los tests
  beforeEach(() => {
    cart = new ShoppingCart();
  });
  
  describe('addItem', () => {
    it('should add a new item to empty cart', () => {
      // Arrange
      const product = createProduct({ id: '1', name: 'Laptop', price: 999 });
      
      // Act
      cart.addItem(product, 1);
      
      // Assert
      expect(cart.getItemCount()).toBe(1);
      expect(cart.isEmpty()).toBe(false);
      expect(cart.getTotal()).toBe(999);
    });
    
    it('should increase quantity when adding existing item', () => {
      // Arrange
      const product = createProduct({ id: '1', name: 'Mouse', price: 25 });
      cart.addItem(product, 2);
      
      // Act
      cart.addItem(product, 3);
      
      // Assert
      expect(cart.getItemCount()).toBe(5);
      expect(cart.getTotal()).toBe(125); // 25 * 5
    });
    
    it('should throw error for zero quantity', () => {
      // Arrange
      const product = createProduct({ id: '1', name: 'Keyboard', price: 50 });
      
      // Act & Assert
      expect(() => cart.addItem(product, 0)).toThrow('Quantity must be positive');
    });
    
    it('should throw error for negative quantity', () => {
      // Arrange
      const product = createProduct({ id: '1', name: 'Monitor', price: 300 });
      
      // Act & Assert
      expect(() => cart.addItem(product, -1)).toThrow('Quantity must be positive');
    });
    
    it('should handle multiple different products', () => {
      // Arrange
      const laptop = createProduct({ id: '1', name: 'Laptop', price: 1000 });
      const mouse = createProduct({ id: '2', name: 'Mouse', price: 50 });
      const keyboard = createProduct({ id: '3', name: 'Keyboard', price: 100 });
      
      // Act
      cart.addItem(laptop, 1);
      cart.addItem(mouse, 2);
      cart.addItem(keyboard, 1);
      
      // Assert
      expect(cart.getItemCount()).toBe(4);
      expect(cart.getTotal()).toBe(1200); // 1000 + 100 + 100
    });
  });
  
  describe('removeItem', () => {
    it('should remove item from cart', () => {
      // Arrange
      const product = createProduct({ id: '1', name: 'Phone', price: 800 });
      cart.addItem(product, 2);
      
      // Act
      cart.removeItem('1');
      
      // Assert
      expect(cart.isEmpty()).toBe(true);
      expect(cart.getItemCount()).toBe(0);
    });
    
    it('should throw error when removing non-existent item', () => {
      // Arrange - cart is empty
      
      // Act & Assert
      expect(() => cart.removeItem('non-existent')).toThrow('Product not in cart');
    });
  });
  
  describe('applyDiscount', () => {
    it('should apply valid discount code', () => {
      // Arrange
      const product = createProduct({ id: '1', price: 100 });
      cart.addItem(product, 1);
      const futureDate = new Date();
      futureDate.setDate(futureDate.getDate() + 30);
      const discount = createDiscountCode({ percentage: 20, expiryDate: futureDate });
      
      // Act
      cart.applyDiscount(discount);
      
      // Assert
      expect(cart.getTotal()).toBe(80); // 100 - 20%
    });
    
    it('should throw error for expired discount code', () => {
      // Arrange
      const pastDate = new Date();
      pastDate.setDate(pastDate.getDate() - 1);
      const expiredDiscount = createDiscountCode({ percentage: 10, expiryDate: pastDate });
      
      // Act & Assert
      expect(() => cart.applyDiscount(expiredDiscount)).toThrow('Discount code expired');
    });
  });
  
  describe('getTotal', () => {
    it('should calculate total without discount', () => {
      // Arrange
      const product1 = createProduct({ id: '1', price: 50 });
      const product2 = createProduct({ id: '2', price: 30 });
      cart.addItem(product1, 2); // 100
      cart.addItem(product2, 3); // 90
      
      // Act
      const total = cart.getTotal();
      
      // Assert
      expect(total).toBe(190);
    });
    
    it('should calculate total with discount', () => {
      // Arrange
      const product = createProduct({ id: '1', price: 200 });
      cart.addItem(product, 1);
      const discount = createValidDiscountCode(25); // 25% off
      cart.applyDiscount(discount);
      
      // Act
      const total = cart.getTotal();
      
      // Assert
      expect(total).toBe(150); // 200 - 25%
    });
  });
});

// Test helpers (Test Data Builders)
function createProduct(overrides?: Partial<Product>): Product {
  return {
    id: '1',
    name: 'Test Product',
    price: 10,
    description: 'A test product',
    ...overrides
  };
}

function createDiscountCode(overrides?: Partial<DiscountCode>): DiscountCode {
  return {
    code: 'TEST10',
    percentage: 10,
    expiryDate: new Date('2025-12-31'),
    ...overrides
  };
}

function createValidDiscountCode(percentage: number): DiscountCode {
  const futureDate = new Date();
  futureDate.setDate(futureDate.getDate() + 30);
  return createDiscountCode({ percentage, expiryDate: futureDate });
}

// ============================================
// EJEMPLO 2: Testing con Mocks y Stubs
// ============================================

// Código con dependencias externas
class OrderService {
  constructor(
    private repository: OrderRepository,
    private emailService: EmailService,
    private inventoryService: InventoryService,
    private paymentService: PaymentService
  ) {}
  
  async createOrder(orderData: CreateOrderDto): Promise<Order> {
    // Verificar inventario
    for (const item of orderData.items) {
      const available = await this.inventoryService.checkAvailability(item.productId, item.quantity);
      if (!available) {
        throw new Error(`Product ${item.productId} not available`);
      }
    }
    
    // Procesar pago
    const total = this.calculateTotal(orderData.items);
    const paymentResult = await this.paymentService.processPayment({
      amount: total,
      customerId: orderData.customerId,
      method: orderData.paymentMethod
    });
    
    if (!paymentResult.success) {
      throw new Error('Payment failed');
    }
    
    // Crear orden
    const order: Order = {
      id: this.generateOrderId(),
      customerId: orderData.customerId,
      items: orderData.items,
      total,
      status: 'confirmed',
      paymentId: paymentResult.transactionId,
      createdAt: new Date()
    };
    
    // Guardar orden
    await this.repository.save(order);
    
    // Actualizar inventario
    for (const item of orderData.items) {
      await this.inventoryService.reduceStock(item.productId, item.quantity);
    }
    
    // Enviar confirmación
    await this.emailService.sendOrderConfirmation(order);
    
    return order;
  }
  
  async cancelOrder(orderId: string): Promise<void> {
    const order = await this.repository.findById(orderId);
    
    if (!order) {
      throw new Error('Order not found');
    }
    
    if (order.status === 'shipped') {
      throw new Error('Cannot cancel shipped order');
    }
    
    // Reembolsar pago
    await this.paymentService.refund(order.paymentId);
    
    // Actualizar estado
    order.status = 'cancelled';
    await this.repository.update(order);
    
    // Restaurar inventario
    for (const item of order.items) {
      await this.inventoryService.increaseStock(item.productId, item.quantity);
    }
    
    // Notificar cliente
    await this.emailService.sendCancellationNotice(order);
  }
  
  private calculateTotal(items: OrderItem[]): number {
    return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }
  
  private generateOrderId(): string {
    return `ORD-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// ✅ Tests con mocks bien organizados
describe('OrderService', () => {
  let orderService: OrderService;
  let mockRepository: jest.Mocked<OrderRepository>;
  let mockEmailService: jest.Mocked<EmailService>;
  let mockInventoryService: jest.Mocked<InventoryService>;
  let mockPaymentService: jest.Mocked<PaymentService>;
  
  beforeEach(() => {
    // Crear mocks
    mockRepository = createMockOrderRepository();
    mockEmailService = createMockEmailService();
    mockInventoryService = createMockInventoryService();
    mockPaymentService = createMockPaymentService();
    
    // Crear instancia con mocks
    orderService = new OrderService(
      mockRepository,
      mockEmailService,
      mockInventoryService,
      mockPaymentService
    );
  });
  
  describe('createOrder', () => {
    it('should create order successfully with all validations', async () => {
      // Arrange
      const orderData = createOrderData();
      
      // Setup mock responses
      mockInventoryService.checkAvailability.mockResolvedValue(true);
      mockPaymentService.processPayment.mockResolvedValue({
        success: true,
        transactionId: 'TXN-123'
      });
      mockRepository.save.mockResolvedValue(undefined);
      mockInventoryService.reduceStock.mockResolvedValue(undefined);
      mockEmailService.sendOrderConfirmation.mockResolvedValue(undefined);
      
      // Act
      const result = await orderService.createOrder(orderData);
      
      // Assert - verificar resultado
      expect(result).toMatchObject({
        customerId: orderData.customerId,
        items: orderData.items,
        total: 150, // 50 * 2 + 25 * 2
        status: 'confirmed',
        paymentId: 'TXN-123'
      });
      
      // Assert - verificar que se llamaron las dependencias correctamente
      expect(mockInventoryService.checkAvailability).toHaveBeenCalledTimes(2);
      expect(mockInventoryService.checkAvailability).toHaveBeenCalledWith('PROD-1', 2);
      expect(mockInventoryService.checkAvailability).toHaveBeenCalledWith('PROD-2', 2);
      
      expect(mockPaymentService.processPayment).toHaveBeenCalledWith({
        amount: 150,
        customerId: 'CUST-123',
        method: 'credit_card'
      });
      
      expect(mockRepository.save).toHaveBeenCalledWith(
        expect.objectContaining({
          customerId: 'CUST-123',
          total: 150,
          status: 'confirmed'
        })
      );
      
      expect(mockInventoryService.reduceStock).toHaveBeenCalledTimes(2);
      expect(mockEmailService.sendOrderConfirmation).toHaveBeenCalledTimes(1);
    });
    
    it('should throw error when product is not available', async () => {
      // Arrange
      const orderData = createOrderData();
      mockInventoryService.checkAvailability
        .mockResolvedValueOnce(true)  // First product available
        .mockResolvedValueOnce(false); // Second product not available
      
      // Act & Assert
      await expect(orderService.createOrder(orderData))
        .rejects
        .toThrow('Product PROD-2 not available');
      
      // Verify that payment was not processed
      expect(mockPaymentService.processPayment).not.toHaveBeenCalled();
      expect(mockRepository.save).not.toHaveBeenCalled();
    });
    
    it('should throw error when payment fails', async () => {
      // Arrange
      const orderData = createOrderData();
      mockInventoryService.checkAvailability.mockResolvedValue(true);
      mockPaymentService.processPayment.mockResolvedValue({
        success: false,
        error: 'Insufficient funds'
      });
      
      // Act & Assert
      await expect(orderService.createOrder(orderData))
        .rejects
        .toThrow('Payment failed');
      
      // Verify that order was not saved
      expect(mockRepository.save).not.toHaveBeenCalled();
      expect(mockInventoryService.reduceStock).not.toHaveBeenCalled();
      expect(mockEmailService.sendOrderConfirmation).not.toHaveBeenCalled();
    });
    
    it('should rollback inventory if email fails', async () => {
      // Arrange
      const orderData = createOrderData();
      mockInventoryService.checkAvailability.mockResolvedValue(true);
      mockPaymentService.processPayment.mockResolvedValue({
        success: true,
        transactionId: 'TXN-123'
      });
      mockRepository.save.mockResolvedValue(undefined);
      mockInventoryService.reduceStock.mockResolvedValue(undefined);
      mockEmailService.sendOrderConfirmation.mockRejectedValue(new Error('Email service down'));
      
      // Act & Assert
      await expect(orderService.createOrder(orderData))
        .rejects
        .toThrow('Email service down');
      
      // En una implementación real, aquí verificaríamos el rollback
      // Por ahora, solo verificamos que se intentó enviar el email
      expect(mockEmailService.sendOrderConfirmation).toHaveBeenCalled();
    });
  });
  
  describe('cancelOrder', () => {
    it('should cancel order successfully', async () => {
      // Arrange
      const order = createOrder({ status: 'confirmed' });
      mockRepository.findById.mockResolvedValue(order);
      mockPaymentService.refund.mockResolvedValue({ success: true });
      mockRepository.update.mockResolvedValue(undefined);
      mockInventoryService.increaseStock.mockResolvedValue(undefined);
      mockEmailService.sendCancellationNotice.mockResolvedValue(undefined);
      
      // Act
      await orderService.cancelOrder('ORD-123');
      
      // Assert
      expect(mockPaymentService.refund).toHaveBeenCalledWith('TXN-123');
      expect(mockRepository.update).toHaveBeenCalledWith(
        expect.objectContaining({
          status: 'cancelled'
        })
      );
      expect(mockInventoryService.increaseStock).toHaveBeenCalledTimes(2);
      expect(mockEmailService.sendCancellationNotice).toHaveBeenCalled();
    });
    
    it('should throw error for non-existent order', async () => {
      // Arrange
      mockRepository.findById.mockResolvedValue(null);
      
      // Act & Assert
      await expect(orderService.cancelOrder('NON-EXISTENT'))
        .rejects
        .toThrow('Order not found');
    });
    
    it('should throw error for shipped order', async () => {
      // Arrange
      const shippedOrder = createOrder({ status: 'shipped' });
      mockRepository.findById.mockResolvedValue(shippedOrder);
      
      // Act & Assert
      await expect(orderService.cancelOrder('ORD-123'))
        .rejects
        .toThrow('Cannot cancel shipped order');
      
      // Verify no actions were taken
      expect(mockPaymentService.refund).not.toHaveBeenCalled();
      expect(mockRepository.update).not.toHaveBeenCalled();
    });
  });
});

// Mock factories
function createMockOrderRepository(): jest.Mocked<OrderRepository> {
  return {
    save: jest.fn(),
    findById: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
    findByCustomer: jest.fn()
  };
}

function createMockEmailService(): jest.Mocked<EmailService> {
  return {
    sendOrderConfirmation: jest.fn(),
    sendCancellationNotice: jest.fn(),
    sendShippingNotification: jest.fn()
  };
}

function createMockInventoryService(): jest.Mocked<InventoryService> {
  return {
    checkAvailability: jest.fn(),
    reduceStock: jest.fn(),
    increaseStock: jest.fn(),
    getStock: jest.fn()
  };
}

function createMockPaymentService(): jest.Mocked<PaymentService> {
  return {
    processPayment: jest.fn(),
    refund: jest.fn(),
    getTransactionStatus: jest.fn()
  };
}

// Test data builders
function createOrderData(overrides?: Partial<CreateOrderDto>): CreateOrderDto {
  return {
    customerId: 'CUST-123',
    items: [
      { productId: 'PROD-1', quantity: 2, price: 50 },
      { productId: 'PROD-2', quantity: 2, price: 25 }
    ],
    paymentMethod: 'credit_card',
    shippingAddress: {
      street: '123 Main St',
      city: 'Seattle',
      state: 'WA',
      zip: '98101'
    },
    ...overrides
  };
}

function createOrder(overrides?: Partial<Order>): Order {
  return {
    id: 'ORD-123',
    customerId: 'CUST-123',
    items: [
      { productId: 'PROD-1', quantity: 2, price: 50 },
      { productId: 'PROD-2', quantity: 2, price: 25 }
    ],
    total: 150,
    status: 'confirmed',
    paymentId: 'TXN-123',
    createdAt: new Date('2024-01-01'),
    ...overrides
  };
}

// ============================================
// EJEMPLO 3: Test Driven Development (TDD)
// ============================================

// Empezamos con los tests ANTES de escribir el código

describe('PasswordValidator - TDD Example', () => {
  let validator: PasswordValidator;
  
  beforeEach(() => {
    validator = new PasswordValidator();
  });
  
  describe('Basic validation', () => {
    it('should reject empty password', () => {
      const result = validator.validate('');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password is required');
    });
    
    it('should reject password shorter than 8 characters', () => {
      const result = validator.validate('Pass1!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password must be at least 8 characters');
    });
    
    it('should reject password longer than 128 characters', () => {
      const longPassword = 'a'.repeat(129);
      const result = validator.validate(longPassword);
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password must not exceed 128 characters');
    });
  });
  
  describe('Character requirements', () => {
    it('should require at least one uppercase letter', () => {
      const result = validator.validate('password123!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password must contain at least one uppercase letter');
    });
    
    it('should require at least one lowercase letter', () => {
      const result = validator.validate('PASSWORD123!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password must contain at least one lowercase letter');
    });
    
    it('should require at least one number', () => {
      const result = validator.validate('Password!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password must contain at least one number');
    });
    
    it('should require at least one special character', () => {
      const result = validator.validate('Password123');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password must contain at least one special character');
    });
  });
  
  describe('Common password patterns', () => {
    it('should reject common passwords', () => {
      const commonPasswords = ['Password123!', 'Admin123!', 'Welcome123!'];
      
      commonPasswords.forEach(password => {
        const result = validator.validate(password);
        expect(result.isValid).toBe(false);
        expect(result.errors).toContain('Password is too common');
      });
    });
    
    it('should reject passwords with sequential characters', () => {
      const result = validator.validate('Abcd1234!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password contains sequential characters');
    });
    
    it('should reject passwords with repeated characters', () => {
      const result = validator.validate('Paaasword123!');
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('Password contains too many repeated characters');
    });
  });
  
  describe('Valid passwords', () => {
    it('should accept valid password with all requirements', () => {
      const result = validator.validate('MyS3cur3P@ssw0rd');
      expect(result.isValid).toBe(true);
      expect(result.errors).toHaveLength(0);
    });
    
    it('should return strength score for valid passwords', () => {
      const weakResult = validator.validate('MyPass1!');
      const strongResult = validator.validate('MyV3ryS3cur3P@ssw0rd!2024');
      
      expect(weakResult.strength).toBe('weak');
      expect(strongResult.strength).toBe('strong');
    });
  });
  
  describe('Configuration', () => {
    it('should allow custom configuration', () => {
      const customValidator = new PasswordValidator({
        minLength: 12,
        requireUppercase: true,
        requireLowercase: true,
        requireNumbers: true,
        requireSpecialChars: false
      });
      
      const result = customValidator.validate('MyPassword123');
      expect(result.isValid).toBe(true);
    });
  });
});

// Ahora implementamos el código para pasar los tests
class PasswordValidator {
  private config: PasswordConfig;
  private commonPasswords = new Set([
    'password123!', 'admin123!', 'welcome123!', 'letmein123!', 'qwerty123!'
  ]);
  
  constructor(config?: Partial<PasswordConfig>) {
    this.config = {
      minLength: 8,
      maxLength: 128,
      requireUppercase: true,
      requireLowercase: true,
      requireNumbers: true,
      requireSpecialChars: true,
      ...config
    };
  }
  
  validate(password: string): ValidationResult {
    const errors: string[] = [];
    
    // Validaciones básicas
    if (!password) {
      errors.push('Password is required');
      return { isValid: false, errors };
    }
    
    if (password.length < this.config.minLength) {
      errors.push(`Password must be at least ${this.config.minLength} characters`);
    }
    
    if (password.length > this.config.maxLength) {
      errors.push(`Password must not exceed ${this.config.maxLength} characters`);
    }
    
    // Validaciones de caracteres
    if (this.config.requireUppercase && !/[A-Z]/.test(password)) {
      errors.push('Password must contain at least one uppercase letter');
    }
    
    if (this.config.requireLowercase && !/[a-z]/.test(password)) {
      errors.push('Password must contain at least one lowercase letter');
    }
    
    if (this.config.requireNumbers && !/\d/.test(password)) {
      errors.push('Password must contain at least one number');
    }
    
    if (this.config.requireSpecialChars && !/[!@#$%^&*(),.?":{}|<>]/.test(password)) {
      errors.push('Password must contain at least one special character');
    }
    
    // Validaciones de patrones
    if (this.isCommonPassword(password)) {
      errors.push('Password is too common');
    }
    
    if (this.hasSequentialCharacters(password)) {
      errors.push('Password contains sequential characters');
    }
    
    if (this.hasRepeatedCharacters(password)) {
      errors.push('Password contains too many repeated characters');
    }
    
    // Calcular fuerza si es válido
    const isValid = errors.length === 0;
    const strength = isValid ? this.calculateStrength(password) : undefined;
    
    return {
      isValid,
      errors,
      strength
    };
  }
  
  private isCommonPassword(password: string): boolean {
    return this.commonPasswords.has(password.toLowerCase());
  }
  
  private hasSequentialCharacters(password: string): boolean {
    const sequences = ['abcd', 'bcde', 'cdef', '1234', '2345', '3456'];
    const lowerPassword = password.toLowerCase();
    
    return sequences.some(seq => lowerPassword.includes(seq));
  }
  
  private hasRepeatedCharacters(password: string): boolean {
    return /(.)\1{2,}/.test(password);
  }
  
  private calculateStrength(password: string): 'weak' | 'medium' | 'strong' {
    let score = 0;
    
    // Length scoring
    if (password.length >= 12) score++;
    if (password.length >= 16) score++;
    
    // Complexity scoring
    if (/[A-Z].*[A-Z]/.test(password)) score++; // Multiple uppercase
    if (/[a-z].*[a-z]/.test(password)) score++; // Multiple lowercase
    if (/\d.*\d/.test(password)) score++; // Multiple numbers
    if (/[!@#$%^&*()].*[!@#$%^&*()]/.test(password)) score++; // Multiple special
    
    // Variety scoring
    const uniqueChars = new Set(password).size;
    if (uniqueChars > password.length * 0.7) score++;
    
    if (score >= 6) return 'strong';
    if (score >= 3) return 'medium';
    return 'weak';
  }
}

// ============================================
// EJEMPLO 4: Integration Testing
// ============================================

// Tests de integración que verifican múltiples componentes trabajando juntos

describe('Order Processing Integration', () => {
  let orderService: OrderService;
  let database: TestDatabase;
  let emailSpy: EmailServiceSpy;
  
  beforeAll(async () => {
    // Setup de base de datos de prueba
    database = await TestDatabase.create();
    await database.migrate();
  });
  
  afterAll(async () => {
    await database.close();
  });
  
  beforeEach(async () => {
    // Limpiar datos entre tests
    await database.clear();
    
    // Crear servicios reales pero con algunos mocks
    const repository = new RealOrderRepository(database);
    emailSpy = new EmailServiceSpy();
    const inventoryService = new RealInventoryService(database);
    const paymentService = new MockPaymentService(); // Mock para no cobrar de verdad
    
    orderService = new OrderService(
      repository,
      emailSpy,
      inventoryService,
      paymentService
    );
    
    // Seed data
    await seedTestData(database);
  });
  
  it('should process complete order flow', async () => {
    // Arrange
    const customerId = await createTestCustomer(database);
    const productId = await createTestProduct(database, { stock: 10 });
    
    const orderData: CreateOrderDto = {
      customerId,
      items: [{ productId, quantity: 2, price: 50 }],
      paymentMethod: 'credit_card',
      shippingAddress: createTestAddress()
    };
    
    // Act
    const order = await orderService.createOrder(orderData);
    
    // Assert - verificar que la orden se creó correctamente
    expect(order.id).toBeDefined();
    expect(order.status).toBe('confirmed');
    
    // Assert - verificar que se guardó en la base de datos
    const savedOrder = await database.query(
      'SELECT * FROM orders WHERE id = ?',
      [order.id]
    );
    expect(savedOrder).toBeDefined();
    
    // Assert - verificar que se actualizó el inventario
    const updatedStock = await database.query(
      'SELECT stock FROM products WHERE id = ?',
      [productId]
    );
    expect(updatedStock.stock).toBe(8); // 10 - 2
    
    // Assert - verificar que se envió el email
    expect(emailSpy.sentEmails).toHaveLength(1);
    expect(emailSpy.sentEmails[0].type).toBe('order_confirmation');
  });
  
  it('should handle concurrent orders correctly', async () => {
    // Arrange
    const customerId = await createTestCustomer(database);
    const productId = await createTestProduct(database, { stock: 5 });
    
    const orderData1 = createOrderData({ customerId, productId, quantity: 3 });
    const orderData2 = createOrderData({ customerId, productId, quantity: 3 });
    
    // Act - crear órdenes concurrentemente
    const results = await Promise.allSettled([
      orderService.createOrder(orderData1),
      orderService.createOrder(orderData2)
    ]);
    
    // Assert - una debe succeeder, otra debe fallar
    const succeeded = results.filter(r => r.status === 'fulfilled');
    const failed = results.filter(r => r.status === 'rejected');
    
    expect(succeeded).toHaveLength(1);
    expect(failed).toHaveLength(1);
    
    // Verificar que el inventario es correcto
    const finalStock = await database.query(
      'SELECT stock FROM products WHERE id = ?',
      [productId]
    );
    expect(finalStock.stock).toBe(2); // 5 - 3
  });
});

// Interfaces y tipos auxiliares
interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
}

interface CartItem {
  product: Product;
  quantity: number;
}

interface DiscountCode {
  code: string;
  percentage: number;
  expiryDate: Date;
}

interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

interface Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  total: number;
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  paymentId?: string;
  createdAt: Date;
}

interface CreateOrderDto {
  customerId: string;
  items: OrderItem[];
  paymentMethod: string;
  shippingAddress: any;
}

interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
  update(order: Order): Promise<void>;
  delete(id: string): Promise<void>;
  findByCustomer(customerId: string): Promise<Order[]>;
}

interface EmailService {
  sendOrderConfirmation(order: Order): Promise<void>;
  sendCancellationNotice(order: Order): Promise<void>;
  sendShippingNotification(order: Order): Promise<void>;
}

interface InventoryService {
  checkAvailability(productId: string, quantity: number): Promise<boolean>;
  reduceStock(productId: string, quantity: number): Promise<void>;
  increaseStock(productId: string, quantity: number): Promise<void>;
  getStock(productId: string): Promise<number>;
}

interface PaymentService {
  processPayment(data: any): Promise<any>;
  refund(transactionId: string): Promise<any>;
  getTransactionStatus(transactionId: string): Promise<any>;
}

interface ValidationResult {
  isValid: boolean;
  errors: string[];
  strength?: 'weak' | 'medium' | 'strong';
}

interface PasswordConfig {
  minLength: number;
  maxLength: number;
  requireUppercase: boolean;
  requireLowercase: boolean;
  requireNumbers: boolean;
  requireSpecialChars: boolean;
}

// Test utilities
class TestDatabase {
  static async create(): Promise<TestDatabase> {
    // Crear base de datos en memoria para tests
    return new TestDatabase();
  }
  
  async migrate(): Promise<void> {
    // Ejecutar migraciones
  }
  
  async clear(): Promise<void> {
    // Limpiar todas las tablas
  }
  
  async close(): Promise<void> {
    // Cerrar conexión
  }
  
  async query(sql: string, params?: any[]): Promise<any> {
    // Ejecutar query
    return {};
  }
}

class EmailServiceSpy implements EmailService {
  sentEmails: any[] = [];
  
  async sendOrderConfirmation(order: Order): Promise<void> {
    this.sentEmails.push({ type: 'order_confirmation', order });
  }
  
  async sendCancellationNotice(order: Order): Promise<void> {
    this.sentEmails.push({ type: 'cancellation', order });
  }
  
  async sendShippingNotification(order: Order): Promise<void> {
    this.sentEmails.push({ type: 'shipping', order });
  }
}

async function seedTestData(database: TestDatabase): Promise<void> {
  // Insertar datos de prueba
}

async function createTestCustomer(database: TestDatabase): Promise<string> {
  return 'CUST-TEST-123';
}

async function createTestProduct(database: TestDatabase, data: any): Promise<string> {
  return 'PROD-TEST-123';
}

function createTestAddress(): any {
  return {
    street: '123 Test St',
    city: 'Test City',
    state: 'TS',
    zip: '12345'
  };
}
```

### Mejores prácticas de testing:

1. **Tests independientes**: Cada test debe poder ejecutarse solo
2. **Tests determinísticos**: Siempre el mismo resultado
3. **Tests rápidos**: Los unit tests deben ser milisegundos
4. **Tests descriptivos**: Los nombres deben explicar qué se está testeando
5. **Un assert por test**: Idealmente, verificar una sola cosa
6. **Usar test builders**: Para crear datos de prueba fácilmente
7. **Mockear dependencias externas**: En unit tests
8. **Tests de regresión**: Agregar test cuando se corrige un bug

El testing bien hecho es fundamental para mantener código de calidad y poder refactorizar con confianza.