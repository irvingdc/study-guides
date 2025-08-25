# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 9: Mejores Prácticas de Mantenimiento
### Para Entrevista Senior Software Engineer

---

## 9. Mejores Prácticas de Mantenimiento - Construyendo Software Sostenible

El mantenimiento consume el 60-80% del costo total del software durante su ciclo de vida. Un código bien mantenido no es solo código que funciona hoy, sino código que puede evolucionar mañana. Como Senior Software Engineer, tu responsabilidad es crear sistemas que otros desarrolladores (incluyéndote a ti mismo en 6 meses) puedan entender, modificar y extender sin miedo.

### Por qué el mantenimiento es crítico

**El problema del código legacy:**
- El código se vuelve legacy desde el momento en que se escribe
- Sin mantenimiento activo, la deuda técnica crece exponencialmente
- Los costos de cambio aumentan con el tiempo
- El conocimiento del sistema se pierde cuando los desarrolladores se van

**El valor del buen mantenimiento:**
- Reduce el tiempo de desarrollo de nuevas features
- Minimiza bugs en producción
- Facilita el onboarding de nuevos desarrolladores
- Permite escalar el equipo y el producto
- Reduce el burnout del equipo

### Principios fundamentales del código mantenible

1. **Claridad sobre cleverness**: El código inteligente que nadie entiende es una liability
2. **Consistencia sobre perfección**: Mejor consistentemente bueno que inconsistentemente perfecto
3. **Documentación como ciudadano de primera clase**: La documentación es parte del código
4. **Testing como red de seguridad**: Los tests permiten cambios seguros
5. **Refactoring continuo**: Pequeñas mejoras constantes vs. grandes reescrituras

---

## 9.1 Documentación Efectiva

### Explicación Conceptual Profunda

La documentación efectiva no es sobre escribir más, sino sobre comunicar mejor. Es el puente entre la intención del autor y la comprensión del lector. La documentación debe responder no solo "qué" hace el código, sino "por qué" existe y "cómo" encaja en el sistema mayor.

#### Niveles de documentación:

1. **Documentación arquitectónica**: Decisiones de alto nivel, trade-offs, patterns
2. **Documentación de API**: Contratos, interfaces públicas, ejemplos de uso
3. **Documentación de código**: Comments inline, JSDoc, tipos
4. **Documentación operacional**: Deploy, monitoring, troubleshooting
5. **Documentación de procesos**: Workflows, contribución guidelines

#### Principios de buena documentación:

- **Proximidad**: La documentación debe estar cerca del código que documenta
- **Actualidad**: Documentación desactualizada es peor que no documentación
- **Ejemplos**: Un ejemplo vale más que mil palabras
- **Contexto**: Explica el "por qué", no solo el "qué"
- **Audiencia**: Escribe para tu lector, no para ti

### Implementación Práctica con Explicación Detallada

```typescript
// ============================================
// EJEMPLO 1: Documentación de Sistema Completa
// ============================================

/**
 * Sistema de Procesamiento de Pagos
 * 
 * ARQUITECTURA:
 * Este sistema implementa el patrón Saga para manejar transacciones
 * distribuidas across múltiples servicios. Cada paso de la transacción
 * es compensable para garantizar consistencia eventual.
 * 
 * FLUJO:
 * 1. Validación de pago -> PaymentValidator
 * 2. Autorización -> PaymentGateway
 * 3. Captura -> PaymentProcessor
 * 4. Notificación -> NotificationService
 * 
 * CONSIDERACIONES DE DISEÑO:
 * - Idempotencia: Todas las operaciones son idempotentes usando transaction IDs
 * - Retry policy: Exponential backoff con jitter, max 3 intentos
 * - Timeouts: 30s para gateway, 10s para servicios internos
 * - Rate limiting: 100 req/s por merchant
 * 
 * SEGURIDAD:
 * - PCI DSS compliance: No almacenamos datos de tarjetas
 * - Tokenización: Usamos tokens del payment gateway
 * - Audit trail: Todas las operaciones son logged
 * 
 * MONITOREO:
 * - Métricas: payment_success_rate, payment_latency, gateway_errors
 * - Alertas: Failure rate > 1%, Latency P99 > 2s
 * - Dashboards: Grafana dashboard ID: payment-system-001
 * 
 * DEPENDENCIAS:
 * - Payment Gateway: Stripe API v2023-10-16
 * - Database: PostgreSQL 14+ con row-level security
 * - Cache: Redis 7+ para idempotency keys
 * - Queue: RabbitMQ para procesamiento asíncrono
 * 
 * MANTENIMIENTO:
 * - Owner: Payments Team (@payments-team)
 * - On-call: PagerDuty rotation "payments-oncall"
 * - Runbook: https://wiki.company.com/payments/runbook
 * 
 * @module PaymentSystem
 * @since 2023-01-15
 * @version 2.4.0
 */

// Interfaz principal con documentación detallada
interface PaymentRequest {
  /**
   * ID único de la transacción. Usado para idempotencia.
   * Formato: UUID v4
   * @example "550e8400-e29b-41d4-a716-446655440000"
   */
  transactionId: string;

  /**
   * Monto en centavos para evitar problemas de punto flotante
   * @min 50 - Mínimo $0.50
   * @max 99999999 - Máximo $999,999.99
   */
  amountCents: number;

  /**
   * Código de moneda ISO 4217
   * @pattern ^[A-Z]{3}$
   * @example "USD", "EUR", "GBP"
   */
  currency: string;

  /**
   * Token de pago obtenido del frontend via Stripe.js
   * Este token expira después de 1 uso o 5 minutos
   */
  paymentToken: string;

  /**
   * Metadata del merchant para reconciliación
   */
  merchantData: {
    /** ID interno del merchant en nuestro sistema */
    merchantId: string;
    
    /** Referencia de orden del merchant para tracking */
    orderReference: string;
    
    /** Categoría MCC para risk assessment */
    mccCode: string;
  };

  /**
   * Información del cliente para fraud detection
   */
  customerInfo: {
    /** IP del cliente para geolocalización */
    ipAddress: string;
    
    /** User agent para device fingerprinting */
    userAgent: string;
    
    /** Email hasheado para velocity checks */
    emailHash: string;
  };
}

/**
 * Procesador principal de pagos con manejo robusto de errores
 * 
 * Esta clase coordina todo el flujo de pago implementando:
 * - Circuit breaker para el payment gateway
 * - Retry logic con exponential backoff
 * - Distributed tracing para debugging
 * - Métricas detalladas para monitoring
 */
class PaymentProcessor {
  private readonly logger = new Logger('PaymentProcessor');
  private readonly metrics = new MetricsCollector('payments');
  private readonly circuitBreaker: CircuitBreaker;
  
  constructor(
    private readonly gateway: PaymentGateway,
    private readonly validator: PaymentValidator,
    private readonly repository: PaymentRepository,
    private readonly notifier: NotificationService
  ) {
    // Circuit breaker configuration
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,        // Abre después de 5 fallos
      resetTimeout: 60000,         // Intenta reset después de 1 minuto
      monitoringPeriod: 120000,    // Ventana de monitoring de 2 minutos
      requestTimeout: 30000        // Timeout de 30 segundos
    });
  }

  /**
   * Procesa un pago completo con todas las validaciones y compensaciones
   * 
   * FLUJO DETALLADO:
   * 1. Validación de business rules
   * 2. Check de idempotencia
   * 3. Risk assessment
   * 4. Autorización con gateway
   * 5. Persistencia de transacción
   * 6. Captura de fondos
   * 7. Notificación de resultado
   * 
   * MANEJO DE ERRORES:
   * - ValidationError: Retorna inmediatamente, no retry
   * - GatewayTimeout: Retry con backoff, max 3 intentos
   * - InsufficientFunds: No retry, notifica al cliente
   * - SystemError: Retry, alerta a on-call si persiste
   * 
   * COMPENSACIÓN:
   * Si falla después de autorización, ejecuta reversal automático
   * 
   * @param request - Datos del pago a procesar
   * @returns Resultado del procesamiento con transaction ID
   * @throws {ValidationError} Si los datos son inválidos
   * @throws {PaymentGatewayError} Si el gateway rechaza el pago
   * @throws {SystemError} Si hay un error interno del sistema
   * 
   * @example
   * ```typescript
   * try {
   *   const result = await processor.processPayment({
   *     transactionId: generateUUID(),
   *     amountCents: 10000, // $100.00
   *     currency: 'USD',
   *     paymentToken: 'tok_visa',
   *     merchantData: { ... },
   *     customerInfo: { ... }
   *   });
   *   console.log('Payment successful:', result.chargeId);
   * } catch (error) {
   *   if (error instanceof ValidationError) {
   *     // Handle validation error - no retry
   *   } else if (error instanceof PaymentGatewayError) {
   *     // Handle gateway error - maybe retry
   *   }
   * }
   * ```
   */
  async processPayment(request: PaymentRequest): Promise<PaymentResult> {
    const startTime = Date.now();
    const traceId = this.generateTraceId();
    
    this.logger.info('Starting payment processing', {
      traceId,
      transactionId: request.transactionId,
      amount: request.amountCents,
      currency: request.currency
    });

    try {
      // Step 1: Validación comprehensiva
      await this.validateRequest(request);
      
      // Step 2: Idempotency check
      const existing = await this.checkIdempotency(request.transactionId);
      if (existing) {
        this.logger.info('Idempotent request, returning existing result', {
          traceId,
          transactionId: request.transactionId
        });
        return existing;
      }
      
      // Step 3: Risk assessment
      const riskScore = await this.assessRisk(request);
      if (riskScore > 0.8) {
        throw new HighRiskTransactionError('Transaction flagged as high risk');
      }
      
      // Step 4: Gateway authorization con circuit breaker
      const authorization = await this.circuitBreaker.execute(async () => {
        return await this.gateway.authorize({
          token: request.paymentToken,
          amount: request.amountCents,
          currency: request.currency,
          metadata: {
            merchantId: request.merchantData.merchantId,
            traceId
          }
        });
      });
      
      // Step 5: Persistir transacción
      const transaction = await this.repository.createTransaction({
        id: request.transactionId,
        authorizationId: authorization.id,
        amount: request.amountCents,
        currency: request.currency,
        status: 'authorized',
        merchantId: request.merchantData.merchantId,
        createdAt: new Date()
      });
      
      try {
        // Step 6: Capturar fondos
        const capture = await this.gateway.capture(authorization.id);
        
        // Step 7: Actualizar estado
        await this.repository.updateTransaction(transaction.id, {
          status: 'captured',
          captureId: capture.id,
          capturedAt: new Date()
        });
        
        // Step 8: Notificar resultado
        await this.notifier.notifyPaymentSuccess({
          transactionId: transaction.id,
          merchantId: request.merchantData.merchantId,
          orderReference: request.merchantData.orderReference
        });
        
        // Métricas de éxito
        this.metrics.recordSuccess('payment_processed', Date.now() - startTime);
        
        return {
          success: true,
          transactionId: transaction.id,
          chargeId: capture.chargeId,
          processedAt: new Date()
        };
        
      } catch (captureError) {
        // COMPENSACIÓN: Reversar autorización si falla la captura
        this.logger.error('Capture failed, initiating reversal', {
          traceId,
          error: captureError,
          authorizationId: authorization.id
        });
        
        await this.compensateFailedPayment(authorization.id, transaction.id);
        throw captureError;
      }
      
    } catch (error) {
      // Logging detallado de errores
      this.logger.error('Payment processing failed', {
        traceId,
        transactionId: request.transactionId,
        error: error.message,
        stack: error.stack,
        duration: Date.now() - startTime
      });
      
      // Métricas de fallo
      this.metrics.recordFailure('payment_processed', error.constructor.name);
      
      // Re-throw con contexto adicional
      throw new PaymentProcessingError(
        `Payment failed: ${error.message}`,
        { 
          originalError: error,
          traceId,
          transactionId: request.transactionId
        }
      );
    }
  }

  /**
   * Valida exhaustivamente el request de pago
   * 
   * VALIDACIONES:
   * - Formato de campos
   * - Business rules (montos, límites)
   * - Merchant status
   * - Velocity checks
   * - Blacklist checks
   */
  private async validateRequest(request: PaymentRequest): Promise<void> {
    // Validación de formato
    if (!this.isValidUUID(request.transactionId)) {
      throw new ValidationError('Invalid transaction ID format');
    }
    
    if (request.amountCents < 50 || request.amountCents > 99999999) {
      throw new ValidationError('Amount out of allowed range');
    }
    
    if (!this.isValidCurrency(request.currency)) {
      throw new ValidationError('Invalid or unsupported currency');
    }
    
    // Validación de merchant
    const merchant = await this.repository.getMerchant(request.merchantData.merchantId);
    if (!merchant || merchant.status !== 'active') {
      throw new ValidationError('Invalid or inactive merchant');
    }
    
    // Velocity check
    const recentTransactions = await this.repository.getRecentTransactions(
      request.customerInfo.emailHash,
      3600000 // última hora
    );
    
    if (recentTransactions.length > 10) {
      throw new ValidationError('Velocity limit exceeded');
    }
    
    // Blacklist check
    if (await this.isBlacklisted(request.customerInfo.ipAddress)) {
      throw new ValidationError('Blocked IP address');
    }
  }

  /**
   * Compensa una transacción fallida
   * 
   * ACCIONES DE COMPENSACIÓN:
   * 1. Reversar autorización en gateway
   * 2. Marcar transacción como failed
   * 3. Notificar al merchant
   * 4. Log para audit trail
   */
  private async compensateFailedPayment(
    authorizationId: string,
    transactionId: string
  ): Promise<void> {
    try {
      // Intentar reversar con retry
      await this.retryWithBackoff(async () => {
        await this.gateway.reverse(authorizationId);
      }, 3);
      
      // Actualizar estado en DB
      await this.repository.updateTransaction(transactionId, {
        status: 'reversed',
        reversedAt: new Date()
      });
      
      this.logger.info('Payment compensation completed', {
        transactionId,
        authorizationId
      });
      
    } catch (compensationError) {
      // Si la compensación falla, alertar para intervención manual
      this.logger.critical('Compensation failed - manual intervention required', {
        transactionId,
        authorizationId,
        error: compensationError
      });
      
      // Alertar a on-call
      await this.notifier.alertOnCall({
        severity: 'critical',
        message: `Failed compensation for transaction ${transactionId}`,
        context: { authorizationId, error: compensationError.message }
      });
    }
  }

  private generateTraceId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  private isValidUUID(uuid: string): boolean {
    const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    return uuidRegex.test(uuid);
  }

  private isValidCurrency(currency: string): boolean {
    const supportedCurrencies = ['USD', 'EUR', 'GBP', 'CAD', 'AUD'];
    return supportedCurrencies.includes(currency);
  }

  private async retryWithBackoff<T>(
    fn: () => Promise<T>,
    maxRetries: number
  ): Promise<T> {
    let lastError: Error;
    
    for (let i = 0; i < maxRetries; i++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;
        
        if (i < maxRetries - 1) {
          const delay = Math.min(1000 * Math.pow(2, i) + Math.random() * 1000, 30000);
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }
    
    throw lastError!;
  }
}

// ============================================
// EJEMPLO 2: Estructura de Proyecto Mantenible
// ============================================

/**
 * Estructura de carpetas para máxima mantenibilidad
 * 
 * src/
 * ├── core/                    # Lógica de negocio pura
 * │   ├── domain/             # Entidades y value objects
 * │   ├── use-cases/          # Casos de uso de la aplicación
 * │   └── interfaces/         # Puertos para adaptadores
 * │
 * ├── infrastructure/          # Implementaciones técnicas
 * │   ├── database/           # Repositorios y migraciones
 * │   ├── http/              # Controladores y middleware
 * │   ├── messaging/         # Queues y event handlers
 * │   └── external/          # Integraciones con servicios externos
 * │
 * ├── shared/                  # Código compartido
 * │   ├── errors/            # Custom error classes
 * │   ├── utils/             # Utilidades genéricas
 * │   ├── types/             # TypeScript types compartidos
 * │   └── constants/         # Constantes de la aplicación
 * │
 * ├── config/                  # Configuración
 * │   ├── database.ts        # Config de DB
 * │   ├── app.ts            # Config de aplicación
 * │   └── environments/      # Configs por ambiente
 * │
 * └── tests/                   # Tests organizados por tipo
 *     ├── unit/              # Tests unitarios
 *     ├── integration/       # Tests de integración
 *     ├── e2e/              # Tests end-to-end
 *     └── fixtures/         # Data de prueba
 */

// Ejemplo de organización modular
namespace ModularArchitecture {
  
  // ============ CORE DOMAIN ============
  
  /**
   * Entidad de dominio con lógica de negocio encapsulada
   * 
   * PRINCIPIOS:
   * - Inmutabilidad: Métodos retornan nuevas instancias
   * - Validación: El estado siempre es válido
   * - Rich domain model: La lógica está en el dominio
   */
  export class Order {
    private constructor(
      private readonly id: string,
      private readonly customerId: string,
      private readonly items: OrderItem[],
      private readonly status: OrderStatus,
      private readonly createdAt: Date,
      private readonly updatedAt: Date
    ) {
      // Constructor privado garantiza creación válida
      this.validate();
    }

    /**
     * Factory method con validación
     * 
     * @throws {InvalidOrderError} Si los datos son inválidos
     */
    static create(data: CreateOrderData): Order {
      if (!data.items || data.items.length === 0) {
        throw new InvalidOrderError('Order must have at least one item');
      }

      const now = new Date();
      return new Order(
        generateId(),
        data.customerId,
        data.items.map(item => OrderItem.create(item)),
        OrderStatus.PENDING,
        now,
        now
      );
    }

    /**
     * Reconstitución desde persistencia
     */
    static fromPersistence(data: OrderPersistenceData): Order {
      return new Order(
        data.id,
        data.customerId,
        data.items.map(item => OrderItem.fromPersistence(item)),
        data.status as OrderStatus,
        new Date(data.createdAt),
        new Date(data.updatedAt)
      );
    }

    /**
     * Método de negocio con validación de invariantes
     */
    confirm(): Order {
      if (this.status !== OrderStatus.PENDING) {
        throw new InvalidOrderStateError(
          `Cannot confirm order in status ${this.status}`
        );
      }

      return new Order(
        this.id,
        this.customerId,
        this.items,
        OrderStatus.CONFIRMED,
        this.createdAt,
        new Date()
      );
    }

    /**
     * Cálculo derivado con lógica de negocio
     */
    calculateTotal(): Money {
      return this.items.reduce(
        (total, item) => total.add(item.calculateSubtotal()),
        Money.zero(this.getCurrency())
      );
    }

    private validate(): void {
      if (!this.id || !this.customerId) {
        throw new InvalidOrderError('Missing required fields');
      }

      if (this.items.length === 0) {
        throw new InvalidOrderError('Order must have items');
      }

      // Validar que todos los items tengan la misma moneda
      const currencies = new Set(this.items.map(item => item.getCurrency()));
      if (currencies.size > 1) {
        throw new InvalidOrderError('All items must have the same currency');
      }
    }

    private getCurrency(): string {
      return this.items[0].getCurrency();
    }

    // Getters inmutables
    getId(): string { return this.id; }
    getCustomerId(): string { return this.customerId; }
    getItems(): ReadonlyArray<OrderItem> { return [...this.items]; }
    getStatus(): OrderStatus { return this.status; }
    getCreatedAt(): Date { return new Date(this.createdAt); }
    getUpdatedAt(): Date { return new Date(this.updatedAt); }
  }

  // ============ USE CASES ============

  /**
   * Caso de uso con dependencias inyectadas
   * 
   * RESPONSABILIDADES:
   * - Orquestar la lógica de negocio
   * - Coordinar entre diferentes dominios
   * - Manejar transacciones
   * - Emitir eventos
   */
  export class ConfirmOrderUseCase {
    constructor(
      private readonly orderRepository: OrderRepository,
      private readonly paymentService: PaymentService,
      private readonly inventoryService: InventoryService,
      private readonly eventBus: EventBus,
      private readonly logger: Logger
    ) {}

    /**
     * Ejecuta el caso de uso con manejo completo de errores
     * 
     * FLUJO:
     * 1. Cargar orden
     * 2. Validar disponibilidad de inventario
     * 3. Procesar pago
     * 4. Confirmar orden
     * 5. Reservar inventario
     * 6. Emitir eventos
     * 
     * COMPENSACIÓN:
     * Si falla después del pago, se hace refund automático
     */
    async execute(command: ConfirmOrderCommand): Promise<ConfirmOrderResult> {
      this.logger.info('Starting order confirmation', { orderId: command.orderId });

      // Comenzar transacción
      const transaction = await this.orderRepository.beginTransaction();

      try {
        // 1. Cargar orden con lock pessimista
        const order = await this.orderRepository.findByIdWithLock(
          command.orderId,
          transaction
        );

        if (!order) {
          throw new OrderNotFoundError(command.orderId);
        }

        // 2. Validar inventario
        const availabilityCheck = await this.inventoryService.checkAvailability(
          order.getItems().map(item => ({
            productId: item.getProductId(),
            quantity: item.getQuantity()
          }))
        );

        if (!availabilityCheck.available) {
          throw new InsufficientInventoryError(availabilityCheck.unavailableItems);
        }

        // 3. Procesar pago
        const paymentResult = await this.paymentService.processPayment({
          orderId: order.getId(),
          amount: order.calculateTotal(),
          customerId: order.getCustomerId()
        });

        try {
          // 4. Confirmar orden
          const confirmedOrder = order.confirm();

          // 5. Persistir cambios
          await this.orderRepository.save(confirmedOrder, transaction);

          // 6. Reservar inventario
          await this.inventoryService.reserve({
            orderId: order.getId(),
            items: order.getItems().map(item => ({
              productId: item.getProductId(),
              quantity: item.getQuantity()
            }))
          });

          // 7. Commit transacción
          await transaction.commit();

          // 8. Emitir eventos (fuera de transacción)
          await this.eventBus.publish(new OrderConfirmedEvent({
            orderId: confirmedOrder.getId(),
            customerId: confirmedOrder.getCustomerId(),
            total: confirmedOrder.calculateTotal(),
            confirmedAt: confirmedOrder.getUpdatedAt()
          }));

          this.logger.info('Order confirmed successfully', {
            orderId: confirmedOrder.getId(),
            paymentId: paymentResult.paymentId
          });

          return {
            success: true,
            orderId: confirmedOrder.getId(),
            paymentId: paymentResult.paymentId,
            confirmedAt: confirmedOrder.getUpdatedAt()
          };

        } catch (error) {
          // Compensar pago si falla después de cobrar
          this.logger.error('Order confirmation failed after payment, initiating refund', {
            orderId: order.getId(),
            paymentId: paymentResult.paymentId,
            error
          });

          await this.paymentService.refund({
            paymentId: paymentResult.paymentId,
            reason: 'Order confirmation failed'
          });

          throw error;
        }

      } catch (error) {
        // Rollback transacción
        await transaction.rollback();

        this.logger.error('Order confirmation failed', {
          orderId: command.orderId,
          error: error.message
        });

        throw error;
      }
    }
  }

  // ============ INFRASTRUCTURE ============

  /**
   * Repositorio con abstracción de persistencia
   * 
   * PATRONES:
   * - Repository pattern para aislar persistencia
   * - Unit of Work con transacciones
   * - Specification pattern para queries complejas
   */
  export class PostgresOrderRepository implements OrderRepository {
    constructor(
      private readonly db: DatabaseConnection,
      private readonly mapper: OrderMapper
    ) {}

    async findById(id: string): Promise<Order | null> {
      const result = await this.db.query(
        `SELECT * FROM orders WHERE id = $1`,
        [id]
      );

      if (!result.rows[0]) {
        return null;
      }

      // Cargar items relacionados
      const items = await this.db.query(
        `SELECT * FROM order_items WHERE order_id = $1`,
        [id]
      );

      return this.mapper.toDomain({
        ...result.rows[0],
        items: items.rows
      });
    }

    async findByIdWithLock(
      id: string, 
      transaction: Transaction
    ): Promise<Order | null> {
      const result = await transaction.query(
        `SELECT * FROM orders WHERE id = $1 FOR UPDATE`,
        [id]
      );

      if (!result.rows[0]) {
        return null;
      }

      const items = await transaction.query(
        `SELECT * FROM order_items WHERE order_id = $1`,
        [id]
      );

      return this.mapper.toDomain({
        ...result.rows[0],
        items: items.rows
      });
    }

    async save(order: Order, transaction?: Transaction): Promise<void> {
      const data = this.mapper.toPersistence(order);
      const conn = transaction || this.db;

      // Upsert order
      await conn.query(
        `INSERT INTO orders (id, customer_id, status, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5)
         ON CONFLICT (id) 
         DO UPDATE SET 
           status = EXCLUDED.status,
           updated_at = EXCLUDED.updated_at`,
        [data.id, data.customerId, data.status, data.createdAt, data.updatedAt]
      );

      // Upsert items
      for (const item of data.items) {
        await conn.query(
          `INSERT INTO order_items (id, order_id, product_id, quantity, price)
           VALUES ($1, $2, $3, $4, $5)
           ON CONFLICT (id) 
           DO UPDATE SET 
             quantity = EXCLUDED.quantity,
             price = EXCLUDED.price`,
          [item.id, data.id, item.productId, item.quantity, item.price]
        );
      }
    }

    async beginTransaction(): Promise<Transaction> {
      return await this.db.beginTransaction();
    }
  }
}

// ============================================
// EJEMPLO 3: Testing Estratégico
// ============================================

/**
 * Estrategia de testing para mantenibilidad
 * 
 * PIRÁMIDE DE TESTING:
 * - 70% Unit tests: Rápidos, aislados, determinísticos
 * - 20% Integration tests: Verifican interacciones
 * - 10% E2E tests: Validan flujos críticos
 * 
 * PRINCIPIOS:
 * - Tests como documentación viva
 * - Test behavior, not implementation
 * - Arrange-Act-Assert pattern
 * - Test data builders para setup complejo
 */

// Test builder para datos complejos
class OrderTestBuilder {
  private id = 'test-order-123';
  private customerId = 'customer-456';
  private items: OrderItem[] = [this.defaultItem()];
  private status = OrderStatus.PENDING;
  
  withId(id: string): this {
    this.id = id;
    return this;
  }
  
  withCustomerId(customerId: string): this {
    this.customerId = customerId;
    return this;
  }
  
  withItems(items: OrderItem[]): this {
    this.items = items;
    return this;
  }
  
  withStatus(status: OrderStatus): this {
    this.status = status;
    return this;
  }
  
  withEmptyItems(): this {
    this.items = [];
    return this;
  }
  
  build(): Order {
    return Order.fromPersistence({
      id: this.id,
      customerId: this.customerId,
      items: this.items.map(item => ({
        id: item.getId(),
        productId: item.getProductId(),
        quantity: item.getQuantity(),
        price: item.getPrice().getAmount()
      })),
      status: this.status,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    });
  }
  
  private defaultItem(): OrderItem {
    return OrderItem.create({
      productId: 'product-789',
      quantity: 2,
      price: Money.create(1000, 'USD')
    });
  }
}

// Tests expressivos y mantenibles
describe('Order Confirmation Use Case', () => {
  let useCase: ConfirmOrderUseCase;
  let orderRepository: MockOrderRepository;
  let paymentService: MockPaymentService;
  let inventoryService: MockInventoryService;
  let eventBus: MockEventBus;
  
  beforeEach(() => {
    // Setup con mocks configurables
    orderRepository = new MockOrderRepository();
    paymentService = new MockPaymentService();
    inventoryService = new MockInventoryService();
    eventBus = new MockEventBus();
    
    useCase = new ConfirmOrderUseCase(
      orderRepository,
      paymentService,
      inventoryService,
      eventBus,
      new MockLogger()
    );
  });
  
  describe('Happy Path', () => {
    it('should confirm order when all conditions are met', async () => {
      // Arrange
      const order = new OrderTestBuilder()
        .withStatus(OrderStatus.PENDING)
        .build();
      
      orderRepository.setOrder(order);
      inventoryService.setAvailable(true);
      paymentService.setSuccessful(true);
      
      // Act
      const result = await useCase.execute({
        orderId: order.getId()
      });
      
      // Assert
      expect(result.success).toBe(true);
      expect(result.orderId).toBe(order.getId());
      expect(result.paymentId).toBeDefined();
      
      // Verify side effects
      expect(orderRepository.savedOrders).toHaveLength(1);
      expect(orderRepository.savedOrders[0].getStatus()).toBe(OrderStatus.CONFIRMED);
      
      expect(eventBus.publishedEvents).toHaveLength(1);
      expect(eventBus.publishedEvents[0]).toBeInstanceOf(OrderConfirmedEvent);
    });
  });
  
  describe('Error Scenarios', () => {
    it('should throw when order not found', async () => {
      // Arrange
      orderRepository.setOrder(null);
      
      // Act & Assert
      await expect(useCase.execute({ orderId: 'non-existent' }))
        .rejects
        .toThrow(OrderNotFoundError);
      
      // Verify no side effects
      expect(paymentService.processedPayments).toHaveLength(0);
      expect(inventoryService.reservations).toHaveLength(0);
      expect(eventBus.publishedEvents).toHaveLength(0);
    });
    
    it('should throw when inventory not available', async () => {
      // Arrange
      const order = new OrderTestBuilder().build();
      orderRepository.setOrder(order);
      inventoryService.setAvailable(false);
      
      // Act & Assert
      await expect(useCase.execute({ orderId: order.getId() }))
        .rejects
        .toThrow(InsufficientInventoryError);
      
      // Verify no payment was attempted
      expect(paymentService.processedPayments).toHaveLength(0);
    });
    
    it('should refund payment when confirmation fails after payment', async () => {
      // Arrange
      const order = new OrderTestBuilder().build();
      orderRepository.setOrder(order);
      inventoryService.setAvailable(true);
      paymentService.setSuccessful(true);
      orderRepository.setSaveError(new Error('Database error'));
      
      // Act & Assert
      await expect(useCase.execute({ orderId: order.getId() }))
        .rejects
        .toThrow('Database error');
      
      // Verify refund was issued
      expect(paymentService.refunds).toHaveLength(1);
      expect(paymentService.refunds[0].reason).toBe('Order confirmation failed');
    });
  });
  
  describe('Edge Cases', () => {
    it('should handle concurrent order confirmations', async () => {
      // Test para race conditions
      const order = new OrderTestBuilder().build();
      orderRepository.setOrder(order);
      inventoryService.setAvailable(true);
      paymentService.setSuccessful(true);
      
      // Simular confirmaciones concurrentes
      const promise1 = useCase.execute({ orderId: order.getId() });
      const promise2 = useCase.execute({ orderId: order.getId() });
      
      const results = await Promise.allSettled([promise1, promise2]);
      
      // Una debe tener éxito, la otra debe fallar por lock
      const successes = results.filter(r => r.status === 'fulfilled');
      const failures = results.filter(r => r.status === 'rejected');
      
      expect(successes).toHaveLength(1);
      expect(failures).toHaveLength(1);
    });
  });
});

// ============================================
// EJEMPLO 4: Monitoreo y Observabilidad
// ============================================

/**
 * Sistema de monitoreo para mantenibilidad
 * 
 * MÉTRICAS CLAVE:
 * - Business metrics: Conversion, revenue, user engagement
 * - Technical metrics: Latency, error rate, throughput
 * - Infrastructure metrics: CPU, memory, disk I/O
 * - Custom metrics: Domain-specific indicators
 */
class ObservabilityService {
  private readonly metrics: MetricsCollector;
  private readonly tracer: Tracer;
  private readonly logger: StructuredLogger;
  
  constructor() {
    this.metrics = new MetricsCollector({
      prefix: 'app',
      defaultTags: {
        environment: process.env.NODE_ENV,
        service: 'payment-service',
        version: process.env.APP_VERSION
      }
    });
    
    this.tracer = new Tracer({
      serviceName: 'payment-service',
      samplingRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0
    });
    
    this.logger = new StructuredLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: 'json'
    });
  }
  
  /**
   * Wrapper para instrumentar funciones automáticamente
   * 
   * INSTRUMENTACIÓN:
   * - Distributed tracing
   * - Métricas de performance
   * - Logging estructurado
   * - Error tracking
   */
  instrument<T extends (...args: any[]) => Promise<any>>(
    fn: T,
    options: InstrumentOptions
  ): T {
    const self = this;
    
    return async function instrumented(
      this: any,
      ...args: Parameters<T>
    ): Promise<ReturnType<T>> {
      // Start span for distributed tracing
      const span = self.tracer.startSpan(options.name, {
        attributes: options.attributes || {}
      });
      
      // Contexto para logging
      const context = {
        traceId: span.getTraceId(),
        spanId: span.getSpanId(),
        operation: options.name,
        ...options.metadata
      };
      
      self.logger.debug('Operation started', context);
      
      const startTime = Date.now();
      
      try {
        // Ejecutar función
        const result = await fn.apply(this, args);
        
        // Métricas de éxito
        const duration = Date.now() - startTime;
        self.metrics.histogram(`${options.name}.duration`, duration, options.tags);
        self.metrics.increment(`${options.name}.success`, 1, options.tags);
        
        // Logging
        self.logger.info('Operation completed', {
          ...context,
          duration,
          success: true
        });
        
        // Span attributes
        span.setAttributes({
          'operation.duration': duration,
          'operation.success': true
        });
        
        return result;
        
      } catch (error) {
        // Métricas de error
        const duration = Date.now() - startTime;
        self.metrics.histogram(`${options.name}.duration`, duration, {
          ...options.tags,
          error: error.constructor.name
        });
        self.metrics.increment(`${options.name}.error`, 1, {
          ...options.tags,
          error_type: error.constructor.name
        });
        
        // Logging de error
        self.logger.error('Operation failed', {
          ...context,
          duration,
          success: false,
          error: {
            message: error.message,
            stack: error.stack,
            type: error.constructor.name
          }
        });
        
        // Span error
        span.setAttributes({
          'operation.duration': duration,
          'operation.success': false,
          'error.type': error.constructor.name,
          'error.message': error.message
        });
        span.setStatus({ code: SpanStatusCode.ERROR });
        
        throw error;
        
      } finally {
        span.end();
      }
    } as T;
  }
  
  /**
   * Health checks para monitoring
   */
  async checkHealth(): Promise<HealthStatus> {
    const checks: HealthCheck[] = [];
    
    // Database health
    const dbCheck = await this.checkDatabase();
    checks.push({
      name: 'database',
      status: dbCheck.healthy ? 'healthy' : 'unhealthy',
      latency: dbCheck.latency,
      error: dbCheck.error
    });
    
    // Redis health
    const redisCheck = await this.checkRedis();
    checks.push({
      name: 'redis',
      status: redisCheck.healthy ? 'healthy' : 'unhealthy',
      latency: redisCheck.latency,
      error: redisCheck.error
    });
    
    // External services
    const paymentGatewayCheck = await this.checkPaymentGateway();
    checks.push({
      name: 'payment_gateway',
      status: paymentGatewayCheck.healthy ? 'healthy' : 'degraded',
      latency: paymentGatewayCheck.latency,
      error: paymentGatewayCheck.error
    });
    
    // Overall status
    const unhealthyChecks = checks.filter(c => c.status === 'unhealthy');
    const degradedChecks = checks.filter(c => c.status === 'degraded');
    
    let overallStatus: 'healthy' | 'degraded' | 'unhealthy';
    if (unhealthyChecks.length > 0) {
      overallStatus = 'unhealthy';
    } else if (degradedChecks.length > 0) {
      overallStatus = 'degraded';
    } else {
      overallStatus = 'healthy';
    }
    
    return {
      status: overallStatus,
      timestamp: new Date(),
      checks,
      metadata: {
        version: process.env.APP_VERSION,
        uptime: process.uptime(),
        memory: process.memoryUsage()
      }
    };
  }
  
  /**
   * Custom business metrics
   */
  recordBusinessMetric(name: string, value: number, tags?: Record<string, string>): void {
    this.metrics.gauge(`business.${name}`, value, tags);
    
    // Alert si métrica crítica está fuera de rango
    if (this.isCriticalMetric(name)) {
      this.checkThresholds(name, value);
    }
  }
  
  private isCriticalMetric(name: string): boolean {
    const criticalMetrics = [
      'payment.success_rate',
      'order.completion_rate',
      'api.error_rate',
      'database.connection_pool.available'
    ];
    return criticalMetrics.includes(name);
  }
  
  private checkThresholds(metric: string, value: number): void {
    const thresholds: Record<string, { min?: number; max?: number }> = {
      'payment.success_rate': { min: 0.95 },
      'order.completion_rate': { min: 0.80 },
      'api.error_rate': { max: 0.01 },
      'database.connection_pool.available': { min: 5 }
    };
    
    const threshold = thresholds[metric];
    if (!threshold) return;
    
    if (threshold.min !== undefined && value < threshold.min) {
      this.logger.critical(`Metric ${metric} below threshold`, {
        metric,
        value,
        threshold: threshold.min,
        type: 'below_minimum'
      });
      
      // Trigger alert
      this.sendAlert({
        severity: 'critical',
        metric,
        value,
        threshold: threshold.min,
        message: `${metric} is ${value}, below minimum threshold of ${threshold.min}`
      });
    }
    
    if (threshold.max !== undefined && value > threshold.max) {
      this.logger.critical(`Metric ${metric} above threshold`, {
        metric,
        value,
        threshold: threshold.max,
        type: 'above_maximum'
      });
      
      // Trigger alert
      this.sendAlert({
        severity: 'critical',
        metric,
        value,
        threshold: threshold.max,
        message: `${metric} is ${value}, above maximum threshold of ${threshold.max}`
      });
    }
  }
}
```

### Mejores prácticas adicionales:

1. **Code Reviews efectivos**: Enfócate en arquitectura, no en estilo
2. **Refactoring continuo**: Boy Scout Rule - deja el código mejor de lo que lo encontraste
3. **Deprecation strategy**: Marca código obsoleto y provee migración paths
4. **Performance budgets**: Define límites de performance y monitoréalos
5. **Dependency management**: Actualiza regularmente, audita vulnerabilidades
6. **Feature flags**: Permite rollbacks seguros y experimentación
7. **Runbooks**: Documenta procedimientos operacionales

El código mantenible es una inversión en el futuro del producto y la salud mental del equipo.