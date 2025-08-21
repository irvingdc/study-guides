# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 3: Code Smells y Técnicas de Refactoring
### Para Entrevista Senior Software Engineer - Zillow

---

## 3. Code Smells - Identificando Problemas en el Código

Los "Code Smells" (olores de código) son indicadores de que algo podría estar mal en tu código. No son bugs ni errores que impidan que el programa funcione, pero son señales de debilidades en el diseño que pueden ralentizar el desarrollo o aumentar el riesgo de bugs y fallas en el futuro.

### ¿Por qué es importante conocer los Code Smells?

1. **Detección temprana de problemas**: Identificar problemas antes de que se vuelvan críticos
2. **Mejora continua del código**: Saber qué refactorizar y cuándo
3. **Comunicación en el equipo**: Vocabulario común para discutir problemas
4. **Mantenibilidad a largo plazo**: Prevenir la deuda técnica
5. **Calidad del código**: Mantener estándares altos consistentemente

### Categorías de Code Smells:

- **Bloaters**: Código que ha crecido demasiado y es difícil de manejar
- **Object-Orientation Abusers**: Mal uso de conceptos de orientación a objetos
- **Change Preventers**: Código que hace difícil modificar el sistema
- **Dispensables**: Código innecesario que debería eliminarse
- **Couplers**: Excesivo acoplamiento entre clases

---

## 3.1 Long Method (Método Largo)

### Explicación Detallada

Un método largo es uno que intenta hacer demasiado. Como regla general, si un método tiene más de 10-20 líneas, probablemente está haciendo más de una cosa. Los métodos largos son difíciles de entender, testear, reutilizar y mantener.

#### Señales de que un método es muy largo:

1. **No puedes ver todo el método en tu pantalla sin scrollear**
2. **Necesitas comentarios para explicar secciones del método**
3. **Tiene múltiples niveles de indentación**
4. **Es difícil elegir un nombre descriptivo para el método**
5. **Contiene múltiples responsabilidades o algoritmos**
6. **Usa muchas variables locales**
7. **Es difícil de testear completamente**

#### Problemas que causa:

- **Dificulta la comprensión**: Más código = más complejidad cognitiva
- **Dificulta el testing**: Muchos caminos de ejecución que testear
- **Dificulta la reutilización**: No puedes reutilizar partes específicas
- **Dificulta el debugging**: Es difícil aislar dónde está el problema
- **Aumenta el acoplamiento**: Suele mezclar múltiples conceptos

#### Técnicas de refactorización:

1. **Extract Method**: Extraer partes en métodos separados
2. **Replace Temp with Query**: Reemplazar variables temporales con métodos
3. **Introduce Parameter Object**: Agrupar parámetros relacionados
4. **Preserve Whole Object**: Pasar el objeto completo en lugar de sus propiedades
5. **Replace Method with Method Object**: Convertir el método en su propia clase

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// PROBLEMA: Método extremadamente largo
// ============================================

class OrderProcessor {
  // ❌ PROBLEMA: Este método tiene 100+ líneas y hace DEMASIADAS cosas
  processOrder(orderData: any): void {
    console.log('Starting order processing...');
    
    // Problema 1: Validación mezclada con lógica de negocio
    if (!orderData) {
      throw new Error('Order data is required');
    }
    
    if (!orderData.customer) {
      throw new Error('Customer information is required');
    }
    
    if (!orderData.customer.email || !orderData.customer.email.includes('@')) {
      throw new Error('Valid customer email is required');
    }
    
    if (!orderData.customer.name || orderData.customer.name.trim().length < 2) {
      throw new Error('Customer name must be at least 2 characters');
    }
    
    if (!orderData.items || !Array.isArray(orderData.items) || orderData.items.length === 0) {
      throw new Error('Order must contain at least one item');
    }
    
    // Problema 2: Lógica de inventario mezclada
    console.log('Checking inventory...');
    const unavailableItems = [];
    for (let i = 0; i < orderData.items.length; i++) {
      const item = orderData.items[i];
      
      // Simulación de verificación de inventario
      const availableQuantity = Math.floor(Math.random() * 100);
      if (availableQuantity < item.quantity) {
        unavailableItems.push({
          name: item.name,
          requested: item.quantity,
          available: availableQuantity
        });
      }
    }
    
    if (unavailableItems.length > 0) {
      console.error('Some items are not available:', unavailableItems);
      // Enviar notificación de items no disponibles
      const emailBody = `
        Dear ${orderData.customer.name},
        
        The following items in your order are not available:
        ${unavailableItems.map(item => 
          `- ${item.name}: Requested ${item.requested}, Available ${item.available}`
        ).join('\n')}
        
        Please update your order.
      `;
      console.log('Sending email:', emailBody);
      throw new Error('Some items are not available');
    }
    
    // Problema 3: Cálculo de precios mezclado
    console.log('Calculating prices...');
    let subtotal = 0;
    let totalWeight = 0;
    
    for (const item of orderData.items) {
      const itemPrice = item.price * item.quantity;
      subtotal += itemPrice;
      
      // Calcular peso para envío
      const itemWeight = item.weight || 0.5; // peso default 0.5 kg
      totalWeight += itemWeight * item.quantity;
    }
    
    // Problema 4: Lógica de descuentos compleja
    console.log('Applying discounts...');
    let discountAmount = 0;
    
    // Descuento por volumen
    if (subtotal > 1000) {
      discountAmount += subtotal * 0.1; // 10% descuento
    } else if (subtotal > 500) {
      discountAmount += subtotal * 0.05; // 5% descuento
    }
    
    // Descuento por cliente frecuente
    if (orderData.customer.loyaltyPoints > 1000) {
      discountAmount += subtotal * 0.05; // 5% adicional
    }
    
    // Descuento por código promocional
    if (orderData.promoCode) {
      if (orderData.promoCode === 'SAVE20') {
        discountAmount += subtotal * 0.2;
      } else if (orderData.promoCode === 'SAVE10') {
        discountAmount += subtotal * 0.1;
      } else if (orderData.promoCode === 'FREESHIP') {
        // Se aplicará más adelante en el cálculo de envío
      }
    }
    
    // Asegurar que el descuento no exceda el 50%
    if (discountAmount > subtotal * 0.5) {
      discountAmount = subtotal * 0.5;
    }
    
    const discountedSubtotal = subtotal - discountAmount;
    
    // Problema 5: Cálculo de envío complejo
    console.log('Calculating shipping...');
    let shippingCost = 0;
    
    if (orderData.shippingMethod === 'express') {
      shippingCost = 25 + (totalWeight * 2);
    } else if (orderData.shippingMethod === 'standard') {
      shippingCost = 10 + (totalWeight * 1);
    } else if (orderData.shippingMethod === 'economy') {
      shippingCost = 5 + (totalWeight * 0.5);
    }
    
    // Envío gratis para órdenes grandes o con código promocional
    if (discountedSubtotal > 100 || orderData.promoCode === 'FREESHIP') {
      shippingCost = 0;
    }
    
    // Problema 6: Cálculo de impuestos
    console.log('Calculating taxes...');
    let taxRate = 0;
    
    // Diferentes tasas de impuesto por estado
    switch (orderData.shippingAddress.state) {
      case 'CA':
        taxRate = 0.0725;
        break;
      case 'NY':
        taxRate = 0.08;
        break;
      case 'TX':
        taxRate = 0.0625;
        break;
      case 'FL':
        taxRate = 0.06;
        break;
      default:
        taxRate = 0.05;
    }
    
    const taxAmount = (discountedSubtotal + shippingCost) * taxRate;
    const total = discountedSubtotal + shippingCost + taxAmount;
    
    // Problema 7: Procesamiento de pago
    console.log('Processing payment...');
    let paymentSuccessful = false;
    
    if (orderData.paymentMethod === 'credit_card') {
      // Validar tarjeta de crédito
      if (!orderData.paymentDetails.cardNumber || 
          orderData.paymentDetails.cardNumber.replace(/\s/g, '').length !== 16) {
        throw new Error('Invalid credit card number');
      }
      
      if (!orderData.paymentDetails.cvv || 
          orderData.paymentDetails.cvv.length !== 3) {
        throw new Error('Invalid CVV');
      }
      
      // Procesar pago
      console.log(`Charging ${total} to credit card...`);
      paymentSuccessful = Math.random() > 0.1; // 90% success rate simulado
      
    } else if (orderData.paymentMethod === 'paypal') {
      console.log(`Processing PayPal payment for ${total}...`);
      paymentSuccessful = Math.random() > 0.05; // 95% success rate simulado
      
    } else if (orderData.paymentMethod === 'bitcoin') {
      console.log(`Processing Bitcoin payment for ${total}...`);
      paymentSuccessful = Math.random() > 0.2; // 80% success rate simulado
    }
    
    if (!paymentSuccessful) {
      throw new Error('Payment failed');
    }
    
    // Problema 8: Creación y guardado de orden
    console.log('Creating order record...');
    const order = {
      id: 'ORD-' + Date.now() + '-' + Math.random().toString(36).substr(2, 9),
      customerId: orderData.customer.id,
      items: orderData.items,
      subtotal: subtotal,
      discount: discountAmount,
      shipping: shippingCost,
      tax: taxAmount,
      total: total,
      status: 'confirmed',
      paymentMethod: orderData.paymentMethod,
      shippingAddress: orderData.shippingAddress,
      createdAt: new Date()
    };
    
    // Guardar en base de datos (simulado)
    console.log('Saving order to database:', order);
    
    // Problema 9: Actualización de inventario
    console.log('Updating inventory...');
    for (const item of orderData.items) {
      console.log(`Reducing inventory for ${item.name} by ${item.quantity}`);
    }
    
    // Problema 10: Envío de notificaciones
    console.log('Sending notifications...');
    
    // Email de confirmación
    const confirmationEmail = `
      Dear ${orderData.customer.name},
      
      Your order ${order.id} has been confirmed!
      
      Order Summary:
      ${orderData.items.map(item => 
        `- ${item.name}: ${item.quantity} x $${item.price}`
      ).join('\n')}
      
      Subtotal: $${subtotal.toFixed(2)}
      Discount: -$${discountAmount.toFixed(2)}
      Shipping: $${shippingCost.toFixed(2)}
      Tax: $${taxAmount.toFixed(2)}
      Total: $${total.toFixed(2)}
      
      Thank you for your purchase!
    `;
    console.log('Sending confirmation email:', confirmationEmail);
    
    // SMS de confirmación
    if (orderData.customer.phone) {
      const smsMessage = `Your order ${order.id} for $${total.toFixed(2)} has been confirmed!`;
      console.log('Sending SMS:', smsMessage);
    }
    
    // Notificación al warehouse
    console.log('Notifying warehouse about new order...');
    
    console.log('Order processing completed successfully!');
  }
}

// ============================================
// SOLUCIÓN: Refactorización usando Extract Method
// ============================================

// Interfaces y tipos bien definidos
interface Customer {
  id: string;
  name: string;
  email: string;
  phone?: string;
  loyaltyPoints: number;
}

interface OrderItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
  weight?: number;
}

interface ShippingAddress {
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

interface PaymentDetails {
  cardNumber?: string;
  cvv?: string;
  paypalEmail?: string;
  bitcoinAddress?: string;
}

interface OrderData {
  customer: Customer;
  items: OrderItem[];
  shippingMethod: 'express' | 'standard' | 'economy';
  shippingAddress: ShippingAddress;
  paymentMethod: 'credit_card' | 'paypal' | 'bitcoin';
  paymentDetails: PaymentDetails;
  promoCode?: string;
}

interface Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  pricing: OrderPricing;
  status: OrderStatus;
  paymentMethod: string;
  shippingAddress: ShippingAddress;
  createdAt: Date;
}

interface OrderPricing {
  subtotal: number;
  discount: number;
  shipping: number;
  tax: number;
  total: number;
}

type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';

// ✅ SOLUCIÓN: Clase refactorizada con métodos pequeños y enfocados
class RefactoredOrderProcessor {
  // Constantes extraídas para fácil configuración
  private readonly VOLUME_DISCOUNT_THRESHOLD_HIGH = 1000;
  private readonly VOLUME_DISCOUNT_THRESHOLD_LOW = 500;
  private readonly VOLUME_DISCOUNT_RATE_HIGH = 0.1;
  private readonly VOLUME_DISCOUNT_RATE_LOW = 0.05;
  private readonly LOYALTY_DISCOUNT_THRESHOLD = 1000;
  private readonly LOYALTY_DISCOUNT_RATE = 0.05;
  private readonly MAX_DISCOUNT_RATE = 0.5;
  private readonly FREE_SHIPPING_THRESHOLD = 100;
  private readonly DEFAULT_ITEM_WEIGHT = 0.5;
  
  // Inyección de dependencias para mejor testing
  constructor(
    private inventoryService: InventoryService,
    private paymentService: PaymentService,
    private notificationService: NotificationService,
    private orderRepository: OrderRepository,
    private logger: Logger
  ) {}
  
  // Método principal ahora es corto y claro
  async processOrder(orderData: OrderData): Promise<Order> {
    this.logger.info('Starting order processing...');
    
    try {
      // Cada paso es un método separado con una responsabilidad clara
      this.validateOrderData(orderData);
      await this.checkInventoryAvailability(orderData.items);
      
      const pricing = this.calculateOrderPricing(orderData);
      await this.processPayment(orderData, pricing.total);
      
      const order = this.createOrder(orderData, pricing);
      await this.saveOrder(order);
      
      await this.updateInventory(orderData.items);
      await this.sendNotifications(order, orderData.customer);
      
      this.logger.info('Order processing completed successfully!');
      return order;
      
    } catch (error) {
      this.logger.error('Order processing failed:', error);
      throw error;
    }
  }
  
  // Validación extraída y organizada
  private validateOrderData(orderData: OrderData): void {
    this.validateCustomer(orderData.customer);
    this.validateItems(orderData.items);
    this.validateShippingAddress(orderData.shippingAddress);
    this.validatePaymentDetails(orderData);
  }
  
  private validateCustomer(customer: Customer): void {
    if (!customer) {
      throw new ValidationError('Customer information is required');
    }
    
    if (!this.isValidEmail(customer.email)) {
      throw new ValidationError('Valid customer email is required');
    }
    
    if (!this.isValidName(customer.name)) {
      throw new ValidationError('Customer name must be at least 2 characters');
    }
  }
  
  private validateItems(items: OrderItem[]): void {
    if (!items || !Array.isArray(items) || items.length === 0) {
      throw new ValidationError('Order must contain at least one item');
    }
    
    items.forEach(item => this.validateItem(item));
  }
  
  private validateItem(item: OrderItem): void {
    if (!item.name || !item.id) {
      throw new ValidationError('Item must have name and ID');
    }
    
    if (item.quantity <= 0) {
      throw new ValidationError('Item quantity must be positive');
    }
    
    if (item.price < 0) {
      throw new ValidationError('Item price cannot be negative');
    }
  }
  
  private validateShippingAddress(address: ShippingAddress): void {
    if (!address) {
      throw new ValidationError('Shipping address is required');
    }
    
    const requiredFields = ['street', 'city', 'state', 'zipCode', 'country'];
    for (const field of requiredFields) {
      if (!address[field]) {
        throw new ValidationError(`Shipping address ${field} is required`);
      }
    }
  }
  
  private validatePaymentDetails(orderData: OrderData): void {
    const validator = this.getPaymentValidator(orderData.paymentMethod);
    validator.validate(orderData.paymentDetails);
  }
  
  // Verificación de inventario extraída
  private async checkInventoryAvailability(items: OrderItem[]): Promise<void> {
    this.logger.info('Checking inventory...');
    
    const availabilityResults = await Promise.all(
      items.map(item => this.inventoryService.checkAvailability(item))
    );
    
    const unavailableItems = availabilityResults.filter(result => !result.isAvailable);
    
    if (unavailableItems.length > 0) {
      await this.handleUnavailableItems(unavailableItems);
      throw new InventoryError('Some items are not available', unavailableItems);
    }
  }
  
  private async handleUnavailableItems(unavailableItems: InventoryCheckResult[]): Promise<void> {
    this.logger.warn('Some items are not available:', unavailableItems);
    // La lógica de notificación está delegada al servicio de notificaciones
    await this.notificationService.notifyUnavailableItems(unavailableItems);
  }
  
  // Cálculo de precios extraído y organizado
  private calculateOrderPricing(orderData: OrderData): OrderPricing {
    this.logger.info('Calculating order pricing...');
    
    const subtotal = this.calculateSubtotal(orderData.items);
    const totalWeight = this.calculateTotalWeight(orderData.items);
    const discount = this.calculateTotalDiscount(subtotal, orderData);
    const discountedSubtotal = subtotal - discount;
    const shipping = this.calculateShipping(
      orderData.shippingMethod,
      totalWeight,
      discountedSubtotal,
      orderData.promoCode
    );
    const tax = this.calculateTax(discountedSubtotal, shipping, orderData.shippingAddress.state);
    const total = discountedSubtotal + shipping + tax;
    
    return {
      subtotal,
      discount,
      shipping,
      tax,
      total
    };
  }
  
  private calculateSubtotal(items: OrderItem[]): number {
    return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }
  
  private calculateTotalWeight(items: OrderItem[]): number {
    return items.reduce((total, item) => {
      const itemWeight = item.weight || this.DEFAULT_ITEM_WEIGHT;
      return total + (itemWeight * item.quantity);
    }, 0);
  }
  
  private calculateTotalDiscount(subtotal: number, orderData: OrderData): number {
    const discounts = [
      this.calculateVolumeDiscount(subtotal),
      this.calculateLoyaltyDiscount(subtotal, orderData.customer.loyaltyPoints),
      this.calculatePromoCodeDiscount(subtotal, orderData.promoCode)
    ];
    
    const totalDiscount = discounts.reduce((sum, discount) => sum + discount, 0);
    
    // Aplicar límite máximo de descuento
    return Math.min(totalDiscount, subtotal * this.MAX_DISCOUNT_RATE);
  }
  
  private calculateVolumeDiscount(subtotal: number): number {
    if (subtotal > this.VOLUME_DISCOUNT_THRESHOLD_HIGH) {
      return subtotal * this.VOLUME_DISCOUNT_RATE_HIGH;
    } else if (subtotal > this.VOLUME_DISCOUNT_THRESHOLD_LOW) {
      return subtotal * this.VOLUME_DISCOUNT_RATE_LOW;
    }
    return 0;
  }
  
  private calculateLoyaltyDiscount(subtotal: number, loyaltyPoints: number): number {
    if (loyaltyPoints > this.LOYALTY_DISCOUNT_THRESHOLD) {
      return subtotal * this.LOYALTY_DISCOUNT_RATE;
    }
    return 0;
  }
  
  private calculatePromoCodeDiscount(subtotal: number, promoCode?: string): number {
    if (!promoCode) return 0;
    
    const promoDiscounts: Record<string, number> = {
      'SAVE20': 0.2,
      'SAVE10': 0.1,
      'SAVE5': 0.05
    };
    
    const discountRate = promoDiscounts[promoCode] || 0;
    return subtotal * discountRate;
  }
  
  private calculateShipping(
    method: string,
    weight: number,
    subtotal: number,
    promoCode?: string
  ): number {
    // Envío gratis para órdenes grandes o código promocional
    if (subtotal > this.FREE_SHIPPING_THRESHOLD || promoCode === 'FREESHIP') {
      return 0;
    }
    
    const shippingRates: Record<string, { base: number; perKg: number }> = {
      'express': { base: 25, perKg: 2 },
      'standard': { base: 10, perKg: 1 },
      'economy': { base: 5, perKg: 0.5 }
    };
    
    const rate = shippingRates[method] || shippingRates['standard'];
    return rate.base + (weight * rate.perKg);
  }
  
  private calculateTax(subtotal: number, shipping: number, state: string): number {
    const taxRates: Record<string, number> = {
      'CA': 0.0725,
      'NY': 0.08,
      'TX': 0.0625,
      'FL': 0.06
    };
    
    const taxRate = taxRates[state] || 0.05;
    return (subtotal + shipping) * taxRate;
  }
  
  // Procesamiento de pago extraído
  private async processPayment(orderData: OrderData, amount: number): Promise<void> {
    this.logger.info(`Processing payment of $${amount.toFixed(2)}...`);
    
    const paymentRequest: PaymentRequest = {
      amount,
      method: orderData.paymentMethod,
      details: orderData.paymentDetails,
      customer: orderData.customer
    };
    
    const result = await this.paymentService.processPayment(paymentRequest);
    
    if (!result.success) {
      throw new PaymentError('Payment failed', result.error);
    }
  }
  
  // Creación de orden simplificada
  private createOrder(orderData: OrderData, pricing: OrderPricing): Order {
    return {
      id: this.generateOrderId(),
      customerId: orderData.customer.id,
      items: orderData.items,
      pricing,
      status: 'confirmed',
      paymentMethod: orderData.paymentMethod,
      shippingAddress: orderData.shippingAddress,
      createdAt: new Date()
    };
  }
  
  private generateOrderId(): string {
    const timestamp = Date.now();
    const random = Math.random().toString(36).substr(2, 9);
    return `ORD-${timestamp}-${random}`.toUpperCase();
  }
  
  // Operaciones de base de datos delegadas
  private async saveOrder(order: Order): Promise<void> {
    this.logger.info('Saving order to database...');
    await this.orderRepository.save(order);
  }
  
  private async updateInventory(items: OrderItem[]): Promise<void> {
    this.logger.info('Updating inventory...');
    await this.inventoryService.reduceInventory(items);
  }
  
  // Notificaciones delegadas
  private async sendNotifications(order: Order, customer: Customer): Promise<void> {
    this.logger.info('Sending notifications...');
    
    await Promise.all([
      this.notificationService.sendOrderConfirmationEmail(order, customer),
      this.notificationService.sendOrderConfirmationSMS(order, customer),
      this.notificationService.notifyWarehouse(order)
    ]);
  }
  
  // Métodos auxiliares de validación
  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
  
  private isValidName(name: string): boolean {
    return name && name.trim().length >= 2;
  }
  
  private getPaymentValidator(method: string): PaymentValidator {
    const validators: Record<string, PaymentValidator> = {
      'credit_card': new CreditCardValidator(),
      'paypal': new PayPalValidator(),
      'bitcoin': new BitcoinValidator()
    };
    
    return validators[method] || new DefaultPaymentValidator();
  }
}

// ============================================
// Servicios extraídos (Single Responsibility)
// ============================================

// Servicio de inventario
class InventoryService {
  async checkAvailability(item: OrderItem): Promise<InventoryCheckResult> {
    // Lógica de verificación de inventario
    const availableQuantity = await this.getAvailableQuantity(item.id);
    
    return {
      itemId: item.id,
      itemName: item.name,
      requested: item.quantity,
      available: availableQuantity,
      isAvailable: availableQuantity >= item.quantity
    };
  }
  
  async reduceInventory(items: OrderItem[]): Promise<void> {
    for (const item of items) {
      await this.updateQuantity(item.id, -item.quantity);
    }
  }
  
  private async getAvailableQuantity(itemId: string): Promise<number> {
    // Simulación - en realidad consultaría la base de datos
    return Math.floor(Math.random() * 100);
  }
  
  private async updateQuantity(itemId: string, delta: number): Promise<void> {
    console.log(`Updating inventory for ${itemId} by ${delta}`);
  }
}

// Servicio de pagos
class PaymentService {
  async processPayment(request: PaymentRequest): Promise<PaymentResult> {
    const processor = this.getPaymentProcessor(request.method);
    return await processor.process(request);
  }
  
  private getPaymentProcessor(method: string): PaymentProcessor {
    const processors: Record<string, PaymentProcessor> = {
      'credit_card': new CreditCardProcessor(),
      'paypal': new PayPalProcessor(),
      'bitcoin': new BitcoinProcessor()
    };
    
    return processors[method] || new DefaultPaymentProcessor();
  }
}

// Servicio de notificaciones
class NotificationService {
  async sendOrderConfirmationEmail(order: Order, customer: Customer): Promise<void> {
    const emailContent = this.generateEmailContent(order, customer);
    await this.sendEmail(customer.email, 'Order Confirmation', emailContent);
  }
  
  async sendOrderConfirmationSMS(order: Order, customer: Customer): Promise<void> {
    if (!customer.phone) return;
    
    const smsContent = `Your order ${order.id} for $${order.pricing.total.toFixed(2)} has been confirmed!`;
    await this.sendSMS(customer.phone, smsContent);
  }
  
  async notifyWarehouse(order: Order): Promise<void> {
    console.log(`Notifying warehouse about order ${order.id}`);
  }
  
  async notifyUnavailableItems(items: InventoryCheckResult[]): Promise<void> {
    // Lógica de notificación para items no disponibles
  }
  
  private generateEmailContent(order: Order, customer: Customer): string {
    return `
      Dear ${customer.name},
      
      Your order ${order.id} has been confirmed!
      
      Order Summary:
      ${order.items.map(item => 
        `- ${item.name}: ${item.quantity} x $${item.price}`
      ).join('\n')}
      
      Subtotal: $${order.pricing.subtotal.toFixed(2)}
      Discount: -$${order.pricing.discount.toFixed(2)}
      Shipping: $${order.pricing.shipping.toFixed(2)}
      Tax: $${order.pricing.tax.toFixed(2)}
      Total: $${order.pricing.total.toFixed(2)}
      
      Thank you for your purchase!
    `;
  }
  
  private async sendEmail(to: string, subject: string, body: string): Promise<void> {
    console.log(`Sending email to ${to}: ${subject}`);
  }
  
  private async sendSMS(to: string, message: string): Promise<void> {
    console.log(`Sending SMS to ${to}: ${message}`);
  }
}

// Interfaces y clases auxiliares
interface InventoryCheckResult {
  itemId: string;
  itemName: string;
  requested: number;
  available: number;
  isAvailable: boolean;
}

interface PaymentRequest {
  amount: number;
  method: string;
  details: PaymentDetails;
  customer: Customer;
}

interface PaymentResult {
  success: boolean;
  transactionId?: string;
  error?: string;
}

interface PaymentValidator {
  validate(details: PaymentDetails): void;
}

interface PaymentProcessor {
  process(request: PaymentRequest): Promise<PaymentResult>;
}

interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

interface Logger {
  info(message: string, data?: any): void;
  warn(message: string, data?: any): void;
  error(message: string, error?: any): void;
}

// Clases de error personalizadas
class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

class InventoryError extends Error {
  constructor(message: string, public unavailableItems: InventoryCheckResult[]) {
    super(message);
    this.name = 'InventoryError';
  }
}

class PaymentError extends Error {
  constructor(message: string, public details?: string) {
    super(message);
    this.name = 'PaymentError';
  }
}

// Implementaciones de validadores
class CreditCardValidator implements PaymentValidator {
  validate(details: PaymentDetails): void {
    if (!details.cardNumber || details.cardNumber.replace(/\s/g, '').length !== 16) {
      throw new ValidationError('Invalid credit card number');
    }
    
    if (!details.cvv || details.cvv.length !== 3) {
      throw new ValidationError('Invalid CVV');
    }
  }
}

class PayPalValidator implements PaymentValidator {
  validate(details: PaymentDetails): void {
    if (!details.paypalEmail || !details.paypalEmail.includes('@')) {
      throw new ValidationError('Invalid PayPal email');
    }
  }
}

class BitcoinValidator implements PaymentValidator {
  validate(details: PaymentDetails): void {
    if (!details.bitcoinAddress || details.bitcoinAddress.length < 26) {
      throw new ValidationError('Invalid Bitcoin address');
    }
  }
}

class DefaultPaymentValidator implements PaymentValidator {
  validate(details: PaymentDetails): void {
    // Validación por defecto
  }
}

// Implementaciones de procesadores de pago
class CreditCardProcessor implements PaymentProcessor {
  async process(request: PaymentRequest): Promise<PaymentResult> {
    console.log(`Processing credit card payment for $${request.amount}`);
    // Simulación de procesamiento
    const success = Math.random() > 0.1;
    
    return {
      success,
      transactionId: success ? `CC-${Date.now()}` : undefined,
      error: success ? undefined : 'Card declined'
    };
  }
}

class PayPalProcessor implements PaymentProcessor {
  async process(request: PaymentRequest): Promise<PaymentResult> {
    console.log(`Processing PayPal payment for $${request.amount}`);
    const success = Math.random() > 0.05;
    
    return {
      success,
      transactionId: success ? `PP-${Date.now()}` : undefined,
      error: success ? undefined : 'PayPal authorization failed'
    };
  }
}

class BitcoinProcessor implements PaymentProcessor {
  async process(request: PaymentRequest): Promise<PaymentResult> {
    console.log(`Processing Bitcoin payment for $${request.amount}`);
    const success = Math.random() > 0.2;
    
    return {
      success,
      transactionId: success ? `BTC-${Date.now()}` : undefined,
      error: success ? undefined : 'Bitcoin transaction failed'
    };
  }
}

class DefaultPaymentProcessor implements PaymentProcessor {
  async process(request: PaymentRequest): Promise<PaymentResult> {
    return {
      success: false,
      error: 'Payment method not supported'
    };
  }
}
```

### Beneficios de la refactorización:

1. **Métodos pequeños y enfocados**: Cada método tiene una responsabilidad clara
2. **Fácil de testear**: Puedes testear cada método independientemente
3. **Fácil de entender**: El flujo principal es claro y legible
4. **Fácil de mantener**: Los cambios están localizados
5. **Reutilizable**: Los métodos extraídos pueden reutilizarse
6. **Mejor manejo de errores**: Errores específicos para cada caso
7. **Principio DRY**: No hay duplicación de código

---

## 3.2 Duplicate Code (Código Duplicado)

### Explicación Detallada

El código duplicado es uno de los code smells más comunes y dañinos. Ocurre cuando la misma lógica aparece en múltiples lugares. Esto viola el principio DRY (Don't Repeat Yourself) y hace que el mantenimiento sea mucho más difícil.

#### Tipos de duplicación:

1. **Duplicación exacta**: El mismo código copiado y pegado
2. **Duplicación con variaciones menores**: Código casi idéntico con pequeñas diferencias
3. **Duplicación estructural**: Misma estructura pero diferentes detalles
4. **Duplicación semántica**: Diferente código que hace lo mismo
5. **Duplicación de datos**: Misma información en múltiples lugares

#### Problemas que causa:

- **Mantenimiento difícil**: Cambios deben hacerse en múltiples lugares
- **Inconsistencias**: Es fácil olvidar actualizar todas las copias
- **Bugs multiplicados**: Un bug en el código duplicado existe en múltiples lugares
- **Código más largo**: Más líneas de código sin agregar valor
- **Confusión**: No está claro cuál versión es la "correcta"

#### Técnicas de refactorización:

1. **Extract Method**: Extraer código común en un método
2. **Pull Up Method**: Mover método duplicado a clase padre
3. **Form Template Method**: Crear método template con pasos comunes
4. **Replace Algorithm**: Reemplazar implementaciones diferentes con una mejor
5. **Extract Class**: Crear una clase para encapsular la lógica común

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// PROBLEMA: Código duplicado en múltiples formas
// ============================================

// ❌ PROBLEMA 1: Duplicación exacta en múltiples clases
class EmailValidator {
  validate(email: string): boolean {
    // Código duplicado #1
    if (!email || email.trim().length === 0) {
      return false;
    }
    
    const emailParts = email.split('@');
    if (emailParts.length !== 2) {
      return false;
    }
    
    const [localPart, domain] = emailParts;
    
    if (localPart.length === 0 || localPart.length > 64) {
      return false;
    }
    
    const domainParts = domain.split('.');
    if (domainParts.length < 2) {
      return false;
    }
    
    for (const part of domainParts) {
      if (part.length === 0 || part.length > 63) {
        return false;
      }
    }
    
    const validChars = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+$/;
    if (!validChars.test(localPart)) {
      return false;
    }
    
    return true;
  }
}

class UserRegistrationForm {
  validateEmail(email: string): boolean {
    // Código duplicado #2 - Exactamente el mismo que arriba!
    if (!email || email.trim().length === 0) {
      return false;
    }
    
    const emailParts = email.split('@');
    if (emailParts.length !== 2) {
      return false;
    }
    
    const [localPart, domain] = emailParts;
    
    if (localPart.length === 0 || localPart.length > 64) {
      return false;
    }
    
    const domainParts = domain.split('.');
    if (domainParts.length < 2) {
      return false;
    }
    
    for (const part of domainParts) {
      if (part.length === 0 || part.length > 63) {
        return false;
      }
    }
    
    const validChars = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+$/;
    if (!validChars.test(localPart)) {
      return false;
    }
    
    return true;
  }
  
  // Más métodos...
}

class NewsletterSubscription {
  isValidEmail(email: string): boolean {
    // Código duplicado #3 - Y otra vez!
    if (!email || email.trim().length === 0) {
      return false;
    }
    
    const emailParts = email.split('@');
    if (emailParts.length !== 2) {
      return false;
    }
    
    const [localPart, domain] = emailParts;
    
    if (localPart.length === 0 || localPart.length > 64) {
      return false;
    }
    
    const domainParts = domain.split('.');
    if (domainParts.length < 2) {
      return false;
    }
    
    for (const part of domainParts) {
      if (part.length === 0 || part.length > 63) {
        return false;
      }
    }
    
    const validChars = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+$/;
    if (!validChars.test(localPart)) {
      return false;
    }
    
    return true;
  }
}

// ❌ PROBLEMA 2: Duplicación con variaciones menores
class ProductService {
  async getProduct(id: string): Promise<Product | null> {
    // Patrón duplicado #1
    console.log(`Fetching product ${id}`);
    
    try {
      const cacheKey = `product:${id}`;
      
      // Verificar caché
      const cached = await cache.get(cacheKey);
      if (cached) {
        console.log(`Cache hit for product ${id}`);
        return JSON.parse(cached);
      }
      
      // Buscar en base de datos
      const product = await database.query(
        'SELECT * FROM products WHERE id = ?',
        [id]
      );
      
      if (!product) {
        console.log(`Product ${id} not found`);
        return null;
      }
      
      // Guardar en caché
      await cache.set(cacheKey, JSON.stringify(product), 3600);
      
      console.log(`Product ${id} fetched from database`);
      return product;
      
    } catch (error) {
      console.error(`Error fetching product ${id}:`, error);
      throw new Error(`Failed to fetch product: ${error.message}`);
    }
  }
}

class UserService {
  async getUser(id: string): Promise<User | null> {
    // Patrón duplicado #2 - Casi idéntico, solo cambian los nombres
    console.log(`Fetching user ${id}`);
    
    try {
      const cacheKey = `user:${id}`;
      
      // Verificar caché (duplicado)
      const cached = await cache.get(cacheKey);
      if (cached) {
        console.log(`Cache hit for user ${id}`);
        return JSON.parse(cached);
      }
      
      // Buscar en base de datos (duplicado con variación)
      const user = await database.query(
        'SELECT * FROM users WHERE id = ?',
        [id]
      );
      
      if (!user) {
        console.log(`User ${id} not found`);
        return null;
      }
      
      // Guardar en caché (duplicado)
      await cache.set(cacheKey, JSON.stringify(user), 3600);
      
      console.log(`User ${id} fetched from database`);
      return user;
      
    } catch (error) {
      console.error(`Error fetching user ${id}:`, error);
      throw new Error(`Failed to fetch user: ${error.message}`);
    }
  }
}

class OrderService {
  async getOrder(id: string): Promise<Order | null> {
    // Patrón duplicado #3 - Y otra vez el mismo patrón!
    console.log(`Fetching order ${id}`);
    
    try {
      const cacheKey = `order:${id}`;
      
      // Verificar caché (duplicado otra vez)
      const cached = await cache.get(cacheKey);
      if (cached) {
        console.log(`Cache hit for order ${id}`);
        return JSON.parse(cached);
      }
      
      // Buscar en base de datos (duplicado con otra variación)
      const order = await database.query(
        'SELECT * FROM orders WHERE id = ?',
        [id]
      );
      
      if (!order) {
        console.log(`Order ${id} not found`);
        return null;
      }
      
      // Guardar en caché (duplicado)
      await cache.set(cacheKey, JSON.stringify(order), 3600);
      
      console.log(`Order ${id} fetched from database`);
      return order;
      
    } catch (error) {
      console.error(`Error fetching order ${id}:`, error);
      throw new Error(`Failed to fetch order: ${error.message}`);
    }
  }
}

// ❌ PROBLEMA 3: Duplicación estructural
class CreditCardPayment {
  process(amount: number): PaymentResult {
    // Estructura duplicada #1
    console.log('Processing credit card payment...');
    
    // Validación
    if (amount <= 0) {
      return {
        success: false,
        error: 'Amount must be positive'
      };
    }
    
    if (amount > 10000) {
      return {
        success: false,
        error: 'Amount exceeds maximum limit'
      };
    }
    
    // Calcular fee
    const fee = amount * 0.029 + 0.30; // 2.9% + $0.30
    const total = amount + fee;
    
    // Procesar
    try {
      const transactionId = this.processWithProvider(total);
      
      // Log transacción
      console.log(`Transaction ${transactionId} completed`);
      this.saveTransaction({
        id: transactionId,
        amount: amount,
        fee: fee,
        total: total,
        type: 'credit_card',
        timestamp: new Date()
      });
      
      return {
        success: true,
        transactionId: transactionId,
        amount: total
      };
      
    } catch (error) {
      console.error('Payment failed:', error);
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  private processWithProvider(amount: number): string {
    // Simulación
    return `CC-${Date.now()}`;
  }
  
  private saveTransaction(data: any): void {
    // Guardar en DB
  }
}

class PayPalPayment {
  process(amount: number): PaymentResult {
    // Estructura duplicada #2 - Misma estructura, diferentes detalles
    console.log('Processing PayPal payment...');
    
    // Validación (duplicada)
    if (amount <= 0) {
      return {
        success: false,
        error: 'Amount must be positive'
      };
    }
    
    if (amount > 10000) {
      return {
        success: false,
        error: 'Amount exceeds maximum limit'
      };
    }
    
    // Calcular fee (diferente cálculo pero misma estructura)
    const fee = amount * 0.034 + 0.49; // 3.4% + $0.49
    const total = amount + fee;
    
    // Procesar (estructura duplicada)
    try {
      const transactionId = this.processWithPayPal(total);
      
      // Log transacción (duplicado)
      console.log(`Transaction ${transactionId} completed`);
      this.saveTransaction({
        id: transactionId,
        amount: amount,
        fee: fee,
        total: total,
        type: 'paypal',
        timestamp: new Date()
      });
      
      return {
        success: true,
        transactionId: transactionId,
        amount: total
      };
      
    } catch (error) {
      console.error('Payment failed:', error);
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  private processWithPayPal(amount: number): string {
    // Simulación
    return `PP-${Date.now()}`;
  }
  
  private saveTransaction(data: any): void {
    // Guardar en DB
  }
}

// ============================================
// SOLUCIÓN: Eliminar duplicación con diferentes técnicas
// ============================================

// ✅ SOLUCIÓN 1: Extract Method/Class para validación de email
class EmailValidatorService {
  private readonly MAX_LOCAL_LENGTH = 64;
  private readonly MAX_DOMAIN_PART_LENGTH = 63;
  private readonly VALID_LOCAL_CHARS = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+$/;
  
  validate(email: string): ValidationResult {
    // Una única implementación reutilizable
    if (!email || email.trim().length === 0) {
      return {
        isValid: false,
        error: 'Email is required'
      };
    }
    
    const trimmedEmail = email.trim().toLowerCase();
    const parts = this.splitEmail(trimmedEmail);
    
    if (!parts) {
      return {
        isValid: false,
        error: 'Invalid email format'
      };
    }
    
    const localValidation = this.validateLocalPart(parts.local);
    if (!localValidation.isValid) {
      return localValidation;
    }
    
    const domainValidation = this.validateDomain(parts.domain);
    if (!domainValidation.isValid) {
      return domainValidation;
    }
    
    return { isValid: true };
  }
  
  private splitEmail(email: string): { local: string; domain: string } | null {
    const parts = email.split('@');
    
    if (parts.length !== 2) {
      return null;
    }
    
    return {
      local: parts[0],
      domain: parts[1]
    };
  }
  
  private validateLocalPart(localPart: string): ValidationResult {
    if (localPart.length === 0) {
      return {
        isValid: false,
        error: 'Local part cannot be empty'
      };
    }
    
    if (localPart.length > this.MAX_LOCAL_LENGTH) {
      return {
        isValid: false,
        error: `Local part exceeds ${this.MAX_LOCAL_LENGTH} characters`
      };
    }
    
    if (!this.VALID_LOCAL_CHARS.test(localPart)) {
      return {
        isValid: false,
        error: 'Local part contains invalid characters'
      };
    }
    
    return { isValid: true };
  }
  
  private validateDomain(domain: string): ValidationResult {
    const parts = domain.split('.');
    
    if (parts.length < 2) {
      return {
        isValid: false,
        error: 'Domain must have at least one dot'
      };
    }
    
    for (const part of parts) {
      if (part.length === 0) {
        return {
          isValid: false,
          error: 'Domain parts cannot be empty'
        };
      }
      
      if (part.length > this.MAX_DOMAIN_PART_LENGTH) {
        return {
          isValid: false,
          error: `Domain part exceeds ${this.MAX_DOMAIN_PART_LENGTH} characters`
        };
      }
      
      if (!/^[a-zA-Z0-9-]+$/.test(part)) {
        return {
          isValid: false,
          error: 'Domain contains invalid characters'
        };
      }
    }
    
    return { isValid: true };
  }
}

// Ahora todas las clases usan el servicio común
class RefactoredUserRegistrationForm {
  constructor(private emailValidator: EmailValidatorService) {}
  
  validateEmail(email: string): boolean {
    const result = this.emailValidator.validate(email);
    return result.isValid;
  }
}

class RefactoredNewsletterSubscription {
  constructor(private emailValidator: EmailValidatorService) {}
  
  subscribe(email: string): void {
    const validation = this.emailValidator.validate(email);
    
    if (!validation.isValid) {
      throw new Error(validation.error!);
    }
    
    // Proceder con la suscripción
  }
}

// ✅ SOLUCIÓN 2: Generic Repository Pattern para eliminar duplicación de caché
abstract class CachedRepository<T> {
  constructor(
    protected database: Database,
    protected cache: Cache,
    protected entityName: string,
    protected tableName: string,
    protected cacheTTL: number = 3600
  ) {}
  
  // Método genérico que elimina toda la duplicación
  async getById(id: string): Promise<T | null> {
    this.log(`Fetching ${this.entityName} ${id}`);
    
    try {
      // Verificar caché
      const cached = await this.getFromCache(id);
      if (cached) {
        this.log(`Cache hit for ${this.entityName} ${id}`);
        return cached;
      }
      
      // Buscar en base de datos
      const entity = await this.getFromDatabase(id);
      
      if (!entity) {
        this.log(`${this.entityName} ${id} not found`);
        return null;
      }
      
      // Guardar en caché
      await this.saveToCache(id, entity);
      
      this.log(`${this.entityName} ${id} fetched from database`);
      return entity;
      
    } catch (error) {
      this.logError(`Error fetching ${this.entityName} ${id}:`, error);
      throw new Error(`Failed to fetch ${this.entityName}: ${error.message}`);
    }
  }
  
  protected getCacheKey(id: string): string {
    return `${this.entityName.toLowerCase()}:${id}`;
  }
  
  protected async getFromCache(id: string): Promise<T | null> {
    const cacheKey = this.getCacheKey(id);
    const cached = await this.cache.get(cacheKey);
    
    if (cached) {
      return JSON.parse(cached);
    }
    
    return null;
  }
  
  protected async saveToCache(id: string, entity: T): Promise<void> {
    const cacheKey = this.getCacheKey(id);
    await this.cache.set(cacheKey, JSON.stringify(entity), this.cacheTTL);
  }
  
  protected async getFromDatabase(id: string): Promise<T | null> {
    const query = `SELECT * FROM ${this.tableName} WHERE id = ?`;
    const result = await this.database.query(query, [id]);
    return result ? this.mapToEntity(result) : null;
  }
  
  // Método abstracto que las subclases deben implementar
  protected abstract mapToEntity(data: any): T;
  
  protected log(message: string): void {
    console.log(message);
  }
  
  protected logError(message: string, error: any): void {
    console.error(message, error);
  }
}

// Implementaciones específicas sin duplicación
class ProductRepository extends CachedRepository<Product> {
  constructor(database: Database, cache: Cache) {
    super(database, cache, 'Product', 'products', 3600);
  }
  
  protected mapToEntity(data: any): Product {
    return {
      id: data.id,
      name: data.name,
      price: data.price,
      description: data.description,
      stock: data.stock
    };
  }
  
  // Métodos específicos de productos
  async getByCategory(category: string): Promise<Product[]> {
    // Implementación específica
    return [];
  }
}

class UserRepository extends CachedRepository<User> {
  constructor(database: Database, cache: Cache) {
    super(database, cache, 'User', 'users', 7200); // TTL diferente
  }
  
  protected mapToEntity(data: any): User {
    return {
      id: data.id,
      name: data.name,
      email: data.email,
      role: data.role,
      createdAt: new Date(data.created_at)
    };
  }
  
  // Métodos específicos de usuarios
  async getByEmail(email: string): Promise<User | null> {
    // Implementación específica
    return null;
  }
}

class OrderRepository extends CachedRepository<Order> {
  constructor(database: Database, cache: Cache) {
    super(database, cache, 'Order', 'orders', 1800);
  }
  
  protected mapToEntity(data: any): Order {
    return {
      id: data.id,
      customerId: data.customer_id,
      items: JSON.parse(data.items),
      total: data.total,
      status: data.status,
      createdAt: new Date(data.created_at)
    };
  }
  
  // Métodos específicos de órdenes
  async getByCustomer(customerId: string): Promise<Order[]> {
    // Implementación específica
    return [];
  }
}

// ✅ SOLUCIÓN 3: Template Method Pattern para pagos
abstract class PaymentProcessor {
  // Template method que define la estructura común
  process(amount: number): PaymentResult {
    this.logStart();
    
    // Validación común
    const validationResult = this.validateAmount(amount);
    if (!validationResult.isValid) {
      return {
        success: false,
        error: validationResult.error
      };
    }
    
    // Calcular fee (cada subclase tiene su propia implementación)
    const fee = this.calculateFee(amount);
    const total = amount + fee;
    
    // Procesar pago
    try {
      const transactionId = this.processWithProvider(total);
      
      // Guardar transacción
      const transaction = this.createTransaction(transactionId, amount, fee, total);
      this.saveTransaction(transaction);
      
      this.logSuccess(transactionId);
      
      return {
        success: true,
        transactionId: transactionId,
        amount: total,
        fee: fee
      };
      
    } catch (error) {
      this.logError(error);
      return {
        success: false,
        error: this.getErrorMessage(error)
      };
    }
  }
  
  // Métodos comunes con implementación por defecto
  protected validateAmount(amount: number): ValidationResult {
    if (amount <= 0) {
      return {
        isValid: false,
        error: 'Amount must be positive'
      };
    }
    
    const maxAmount = this.getMaxAmount();
    if (amount > maxAmount) {
      return {
        isValid: false,
        error: `Amount exceeds maximum limit of $${maxAmount}`
      };
    }
    
    return { isValid: true };
  }
  
  protected createTransaction(
    id: string,
    amount: number,
    fee: number,
    total: number
  ): Transaction {
    return {
      id: id,
      amount: amount,
      fee: fee,
      total: total,
      type: this.getPaymentType(),
      timestamp: new Date(),
      status: 'completed'
    };
  }
  
  protected saveTransaction(transaction: Transaction): void {
    // Guardar en base de datos
    console.log('Saving transaction:', transaction);
  }
  
  protected logStart(): void {
    console.log(`Processing ${this.getPaymentType()} payment...`);
  }
  
  protected logSuccess(transactionId: string): void {
    console.log(`Transaction ${transactionId} completed`);
  }
  
  protected logError(error: any): void {
    console.error('Payment failed:', error);
  }
  
  protected getErrorMessage(error: any): string {
    return error.message || 'Payment processing failed';
  }
  
  protected getMaxAmount(): number {
    return 10000; // Default, puede ser sobrescrito
  }
  
  // Métodos abstractos que cada subclase debe implementar
  protected abstract calculateFee(amount: number): number;
  protected abstract processWithProvider(amount: number): string;
  protected abstract getPaymentType(): string;
}

// Implementaciones específicas sin duplicación
class CreditCardProcessor extends PaymentProcessor {
  protected calculateFee(amount: number): number {
    return amount * 0.029 + 0.30; // 2.9% + $0.30
  }
  
  protected processWithProvider(amount: number): string {
    // Lógica específica de tarjeta de crédito
    return `CC-${Date.now()}`;
  }
  
  protected getPaymentType(): string {
    return 'credit_card';
  }
}

class PayPalProcessor extends PaymentProcessor {
  protected calculateFee(amount: number): number {
    return amount * 0.034 + 0.49; // 3.4% + $0.49
  }
  
  protected processWithProvider(amount: number): string {
    // Lógica específica de PayPal
    return `PP-${Date.now()}`;
  }
  
  protected getPaymentType(): string {
    return 'paypal';
  }
  
  protected getMaxAmount(): number {
    return 25000; // PayPal permite montos más altos
  }
}

class StripeProcessor extends PaymentProcessor {
  protected calculateFee(amount: number): number {
    return amount * 0.029 + 0.30; // 2.9% + $0.30
  }
  
  protected processWithProvider(amount: number): string {
    // Lógica específica de Stripe
    return `STR-${Date.now()}`;
  }
  
  protected getPaymentType(): string {
    return 'stripe';
  }
}

// ✅ SOLUCIÓN 4: Composición para eliminar duplicación de comportamiento
class ValidationRule<T> {
  constructor(
    private predicate: (value: T) => boolean,
    private errorMessage: string
  ) {}
  
  validate(value: T): ValidationResult {
    if (!this.predicate(value)) {
      return {
        isValid: false,
        error: this.errorMessage
      };
    }
    
    return { isValid: true };
  }
}

class Validator<T> {
  private rules: ValidationRule<T>[] = [];
  
  addRule(predicate: (value: T) => boolean, errorMessage: string): this {
    this.rules.push(new ValidationRule(predicate, errorMessage));
    return this;
  }
  
  validate(value: T): ValidationResult {
    for (const rule of this.rules) {
      const result = rule.validate(value);
      if (!result.isValid) {
        return result;
      }
    }
    
    return { isValid: true };
  }
}

// Uso de validadores componibles
const amountValidator = new Validator<number>()
  .addRule(amount => amount > 0, 'Amount must be positive')
  .addRule(amount => amount <= 10000, 'Amount exceeds maximum limit')
  .addRule(amount => Number.isFinite(amount), 'Amount must be a valid number');

const emailValidator2 = new Validator<string>()
  .addRule(email => !!email && email.length > 0, 'Email is required')
  .addRule(email => email.includes('@'), 'Email must contain @')
  .addRule(email => email.split('@').length === 2, 'Email must have one @')
  .addRule(email => email.length <= 254, 'Email is too long');

// Interfaces auxiliares
interface ValidationResult {
  isValid: boolean;
  error?: string;
}

interface PaymentResult {
  success: boolean;
  transactionId?: string;
  amount?: number;
  fee?: number;
  error?: string;
}

interface Transaction {
  id: string;
  amount: number;
  fee: number;
  total: number;
  type: string;
  timestamp: Date;
  status: string;
}

interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  stock: number;
}

interface User {
  id: string;
  name: string;
  email: string;
  role: string;
  createdAt: Date;
}

interface Order {
  id: string;
  customerId: string;
  items: any[];
  total: number;
  status: string;
  createdAt: Date;
}

interface Database {
  query(sql: string, params?: any[]): Promise<any>;
}

interface Cache {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, ttl?: number): Promise<void>;
}

// Declaraciones para el ejemplo
declare const cache: Cache;
declare const database: Database;
```

### Principios para evitar duplicación:

1. **DRY (Don't Repeat Yourself)**: Cada pieza de conocimiento debe tener una representación única
2. **Single Source of Truth**: Una sola fuente de verdad para cada concepto
3. **Abstracción**: Identificar patrones comunes y abstraerlos
4. **Composición**: Construir comportamiento complejo desde piezas simples
5. **Reutilización**: Diseñar componentes para ser reutilizables

La eliminación de código duplicado es una de las refactorizaciones más importantes porque mejora dramáticamente la mantenibilidad del código.