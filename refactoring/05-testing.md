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
  private discount = 0;
  
  addItem(product: Product, quantity: number): void {
    if (quantity <= 0) throw new Error('Invalid quantity');
    
    const existing = this.items.find(i => i.id === product.id);
    if (existing) {
      existing.quantity += quantity;
    } else {
      this.items.push({ id: product.id, price: product.price, quantity });
    }
  }
  
  applyDiscount(percentage: number): void {
    this.discount = percentage;
  }
  
  getTotal(): number {
    const subtotal = this.items.reduce(
      (sum, item) => sum + (item.price * item.quantity), 0
    );
    return subtotal * (1 - this.discount / 100);
  }
  
  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

// ✅ Tests bien estructurados usando AAA pattern
describe('ShoppingCart', () => {
  let cart: ShoppingCart;
  
  beforeEach(() => {
    cart = new ShoppingCart();
  });
  
  describe('addItem', () => {
    it('should add item to cart', () => {
      // Arrange
      const product = { id: '1', price: 100 };
      
      // Act
      cart.addItem(product, 2);
      
      // Assert
      expect(cart.getTotal()).toBe(200);
      expect(cart.isEmpty()).toBe(false);
    });
    
    it('should increase quantity for existing item', () => {
      // Arrange
      const product = { id: '1', price: 50 };
      cart.addItem(product, 2);
      
      // Act
      cart.addItem(product, 3);
      
      // Assert
      expect(cart.getTotal()).toBe(250); // 50 * 5
    });
    
    it('should throw error for invalid quantity', () => {
      const product = { id: '1', price: 50 };
      expect(() => cart.addItem(product, 0)).toThrow('Invalid quantity');
      expect(() => cart.addItem(product, -1)).toThrow('Invalid quantity');
    });
  });
  
  describe('applyDiscount', () => {
    it('should apply discount to total', () => {
      // Arrange
      cart.addItem({ id: '1', price: 100 }, 1);
      
      // Act
      cart.applyDiscount(20);
      
      // Assert
      expect(cart.getTotal()).toBe(80); // 100 - 20%
    });
  });
  
  describe('getTotal', () => {
    it('should calculate total with multiple items', () => {
      cart.addItem({ id: '1', price: 50 }, 2);  // 100
      cart.addItem({ id: '2', price: 30 }, 3);  // 90
      expect(cart.getTotal()).toBe(190);
    });
    
    it('should calculate total with discount', () => {
      cart.addItem({ id: '1', price: 200 }, 1);
      cart.applyDiscount(25);
      expect(cart.getTotal()).toBe(150); // 200 - 25%
    });
  });
});

// ============================================
// EJEMPLO 2: Testing con Mocks y Stubs
// ============================================

// Código con dependencias externas
class OrderService {
  constructor(
    private repository: OrderRepository,
    private paymentService: PaymentService,
    private emailService: EmailService
  ) {}
  
  async createOrder(orderData: CreateOrderDto): Promise<Order> {
    // Procesar pago
    const total = orderData.items.reduce(
      (sum, item) => sum + item.price * item.quantity, 0
    );
    
    const payment = await this.paymentService.process(total);
    if (!payment.success) {
      throw new Error('Payment failed');
    }
    
    // Crear y guardar orden
    const order: Order = {
      id: `ORD-${Date.now()}`,
      customerId: orderData.customerId,
      items: orderData.items,
      total,
      status: 'confirmed'
    };
    
    await this.repository.save(order);
    await this.emailService.sendConfirmation(order);
    
    return order;
  }
  
  async cancelOrder(orderId: string): Promise<void> {
    const order = await this.repository.findById(orderId);
    
    if (!order) throw new Error('Order not found');
    if (order.status === 'shipped') throw new Error('Cannot cancel');
    
    order.status = 'cancelled';
    await this.repository.update(order);
  }
}

// ✅ Tests con mocks bien organizados
describe('OrderService', () => {
  let service: OrderService;
  let mockRepo: jest.Mocked<OrderRepository>;
  let mockPayment: jest.Mocked<PaymentService>;
  let mockEmail: jest.Mocked<EmailService>;
  
  beforeEach(() => {
    mockRepo = {
      save: jest.fn(),
      findById: jest.fn(),
      update: jest.fn()
    };
    mockPayment = {
      process: jest.fn()
    };
    mockEmail = {
      sendConfirmation: jest.fn()
    };
    
    service = new OrderService(mockRepo, mockPayment, mockEmail);
  });
  
  describe('createOrder', () => {
    it('should create order successfully', async () => {
      // Arrange
      const orderData = {
        customerId: 'C1',
        items: [{ productId: 'P1', quantity: 2, price: 50 }]
      };
      mockPayment.process.mockResolvedValue({ success: true });
      
      // Act
      const result = await service.createOrder(orderData);
      
      // Assert
      expect(result.total).toBe(100);
      expect(result.status).toBe('confirmed');
      expect(mockRepo.save).toHaveBeenCalled();
      expect(mockEmail.sendConfirmation).toHaveBeenCalled();
    });
    
    it('should throw error when payment fails', async () => {
      const orderData = {
        customerId: 'C1',
        items: [{ productId: 'P1', quantity: 1, price: 100 }]
      };
      mockPayment.process.mockResolvedValue({ success: false });
      
      await expect(service.createOrder(orderData))
        .rejects.toThrow('Payment failed');
      
      expect(mockRepo.save).not.toHaveBeenCalled();
    });
  });
  
  describe('cancelOrder', () => {
    it('should cancel order', async () => {
      const order = { id: '1', status: 'confirmed', total: 100 };
      mockRepo.findById.mockResolvedValue(order);
      
      await service.cancelOrder('1');
      
      expect(mockRepo.update).toHaveBeenCalledWith(
        expect.objectContaining({ status: 'cancelled' })
      );
    });
    
    it('should throw error for shipped order', async () => {
      mockRepo.findById.mockResolvedValue({ id: '1', status: 'shipped' });
      
      await expect(service.cancelOrder('1'))
        .rejects.toThrow('Cannot cancel');
    });
  });
});

// ============================================
// EJEMPLO 3: Test Driven Development (TDD)
// ============================================

// Empezamos con los tests ANTES de escribir el código

describe('PasswordValidator - TDD', () => {
  let validator: PasswordValidator;
  
  beforeEach(() => {
    validator = new PasswordValidator();
  });
  
  it('should reject short passwords', () => {
    const result = validator.validate('Pass1!');
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('Too short');
  });
  
  it('should require uppercase, lowercase, number and special char', () => {
    expect(validator.validate('password123!').isValid).toBe(false); // no uppercase
    expect(validator.validate('PASSWORD123!').isValid).toBe(false); // no lowercase
    expect(validator.validate('Password!').isValid).toBe(false);    // no number
    expect(validator.validate('Password123').isValid).toBe(false);  // no special
  });
  
  it('should reject common passwords', () => {
    const result = validator.validate('Password123!');
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('Too common');
  });
  
  it('should accept valid password', () => {
    const result = validator.validate('MyS3cur3P@ss');
    expect(result.isValid).toBe(true);
    expect(result.strength).toBeDefined();
  });
});

// Ahora implementamos el código para pasar los tests
class PasswordValidator {
  private commonPasswords = new Set(['password123!', 'admin123!']);
  
  validate(password: string): ValidationResult {
    const errors: string[] = [];
    
    if (password.length < 8) {
      errors.push('Too short');
    }
    
    if (!/[A-Z]/.test(password)) errors.push('No uppercase');
    if (!/[a-z]/.test(password)) errors.push('No lowercase');
    if (!/\d/.test(password)) errors.push('No number');
    if (!/[!@#$%^&*]/.test(password)) errors.push('No special char');
    
    if (this.commonPasswords.has(password.toLowerCase())) {
      errors.push('Too common');
    }
    
    const isValid = errors.length === 0;
    const strength = isValid ? this.getStrength(password) : undefined;
    
    return { isValid, errors, strength };
  }
  
  private getStrength(password: string): 'weak' | 'strong' {
    return password.length >= 12 ? 'strong' : 'weak';
  }
}

// ============================================
// EJEMPLO 4: Integration Testing
// ============================================

// Tests de integración que verifican múltiples componentes trabajando juntos

describe('Order Processing Integration', () => {
  let orderService: OrderService;
  let database: TestDatabase;
  
  beforeAll(async () => {
    database = await TestDatabase.create();
  });
  
  beforeEach(async () => {
    await database.clear();
    
    const repository = new RealOrderRepository(database);
    const paymentService = new MockPaymentService();
    const emailService = new MockEmailService();
    
    orderService = new OrderService(repository, paymentService, emailService);
  });
  
  it('should process complete order flow', async () => {
    // Arrange
    const orderData = {
      customerId: 'C1',
      items: [{ productId: 'P1', quantity: 2, price: 50 }]
    };
    
    // Act
    const order = await orderService.createOrder(orderData);
    
    // Assert
    expect(order.id).toBeDefined();
    expect(order.status).toBe('confirmed');
    
    const saved = await database.query('SELECT * FROM orders WHERE id = ?', [order.id]);
    expect(saved).toBeDefined();
  });
  
  it('should handle concurrent orders', async () => {
    const orderData = {
      customerId: 'C1',
      items: [{ productId: 'P1', quantity: 3, price: 50 }]
    };
    
    // Act - crear órdenes concurrentemente
    const results = await Promise.allSettled([
      orderService.createOrder(orderData),
      orderService.createOrder(orderData)
    ]);
    
    // Assert - verificar comportamiento esperado
    expect(results.some(r => r.status === 'fulfilled')).toBe(true);
  });
});

// Interfaces auxiliares
interface Product {
  id: string;
  price: number;
}

interface CartItem {
  id: string;
  price: number;
  quantity: number;
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
  status: 'confirmed' | 'shipped' | 'cancelled';
}

interface CreateOrderDto {
  customerId: string;
  items: OrderItem[];
}

interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
  update(order: Order): Promise<void>;
}

interface PaymentService {
  process(amount: number): Promise<{ success: boolean }>;
}

interface EmailService {
  sendConfirmation(order: Order): Promise<void>;
}

interface ValidationResult {
  isValid: boolean;
  errors: string[];
  strength?: 'weak' | 'strong';
}

// Test utilities
class TestDatabase {
  static async create(): Promise<TestDatabase> {
    return new TestDatabase();
  }
  
  async clear(): Promise<void> {}
  
  async query(sql: string, params?: any[]): Promise<any> {
    return {};
  }
}

class RealOrderRepository implements OrderRepository {
  constructor(private db: TestDatabase) {}
  async save(order: Order): Promise<void> {}
  async findById(id: string): Promise<Order | null> { return null; }
  async update(order: Order): Promise<void> {}
}

class MockPaymentService implements PaymentService {
  async process(amount: number): Promise<{ success: boolean }> {
    return { success: true };
  }
}

class MockEmailService implements EmailService {
  async sendConfirmation(order: Order): Promise<void> {}
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