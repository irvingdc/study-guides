# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 10: Preguntas Comunes de Entrevista
### Para Entrevista Senior Software Engineer

---

## 10. Preguntas Comunes de Entrevista - Preparación Completa

Esta sección cubre las preguntas más frecuentes en entrevistas de refactorización y mantenimiento para posiciones senior. Cada pregunta incluye una respuesta comprehensiva con ejemplos en TypeScript.

---

## 10.1 Preguntas Conceptuales

### Pregunta 1: "¿Cómo identificas código que necesita refactorización?"

**Respuesta Completa:**

Identifico código que necesita refactorización usando múltiples señales y métricas objetivas:

**Indicadores Cuantitativos:**
- **Complejidad ciclomática > 10**: Indica demasiados paths de ejecución
- **Líneas de código por función > 50**: Sugiere violación de SRP
- **Profundidad de anidación > 3**: Dificulta la comprensión
- **Acoplamiento alto**: Muchas dependencias entre módulos
- **Cobertura de tests < 80%**: Indica código difícil de testear
- **Duplicación > 3 instancias**: Violación de DRY

**Indicadores Cualitativos:**
- **Dificultad para agregar features**: Cambios simples requieren modificaciones extensas
- **Bugs recurrentes**: La misma área genera múltiples issues
- **Evitación del equipo**: Nadie quiere tocar ciertas partes
- **Nombres confusos**: Variables/funciones que no expresan intención
- **Comentarios excesivos**: Código que necesita explicación no es claro
- **Violaciones de convenciones**: Inconsistencia con el resto del codebase

**Herramientas que uso:**
- SonarQube para análisis estático
- ESLint con reglas de complejidad
- Bundle analyzers para detectar código muerto
- Git history para identificar hotspots
- Code coverage reports

```typescript
// EJEMPLO: Código que necesita refactorización
class BadUserService {
  // 🚨 Múltiples responsabilidades
  async processUser(data: any) { // 🚨 Tipo 'any'
    // 🚨 Función muy larga (>100 líneas)
    // 🚨 Alta complejidad ciclomática
    if (data.type === 'new') {
      // Validación
      if (!data.email || !data.email.includes('@')) {
        return { error: 'Invalid email' };
      }
      if (!data.password || data.password.length < 8) {
        return { error: 'Invalid password' };
      }
      
      // 🚨 Lógica de negocio mezclada con infraestructura
      const existingUser = await db.query('SELECT * FROM users WHERE email = ?', [data.email]);
      if (existingUser.length > 0) {
        return { error: 'User exists' };
      }
      
      // 🚨 Manipulación directa de datos
      const hashedPassword = await bcrypt.hash(data.password, 10);
      const userId = uuid();
      
      // 🚨 SQL hardcodeado
      await db.query(
        'INSERT INTO users (id, email, password, created_at) VALUES (?, ?, ?, ?)',
        [userId, data.email, hashedPassword, new Date()]
      );
      
      // 🚨 Lógica de email mezclada
      await sendEmail({
        to: data.email,
        subject: 'Welcome',
        body: `Welcome ${data.email}!` // 🚨 Template hardcodeado
      });
      
      // 🚨 Logging inconsistente
      console.log('User created: ' + userId);
      
      return { userId };
      
    } else if (data.type === 'update') {
      // 🚨 Duplicación de validación
      if (!data.email || !data.email.includes('@')) {
        return { error: 'Invalid email' };
      }
      
      // ... más código duplicado
    } else if (data.type === 'delete') {
      // ... más branches
    }
    // 🚨 No hay manejo de caso default
  }
}

// SOLUCIÓN: Código refactorizado
class GoodUserService {
  constructor(
    private userRepository: UserRepository,
    private passwordService: PasswordService,
    private emailService: EmailService,
    private logger: Logger
  ) {}

  async createUser(command: CreateUserCommand): Promise<Result<User>> {
    // Validación separada y reutilizable
    const validationResult = this.validateUserData(command);
    if (validationResult.isFailure()) {
      return Result.fail(validationResult.error);
    }

    // Check de duplicados con abstracción
    const existingUser = await this.userRepository.findByEmail(command.email);
    if (existingUser) {
      return Result.fail(new UserAlreadyExistsError(command.email));
    }

    // Creación de entidad con lógica de dominio
    const hashedPassword = await this.passwordService.hash(command.password);
    const user = User.create({
      email: command.email,
      hashedPassword
    });

    // Persistencia abstraída
    await this.userRepository.save(user);

    // Efectos secundarios desacoplados
    await this.emailService.sendWelcomeEmail(user);
    
    // Logging estructurado
    this.logger.info('User created', { userId: user.id, email: user.email });

    return Result.ok(user);
  }

  private validateUserData(data: CreateUserCommand): ValidationResult {
    const validator = new UserValidator();
    return validator.validate(data);
  }
}
```

---

### Pregunta 2: "¿Cuál es tu estrategia para refactorizar código legacy?"

**Respuesta Completa:**

Mi estrategia para refactorizar código legacy sigue un enfoque sistemático y de bajo riesgo:

**Fase 1: Entendimiento y Estabilización**
1. **Crear tests de caracterización**: Documentan el comportamiento actual
2. **Agregar logging y monitoring**: Entender el uso en producción
3. **Documentar el conocimiento**: Crear diagramas y documentación
4. **Identificar boundaries**: Encontrar los límites naturales del sistema

**Fase 2: Aislamiento**
1. **Introducir interfaces**: Crear contratos claros
2. **Dependency injection**: Hacer el código testeable
3. **Strangler Fig pattern**: Reemplazar gradualmente
4. **Feature toggles**: Permitir rollback seguro

**Fase 3: Refactorización Incremental**
1. **Boy Scout Rule**: Mejorar código que tocas
2. **Refactorings pequeños**: Cambios atómicos y seguros
3. **Continuous Integration**: Validar cada cambio
4. **Métricas de progreso**: Medir mejora objetivamente

```typescript
// EJEMPLO: Estrategia de refactorización de código legacy

// PASO 1: Código legacy original
class LegacyPaymentProcessor {
  processPayment(amount: number, cardNumber: string, cvv: string): boolean {
    // Validación acoplada
    if (amount <= 0) return false;
    if (cardNumber.length !== 16) return false;
    if (cvv.length !== 3) return false;
    
    // Llamada directa a servicio externo
    const result = fetch('https://payment-gateway.com/charge', {
      method: 'POST',
      body: JSON.stringify({ amount, cardNumber, cvv })
    });
    
    // Lógica de negocio mezclada
    if (result.status === 200) {
      // Side effect directo
      database.query(`INSERT INTO payments VALUES (${amount}, '${cardNumber}')`);
      
      // Logging primitivo
      console.log('Payment processed');
      return true;
    }
    
    return false;
  }
}

// PASO 2: Tests de caracterización
describe('LegacyPaymentProcessor - Characterization Tests', () => {
  it('should reject negative amounts', () => {
    const processor = new LegacyPaymentProcessor();
    expect(processor.processPayment(-100, '1234567890123456', '123')).toBe(false);
  });
  
  it('should reject invalid card numbers', () => {
    const processor = new LegacyPaymentProcessor();
    expect(processor.processPayment(100, '123', '123')).toBe(false);
  });
  
  // Documentar TODO el comportamiento actual
});

// PASO 3: Introducir abstracción sin cambiar comportamiento
interface PaymentGateway {
  charge(request: PaymentRequest): Promise<PaymentResult>;
}

class LegacyPaymentGatewayAdapter implements PaymentGateway {
  async charge(request: PaymentRequest): Promise<PaymentResult> {
    // Wrap la llamada legacy
    const response = await fetch('https://payment-gateway.com/charge', {
      method: 'POST',
      body: JSON.stringify({
        amount: request.amount,
        cardNumber: request.cardNumber,
        cvv: request.cvv
      })
    });
    
    return {
      success: response.status === 200,
      transactionId: response.headers.get('x-transaction-id') || ''
    };
  }
}

// PASO 4: Extraer responsabilidades gradualmente
class PaymentValidator {
  validate(request: PaymentRequest): ValidationResult {
    const errors: string[] = [];
    
    if (request.amount <= 0) {
      errors.push('Amount must be positive');
    }
    
    if (!this.isValidCardNumber(request.cardNumber)) {
      errors.push('Invalid card number');
    }
    
    if (!this.isValidCVV(request.cvv)) {
      errors.push('Invalid CVV');
    }
    
    return {
      isValid: errors.length === 0,
      errors
    };
  }
  
  private isValidCardNumber(cardNumber: string): boolean {
    // Luhn algorithm
    return cardNumber.length === 16 && this.luhnCheck(cardNumber);
  }
  
  private isValidCVV(cvv: string): boolean {
    return /^\d{3,4}$/.test(cvv);
  }
  
  private luhnCheck(cardNumber: string): boolean {
    // Implementación del algoritmo de Luhn
    let sum = 0;
    let isEven = false;
    
    for (let i = cardNumber.length - 1; i >= 0; i--) {
      let digit = parseInt(cardNumber[i], 10);
      
      if (isEven) {
        digit *= 2;
        if (digit > 9) {
          digit -= 9;
        }
      }
      
      sum += digit;
      isEven = !isEven;
    }
    
    return sum % 10 === 0;
  }
}

// PASO 5: Nueva implementación coexistiendo con legacy
class ModernPaymentProcessor {
  constructor(
    private validator: PaymentValidator,
    private gateway: PaymentGateway,
    private repository: PaymentRepository,
    private logger: Logger,
    private featureFlags: FeatureFlags
  ) {}
  
  async processPayment(request: PaymentRequest): Promise<PaymentResult> {
    // Feature flag para rollback seguro
    if (!this.featureFlags.isEnabled('modern-payment-processor')) {
      return this.legacyProcess(request);
    }
    
    // Validación
    const validation = this.validator.validate(request);
    if (!validation.isValid) {
      this.logger.warn('Payment validation failed', { 
        errors: validation.errors,
        request 
      });
      return PaymentResult.validationFailed(validation.errors);
    }
    
    try {
      // Procesar pago
      const gatewayResult = await this.gateway.charge(request);
      
      if (gatewayResult.success) {
        // Persistir
        await this.repository.savePayment({
          amount: request.amount,
          cardNumberLast4: request.cardNumber.slice(-4),
          transactionId: gatewayResult.transactionId,
          timestamp: new Date()
        });
        
        this.logger.info('Payment processed successfully', {
          transactionId: gatewayResult.transactionId,
          amount: request.amount
        });
        
        return PaymentResult.success(gatewayResult.transactionId);
      }
      
      return PaymentResult.gatewayDeclined(gatewayResult.declineReason);
      
    } catch (error) {
      this.logger.error('Payment processing failed', { error, request });
      return PaymentResult.systemError();
    }
  }
  
  private async legacyProcess(request: PaymentRequest): Promise<PaymentResult> {
    // Mantener comportamiento legacy mientras migramos
    const processor = new LegacyPaymentProcessor();
    const success = processor.processPayment(
      request.amount,
      request.cardNumber,
      request.cvv
    );
    
    return success 
      ? PaymentResult.success('legacy-transaction')
      : PaymentResult.failed();
  }
}

// PASO 6: Migración gradual con métricas
class PaymentMigrationMonitor {
  private metrics = new MetricsCollector();
  
  recordLegacyCall(): void {
    this.metrics.increment('payment.legacy.calls');
  }
  
  recordModernCall(): void {
    this.metrics.increment('payment.modern.calls');
  }
  
  recordMigrationProgress(): void {
    const totalCalls = this.getTotalCalls();
    const modernCalls = this.getModernCalls();
    const percentage = (modernCalls / totalCalls) * 100;
    
    this.metrics.gauge('payment.migration.percentage', percentage);
    
    if (percentage >= 100) {
      this.logger.info('Migration complete! Legacy code can be removed.');
    }
  }
}
```

---

### Pregunta 3: "¿Cómo manejas la deuda técnica en un proyecto?"

**Respuesta Completa:**

Manejo la deuda técnica como una parte integral del proceso de desarrollo, no como algo separado:

**1. Identificación y Catalogación:**
- Mantengo un registro de deuda técnica en el backlog
- Clasifico por impacto (alto/medio/bajo) y esfuerzo
- Uso herramientas como SonarQube para métricas objetivas
- Documento el costo de no addressing la deuda

**2. Priorización:**
- **Deuda que bloquea features**: Prioridad máxima
- **Deuda que causa bugs frecuentes**: Alta prioridad
- **Deuda que ralentiza desarrollo**: Media prioridad
- **Deuda cosmética**: Baja prioridad

**3. Estrategias de Pago:**
- **20% rule**: Dedicar 20% del sprint a deuda técnica
- **Boy Scout Rule**: Mejorar código que tocas
- **Tech debt sprints**: Sprints dedicados trimestralmente
- **Refactoring as part of feature work**: Incluir en estimaciones

```typescript
// EJEMPLO: Sistema de gestión de deuda técnica

/**
 * Sistema para tracking y gestión de deuda técnica
 */
class TechnicalDebtManager {
  private debtItems: TechnicalDebtItem[] = [];
  private readonly DEBT_THRESHOLD = 100; // Puntos de deuda máximos permitidos
  
  /**
   * Registrar nueva deuda técnica
   */
  registerDebt(debt: TechnicalDebtItem): void {
    // Calcular el costo real de la deuda
    debt.cost = this.calculateDebtCost(debt);
    
    // Agregar al registro
    this.debtItems.push(debt);
    
    // Alertar si superamos threshold
    if (this.getTotalDebtScore() > this.DEBT_THRESHOLD) {
      this.alertHighDebt();
    }
    
    // Logging para visibilidad
    this.logger.info('Technical debt registered', {
      type: debt.type,
      impact: debt.impact,
      cost: debt.cost,
      location: debt.location
    });
  }
  
  /**
   * Calcular el costo de la deuda en términos de:
   * - Tiempo adicional de desarrollo
   * - Riesgo de bugs
   * - Impacto en performance
   */
  private calculateDebtCost(debt: TechnicalDebtItem): DebtCost {
    const baseCost = {
      development: 0,
      maintenance: 0,
      risk: 0
    };
    
    // Costo de desarrollo (horas adicionales por feature)
    switch (debt.impact) {
      case 'high':
        baseCost.development = 8; // 1 día adicional por feature
        break;
      case 'medium':
        baseCost.development = 4; // Medio día adicional
        break;
      case 'low':
        baseCost.development = 1; // 1 hora adicional
        break;
    }
    
    // Costo de mantenimiento (horas por mes)
    if (debt.type === 'code-duplication') {
      baseCost.maintenance = debt.occurrences * 2;
    } else if (debt.type === 'missing-tests') {
      baseCost.maintenance = 5; // Debugging sin tests
    } else if (debt.type === 'complex-function') {
      baseCost.maintenance = 3; // Entender código complejo
    }
    
    // Riesgo (probabilidad de causar bugs)
    if (debt.type === 'missing-tests') {
      baseCost.risk = 0.3; // 30% probabilidad de bugs
    } else if (debt.type === 'tight-coupling') {
      baseCost.risk = 0.2; // 20% probabilidad de cascading failures
    }
    
    return baseCost;
  }
  
  /**
   * Priorizar deuda técnica para el próximo sprint
   */
  prioritizeDebt(): TechnicalDebtItem[] {
    return this.debtItems
      .map(item => ({
        ...item,
        priority: this.calculatePriority(item)
      }))
      .sort((a, b) => b.priority - a.priority)
      .slice(0, 5); // Top 5 items para el sprint
  }
  
  private calculatePriority(debt: TechnicalDebtItem): number {
    let score = 0;
    
    // Impacto en velocidad de desarrollo
    score += debt.cost.development * 10;
    
    // Frecuencia de modificación del código
    score += debt.changeFrequency * 5;
    
    // Riesgo de bugs
    score += debt.cost.risk * 100;
    
    // Número de desarrolladores afectados
    score += debt.affectedDevelopers * 3;
    
    // Edad de la deuda (más vieja = más prioritaria)
    const ageInDays = (Date.now() - debt.createdAt.getTime()) / (1000 * 60 * 60 * 24);
    score += Math.min(ageInDays, 90) / 3;
    
    return score;
  }
  
  /**
   * Generar reporte de deuda técnica para stakeholders
   */
  generateReport(): TechnicalDebtReport {
    const totalCost = this.debtItems.reduce((sum, item) => ({
      development: sum.development + item.cost.development,
      maintenance: sum.maintenance + item.cost.maintenance,
      risk: sum.risk + item.cost.risk
    }), { development: 0, maintenance: 0, risk: 0 });
    
    const report: TechnicalDebtReport = {
      summary: {
        totalItems: this.debtItems.length,
        totalDevelopmentHours: totalCost.development,
        monthlyMaintenanceHours: totalCost.maintenance,
        averageRisk: totalCost.risk / this.debtItems.length,
        estimatedCost: this.calculateMonetaryCost(totalCost)
      },
      
      byCategory: this.groupByCategory(),
      
      byTeam: this.groupByTeam(),
      
      trends: this.analyzeTrends(),
      
      recommendations: this.generateRecommendations(),
      
      proposedActions: this.prioritizeDebt().map(item => ({
        description: item.description,
        estimatedEffort: item.estimatedEffort,
        expectedBenefit: this.calculateBenefit(item),
        assignedTo: item.proposedAssignee
      }))
    };
    
    return report;
  }
  
  /**
   * Integración con el proceso de desarrollo
   */
  integrateWithDevelopment(): void {
    // Hook en PR reviews
    this.onPullRequest((pr) => {
      const addedDebt = this.analyzeCodeForDebt(pr.changes);
      const removedDebt = this.analyzeDebtRemoval(pr.changes);
      
      // Agregar comentario al PR
      pr.addComment(`
        Technical Debt Impact:
        - Added: ${addedDebt.length} items
        - Removed: ${removedDebt.length} items
        - Net change: ${addedDebt.length - removedDebt.length}
        
        ${addedDebt.length > removedDebt.length ? 
          '⚠️ This PR increases technical debt' : 
          '✅ This PR reduces technical debt'}
      `);
      
      // Bloquear si agrega mucha deuda
      if (addedDebt.filter(d => d.impact === 'high').length > 0) {
        pr.requestChanges('High impact technical debt detected. Please address or document.');
      }
    });
    
    // Métricas en dashboards
    this.metrics.gauge('tech_debt.total_items', this.debtItems.length);
    this.metrics.gauge('tech_debt.total_cost_hours', this.getTotalCost());
    this.metrics.gauge('tech_debt.high_priority_items', 
      this.debtItems.filter(d => d.impact === 'high').length
    );
  }
}

// Tipos de deuda técnica
interface TechnicalDebtItem {
  id: string;
  type: 'code-duplication' | 'missing-tests' | 'complex-function' | 
        'tight-coupling' | 'outdated-dependency' | 'missing-documentation';
  description: string;
  location: CodeLocation;
  impact: 'high' | 'medium' | 'low';
  estimatedEffort: number; // Horas para resolver
  createdAt: Date;
  createdBy: string;
  changeFrequency: number; // Veces que se modifica por mes
  affectedDevelopers: number;
  cost: DebtCost;
  priority?: number;
  proposedAssignee?: string;
  occurrences?: number; // Para duplicación
}

interface DebtCost {
  development: number; // Horas adicionales de desarrollo
  maintenance: number; // Horas mensuales de mantenimiento
  risk: number; // Probabilidad de causar bugs (0-1)
}

// Automatización de detección de deuda
class DebtDetector {
  /**
   * Analiza código y detecta deuda automáticamente
   */
  async analyzeCodebase(): Promise<TechnicalDebtItem[]> {
    const detectedDebt: TechnicalDebtItem[] = [];
    
    // Detectar funciones complejas
    const complexFunctions = await this.findComplexFunctions();
    detectedDebt.push(...complexFunctions.map(fn => ({
      id: generateId(),
      type: 'complex-function' as const,
      description: `Function ${fn.name} has cyclomatic complexity of ${fn.complexity}`,
      location: fn.location,
      impact: fn.complexity > 20 ? 'high' : fn.complexity > 10 ? 'medium' : 'low',
      estimatedEffort: Math.ceil(fn.complexity / 5),
      createdAt: new Date(),
      createdBy: 'automated-scan',
      changeFrequency: fn.gitStats.changesLastMonth,
      affectedDevelopers: fn.gitStats.uniqueAuthors,
      cost: { development: 0, maintenance: 0, risk: 0 }
    })));
    
    // Detectar duplicación de código
    const duplications = await this.findCodeDuplication();
    detectedDebt.push(...duplications.map(dup => ({
      id: generateId(),
      type: 'code-duplication' as const,
      description: `${dup.lines} lines duplicated ${dup.occurrences} times`,
      location: dup.locations[0],
      impact: dup.lines > 50 ? 'high' : dup.lines > 20 ? 'medium' : 'low',
      estimatedEffort: dup.occurrences * 2,
      createdAt: new Date(),
      createdBy: 'automated-scan',
      changeFrequency: dup.averageChangeFrequency,
      affectedDevelopers: dup.affectedFiles.length,
      cost: { development: 0, maintenance: 0, risk: 0 },
      occurrences: dup.occurrences
    })));
    
    // Detectar falta de tests
    const untestedCode = await this.findUntestedCode();
    detectedDebt.push(...untestedCode.map(code => ({
      id: generateId(),
      type: 'missing-tests' as const,
      description: `${code.className} has ${code.coverage}% test coverage`,
      location: code.location,
      impact: code.complexity > 10 ? 'high' : 'medium',
      estimatedEffort: code.publicMethods * 2,
      createdAt: new Date(),
      createdBy: 'automated-scan',
      changeFrequency: code.changeFrequency,
      affectedDevelopers: code.authors.length,
      cost: { development: 0, maintenance: 0, risk: 0 }
    })));
    
    return detectedDebt;
  }
}
```

---

### Pregunta 4: "Describe una situación donde tuviste que mejorar el performance de una aplicación"

**Respuesta Completa con Caso Real:**

```typescript
/**
 * CASO REAL: Sistema de búsqueda de propiedades que tardaba 8+ segundos
 * 
 * CONTEXTO:
 * - 5 millones de propiedades en base de datos
 * - Búsquedas complejas con 15+ filtros
 * - 10,000 usuarios concurrentes en peak
 * - Response time P95: 8.3 segundos
 * - CPU usage: 85% promedio
 * 
 * OBJETIVO: Reducir latencia a <2 segundos para P95
 */

// PROBLEMA ORIGINAL: Código ineficiente
class SlowPropertySearchService {
  async searchProperties(criteria: SearchCriteria): Promise<Property[]> {
    // ❌ PROBLEMA 1: Query N+1
    const properties = await db.query('SELECT * FROM properties');
    
    const results = [];
    for (const property of properties) {
      // ❌ PROBLEMA 2: Query en loop (N+1)
      const images = await db.query(
        'SELECT * FROM images WHERE property_id = ?',
        [property.id]
      );
      
      // ❌ PROBLEMA 3: Query adicional en loop
      const amenities = await db.query(
        'SELECT * FROM amenities WHERE property_id = ?',
        [property.id]
      );
      
      // ❌ PROBLEMA 4: Filtrado en memoria en lugar de DB
      if (this.matchesCriteria(property, criteria)) {
        // ❌ PROBLEMA 5: Cálculo costoso sin cache
        const distance = await this.calculateDistance(
          property.latitude,
          property.longitude,
          criteria.userLat,
          criteria.userLng
        );
        
        // ❌ PROBLEMA 6: Transformación pesada
        const enrichedProperty = {
          ...property,
          images,
          amenities,
          distance,
          // ❌ PROBLEMA 7: Serialización costosa
          prettyAddress: await this.geocodeAddress(property.address),
          similarProperties: await this.findSimilar(property)
        };
        
        results.push(enrichedProperty);
      }
    }
    
    // ❌ PROBLEMA 8: Sort en memoria con todos los resultados
    return results.sort((a, b) => a.distance - b.distance);
  }
}

// SOLUCIÓN OPTIMIZADA: Implementación eficiente
class OptimizedPropertySearchService {
  private cache: CacheService;
  private searchIndex: ElasticsearchClient;
  private dbReplica: DatabaseConnection;
  
  constructor(
    cache: CacheService,
    searchIndex: ElasticsearchClient,
    dbReplica: DatabaseConnection
  ) {
    this.cache = cache;
    this.searchIndex = searchIndex;
    this.dbReplica = dbReplica; // Read replica para no impactar writes
  }
  
  /**
   * OPTIMIZACIÓN 1: Usar índice de búsqueda especializado
   * Elasticsearch para búsquedas complejas y geo queries
   */
  async searchProperties(criteria: SearchCriteria): Promise<PropertySearchResult> {
    const startTime = performance.now();
    
    // Check cache first
    const cacheKey = this.generateCacheKey(criteria);
    const cachedResult = await this.cache.get(cacheKey);
    if (cachedResult) {
      this.metrics.increment('search.cache.hit');
      return cachedResult;
    }
    
    try {
      // OPTIMIZACIÓN 2: Query optimizado en Elasticsearch
      const searchQuery = this.buildElasticsearchQuery(criteria);
      const searchResults = await this.searchIndex.search(searchQuery);
      
      // OPTIMIZACIÓN 3: Batch loading con DataLoader
      const propertyIds = searchResults.hits.map(hit => hit._id);
      const properties = await this.batchLoadProperties(propertyIds);
      
      // OPTIMIZACIÓN 4: Paginación desde el inicio
      const paginatedResults = this.paginate(properties, criteria.page, criteria.limit);
      
      // OPTIMIZACIÓN 5: Lazy loading de datos adicionales
      const result: PropertySearchResult = {
        properties: paginatedResults,
        total: searchResults.total,
        page: criteria.page,
        hasMore: searchResults.total > (criteria.page * criteria.limit),
        facets: searchResults.aggregations, // Para filtros dinámicos
        
        // Métodos para cargar datos adicionales on-demand
        async loadImages(propertyId: string) {
          return await this.lazyLoadImages(propertyId);
        },
        async loadDetails(propertyId: string) {
          return await this.lazyLoadDetails(propertyId);
        }
      };
      
      // Cache result con TTL apropiado
      await this.cache.set(cacheKey, result, 300); // 5 minutos
      
      // Métricas de performance
      const duration = performance.now() - startTime;
      this.metrics.histogram('search.duration', duration);
      
      if (duration > 2000) {
        this.logger.warn('Slow search detected', {
          duration,
          criteria,
          resultCount: result.total
        });
      }
      
      return result;
      
    } catch (error) {
      this.metrics.increment('search.error');
      this.logger.error('Search failed', { error, criteria });
      
      // Fallback a búsqueda básica en DB
      return this.fallbackSearch(criteria);
    }
  }
  
  /**
   * OPTIMIZACIÓN 6: Query builder eficiente para Elasticsearch
   */
  private buildElasticsearchQuery(criteria: SearchCriteria): any {
    const query: any = {
      index: 'properties',
      body: {
        query: {
          bool: {
            must: [],
            filter: [],
            should: []
          }
        },
        // OPTIMIZACIÓN 7: Solo traer campos necesarios
        _source: ['id', 'title', 'price', 'location', 'thumbnail'],
        
        // OPTIMIZACIÓN 8: Agregaciones para faceted search
        aggs: {
          price_ranges: {
            histogram: {
              field: 'price',
              interval: 100000
            }
          },
          property_types: {
            terms: {
              field: 'type'
            }
          },
          amenities: {
            terms: {
              field: 'amenities'
            }
          }
        },
        
        // OPTIMIZACIÓN 9: Paginación eficiente con search_after
        size: criteria.limit || 20,
        search_after: criteria.searchAfter,
        
        // OPTIMIZACIÓN 10: Ordenamiento optimizado
        sort: this.buildSort(criteria)
      }
    };
    
    // Aplicar filtros eficientemente
    if (criteria.priceMin || criteria.priceMax) {
      query.body.query.bool.filter.push({
        range: {
          price: {
            gte: criteria.priceMin,
            lte: criteria.priceMax
          }
        }
      });
    }
    
    // OPTIMIZACIÓN 11: Geo query eficiente
    if (criteria.location) {
      query.body.query.bool.filter.push({
        geo_distance: {
          distance: criteria.radius || '10km',
          location: {
            lat: criteria.location.lat,
            lon: criteria.location.lng
          }
        }
      });
    }
    
    // Full text search con boost
    if (criteria.searchTerm) {
      query.body.query.bool.must.push({
        multi_match: {
          query: criteria.searchTerm,
          fields: [
            'title^3',      // Boost título
            'description^2', // Boost descripción
            'address'
          ],
          type: 'best_fields',
          fuzziness: 'AUTO'
        }
      });
    }
    
    return query;
  }
  
  /**
   * OPTIMIZACIÓN 12: Batch loading con DataLoader pattern
   */
  private async batchLoadProperties(ids: string[]): Promise<Property[]> {
    // DataLoader automatically batches and caches
    const loader = new DataLoader(async (batchIds: string[]) => {
      // OPTIMIZACIÓN 13: Single query con JOIN eficiente
      const query = `
        SELECT 
          p.*,
          COALESCE(
            json_agg(
              DISTINCT jsonb_build_object(
                'id', i.id,
                'url', i.url,
                'thumbnail', i.thumbnail
              )
            ) FILTER (WHERE i.id IS NOT NULL),
            '[]'
          ) as images,
          COALESCE(
            array_agg(
              DISTINCT a.name
            ) FILTER (WHERE a.id IS NOT NULL),
            ARRAY[]::text[]
          ) as amenities
        FROM properties p
        LEFT JOIN images i ON i.property_id = p.id AND i.is_primary = true
        LEFT JOIN property_amenities pa ON pa.property_id = p.id
        LEFT JOIN amenities a ON a.id = pa.amenity_id
        WHERE p.id = ANY($1)
        GROUP BY p.id
      `;
      
      const results = await this.dbReplica.query(query, [batchIds]);
      
      // Mantener orden de IDs
      const propertyMap = new Map(
        results.rows.map(row => [row.id, this.mapToProperty(row)])
      );
      
      return batchIds.map(id => propertyMap.get(id)!);
    });
    
    // Cargar en paralelo pero con batching automático
    return await Promise.all(ids.map(id => loader.load(id)));
  }
  
  /**
   * OPTIMIZACIÓN 14: Lazy loading para datos no críticos
   */
  private async lazyLoadImages(propertyId: string): Promise<PropertyImage[]> {
    const cacheKey = `images:${propertyId}`;
    
    // Check cache
    const cached = await this.cache.get(cacheKey);
    if (cached) return cached;
    
    // Load from CDN-optimized table
    const images = await this.dbReplica.query(
      `SELECT id, url, thumbnail, caption, width, height
       FROM images 
       WHERE property_id = $1
       ORDER BY display_order`,
      [propertyId]
    );
    
    const result = images.rows.map(img => ({
      ...img,
      // OPTIMIZACIÓN 15: URLs de CDN con responsive images
      urls: {
        thumbnail: `${CDN_URL}/thumb/${img.id}`,
        small: `${CDN_URL}/small/${img.id}`,
        medium: `${CDN_URL}/medium/${img.id}`,
        large: `${CDN_URL}/large/${img.id}`
      }
    }));
    
    // Cache por 1 hora
    await this.cache.set(cacheKey, result, 3600);
    
    return result;
  }
  
  /**
   * OPTIMIZACIÓN 16: Circuit breaker para servicios externos
   */
  private async fallbackSearch(criteria: SearchCriteria): Promise<PropertySearchResult> {
    this.logger.warn('Using fallback search', { criteria });
    
    // Query simplificado directo a DB
    const query = `
      SELECT id, title, price, latitude, longitude
      FROM properties
      WHERE price BETWEEN $1 AND $2
      LIMIT $3 OFFSET $4
    `;
    
    const results = await this.dbReplica.query(query, [
      criteria.priceMin || 0,
      criteria.priceMax || 999999999,
      criteria.limit || 20,
      ((criteria.page || 1) - 1) * (criteria.limit || 20)
    ]);
    
    return {
      properties: results.rows,
      total: results.rows.length,
      page: criteria.page || 1,
      hasMore: false,
      facets: {},
      loadImages: async () => [],
      loadDetails: async () => null
    };
  }
  
  /**
   * OPTIMIZACIÓN 17: Cache key generation con normalization
   */
  private generateCacheKey(criteria: SearchCriteria): string {
    // Normalizar y ordenar criteria para mejor hit rate
    const normalized = {
      ...criteria,
      // Round location para mejor cache hit rate
      location: criteria.location ? {
        lat: Math.round(criteria.location.lat * 1000) / 1000,
        lng: Math.round(criteria.location.lng * 1000) / 1000
      } : null,
      // Ordenar arrays para consistencia
      amenities: criteria.amenities?.sort(),
      propertyTypes: criteria.propertyTypes?.sort()
    };
    
    return `search:${crypto.createHash('md5')
      .update(JSON.stringify(normalized))
      .digest('hex')}`;
  }
  
  /**
   * OPTIMIZACIÓN 18: Connection pooling y read replicas
   */
  private initializeConnections(): void {
    // Pool de conexiones optimizado
    this.dbReplica = new Pool({
      host: process.env.DB_REPLICA_HOST,
      max: 20, // Max conexiones
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
      statement_timeout: 5000, // Kill queries lentas
      query_timeout: 5000
    });
    
    // Preparar statements frecuentes
    this.dbReplica.on('connect', async (client) => {
      await client.query('PREPARE property_by_id AS SELECT * FROM properties WHERE id = $1');
      await client.query('PREPARE images_by_property AS SELECT * FROM images WHERE property_id = $1');
    });
  }
}

/**
 * RESULTADOS DE LA OPTIMIZACIÓN:
 * 
 * ANTES:
 * - P50: 4.2s
 * - P95: 8.3s
 * - P99: 12.1s
 * - Throughput: 120 req/s
 * - CPU: 85%
 * - Memory: 4.2GB
 * 
 * DESPUÉS:
 * - P50: 180ms (-95.7%)
 * - P95: 1.2s (-85.5%)
 * - P99: 2.1s (-82.6%)
 * - Throughput: 1,850 req/s (+1,441%)
 * - CPU: 35% (-58.8%)
 * - Memory: 1.8GB (-57.1%)
 * 
 * TÉCNICAS CLAVE QUE FUNCIONARON:
 * 1. Elasticsearch para búsquedas complejas: -70% latency
 * 2. DataLoader pattern: Eliminó N+1 queries
 * 3. Caching estratégico: 65% cache hit rate
 * 4. Read replicas: Distribuyó carga
 * 5. Lazy loading: Redujo payload 80%
 * 6. CDN para imágenes: -60% bandwidth
 * 7. Connection pooling: -40% connection overhead
 */
```

---

### Pregunta 5: "¿Cómo aseguras la calidad del código en tu equipo?"

**Respuesta Completa:**

Aseguro la calidad del código mediante un enfoque multi-capa que combina herramientas, procesos y cultura:

```typescript
/**
 * Sistema completo de aseguramiento de calidad
 */
class CodeQualityAssurance {
  
  // ============================================
  // 1. AUTOMATED QUALITY GATES
  // ============================================
  
  /**
   * Pre-commit hooks para validación local
   */
  setupPreCommitHooks(): void {
    // .husky/pre-commit
    const preCommitConfig = `
#!/bin/sh
# Linting
npm run lint:fix

# Type checking
npm run typecheck

# Format
npm run prettier:fix

# Tests afectados
npm run test:affected

# Complejidad
npm run complexity:check

# Security
npm run security:audit
    `;
    
    // Configuración de complejidad máxima
    const eslintConfig = {
      rules: {
        'complexity': ['error', 10],
        'max-lines-per-function': ['error', 50],
        'max-depth': ['error', 3],
        'max-nested-callbacks': ['error', 3],
        'max-params': ['error', 4]
      }
    };
  }
  
  /**
   * CI/CD Pipeline con quality gates
   */
  setupCIPipeline(): GitHubActionsWorkflow {
    return {
      name: 'Code Quality Pipeline',
      on: ['pull_request', 'push'],
      jobs: {
        quality: {
          steps: [
            // Static analysis
            {
              name: 'SonarQube Analysis',
              run: 'sonar-scanner',
              with: {
                qualityGate: {
                  coverage: 80,
                  duplications: 3,
                  maintainabilityRating: 'A',
                  securityRating: 'A',
                  reliabilityRating: 'A'
                }
              }
            },
            
            // Security scanning
            {
              name: 'Security Audit',
              run: 'npm audit --audit-level=moderate'
            },
            
            // Dependency check
            {
              name: 'License Check',
              run: 'license-checker --onlyAllow "MIT;Apache-2.0;BSD"'
            },
            
            // Performance testing
            {
              name: 'Performance Budget',
              run: 'bundlesize',
              with: {
                maxSize: '200kb',
                compression: 'gzip'
              }
            },
            
            // Test coverage
            {
              name: 'Test Coverage',
              run: 'jest --coverage',
              with: {
                coverageThreshold: {
                  global: {
                    branches: 80,
                    functions: 80,
                    lines: 80,
                    statements: 80
                  }
                }
              }
            }
          ]
        }
      }
    };
  }
  
  // ============================================
  // 2. CODE REVIEW PROCESS
  // ============================================
  
  /**
   * Structured code review con checklist
   */
  class CodeReviewChecklist {
    private checks = [
      // Funcionalidad
      {
        category: 'Functionality',
        items: [
          'Does the code do what it\'s supposed to do?',
          'Are edge cases handled?',
          'Is error handling appropriate?',
          'Are there any obvious bugs?'
        ]
      },
      
      // Design
      {
        category: 'Design',
        items: [
          'Is the code well-structured?',
          'Does it follow SOLID principles?',
          'Is it at the right abstraction level?',
          'Could it be simpler?'
        ]
      },
      
      // Performance
      {
        category: 'Performance',
        items: [
          'Are there any N+1 queries?',
          'Is caching used appropriately?',
          'Are there unnecessary loops or computations?',
          'Will it scale?'
        ]
      },
      
      // Security
      {
        category: 'Security',
        items: [
          'Is input validated?',
          'Are SQL queries parameterized?',
          'Is sensitive data protected?',
          'Are permissions checked?'
        ]
      },
      
      // Maintainability
      {
        category: 'Maintainability',
        items: [
          'Is the code self-documenting?',
          'Are names meaningful?',
          'Is it testable?',
          'Will future developers understand it?'
        ]
      },
      
      // Testing
      {
        category: 'Testing',
        items: [
          'Are there sufficient tests?',
          'Do tests cover edge cases?',
          'Are tests maintainable?',
          'Do tests actually test behavior?'
        ]
      }
    ];
    
    /**
     * Automated review helper
     */
    async automatedReview(pr: PullRequest): Promise<ReviewResult> {
      const issues: Issue[] = [];
      
      // Check for large PRs
      if (pr.additions + pr.deletions > 400) {
        issues.push({
          level: 'warning',
          message: 'PR is too large. Consider breaking it down.',
          suggestion: 'PRs should be <400 lines for effective review'
        });
      }
      
      // Check for missing tests
      const hasTests = pr.files.some(f => 
        f.includes('.test.') || f.includes('.spec.')
      );
      if (!hasTests && pr.additions > 50) {
        issues.push({
          level: 'error',
          message: 'No tests found for new code',
          suggestion: 'Add tests for new functionality'
        });
      }
      
      // Check for console.logs
      const hasConsoleLogs = pr.changes.some(change => 
        change.includes('console.log')
      );
      if (hasConsoleLogs) {
        issues.push({
          level: 'warning',
          message: 'console.log found',
          suggestion: 'Use proper logging library'
        });
      }
      
      // Check for TODO comments
      const hasTodos = pr.changes.some(change => 
        change.includes('TODO') || change.includes('FIXME')
      );
      if (hasTodos) {
        issues.push({
          level: 'info',
          message: 'TODO/FIXME comments found',
          suggestion: 'Create tickets for TODOs'
        });
      }
      
      return {
        approved: issues.filter(i => i.level === 'error').length === 0,
        issues,
        suggestions: this.generateSuggestions(pr)
      };
    }
  }
  
  // ============================================
  // 3. TESTING STRATEGY
  // ============================================
  
  /**
   * Comprehensive testing approach
   */
  class TestingStrategy {
    /**
     * Test structure enforcer
     */
    enforceTestStructure(): void {
      // Every production file must have a test file
      const testCoverage = new Map<string, string>();
      
      // Scan for production files
      glob.sync('src/**/*.ts')
        .filter(file => !file.includes('.test.') && !file.includes('.spec.'))
        .forEach(file => {
          const testFile = file.replace('.ts', '.test.ts');
          if (!fs.existsSync(testFile)) {
            console.warn(`Missing test file for ${file}`);
            this.createTestTemplate(file, testFile);
          }
        });
    }
    
    /**
     * Test template generator
     */
    createTestTemplate(sourceFile: string, testFile: string): void {
      const className = path.basename(sourceFile, '.ts');
      
      const template = `
import { ${className} } from './${className}';

describe('${className}', () => {
  let instance: ${className};
  
  beforeEach(() => {
    // Setup
    instance = new ${className}();
  });
  
  afterEach(() => {
    // Cleanup
    jest.clearAllMocks();
  });
  
  describe('initialization', () => {
    it('should create instance', () => {
      expect(instance).toBeDefined();
    });
  });
  
  // TODO: Add actual tests
  describe.todo('main functionality');
  describe.todo('edge cases');
  describe.todo('error handling');
});
      `;
      
      fs.writeFileSync(testFile, template);
    }
    
    /**
     * Mutation testing para validar calidad de tests
     */
    async runMutationTesting(): Promise<MutationTestResult> {
      const config = {
        mutator: 'typescript',
        packageManager: 'npm',
        reporters: ['html', 'clear-text', 'progress'],
        testRunner: 'jest',
        coverageAnalysis: 'perTest',
        thresholds: {
          high: 80,
          low: 60,
          break: 50
        }
      };
      
      // Run Stryker mutation testing
      const result = await stryker.run(config);
      
      if (result.mutationScore < 60) {
        throw new Error('Mutation score too low. Tests are not effective.');
      }
      
      return result;
    }
  }
  
  // ============================================
  // 4. DOCUMENTATION STANDARDS
  // ============================================
  
  /**
   * Documentation enforcement
   */
  class DocumentationStandards {
    /**
     * JSDoc completeness checker
     */
    checkDocumentation(file: SourceFile): DocumentationIssue[] {
      const issues: DocumentationIssue[] = [];
      
      // Check all exported functions
      file.exportedFunctions.forEach(func => {
        if (!func.jsDoc) {
          issues.push({
            type: 'missing-jsdoc',
            location: func.location,
            message: `Function ${func.name} lacks documentation`
          });
        } else {
          // Check JSDoc completeness
          if (!func.jsDoc.description) {
            issues.push({
              type: 'incomplete-jsdoc',
              location: func.location,
              message: 'Missing description'
            });
          }
          
          // Check parameters
          func.parameters.forEach(param => {
            if (!func.jsDoc.params[param.name]) {
              issues.push({
                type: 'undocumented-param',
                location: func.location,
                message: `Parameter ${param.name} not documented`
              });
            }
          });
          
          // Check return type
          if (func.returnType && !func.jsDoc.returns) {
            issues.push({
              type: 'undocumented-return',
              location: func.location,
              message: 'Return value not documented'
            });
          }
        }
      });
      
      // Check README exists
      if (!fs.existsSync(path.join(file.directory, 'README.md'))) {
        issues.push({
          type: 'missing-readme',
          location: file.directory,
          message: 'Module lacks README'
        });
      }
      
      return issues;
    }
    
    /**
     * Generate documentation automatically
     */
    generateDocumentation(): void {
      // TypeDoc configuration
      const typeDocConfig = {
        entryPoints: ['src/index.ts'],
        out: 'docs',
        plugin: ['typedoc-plugin-markdown'],
        exclude: ['**/*.test.ts', '**/*.spec.ts'],
        readme: 'README.md',
        gitRevision: 'main',
        includeVersion: true,
        categorizeByGroup: true
      };
      
      // Generate API docs
      execSync('typedoc', typeDocConfig);
      
      // Generate architecture diagrams
      this.generateArchitectureDiagrams();
      
      // Generate dependency graphs
      this.generateDependencyGraphs();
    }
  }
  
  // ============================================
  // 5. TEAM CULTURE & PRACTICES
  // ============================================
  
  /**
   * Foster quality culture in team
   */
  class QualityCulture {
    /**
     * Pair programming sessions
     */
    organizePairProgramming(): void {
      // Rotate pairs weekly
      const schedule = this.generatePairRotation();
      
      // Focus areas for pairing
      const focusAreas = [
        'Complex algorithm implementation',
        'Critical bug fixes',
        'New feature architecture',
        'Performance optimizations',
        'Security implementations'
      ];
      
      // Track effectiveness
      this.metrics.track('pair_programming_hours');
      this.metrics.track('defects_after_pairing');
    }
    
    /**
     * Knowledge sharing sessions
     */
    organizeKnowledgeSharing(): void {
      const sessions = [
        {
          type: 'Code Review Club',
          frequency: 'weekly',
          format: 'Review interesting PRs together',
          duration: '1 hour'
        },
        {
          type: 'Architecture Decision Records',
          frequency: 'per decision',
          format: 'Document and discuss architectural choices',
          duration: '30 minutes'
        },
        {
          type: 'Post-Mortem Reviews',
          frequency: 'per incident',
          format: 'Blameless analysis of issues',
          duration: '1 hour'
        },
        {
          type: 'Tech Talks',
          frequency: 'bi-weekly',
          format: 'Team members present learnings',
          duration: '30 minutes'
        }
      ];
    }
    
    /**
     * Quality metrics dashboard
     */
    createQualityDashboard(): Dashboard {
      return {
        widgets: [
          {
            title: 'Code Coverage Trend',
            metric: 'test.coverage',
            visualization: 'line-chart',
            threshold: { min: 80, target: 90 }
          },
          {
            title: 'Technical Debt',
            metric: 'sonar.debt',
            visualization: 'gauge',
            threshold: { max: 100 }
          },
          {
            title: 'Build Success Rate',
            metric: 'ci.success_rate',
            visualization: 'percentage',
            threshold: { min: 95 }
          },
          {
            title: 'Mean Time to Review',
            metric: 'pr.review_time',
            visualization: 'histogram',
            threshold: { max: '4 hours' }
          },
          {
            title: 'Defect Escape Rate',
            metric: 'quality.defect_escape_rate',
            visualization: 'trend',
            threshold: { max: 0.05 }
          }
        ],
        alerts: [
          {
            condition: 'coverage < 75',
            action: 'Block deployments'
          },
          {
            condition: 'critical_vulnerabilities > 0',
            action: 'Page on-call'
          }
        ]
      };
    }
  }
}
```

---

## 10.2 Preguntas de Escenarios Prácticos

### Pregunta 6: "Te asignan mantener un sistema legacy sin documentación ni tests. ¿Cuál es tu plan de acción?"

**Respuesta Detallada con Plan de 90 días:**

```typescript
/**
 * PLAN DE ACCIÓN: Sistema Legacy Sin Documentación ni Tests
 * 
 * CONTEXTO ASUMIDO:
 * - Sistema en producción con usuarios activos
 * - Código de 5+ años sin mantenimiento activo
 * - Equipo original ya no está disponible
 * - Business critical pero con issues frecuentes
 */

class LegacySystemRecoveryPlan {
  
  // ============================================
  // SEMANA 1-2: DISCOVERY & STABILIZATION
  // ============================================
  
  async week1_2_Discovery(): Promise<DiscoveryReport> {
    const report: DiscoveryReport = {
      systemOverview: null,
      criticalPaths: [],
      dependencies: [],
      risks: [],
      quickWins: []
    };
    
    // Día 1-2: Setup inicial
    const setupTasks = [
      'Setup development environment',
      'Get production access (read-only)',
      'Setup monitoring and alerting',
      'Create backup of production data',
      'Setup version control if missing'
    ];
    
    // Día 3-5: Análisis estático
    const staticAnalysis = await this.performStaticAnalysis();
    
    // Día 6-8: Análisis dinámico
    const dynamicAnalysis = await this.performDynamicAnalysis();
    
    // Día 9-10: Documentación inicial
    await this.createInitialDocumentation(staticAnalysis, dynamicAnalysis);
    
    return report;
  }
  
  /**
   * Análisis estático del código
   */
  private async performStaticAnalysis(): Promise<StaticAnalysisResult> {
    // 1. Estructura del proyecto
    const projectStructure = await this.analyzeProjectStructure();
    
    // 2. Dependencias
    const dependencies = await this.analyzeDependencies();
    
    // 3. Complejidad
    const complexity = await this.analyzeComplexity();
    
    // 4. Identificar hotspots
    const hotspots = await this.identifyHotspots();
    
    // 5. Detectar patrones
    const patterns = await this.detectPatterns();
    
    return {
      structure: projectStructure,
      dependencies,
      complexity,
      hotspots,
      patterns,
      
      // Métricas clave
      metrics: {
        totalLines: 50000,
        totalFiles: 
        duplicatePercentage: 23,
        averageComplexity: 15,
        testCoverage: 0
      }
    };
  }
  
  /**
   * Herramientas para análisis
   */
  private analyzeProjectStructure(): ProjectStructure {
    // Usar herramientas como dependency-cruiser
    const dependencyGraph = execSync('dependency-cruiser src --output-type dot');
    
    // Analizar estructura de carpetas
    const structure = {
      '/src': {
        '/controllers': 'HTTP endpoints',
        '/services': 'Business logic',
        '/models': 'Data models',
        '/utils': 'Utilities',
        '/legacy': 'Old code to refactor'
      }
    };
    
    // Identificar entry points
    const entryPoints = [
      'src/index.js',
      'src/server.js',
      'src/app.js'
    ];
    
    return { dependencyGraph, structure, entryPoints };
  }
  
  /**
   * Análisis dinámico - Sistema en ejecución
   */
  private async performDynamicAnalysis(): Promise<DynamicAnalysisResult> {
    // 1. Instrumentar código para tracing
    await this.instrumentCode();
    
    // 2. Capturar tráfico de producción
    const productionTraffic = await this.captureProductionTraffic();
    
    // 3. Identificar flujos críticos
    const criticalFlows = await this.identifyCriticalFlows(productionTraffic);
    
    // 4. Mapear base de datos
    const databaseSchema = await this.extractDatabaseSchema();
    
    // 5. Identificar integraciones externas
    const externalIntegrations = await this.identifyExternalIntegrations();
    
    return {
      criticalFlows,
      databaseSchema,
      externalIntegrations,
      performanceBaseline: await this.measurePerformanceBaseline(),
      errorPatterns: await this.analyzeErrorLogs()
    };
  }
  
  // ============================================
  // SEMANA 3-4: CHARACTERIZATION TESTS
  // ============================================
  
  /**
   * Crear tests que documenten comportamiento actual
   */
  async week3_4_CharacterizationTests(): Promise<void> {
    // Estrategia: Empezar con los flujos más críticos
    const criticalPaths = await this.getCriticalPaths();
    
    for (const path of criticalPaths) {
      await this.createCharacterizationTest(path);
    }
  }
  
  /**
   * Ejemplo de characterization test
   */
  private createCharacterizationTest(path: CriticalPath): void {
    const testFile = `
// Characterization test for: ${path.name}
// Purpose: Document current behavior (even if incorrect)
// Created: ${new Date().toISOString()}

describe('${path.name} - Current Behavior', () => {
  let originalFunction;
  let capturedInputs = [];
  let capturedOutputs = [];
  
  beforeAll(() => {
    // Setup spies to capture behavior
    originalFunction = ${path.functionName};
    
    jest.spyOn(global, '${path.functionName}').mockImplementation((...args) => {
      capturedInputs.push(args);
      const result = originalFunction.apply(this, args);
      capturedOutputs.push(result);
      return result;
    });
  });
  
  describe('Happy Path', () => {
    it('should handle normal input', async () => {
      // Record actual behavior
      const input = ${JSON.stringify(path.sampleInput)};
      const result = await ${path.functionName}(input);
      
      // Document what it ACTUALLY does (not what it SHOULD do)
      expect(result).toMatchSnapshot();
      
      // Document side effects
      expect(database.queries).toMatchSnapshot();
      expect(externalAPICalls).toMatchSnapshot();
    });
  });
  
  describe('Edge Cases', () => {
    it('should handle null input', async () => {
      // Document actual behavior with null
      const result = await ${path.functionName}(null);
      expect(result).toMatchSnapshot();
    });
    
    it('should handle empty input', async () => {
      const result = await ${path.functionName}({});
      expect(result).toMatchSnapshot();
    });
    
    it('should handle invalid input', async () => {
      const result = await ${path.functionName}('invalid');
      expect(result).toMatchSnapshot();
    });
  });
  
  describe('Error Cases', () => {
    it('should handle database errors', async () => {
      database.mockError(new Error('Connection lost'));
      
      try {
        await ${path.functionName}(validInput);
      } catch (error) {
        // Document error handling
        expect(error).toMatchSnapshot();
      }
    });
  });
  
  afterAll(() => {
    // Generate report
    console.log('Captured behaviors:', {
      inputs: capturedInputs,
      outputs: capturedOutputs,
      patterns: analyzePatterns(capturedInputs, capturedOutputs)
    });
  });
});
    `;
    
    fs.writeFileSync(`tests/characterization/${path.name}.test.js`, testFile);
  }
  
  // ============================================
  // SEMANA 5-8: INCREMENTAL IMPROVEMENTS
  // ============================================
  
  /**
   * Mejoras incrementales sin romper funcionalidad
   */
  async week5_8_IncrementalImprovements(): Promise<void> {
    // 1. Agregar logging estructurado
    await this.addStructuredLogging();
    
    // 2. Agregar monitoring
    await this.addMonitoring();
    
    // 3. Crear abstracción para partes críticas
    await this.createAbstractionLayer();
    
    // 4. Extraer configuración
    await this.extractConfiguration();
    
    // 5. Mejorar manejo de errores
    await this.improveErrorHandling();
  }
  
  /**
   * Agregar logging sin cambiar lógica
   */
  private async addStructuredLogging(): Promise<void> {
    // Wrapper pattern para agregar logging
    const addLoggingWrapper = (originalFunction: Function, name: string) => {
      return function(...args: any[]) {
        const startTime = Date.now();
        const correlationId = generateCorrelationId();
        
        logger.info(`${name} started`, {
          correlationId,
          args: sanitizeArgs(args),
          timestamp: new Date()
        });
        
        try {
          const result = originalFunction.apply(this, args);
          
          // Handle promises
          if (result && typeof result.then === 'function') {
            return result
              .then((value: any) => {
                logger.info(`${name} completed`, {
                  correlationId,
                  duration: Date.now() - startTime,
                  success: true
                });
                return value;
              })
              .catch((error: Error) => {
                logger.error(`${name} failed`, {
                  correlationId,
                  duration: Date.now() - startTime,
                  error: error.message,
                  stack: error.stack
                });
                throw error;
              });
          }
          
          logger.info(`${name} completed`, {
            correlationId,
            duration: Date.now() - startTime,
            success: true
          });
          
          return result;
          
        } catch (error) {
          logger.error(`${name} failed`, {
            correlationId,
            duration: Date.now() - startTime,
            error: error.message,
            stack: error.stack
          });
          throw error;
        }
      };
    };
    
    // Aplicar a funciones críticas
    criticalFunctions.forEach(func => {
      module.exports[func.name] = addLoggingWrapper(func, func.name);
    });
  }
  
  // ============================================
  // SEMANA 9-12: REFACTORING STRATEGY
  // ============================================
  
  /**
   * Estrategia de refactoring a largo plazo
   */
  async week9_12_RefactoringStrategy(): Promise<RefactoringPlan> {
    return {
      phase1: {
        name: 'Strangler Fig Pattern',
        duration: '3 months',
        approach: `
          1. Crear nueva API alongside legacy
          2. Proxy requests gradualmente
          3. Migrar endpoint por endpoint
          4. Monitor ambos sistemas
          5. Deprecar legacy gradualmente
        `,
        implementation: this.implementStranglerFig()
      },
      
      phase2: {
        name: 'Extract Services',
        duration: '2 months',
        approach: `
          1. Identificar bounded contexts
          2. Extraer servicios independientes
          3. Implementar API Gateway
          4. Migrar datos gradualmente
        `,
        implementation: this.extractServices()
      },
      
      phase3: {
        name: 'Modernize Stack',
        duration: '2 months',
        approach: `
          1. Actualizar dependencias
          2. Migrar a TypeScript
          3. Implementar CI/CD moderno
          4. Containerizar aplicaciones
        `,
        implementation: this.modernizeStack()
      }
    };
  }
  
  /**
   * Implementación del Strangler Fig Pattern
   */
  private implementStranglerFig(): StranglerFigImplementation {
    // 1. Crear proxy router
    const proxyRouter = `
class LegacyProxyRouter {
  private featureFlags: FeatureFlags;
  private legacyApp: Express;
  private modernApp: Express;
  
  constructor() {
    this.featureFlags = new FeatureFlags();
    this.legacyApp = require('./legacy/app');
    this.modernApp = require('./modern/app');
  }
  
  route(req: Request, res: Response): void {
    const endpoint = req.path;
    
    // Check feature flag for this endpoint
    if (this.featureFlags.isModernEnabled(endpoint)) {
      // Log para monitoring
      logger.info('Routing to modern app', { endpoint });
      
      // Route to modern implementation
      this.modernApp(req, res);
      
    } else {
      // Route to legacy
      logger.info('Routing to legacy app', { endpoint });
      this.legacyApp(req, res);
    }
  }
  
  // Gradual migration
  async migrateEndpoint(endpoint: string): Promise<void> {
    // 1. Implement in modern app
    await this.implementModernEndpoint(endpoint);
    
    // 2. Test thoroughly
    await this.runComparisonTests(endpoint);
    
    // 3. Enable for small percentage
    await this.featureFlags.enablePercentage(endpoint, 5);
    
    // 4. Monitor metrics
    await this.monitorEndpoint(endpoint);
    
    // 5. Gradually increase percentage
    for (const percentage of [10, 25, 50, 75, 100]) {
      await this.waitForStableMetrics(endpoint);
      await this.featureFlags.enablePercentage(endpoint, percentage);
    }
  }
}
    `;
    
    // 2. Comparison testing
    const comparisonTesting = `
class ComparisonTester {
  async compareImplementations(endpoint: string, testCases: TestCase[]): Promise<ComparisonResult> {
    const results = [];
    
    for (const testCase of testCases) {
      const legacyResult = await this.callLegacy(endpoint, testCase);
      const modernResult = await this.callModern(endpoint, testCase);
      
      const comparison = {
        testCase,
        legacy: legacyResult,
        modern: modernResult,
        match: this.compareResults(legacyResult, modernResult),
        differences: this.findDifferences(legacyResult, modernResult)
      };
      
      results.push(comparison);
    }
    
    return {
      endpoint,
      totalTests: testCases.length,
      matching: results.filter(r => r.match).length,
      differences: results.filter(r => !r.match),
      recommendation: this.generateRecommendation(results)
    };
  }
}
    `;
    
    return { proxyRouter, comparisonTesting };
  }
  
  // ============================================
  // MÉTRICAS Y MONITOREO
  // ============================================
  
  /**
   * Dashboard de progreso
   */
  createProgressDashboard(): ProgressDashboard {
    return {
      metrics: [
        {
          name: 'Test Coverage',
          current: 0,
          target: 80,
          trend: 'increasing'
        },
        {
          name: 'Technical Debt (hours)',
          current: 500,
          target: 100,
          trend: 'decreasing'
        },
        {
          name: 'Critical Bugs',
          current: 15,
          target: 0,
          trend: 'decreasing'
        },
        {
          name: 'Documentation Coverage',
          current: 10,
          target: 90,
          trend: 'increasing'
        },
        {
          name: 'Endpoints Migrated',
          current: 0,
          target: 45,
          trend: 'increasing'
        }
      ],
      
      milestones: [
        {
          week: 2,
          goal: 'Complete system analysis',
          status: 'completed'
        },
        {
          week: 4,
          goal: 'Characterization tests for critical paths',
          status: 'in-progress'
        },
        {
          week: 8,
          goal: 'Monitoring and alerting in place',
          status: 'pending'
        },
        {
          week: 12,
          goal: 'First service extracted',
          status: 'pending'
        }
      ],
      
      risks: [
        {
          risk: 'Data corruption during migration',
          mitigation: 'Dual writes with validation',
          severity: 'high'
        },
        {
          risk: 'Performance degradation',
          mitigation: 'Load testing and gradual rollout',
          severity: 'medium'
        },
        {
          risk: 'Knowledge gaps',
          mitigation: 'Pair programming and documentation',
          severity: 'medium'
        }
      ]
    };
  }
}
```

---

### Pregunta 7: "¿Cómo manejarías un code review donde no estás de acuerdo con el approach?"

**Respuesta Profesional:**

Manejaría esta situación con profesionalismo y enfoque en el mejor resultado para el equipo:

1. **Primero entender**: Asumo buenas intenciones y busco entender el razonamiento
2. **Hacer preguntas, no acusaciones**: "¿Consideraste X?" en lugar de "Esto está mal"
3. **Proporcionar alternativas con datos**: Mostrar pros/contras objetivamente
4. **Estar abierto a estar equivocado**: Mi approach puede no ser el mejor
5. **Escalar constructivamente si es necesario**: Involucrar al team lead para decisiones arquitectónicas

```typescript
// Ejemplo de feedback constructivo en code review:

/*
 * Original PR comment (mal approach):
 * "This is wrong. You should never use any type."
 * 
 * Mejor approach:
 */

const constructiveReview = {
  comment: `
    I notice we're using 'any' type here. While this works, I'm concerned about:
    
    1. We lose TypeScript's type safety benefits
    2. Potential runtime errors that could be caught at compile time
    3. Makes refactoring harder in the future
    
    What do you think about using a more specific type? For example:
    
    \`\`\`typescript
    interface UserData {
      id: string;
      email: string;
      metadata?: Record<string, unknown>;
    }
    \`\`\`
    
    This would give us type safety while still being flexible.
    
    If there's a specific reason for using 'any' that I'm missing, 
    I'd love to understand the context better!
  `,
  
  // Si hay desacuerdo fundamental
  escalation: `
    I see we have different perspectives on this approach. 
    Both have merit:
    
    Your approach:
    ✅ Simpler implementation
    ✅ Faster to implement
    ❓ May have performance implications at scale
    
    Alternative approach:
    ✅ Better performance characteristics
    ✅ More maintainable long-term
    ❓ More complex initial implementation
    
    Since this affects our architecture, should we:
    1. Quick sync to discuss trade-offs?
    2. Create a small POC to compare both approaches?
    3. Get input from the team in our next tech discussion?
    
    What would you prefer?
  `
};
```

La clave es mantener el foco en el código, no en las personas, y buscar el mejor resultado para el proyecto.

---

## 10.3 Preparación Final

### Consejos para la Entrevista:

1. **Estructura tus respuestas**: Situación → Acción → Resultado
2. **Usa ejemplos reales**: Menciona proyectos específicos cuando sea posible
3. **Muestra pensamiento sistemático**: No solo soluciones puntuales
4. **Demuestra liderazgo técnico**: Cómo influencias mejores prácticas
5. **Sé honesto sobre trade-offs**: No hay soluciones perfectas
6. **Pregunta cuando no sepas**: Muestra curiosidad y humildad

### Preguntas para hacer al entrevistador:

1. "¿Cuáles son los principales desafíos de mantenimiento que enfrenta el equipo?"
2. "¿Qué métricas usan para medir la calidad del código?"
3. "¿Cómo manejan la deuda técnica en los sprints?"
4. "¿Qué herramientas y procesos tienen para code review?"
5. "¿Cuál es la estrategia para modernizar sistemas legacy?"

### Preparación técnica final:

- Repasa patrones de refactoring comunes
- Practica identificar code smells rápidamente
- Ten ejemplos de mejoras de performance listas
- Prepara historias de debugging complejos
- Conoce herramientas modernas de quality assurance

¡Éxito en tu entrevista técnica!