# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 8: Debugging y Troubleshooting
### Para Entrevista Senior Software Engineer - Zillow

---

## 8. Debugging y Troubleshooting - Resolviendo Problemas Sistemáticamente

El debugging efectivo es lo que separa a los desarrolladores junior de los senior. No es solo sobre encontrar bugs, sino sobre entender sistemas complejos, diagnosticar problemas sistemáticamente, y prevenir futuros issues.

### Principios del Debugging Efectivo

1. **Reproducibilidad**: Si no puedes reproducirlo, no puedes arreglarlo
2. **Aislamiento**: Reduce el problema al caso más simple
3. **Hipótesis y verificación**: Formula teorías y pruébalas sistemáticamente
4. **Documentación**: Registra lo que intentas y los resultados
5. **Root cause analysis**: No solo arregles síntomas, encuentra la causa raíz

### Tipos de Bugs Comunes

- **Logic errors**: El código hace algo diferente a lo esperado
- **Race conditions**: Problemas de concurrencia y timing
- **Memory leaks**: Recursos que no se liberan
- **Performance issues**: Código que funciona pero es muy lento
- **Integration bugs**: Problemas entre componentes
- **Environment bugs**: Funciona en desarrollo pero no en producción

### Herramientas de Debugging

- **Debuggers**: Chrome DevTools, VS Code Debugger
- **Profilers**: Para analizar performance
- **Memory analyzers**: Para detectar memory leaks
- **Network tools**: Para debugging de APIs
- **Logging**: Estructurado y con niveles apropiados
- **Monitoring**: APM tools, error tracking

---

## 8.1 Técnicas de Debugging

### Explicación Detallada

El debugging sistemático es una habilidad que se desarrolla con experiencia. Las técnicas correctas pueden reducir el tiempo de resolución de horas a minutos.

#### Metodología de Debugging:

1. **Entender el problema**: ¿Qué debería pasar? ¿Qué está pasando?
2. **Reproducir consistentemente**: Crear un caso de prueba mínimo
3. **Aislar el problema**: Usar binary search, comentar código, etc.
4. **Formar hipótesis**: Basadas en evidencia, no suposiciones
5. **Probar hipótesis**: Una a la vez, documentando resultados
6. **Implementar fix**: Con tests para prevenir regresión
7. **Verificar**: Confirmar que el fix funciona y no rompe nada más

#### Técnicas Específicas:

- **Printf debugging**: Logging estratégico
- **Binary search**: Dividir el problema por la mitad
- **Rubber duck debugging**: Explicar el problema en voz alta
- **Git bisect**: Encontrar el commit que introdujo el bug
- **Delta debugging**: Reducir el caso de prueba automáticamente

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Debugging Sistemático con Logging
// ============================================

// Sistema de logging estructurado para debugging
enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3,
  NONE = 4
}

class DebugLogger {
  private static instance: DebugLogger;
  private logLevel: LogLevel = LogLevel.INFO;
  private contexts: Set<string> = new Set();
  private performanceMarks: Map<string, number> = new Map();
  
  static getInstance(): DebugLogger {
    if (!this.instance) {
      this.instance = new DebugLogger();
    }
    return this.instance;
  }
  
  setLogLevel(level: LogLevel): void {
    this.logLevel = level;
  }
  
  enableContext(context: string): void {
    this.contexts.add(context);
  }
  
  disableContext(context: string): void {
    this.contexts.delete(context);
  }
  
  // Logging con contexto para filtrar mensajes
  debug(context: string, message: string, data?: any): void {
    if (this.shouldLog(LogLevel.DEBUG, context)) {
      console.log(this.formatMessage('DEBUG', context, message), data || '');
      this.saveToBuffer('DEBUG', context, message, data);
    }
  }
  
  info(context: string, message: string, data?: any): void {
    if (this.shouldLog(LogLevel.INFO, context)) {
      console.info(this.formatMessage('INFO', context, message), data || '');
      this.saveToBuffer('INFO', context, message, data);
    }
  }
  
  warn(context: string, message: string, data?: any): void {
    if (this.shouldLog(LogLevel.WARN, context)) {
      console.warn(this.formatMessage('WARN', context, message), data || '');
      this.saveToBuffer('WARN', context, message, data);
    }
  }
  
  error(context: string, message: string, error?: Error): void {
    if (this.shouldLog(LogLevel.ERROR, context)) {
      console.error(this.formatMessage('ERROR', context, message));
      if (error) {
        console.error('Stack trace:', error.stack);
        this.saveToBuffer('ERROR', context, message, {
          message: error.message,
          stack: error.stack
        });
      }
    }
  }
  
  // Performance debugging
  startTimer(label: string): void {
    this.performanceMarks.set(label, performance.now());
    this.debug('PERF', `Timer started: ${label}`);
  }
  
  endTimer(label: string): number {
    const start = this.performanceMarks.get(label);
    if (!start) {
      this.warn('PERF', `Timer not found: ${label}`);
      return -1;
    }
    
    const duration = performance.now() - start;
    this.performanceMarks.delete(label);
    
    const level = duration > 1000 ? LogLevel.WARN : LogLevel.DEBUG;
    const message = `Timer ${label}: ${duration.toFixed(2)}ms`;
    
    if (this.shouldLog(level, 'PERF')) {
      console.log(this.formatMessage(level === LogLevel.WARN ? 'WARN' : 'DEBUG', 'PERF', message));
    }
    
    return duration;
  }
  
  // Trace de ejecución
  trace(context: string, functionName: string, args?: any): void {
    if (this.shouldLog(LogLevel.DEBUG, context)) {
      const trace = new Error().stack?.split('\n').slice(2, 5).join('\n');
      console.log(this.formatMessage('TRACE', context, `${functionName} called`));
      if (args) console.log('Arguments:', args);
      console.log('Call stack:', trace);
    }
  }
  
  // Assertions para debugging
  assert(condition: boolean, message: string): void {
    if (!condition) {
      const error = new Error(`Assertion failed: ${message}`);
      this.error('ASSERT', message, error);
      if (process.env.NODE_ENV === 'development') {
        debugger; // Breakpoint automático en desarrollo
      }
      throw error;
    }
  }
  
  // Checkpoint debugging
  checkpoint(label: string, data?: any): void {
    this.debug('CHECKPOINT', `Reached: ${label}`, data);
  }
  
  private shouldLog(level: LogLevel, context: string): boolean {
    // Si el contexto está deshabilitado, no loguear
    if (this.contexts.size > 0 && !this.contexts.has(context) && context !== 'ERROR') {
      return false;
    }
    
    return level >= this.logLevel;
  }
  
  private formatMessage(level: string, context: string, message: string): string {
    const timestamp = new Date().toISOString();
    return `[${timestamp}] [${level}] [${context}] ${message}`;
  }
  
  // Buffer circular para mantener logs recientes
  private logBuffer: Array<{
    timestamp: Date;
    level: string;
    context: string;
    message: string;
    data?: any;
  }> = [];
  
  private readonly MAX_BUFFER_SIZE = 1000;
  
  private saveToBuffer(level: string, context: string, message: string, data?: any): void {
    this.logBuffer.push({
      timestamp: new Date(),
      level,
      context,
      message,
      data
    });
    
    if (this.logBuffer.length > this.MAX_BUFFER_SIZE) {
      this.logBuffer.shift();
    }
  }
  
  // Dump logs para debugging post-mortem
  dumpLogs(filter?: { level?: string; context?: string; since?: Date }): void {
    console.log('=== LOG DUMP START ===');
    
    let logs = this.logBuffer;
    
    if (filter) {
      if (filter.level) {
        logs = logs.filter(log => log.level === filter.level);
      }
      if (filter.context) {
        logs = logs.filter(log => log.context === filter.context);
      }
      if (filter.since) {
        logs = logs.filter(log => log.timestamp >= filter.since);
      }
    }
    
    logs.forEach(log => {
      console.log(`[${log.timestamp.toISOString()}] [${log.level}] [${log.context}] ${log.message}`);
      if (log.data) {
        console.log('  Data:', log.data);
      }
    });
    
    console.log('=== LOG DUMP END ===');
    console.log(`Total logs: ${logs.length}`);
  }
}

// ============================================
// EJEMPLO 2: Debugging de Race Conditions
// ============================================

// Detector de race conditions
class RaceConditionDetector {
  private operations = new Map<string, {
    startTime: number;
    endTime?: number;
    concurrent: string[];
  }>();
  
  private potentialRaces: Array<{
    operation1: string;
    operation2: string;
    overlap: number;
  }> = [];
  
  // Marcar inicio de operación
  startOperation(id: string): void {
    const now = performance.now();
    
    // Detectar operaciones concurrentes
    const concurrent: string[] = [];
    for (const [opId, op] of this.operations) {
      if (!op.endTime) {
        concurrent.push(opId);
      }
    }
    
    this.operations.set(id, {
      startTime: now,
      concurrent
    });
    
    if (concurrent.length > 0) {
      console.warn(`⚠️ Operation ${id} started while ${concurrent.join(', ')} still running`);
    }
  }
  
  // Marcar fin de operación
  endOperation(id: string): void {
    const op = this.operations.get(id);
    if (!op) {
      console.error(`Operation ${id} not found`);
      return;
    }
    
    op.endTime = performance.now();
    
    // Detectar posibles race conditions
    for (const concurrentId of op.concurrent) {
      const concurrentOp = this.operations.get(concurrentId);
      if (concurrentOp && concurrentOp.endTime) {
        const overlap = Math.min(op.endTime, concurrentOp.endTime) - 
                       Math.max(op.startTime, concurrentOp.startTime);
        
        if (overlap > 0) {
          this.potentialRaces.push({
            operation1: id,
            operation2: concurrentId,
            overlap
          });
          
          console.warn(`🔄 Potential race condition detected:
            ${id} and ${concurrentId} overlapped for ${overlap.toFixed(2)}ms`);
        }
      }
    }
  }
  
  // Reporte de análisis
  generateReport(): RaceConditionReport {
    const report: RaceConditionReport = {
      totalOperations: this.operations.size,
      concurrentOperations: 0,
      potentialRaces: this.potentialRaces,
      timeline: []
    };
    
    // Contar operaciones concurrentes
    for (const op of this.operations.values()) {
      if (op.concurrent.length > 0) {
        report.concurrentOperations++;
      }
    }
    
    // Generar timeline
    const events: TimelineEvent[] = [];
    for (const [id, op] of this.operations) {
      events.push({ time: op.startTime, type: 'start', operation: id });
      if (op.endTime) {
        events.push({ time: op.endTime, type: 'end', operation: id });
      }
    }
    
    report.timeline = events.sort((a, b) => a.time - b.time);
    
    return report;
  }
  
  // Limpiar datos
  reset(): void {
    this.operations.clear();
    this.potentialRaces = [];
  }
}

// Wrapper para detectar race conditions automáticamente
function detectRaces<T extends (...args: any[]) => Promise<any>>(
  fn: T,
  operationId: string
): T {
  const detector = raceDetector;
  
  return (async (...args: Parameters<T>): Promise<ReturnType<T>> => {
    const id = `${operationId}-${Date.now()}`;
    detector.startOperation(id);
    
    try {
      const result = await fn(...args);
      return result;
    } finally {
      detector.endOperation(id);
    }
  }) as T;
}

// ============================================
// EJEMPLO 3: Memory Leak Detection
// ============================================

// Detector de memory leaks
class MemoryLeakDetector {
  private trackedObjects = new WeakMap<object, {
    type: string;
    createdAt: Date;
    stack: string;
  }>();
  
  private leakCandidates = new Map<string, {
    count: number;
    instances: WeakRef<object>[];
    lastSeen: Date;
  }>();
  
  private registry: FinalizationRegistry<string>;
  
  constructor() {
    // Registry para detectar cuando objetos son garbage collected
    this.registry = new FinalizationRegistry((type: string) => {
      this.onObjectCollected(type);
    });
  }
  
  // Rastrear objeto
  track(obj: object, type: string): void {
    const stack = new Error().stack || '';
    
    this.trackedObjects.set(obj, {
      type,
      createdAt: new Date(),
      stack
    });
    
    // Registrar para detección de GC
    this.registry.register(obj, type);
    
    // Actualizar candidatos a leak
    if (!this.leakCandidates.has(type)) {
      this.leakCandidates.set(type, {
        count: 0,
        instances: [],
        lastSeen: new Date()
      });
    }
    
    const candidate = this.leakCandidates.get(type)!;
    candidate.count++;
    candidate.instances.push(new WeakRef(obj));
    candidate.lastSeen = new Date();
    
    // Limpiar referencias muertas
    candidate.instances = candidate.instances.filter(ref => ref.deref() !== undefined);
    
    // Advertir si hay muchas instancias
    if (candidate.count > 100) {
      console.warn(`⚠️ Potential memory leak: ${candidate.count} instances of ${type}`);
    }
  }
  
  // Cuando un objeto es recolectado
  private onObjectCollected(type: string): void {
    const candidate = this.leakCandidates.get(type);
    if (candidate) {
      candidate.count--;
      
      if (candidate.count <= 0) {
        this.leakCandidates.delete(type);
      }
    }
  }
  
  // Analizar posibles leaks
  analyze(): MemoryLeakReport {
    const report: MemoryLeakReport = {
      potentialLeaks: [],
      totalTracked: 0,
      memoryUsage: this.getMemoryUsage()
    };
    
    for (const [type, candidate] of this.leakCandidates) {
      const aliveInstances = candidate.instances.filter(ref => ref.deref() !== undefined);
      
      if (aliveInstances.length > 10) {
        report.potentialLeaks.push({
          type,
          count: aliveInstances.length,
          oldestAge: this.getOldestAge(aliveInstances),
          lastSeen: candidate.lastSeen
        });
      }
      
      report.totalTracked += aliveInstances.length;
    }
    
    return report;
  }
  
  private getOldestAge(instances: WeakRef<object>[]): number {
    let oldest = 0;
    
    for (const ref of instances) {
      const obj = ref.deref();
      if (obj) {
        const info = this.trackedObjects.get(obj);
        if (info) {
          const age = Date.now() - info.createdAt.getTime();
          oldest = Math.max(oldest, age);
        }
      }
    }
    
    return oldest;
  }
  
  private getMemoryUsage(): MemoryUsage {
    if (typeof process !== 'undefined' && process.memoryUsage) {
      const usage = process.memoryUsage();
      return {
        heapUsed: usage.heapUsed,
        heapTotal: usage.heapTotal,
        external: usage.external
      };
    }
    
    // Fallback para browser
    if (typeof performance !== 'undefined' && (performance as any).memory) {
      const memory = (performance as any).memory;
      return {
        heapUsed: memory.usedJSHeapSize,
        heapTotal: memory.totalJSHeapSize,
        external: 0
      };
    }
    
    return { heapUsed: 0, heapTotal: 0, external: 0 };
  }
  
  // Forzar garbage collection (solo en desarrollo)
  forceGC(): void {
    if (typeof global !== 'undefined' && (global as any).gc) {
      console.log('Forcing garbage collection...');
      (global as any).gc();
    } else {
      console.warn('Garbage collection not available. Run with --expose-gc flag');
    }
  }
}

// ============================================
// EJEMPLO 4: Performance Debugging
// ============================================

// Profiler para debugging de performance
class PerformanceProfiler {
  private measurements: Map<string, PerformanceMeasurement[]> = new Map();
  private marks: Map<string, number> = new Map();
  private enabled: boolean = true;
  
  enable(): void {
    this.enabled = true;
  }
  
  disable(): void {
    this.enabled = false;
  }
  
  // Marcar inicio de operación
  mark(name: string): void {
    if (!this.enabled) return;
    
    this.marks.set(name, performance.now());
  }
  
  // Medir desde marca
  measure(name: string, startMark: string, endMark?: string): void {
    if (!this.enabled) return;
    
    const start = this.marks.get(startMark);
    if (!start) {
      console.error(`Start mark '${startMark}' not found`);
      return;
    }
    
    const end = endMark ? this.marks.get(endMark) : performance.now();
    if (!end) {
      console.error(`End mark '${endMark}' not found`);
      return;
    }
    
    const duration = end - start;
    
    if (!this.measurements.has(name)) {
      this.measurements.set(name, []);
    }
    
    this.measurements.get(name)!.push({
      duration,
      timestamp: new Date(),
      start,
      end: end as number
    });
    
    // Advertir si es muy lento
    if (duration > 100) {
      console.warn(`⚠️ Slow operation '${name}': ${duration.toFixed(2)}ms`);
    }
  }
  
  // Decorador para medir funciones automáticamente
  measureFunction<T extends (...args: any[]) => any>(
    fn: T,
    name?: string
  ): T {
    const fnName = name || fn.name || 'anonymous';
    const profiler = this;
    
    return function(this: any, ...args: Parameters<T>): ReturnType<T> {
      if (!profiler.enabled) {
        return fn.apply(this, args);
      }
      
      const startMark = `${fnName}-start-${Date.now()}`;
      profiler.mark(startMark);
      
      try {
        const result = fn.apply(this, args);
        
        // Handle async functions
        if (result instanceof Promise) {
          return result.finally(() => {
            profiler.measure(fnName, startMark);
          }) as ReturnType<T>;
        }
        
        profiler.measure(fnName, startMark);
        return result;
      } catch (error) {
        profiler.measure(fnName, startMark);
        throw error;
      }
    } as T;
  }
  
  // Obtener estadísticas
  getStats(name?: string): PerformanceStats | Map<string, PerformanceStats> {
    if (name) {
      const measurements = this.measurements.get(name);
      if (!measurements) {
        return {
          count: 0,
          total: 0,
          average: 0,
          min: 0,
          max: 0,
          median: 0,
          p95: 0,
          p99: 0
        };
      }
      
      return this.calculateStats(measurements);
    }
    
    // Retornar todas las estadísticas
    const allStats = new Map<string, PerformanceStats>();
    
    for (const [key, measurements] of this.measurements) {
      allStats.set(key, this.calculateStats(measurements));
    }
    
    return allStats;
  }
  
  private calculateStats(measurements: PerformanceMeasurement[]): PerformanceStats {
    if (measurements.length === 0) {
      return {
        count: 0,
        total: 0,
        average: 0,
        min: 0,
        max: 0,
        median: 0,
        p95: 0,
        p99: 0
      };
    }
    
    const durations = measurements.map(m => m.duration).sort((a, b) => a - b);
    const total = durations.reduce((sum, d) => sum + d, 0);
    
    return {
      count: durations.length,
      total,
      average: total / durations.length,
      min: durations[0],
      max: durations[durations.length - 1],
      median: this.percentile(durations, 50),
      p95: this.percentile(durations, 95),
      p99: this.percentile(durations, 99)
    };
  }
  
  private percentile(sorted: number[], p: number): number {
    const index = (sorted.length - 1) * (p / 100);
    const lower = Math.floor(index);
    const upper = Math.ceil(index);
    const weight = index - lower;
    
    return sorted[lower] * (1 - weight) + sorted[upper] * weight;
  }
  
  // Generar reporte
  generateReport(): string {
    const stats = this.getStats() as Map<string, PerformanceStats>;
    let report = '=== Performance Report ===\n\n';
    
    // Ordenar por tiempo total descendente
    const sorted = Array.from(stats.entries()).sort((a, b) => b[1].total - a[1].total);
    
    for (const [name, stat] of sorted) {
      report += `${name}:\n`;
      report += `  Count: ${stat.count}\n`;
      report += `  Total: ${stat.total.toFixed(2)}ms\n`;
      report += `  Average: ${stat.average.toFixed(2)}ms\n`;
      report += `  Min: ${stat.min.toFixed(2)}ms\n`;
      report += `  Max: ${stat.max.toFixed(2)}ms\n`;
      report += `  Median: ${stat.median.toFixed(2)}ms\n`;
      report += `  P95: ${stat.p95.toFixed(2)}ms\n`;
      report += `  P99: ${stat.p99.toFixed(2)}ms\n\n`;
    }
    
    return report;
  }
  
  // Limpiar datos
  reset(): void {
    this.measurements.clear();
    this.marks.clear();
  }
}

// ============================================
// EJEMPLO 5: Error Boundary y Recovery
// ============================================

// Sistema de manejo de errores con debugging información
class ErrorBoundary {
  private errorHandlers = new Map<string, ErrorHandler>();
  private errorHistory: ErrorRecord[] = [];
  private readonly MAX_HISTORY = 100;
  
  // Registrar handler para tipo de error
  registerHandler(errorType: string, handler: ErrorHandler): void {
    this.errorHandlers.set(errorType, handler);
  }
  
  // Capturar y manejar error
  async handle(error: Error, context?: any): Promise<ErrorResolution> {
    const errorRecord = this.createErrorRecord(error, context);
    this.errorHistory.push(errorRecord);
    
    // Mantener historial limitado
    if (this.errorHistory.length > this.MAX_HISTORY) {
      this.errorHistory.shift();
    }
    
    // Log detallado para debugging
    this.logError(errorRecord);
    
    // Buscar handler apropiado
    const handler = this.findHandler(error);
    
    if (handler) {
      try {
        const resolution = await handler(error, context);
        errorRecord.resolution = resolution;
        return resolution;
      } catch (handlerError) {
        console.error('Error handler failed:', handlerError);
        errorRecord.handlerFailed = true;
      }
    }
    
    // Fallback resolution
    return {
      handled: false,
      action: 'log',
      message: 'Unhandled error occurred'
    };
  }
  
  private createErrorRecord(error: Error, context?: any): ErrorRecord {
    return {
      timestamp: new Date(),
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      },
      context: context ? this.sanitizeContext(context) : undefined,
      environment: this.captureEnvironment(),
      breadcrumbs: this.getBreadcrumbs()
    };
  }
  
  private sanitizeContext(context: any): any {
    // Remover información sensible
    const sanitized = { ...context };
    
    const sensitiveKeys = ['password', 'token', 'apiKey', 'secret'];
    
    const sanitizeObject = (obj: any): any => {
      if (typeof obj !== 'object' || obj === null) return obj;
      
      const result: any = Array.isArray(obj) ? [] : {};
      
      for (const key in obj) {
        if (sensitiveKeys.includes(key.toLowerCase())) {
          result[key] = '[REDACTED]';
        } else if (typeof obj[key] === 'object') {
          result[key] = sanitizeObject(obj[key]);
        } else {
          result[key] = obj[key];
        }
      }
      
      return result;
    };
    
    return sanitizeObject(sanitized);
  }
  
  private captureEnvironment(): Environment {
    return {
      userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : 'N/A',
      platform: typeof navigator !== 'undefined' ? navigator.platform : process.platform,
      nodeVersion: typeof process !== 'undefined' ? process.version : 'N/A',
      timestamp: new Date().toISOString(),
      memory: this.getMemoryInfo()
    };
  }
  
  private getMemoryInfo(): any {
    if (typeof process !== 'undefined' && process.memoryUsage) {
      return process.memoryUsage();
    }
    
    if (typeof performance !== 'undefined' && (performance as any).memory) {
      return (performance as any).memory;
    }
    
    return null;
  }
  
  private breadcrumbs: Breadcrumb[] = [];
  
  addBreadcrumb(breadcrumb: Breadcrumb): void {
    this.breadcrumbs.push({
      ...breadcrumb,
      timestamp: new Date()
    });
    
    // Mantener solo los últimos 50 breadcrumbs
    if (this.breadcrumbs.length > 50) {
      this.breadcrumbs.shift();
    }
  }
  
  private getBreadcrumbs(): Breadcrumb[] {
    return [...this.breadcrumbs];
  }
  
  private findHandler(error: Error): ErrorHandler | undefined {
    // Buscar handler específico por nombre de error
    if (this.errorHandlers.has(error.name)) {
      return this.errorHandlers.get(error.name);
    }
    
    // Buscar handler por tipo de error
    for (const [type, handler] of this.errorHandlers) {
      if (error.constructor.name === type) {
        return handler;
      }
    }
    
    // Handler genérico
    return this.errorHandlers.get('*');
  }
  
  private logError(record: ErrorRecord): void {
    console.group(`🔴 Error: ${record.error.name}`);
    console.error('Message:', record.error.message);
    console.error('Stack:', record.error.stack);
    
    if (record.context) {
      console.log('Context:', record.context);
    }
    
    console.log('Environment:', record.environment);
    
    if (record.breadcrumbs && record.breadcrumbs.length > 0) {
      console.log('Breadcrumbs:');
      record.breadcrumbs.forEach(b => {
        console.log(`  ${b.timestamp?.toISOString()} - ${b.type}: ${b.message}`);
      });
    }
    
    console.groupEnd();
  }
  
  // Obtener historial de errores para análisis
  getErrorHistory(filter?: { since?: Date; errorType?: string }): ErrorRecord[] {
    let history = [...this.errorHistory];
    
    if (filter) {
      if (filter.since) {
        history = history.filter(r => r.timestamp >= filter.since!);
      }
      if (filter.errorType) {
        history = history.filter(r => r.error.name === filter.errorType);
      }
    }
    
    return history;
  }
  
  // Análisis de patrones de error
  analyzeErrors(): ErrorAnalysis {
    const analysis: ErrorAnalysis = {
      totalErrors: this.errorHistory.length,
      errorTypes: new Map(),
      errorRate: this.calculateErrorRate(),
      mostCommon: null,
      recentTrend: this.getRecentTrend()
    };
    
    // Contar por tipo
    for (const record of this.errorHistory) {
      const count = analysis.errorTypes.get(record.error.name) || 0;
      analysis.errorTypes.set(record.error.name, count + 1);
    }
    
    // Encontrar el más común
    let maxCount = 0;
    for (const [type, count] of analysis.errorTypes) {
      if (count > maxCount) {
        maxCount = count;
        analysis.mostCommon = type;
      }
    }
    
    return analysis;
  }
  
  private calculateErrorRate(): number {
    if (this.errorHistory.length === 0) return 0;
    
    const now = Date.now();
    const oneHourAgo = now - 3600000;
    
    const recentErrors = this.errorHistory.filter(
      r => r.timestamp.getTime() > oneHourAgo
    );
    
    return recentErrors.length; // Errores por hora
  }
  
  private getRecentTrend(): 'increasing' | 'stable' | 'decreasing' {
    if (this.errorHistory.length < 10) return 'stable';
    
    const recent = this.errorHistory.slice(-10);
    const older = this.errorHistory.slice(-20, -10);
    
    const recentRate = recent.length;
    const olderRate = older.length;
    
    if (recentRate > olderRate * 1.5) return 'increasing';
    if (recentRate < olderRate * 0.5) return 'decreasing';
    return 'stable';
  }
}

// Tipos auxiliares
interface RaceConditionReport {
  totalOperations: number;
  concurrentOperations: number;
  potentialRaces: Array<{
    operation1: string;
    operation2: string;
    overlap: number;
  }>;
  timeline: TimelineEvent[];
}

interface TimelineEvent {
  time: number;
  type: 'start' | 'end';
  operation: string;
}

interface MemoryLeakReport {
  potentialLeaks: Array<{
    type: string;
    count: number;
    oldestAge: number;
    lastSeen: Date;
  }>;
  totalTracked: number;
  memoryUsage: MemoryUsage;
}

interface MemoryUsage {
  heapUsed: number;
  heapTotal: number;
  external: number;
}

interface PerformanceMeasurement {
  duration: number;
  timestamp: Date;
  start: number;
  end: number;
}

interface PerformanceStats {
  count: number;
  total: number;
  average: number;
  min: number;
  max: number;
  median: number;
  p95: number;
  p99: number;
}

interface ErrorRecord {
  timestamp: Date;
  error: {
    name: string;
    message: string;
    stack?: string;
  };
  context?: any;
  environment: Environment;
  breadcrumbs?: Breadcrumb[];
  resolution?: ErrorResolution;
  handlerFailed?: boolean;
}

interface Environment {
  userAgent: string;
  platform: string;
  nodeVersion: string;
  timestamp: string;
  memory: any;
}

interface Breadcrumb {
  type: string;
  message: string;
  data?: any;
  timestamp?: Date;
}

interface ErrorResolution {
  handled: boolean;
  action: 'retry' | 'fallback' | 'log' | 'ignore';
  message?: string;
  fallbackValue?: any;
}

interface ErrorAnalysis {
  totalErrors: number;
  errorTypes: Map<string, number>;
  errorRate: number;
  mostCommon: string | null;
  recentTrend: 'increasing' | 'stable' | 'decreasing';
}

type ErrorHandler = (error: Error, context?: any) => Promise<ErrorResolution> | ErrorResolution;

// Instancias globales para los ejemplos
const logger = DebugLogger.getInstance();
const raceDetector = new RaceConditionDetector();
const memoryDetector = new MemoryLeakDetector();
const profiler = new PerformanceProfiler();
const errorBoundary = new ErrorBoundary();

// Configuración de debugging para desarrollo
if (process.env.NODE_ENV === 'development') {
  logger.setLogLevel(LogLevel.DEBUG);
  logger.enableContext('API');
  logger.enableContext('DATABASE');
  logger.enableContext('CACHE');
  
  // Registrar handlers de error
  errorBoundary.registerHandler('NetworkError', async (error, context) => {
    logger.error('NETWORK', 'Network error occurred', error);
    return {
      handled: true,
      action: 'retry',
      message: 'Retrying network request...'
    };
  });
  
  errorBoundary.registerHandler('ValidationError', async (error, context) => {
    logger.warn('VALIDATION', 'Validation failed', context);
    return {
      handled: true,
      action: 'log',
      message: 'Validation error logged'
    };
  });
  
  // Handler genérico
  errorBoundary.registerHandler('*', async (error, context) => {
    logger.error('UNHANDLED', 'Unhandled error', error);
    return {
      handled: false,
      action: 'log',
      message: 'Error logged for investigation'
    };
  });
}
```

### Mejores prácticas de debugging:

1. **Logging estructurado**: Usa contextos y niveles apropiados
2. **Reproducibilidad**: Crea casos de prueba mínimos
3. **Bisección**: Divide el problema por la mitad iterativamente
4. **Instrumentación**: Agrega logging temporal para entender el flujo
5. **Version control**: Usa git bisect para encontrar regresiones
6. **Monitoring**: Implementa alertas para detectar problemas temprano
7. **Post-mortem**: Documenta la causa raíz y cómo prevenirla

El debugging efectivo es una combinación de herramientas, técnicas y experiencia. La clave es ser sistemático y no hacer suposiciones.