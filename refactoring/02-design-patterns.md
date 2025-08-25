# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 2: Design Patterns Esenciales
### Para Entrevista Senior Software Engineer

---

## 2. Design Patterns - Patrones de Diseño Fundamentales

Los patrones de diseño son soluciones probadas y reutilizables a problemas comunes en el desarrollo de software. Fueron popularizados por el libro "Design Patterns: Elements of Reusable Object-Oriented Software" (Gang of Four, 1994). No son código específico, sino descripciones de cómo resolver problemas que pueden adaptarse a diferentes situaciones.

### ¿Por qué son importantes los Design Patterns?

1. **Vocabulario común**: Permiten comunicar ideas complejas de diseño con términos simples
2. **Soluciones probadas**: Han sido refinados a través de años de uso en la industria
3. **Aceleran el desarrollo**: No necesitas reinventar la rueda
4. **Mejoran la mantenibilidad**: Los patrones son reconocibles y entendibles
5. **Facilitan la colaboración**: Otros desarrolladores reconocen los patrones

### Categorías de Design Patterns:

- **Creacionales**: Se enfocan en mecanismos de creación de objetos
- **Estructurales**: Se ocupan de la composición de clases y objetos
- **De Comportamiento**: Se centran en la comunicación entre objetos

---

## 2.1 Factory Pattern (Patrón Creacional)

### Explicación Detallada

El Factory Pattern es un patrón creacional que proporciona una interfaz para crear objetos sin especificar sus clases concretas. Define un método que debe usarse para crear objetos en lugar de llamar directamente al constructor con `new`.

#### ¿Cuándo usar Factory Pattern?

1. **Cuando no sabes de antemano los tipos exactos de objetos que necesitarás crear**
2. **Cuando quieres centralizar la lógica de creación de objetos**
3. **Cuando la creación del objeto requiere lógica compleja**
4. **Cuando quieres desacoplar el código que usa los objetos del código que los crea**
5. **Cuando necesitas controlar el número de instancias creadas**

#### Tipos de Factory Pattern:

- **Simple Factory**: No es un patrón GoF oficial, pero es útil para casos simples
- **Factory Method**: Define una interfaz para crear objetos, pero deja que las subclases decidan qué clase instanciar
- **Abstract Factory**: Proporciona una interfaz para crear familias de objetos relacionados

#### Ventajas:

- **Encapsulación**: La lógica de creación está en un solo lugar
- **Flexibilidad**: Fácil agregar nuevos tipos sin modificar código cliente
- **Desacoplamiento**: El código cliente no depende de clases concretas
- **Reutilización**: La lógica de creación puede reutilizarse

#### Desventajas:

- **Complejidad adicional**: Puede ser excesivo para casos simples
- **Más clases**: Requiere crear clases factory adicionales

### Ejemplo Práctico con Explicación

```typescript
// EJEMPLO 1: Simple Factory Pattern

// 1. Interface común
interface Notification {
  send(message: string, recipient: string): Promise<boolean>;
  getType(): string;
}

// 2. Implementaciones concretas
class EmailNotification implements Notification {
  async send(message: string, recipient: string): Promise<boolean> {
    console.log(`Email to ${recipient}: ${message}`);
    return true;
  }
  
  getType(): string {
    return 'EMAIL';
  }
}

class SMSNotification implements Notification {
  async send(message: string, recipient: string): Promise<boolean> {
    const truncated = message.substring(0, 160);
    console.log(`SMS to ${recipient}: ${truncated}`);
    return true;
  }
  
  getType(): string {
    return 'SMS';
  }
}

class PushNotification implements Notification {
  async send(message: string, recipient: string): Promise<boolean> {
    console.log(`Push to ${recipient}: ${message}`);
    return true;
  }
  
  getType(): string {
    return 'PUSH';
  }
}

// 3. Simple Factory
class NotificationFactory {
  static createNotification(type: string): Notification {
    switch(type) {
      case 'email':
        return new EmailNotification();
      case 'sms':
        return new SMSNotification();
      case 'push':
        return new PushNotification();
      default:
        throw new Error(`Unknown type: ${type}`);
    }
  }
  
  // Crear basado en preferencias del usuario
  static createFromUser(user: User): Notification {
    if (user.prefersPush) {
      return this.createNotification('push');
    } else if (user.prefersSMS) {
      return this.createNotification('sms');
    }
    return this.createNotification('email');
  }
}

// EJEMPLO 2: Factory Method Pattern

// 1. Producto abstracto
abstract class Document {
  abstract getType(): string;
  abstract save(): void;
}

// 2. Productos concretos
class PDFDocument extends Document {
  getType(): string { return 'PDF'; }
  save(): void { console.log('Saving PDF'); }
}

class WordDocument extends Document {
  getType(): string { return 'DOCX'; }
  save(): void { console.log('Saving Word'); }
}

// 3. Creator abstracto con Factory Method
abstract class Application {
  // Factory Method - subclases deciden qué crear
  abstract createDocument(): Document;
  
  newDocument(): Document {
    const doc = this.createDocument();
    console.log(`Created ${doc.getType()}`);
    return doc;
  }
}

// 4. Creators concretos
class PDFApplication extends Application {
  createDocument(): Document {
    return new PDFDocument();
  }
}

class WordApplication extends Application {
  createDocument(): Document {
    return new WordDocument();
  }
}

// EJEMPLO 3: Abstract Factory Pattern
// Crea familias de objetos relacionados

// 1. Interfaces de productos
interface Button {
  render(): void;
}

interface Input {
  render(): void;
}

// 2. Implementaciones por tema
class LightButton implements Button {
  render(): void { console.log('Light button'); }
}

class LightInput implements Input {
  render(): void { console.log('Light input'); }
}

class DarkButton implements Button {
  render(): void { console.log('Dark button'); }
}

class DarkInput implements Input {
  render(): void { console.log('Dark input'); }
}

// 3. Abstract Factory
interface UIFactory {
  createButton(): Button;
  createInput(): Input;
}

// 4. Factories concretas
class LightThemeFactory implements UIFactory {
  createButton(): Button { return new LightButton(); }
  createInput(): Input { return new LightInput(); }
}

class DarkThemeFactory implements UIFactory {
  createButton(): Button { return new DarkButton(); }
  createInput(): Input { return new DarkInput(); }
}

// 5. Uso
class App {
  private factory: UIFactory;
  
  constructor(theme: 'light' | 'dark') {
    this.factory = theme === 'dark' 
      ? new DarkThemeFactory() 
      : new LightThemeFactory();
  }
  
  createUI(): void {
    const button = this.factory.createButton();
    const input = this.factory.createInput();
    button.render();
    input.render();
  }
}

// USO DE LOS FACTORIES

// 1. Simple Factory
const email = NotificationFactory.createNotification('email');
await email.send('Hello', 'user@example.com');

// 2. Factory Method
const pdfApp = new PDFApplication();
const doc = pdfApp.newDocument(); // Crea PDF
doc.save();

// 3. Abstract Factory
const app = new App('dark');
app.createUI(); // Crea UI con tema dark
```

---

## 2.2 Singleton Pattern (Patrón Creacional)

### Explicación Detallada

El Singleton Pattern garantiza que una clase tenga una única instancia y proporciona un punto de acceso global a esa instancia. Es uno de los patrones más simples pero también uno de los más controversiales.

#### ¿Cuándo usar Singleton?

1. **Cuando necesitas exactamente una instancia** (ej: configuración de la aplicación)
2. **Cuando el acceso a esa instancia debe ser global**
3. **Cuando la instancia debe ser lazy-initialized** (creada solo cuando se necesita)
4. **Para recursos compartidos** (ej: pool de conexiones, caché)
5. **Para coordinar acciones en el sistema** (ej: logger, gestor de eventos)

#### Problemas del Singleton:

- **Dificulta el testing**: Es difícil crear mocks o stubs
- **Acopla el código**: Introduce dependencias globales
- **Problemas de concurrencia**: En aplicaciones multi-thread
- **Viola el principio de responsabilidad única**: Gestiona su propia creación
- **Puede enmascarar mal diseño**: A veces es mejor inyectar dependencias

#### Alternativas al Singleton:

- **Inyección de dependencias**: Pasar la instancia como parámetro
- **Service Locator**: Registro central de servicios
- **Módulos**: En JavaScript/TypeScript, los módulos ya son singletons

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Singleton Clásico
// ============================================

class ConfigurationManager {
  // Propiedad estática privada para almacenar la única instancia
  private static instance: ConfigurationManager | null = null;
  
  // Almacén de configuración
  private config: Map<string, any> = new Map();
  private configFilePath: string;
  private isLoaded: boolean = false;
  
  // Constructor PRIVADO - previene new ConfigurationManager()
  private constructor() {
    // Inicialización que solo debe ocurrir una vez
    this.configFilePath = process.env.CONFIG_PATH || './config.json';
    console.log('🔧 ConfigurationManager instance created');
    
    // Cargar configuración inicial
    this.loadConfiguration();
  }
  
  // Método estático para obtener la instancia
  public static getInstance(): ConfigurationManager {
    // Lazy initialization - crear solo cuando se necesita
    if (!ConfigurationManager.instance) {
      console.log('📦 Creating new ConfigurationManager instance...');
      ConfigurationManager.instance = new ConfigurationManager();
    } else {
      console.log('♻️ Returning existing ConfigurationManager instance');
    }
    
    return ConfigurationManager.instance;
  }
  
  // Métodos públicos para gestionar configuración
  private loadConfiguration(): void {
    console.log(`📁 Loading configuration from ${this.configFilePath}`);
    
    // Simular carga de archivo
    // En una aplicación real, esto leería de un archivo o API
    this.config.set('database.host', 'localhost');
    this.config.set('database.port', 5432);
    this.config.set('database.name', 'myapp');
    this.config.set('api.timeout', 30000);
    this.config.set('api.retries', 3);
    this.config.set('cache.ttl', 3600);
    this.config.set('features.darkMode', true);
    this.config.set('features.betaFeatures', false);
    
    this.isLoaded = true;
  }
  
  public get(key: string): any {
    if (!this.isLoaded) {
      throw new Error('Configuration not loaded yet');
    }
    
    // Soportar notación de punto para acceso anidado
    const keys = key.split('.');
    let value = this.config;
    
    for (const k of keys) {
      if (value instanceof Map) {
        value = value.get(k);
      } else if (value && typeof value === 'object') {
        value = value[k];
      } else {
        return undefined;
      }
    }
    
    return value;
  }
  
  public set(key: string, value: any): void {
    console.log(`⚙️ Setting config: ${key} = ${value}`);
    this.config.set(key, value);
  }
  
  public getAll(): Record<string, any> {
    const result: Record<string, any> = {};
    
    for (const [key, value] of this.config.entries()) {
      result[key] = value;
    }
    
    return result;
  }
  
  public reload(): void {
    console.log('🔄 Reloading configuration...');
    this.config.clear();
    this.loadConfiguration();
  }
  
  // Método para resetear el singleton (útil para testing)
  public static resetInstance(): void {
    if (ConfigurationManager.instance) {
      console.log('🗑️ Resetting ConfigurationManager instance');
      ConfigurationManager.instance = null;
    }
  }
}

// ============================================
// EJEMPLO 2: Singleton Thread-Safe (simulado)
// ============================================

class DatabaseConnectionPool {
  private static instance: DatabaseConnectionPool | null = null;
  private static lock: boolean = false;
  
  private connections: Connection[] = [];
  private availableConnections: Connection[] = [];
  private maxConnections: number = 10;
  private activeConnections: number = 0;
  
  private constructor() {
    console.log('🗄️ Creating DatabaseConnectionPool');
    this.initializePool();
  }
  
  // Double-checked locking pattern (simulado para TypeScript)
  public static async getInstance(): Promise<DatabaseConnectionPool> {
    // Primera verificación (sin lock)
    if (!DatabaseConnectionPool.instance) {
      // Simular lock acquisition
      while (DatabaseConnectionPool.lock) {
        await new Promise(resolve => setTimeout(resolve, 10));
      }
      
      DatabaseConnectionPool.lock = true;
      
      try {
        // Segunda verificación (con lock)
        if (!DatabaseConnectionPool.instance) {
          DatabaseConnectionPool.instance = new DatabaseConnectionPool();
        }
      } finally {
        DatabaseConnectionPool.lock = false;
      }
    }
    
    return DatabaseConnectionPool.instance;
  }
  
  private initializePool(): void {
    console.log(`📊 Initializing connection pool with ${this.maxConnections} connections`);
    
    for (let i = 0; i < this.maxConnections; i++) {
      const connection = new Connection(i);
      this.connections.push(connection);
      this.availableConnections.push(connection);
    }
  }
  
  public async getConnection(): Promise<Connection> {
    // Esperar si no hay conexiones disponibles
    while (this.availableConnections.length === 0) {
      console.log('⏳ Waiting for available connection...');
      await new Promise(resolve => setTimeout(resolve, 100));
    }
    
    const connection = this.availableConnections.pop()!;
    this.activeConnections++;
    
    console.log(`✅ Connection ${connection.id} acquired. Active: ${this.activeConnections}`);
    
    return connection;
  }
  
  public releaseConnection(connection: Connection): void {
    if (!this.connections.includes(connection)) {
      throw new Error('Connection does not belong to this pool');
    }
    
    connection.reset();
    this.availableConnections.push(connection);
    this.activeConnections--;
    
    console.log(`🔓 Connection ${connection.id} released. Active: ${this.activeConnections}`);
  }
  
  public getStats(): PoolStats {
    return {
      total: this.maxConnections,
      active: this.activeConnections,
      available: this.availableConnections.length
    };
  }
  
  public async executeQuery(query: string): Promise<any> {
    const connection = await this.getConnection();
    
    try {
      return await connection.query(query);
    } finally {
      this.releaseConnection(connection);
    }
  }
}

// ============================================
// EJEMPLO 3: Singleton con Registry Pattern
// ============================================

// Este patrón permite múltiples "singletons" nombrados

class ServiceRegistry {
  private static instance: ServiceRegistry | null = null;
  private services: Map<string, any> = new Map();
  private factories: Map<string, () => any> = new Map();
  
  private constructor() {
    console.log('📚 ServiceRegistry created');
  }
  
  public static getInstance(): ServiceRegistry {
    if (!ServiceRegistry.instance) {
      ServiceRegistry.instance = new ServiceRegistry();
    }
    return ServiceRegistry.instance;
  }
  
  // Registrar un servicio singleton
  public registerSingleton<T>(name: string, instance: T): void {
    if (this.services.has(name)) {
      console.warn(`⚠️ Service '${name}' is already registered`);
      return;
    }
    
    console.log(`📝 Registering singleton service: ${name}`);
    this.services.set(name, instance);
  }
  
  // Registrar una factory para lazy initialization
  public registerFactory<T>(name: string, factory: () => T): void {
    if (this.factories.has(name)) {
      console.warn(`⚠️ Factory '${name}' is already registered`);
      return;
    }
    
    console.log(`🏭 Registering factory: ${name}`);
    this.factories.set(name, factory);
  }
  
  // Obtener un servicio
  public get<T>(name: string): T {
    // Primero buscar en servicios ya creados
    if (this.services.has(name)) {
      console.log(`♻️ Returning existing service: ${name}`);
      return this.services.get(name);
    }
    
    // Si no existe, buscar factory y crear
    if (this.factories.has(name)) {
      console.log(`🔨 Creating service from factory: ${name}`);
      const factory = this.factories.get(name)!;
      const instance = factory();
      this.services.set(name, instance);
      return instance;
    }
    
    throw new Error(`Service '${name}' not found`);
  }
  
  // Verificar si un servicio existe
  public has(name: string): boolean {
    return this.services.has(name) || this.factories.has(name);
  }
  
  // Limpiar todos los servicios
  public clear(): void {
    console.log('🧹 Clearing all services');
    this.services.clear();
  }
  
  // Obtener lista de servicios registrados
  public list(): string[] {
    const serviceNames = Array.from(this.services.keys());
    const factoryNames = Array.from(this.factories.keys());
    return [...new Set([...serviceNames, ...factoryNames])];
  }
}

// ============================================
// EJEMPLO 4: Module Pattern como Singleton
// ============================================

// En TypeScript/JavaScript, los módulos son naturalmente singletons

const LoggerModule = (() => {
  // Variables privadas
  let logs: LogEntry[] = [];
  let logLevel: LogLevel = LogLevel.INFO;
  let maxLogs: number = 1000;
  
  // Funciones privadas
  function formatMessage(level: LogLevel, message: string): string {
    const timestamp = new Date().toISOString();
    const levelStr = LogLevel[level];
    return `[${timestamp}] [${levelStr}] ${message}`;
  }
  
  function shouldLog(level: LogLevel): boolean {
    return level >= logLevel;
  }
  
  function addLog(entry: LogEntry): void {
    logs.push(entry);
    
    // Mantener solo los últimos maxLogs
    if (logs.length > maxLogs) {
      logs = logs.slice(-maxLogs);
    }
  }
  
  // API pública
  return {
    debug(message: string, data?: any): void {
      if (shouldLog(LogLevel.DEBUG)) {
        const formatted = formatMessage(LogLevel.DEBUG, message);
        console.log(formatted, data || '');
        addLog({ level: LogLevel.DEBUG, message, data, timestamp: new Date() });
      }
    },
    
    info(message: string, data?: any): void {
      if (shouldLog(LogLevel.INFO)) {
        const formatted = formatMessage(LogLevel.INFO, message);
        console.info(formatted, data || '');
        addLog({ level: LogLevel.INFO, message, data, timestamp: new Date() });
      }
    },
    
    warn(message: string, data?: any): void {
      if (shouldLog(LogLevel.WARN)) {
        const formatted = formatMessage(LogLevel.WARN, message);
        console.warn(formatted, data || '');
        addLog({ level: LogLevel.WARN, message, data, timestamp: new Date() });
      }
    },
    
    error(message: string, error?: Error): void {
      if (shouldLog(LogLevel.ERROR)) {
        const formatted = formatMessage(LogLevel.ERROR, message);
        console.error(formatted, error || '');
        addLog({ 
          level: LogLevel.ERROR, 
          message, 
          data: error?.stack, 
          timestamp: new Date() 
        });
      }
    },
    
    setLogLevel(level: LogLevel): void {
      logLevel = level;
      console.log(`📊 Log level set to ${LogLevel[level]}`);
    },
    
    getLogLevel(): LogLevel {
      return logLevel;
    },
    
    getLogs(level?: LogLevel): LogEntry[] {
      if (level !== undefined) {
        return logs.filter(log => log.level === level);
      }
      return [...logs];
    },
    
    clearLogs(): void {
      logs = [];
      console.log('🗑️ Logs cleared');
    },
    
    exportLogs(): string {
      return JSON.stringify(logs, null, 2);
    }
  };
})();

// ============================================
// EJEMPLO 5: Singleton con Lazy Properties
// ============================================

class ApplicationContext {
  private static instance: ApplicationContext | null = null;
  
  // Propiedades lazy-initialized
  private _database?: DatabaseConnectionPool;
  private _cache?: CacheManager;
  private _eventBus?: EventBus;
  private _config?: ConfigurationManager;
  
  private constructor() {
    console.log('🌍 ApplicationContext initialized');
  }
  
  public static getInstance(): ApplicationContext {
    if (!ApplicationContext.instance) {
      ApplicationContext.instance = new ApplicationContext();
    }
    return ApplicationContext.instance;
  }
  
  // Cada propiedad se inicializa solo cuando se accede
  public get database(): DatabaseConnectionPool {
    if (!this._database) {
      console.log('💾 Initializing database connection pool...');
      this._database = DatabaseConnectionPool.getInstance();
    }
    return this._database;
  }
  
  public get cache(): CacheManager {
    if (!this._cache) {
      console.log('💾 Initializing cache manager...');
      this._cache = new CacheManager();
    }
    return this._cache;
  }
  
  public get eventBus(): EventBus {
    if (!this._eventBus) {
      console.log('📡 Initializing event bus...');
      this._eventBus = new EventBus();
    }
    return this._eventBus;
  }
  
  public get config(): ConfigurationManager {
    if (!this._config) {
      console.log('⚙️ Initializing configuration manager...');
      this._config = ConfigurationManager.getInstance();
    }
    return this._config;
  }
  
  // Método para inicializar todo de una vez si es necesario
  public async initialize(): Promise<void> {
    console.log('🚀 Initializing application context...');
    
    // Forzar inicialización de todos los servicios
    await this.database;
    this.cache;
    this.eventBus;
    this.config;
    
    console.log('✅ Application context ready');
  }
  
  // Método para limpiar recursos
  public async shutdown(): Promise<void> {
    console.log('🛑 Shutting down application context...');
    
    // Limpiar recursos en orden inverso
    if (this._eventBus) {
      this._eventBus.removeAllListeners();
    }
    
    if (this._cache) {
      this._cache.clear();
    }
    
    if (this._database) {
      // await this._database.closeAll();
    }
    
    console.log('👋 Application context shut down');
  }
}

// ============================================
// Clases auxiliares para los ejemplos
// ============================================

class Connection {
  constructor(public readonly id: number) {}
  
  async query(sql: string): Promise<any> {
    console.log(`🔍 Connection ${this.id} executing: ${sql}`);
    // Simular query
    await new Promise(resolve => setTimeout(resolve, Math.random() * 100));
    return { rows: [], affectedRows: 0 };
  }
  
  reset(): void {
    // Limpiar estado de la conexión
  }
}

interface PoolStats {
  total: number;
  active: number;
  available: number;
}

enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3
}

interface LogEntry {
  level: LogLevel;
  message: string;
  data?: any;
  timestamp: Date;
}

class CacheManager {
  private cache = new Map<string, CacheEntry>();
  
  set(key: string, value: any, ttl: number = 3600000): void {
    this.cache.set(key, {
      value,
      expiry: Date.now() + ttl
    });
  }
  
  get(key: string): any | null {
    const entry = this.cache.get(key);
    
    if (!entry) return null;
    
    if (Date.now() > entry.expiry) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.value;
  }
  
  clear(): void {
    this.cache.clear();
  }
}

interface CacheEntry {
  value: any;
  expiry: number;
}

class EventBus {
  private listeners = new Map<string, Set<Function>>();
  
  on(event: string, handler: Function): void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);
  }
  
  emit(event: string, ...args: any[]): void {
    const handlers = this.listeners.get(event);
    if (handlers) {
      handlers.forEach(handler => handler(...args));
    }
  }
  
  removeAllListeners(): void {
    this.listeners.clear();
  }
}

// ============================================
// USO DE LOS SINGLETONS
// ============================================

async function demonstrateSingletons() {
  console.log('\n=== SINGLETON PATTERN DEMO ===\n');
  
  // 1. Configuration Manager
  console.log('\n--- Configuration Manager ---');
  const config1 = ConfigurationManager.getInstance();
  const config2 = ConfigurationManager.getInstance();
  
  console.log(`Same instance? ${config1 === config2}`); // true
  
  config1.set('app.name', 'My Application');
  console.log(`App name: ${config2.get('app.name')}`); // My Application
  
  // 2. Database Connection Pool
  console.log('\n--- Database Connection Pool ---');
  const pool = await DatabaseConnectionPool.getInstance();
  
  // Ejecutar queries en paralelo
  const queries = [
    pool.executeQuery('SELECT * FROM users'),
    pool.executeQuery('SELECT * FROM products'),
    pool.executeQuery('SELECT * FROM orders')
  ];
  
  await Promise.all(queries);
  console.log('Pool stats:', pool.getStats());
  
  // 3. Service Registry
  console.log('\n--- Service Registry ---');
  const registry = ServiceRegistry.getInstance();
  
  // Registrar servicios
  registry.registerSingleton('logger', LoggerModule);
  registry.registerFactory('timestamp', () => new Date().toISOString());
  
  const logger = registry.get('logger');
  logger.info('Service registry working!');
  
  const time1 = registry.get('timestamp');
  await new Promise(resolve => setTimeout(resolve, 100));
  const time2 = registry.get('timestamp');
  console.log(`Same timestamp? ${time1 === time2}`); // true (cached)
  
  // 4. Application Context
  console.log('\n--- Application Context ---');
  const context = ApplicationContext.getInstance();
  
  // Los servicios se inicializan solo cuando se acceden
  const db = context.database; // Se inicializa aquí
  const cache = context.cache; // Se inicializa aquí
  
  cache.set('user:123', { name: 'John', age: 30 });
  console.log('Cached user:', cache.get('user:123'));
  
  // 5. Logger Module
  console.log('\n--- Logger Module ---');
  LoggerModule.setLogLevel(LogLevel.DEBUG);
  LoggerModule.debug('Debug message');
  LoggerModule.info('Info message');
  LoggerModule.warn('Warning message');
  LoggerModule.error('Error message', new Error('Sample error'));
  
  console.log('\nRecent logs:', LoggerModule.getLogs().slice(-3));
}

// demonstrateSingletons();
```

### Consideraciones importantes sobre Singleton:

1. **Testing**: Usa métodos para resetear la instancia en tests
2. **Thread Safety**: En lenguajes con threads, necesitas sincronización
3. **Lazy vs Eager**: Decide si inicializar al arranque o cuando se necesite
4. **Alternativas**: Considera inyección de dependencias antes de usar Singleton
5. **Global State**: Ten cuidado con el estado global mutable

El Singleton es útil pero debe usarse con moderación. Muchas veces, lo que parece necesitar un Singleton puede resolverse mejor con inyección de dependencias o un contenedor IoC.

---

## Enfoques Funcionales para Design Patterns

### Factory Pattern Funcional

```typescript
// Factory como función de orden superior
type NotificationType = 'email' | 'sms' | 'push';
type Notifier = (message: string, recipient: string) => Promise<void>;

const notifierFactory = (type: NotificationType): Notifier => {
  const notifiers: Record<NotificationType, Notifier> = {
    email: async (msg, to) => console.log(`Email to ${to}: ${msg}`),
    sms: async (msg, to) => console.log(`SMS to ${to}: ${msg.substring(0, 160)}`),
    push: async (msg, to) => console.log(`Push to ${to}: ${msg}`)
  };
  
  return notifiers[type] || notifiers.email;
};

// Aplicación parcial para crear notificadores especializados
const createNotifier = (type: NotificationType) => 
  (recipient: string) => 
    (message: string) => 
      notifierFactory(type)(message, recipient);

// Uso
const emailUser = createNotifier('email')('user@example.com');
await emailUser('Welcome!');
```

### Singleton Funcional

```typescript
// Módulo con closure (naturalmente singleton)
const ConfigModule = (() => {
  let config: Map<string, any> | null = null;
  
  const initialize = (): Map<string, any> => {
    if (!config) {
      config = new Map([
        ['api.url', 'https://api.example.com'],
        ['api.timeout', 5000]
      ]);
    }
    return config;
  };
  
  return {
    get: (key: string): any => initialize().get(key),
    set: (key: string, value: any): void => initialize().set(key, value)
  };
})();

// Memoización para singleton funcional
const memoizedSingleton = <T>(factory: () => T): (() => T) => {
  let instance: T | null = null;
  return () => instance || (instance = factory());
};

const getDatabase = memoizedSingleton(() => ({
  connect: () => console.log('Connected'),
  query: (sql: string) => console.log(`Query: ${sql}`)
}));
```

### Strategy Pattern Funcional

```typescript
// Estrategias como funciones
type SortStrategy<T> = (items: T[]) => T[];

const sortStrategies = {
  alphabetical: <T extends { name: string }>(items: T[]): T[] =>
    [...items].sort((a, b) => a.name.localeCompare(b.name)),
    
  numerical: <T extends { value: number }>(items: T[]): T[] =>
    [...items].sort((a, b) => a.value - b.value),
    
  byDate: <T extends { date: Date }>(items: T[]): T[] =>
    [...items].sort((a, b) => a.date.getTime() - b.date.getTime())
};

// Aplicar estrategia
const sortWith = <T>(strategy: SortStrategy<T>) => (items: T[]): T[] =>
  strategy(items);

// Uso
const items = [{ name: 'Z' }, { name: 'A' }];
const sorted = sortWith(sortStrategies.alphabetical)(items);
```

### Observer Pattern Funcional

```typescript
// Event emitter funcional
type Listener<T> = (data: T) => void;
type Unsubscribe = () => void;

const createEventEmitter = <T>() => {
  let listeners: Listener<T>[] = [];
  
  return {
    emit: (data: T): void => {
      listeners.forEach(listener => listener(data));
    },
    
    subscribe: (listener: Listener<T>): Unsubscribe => {
      listeners = [...listeners, listener];
      return () => {
        listeners = listeners.filter(l => l !== listener);
      };
    }
  };
};

// Uso
const priceEmitter = createEventEmitter<number>();
const unsubscribe = priceEmitter.subscribe(price => 
  console.log(`Price: ${price}`)
);

priceEmitter.emit(100);
unsubscribe();
```

### Decorator Pattern Funcional

```typescript
// Decoradores como funciones de orden superior
type Handler = (req: Request) => Response;

const withLogging = (handler: Handler): Handler => 
  (req) => {
    console.log(`Request: ${req.url}`);
    const response = handler(req);
    console.log(`Response: ${response.status}`);
    return response;
  };

const withAuth = (handler: Handler): Handler =>
  (req) => {
    if (!req.headers.authorization) {
      return { status: 401, body: 'Unauthorized' };
    }
    return handler(req);
  };

// Composición de decoradores
import { pipe } from 'fp-ts/function';

const handler: Handler = (req) => ({ 
  status: 200, 
  body: 'Success' 
});

const decoratedHandler = pipe(
  handler,
  withAuth,
  withLogging
);
```