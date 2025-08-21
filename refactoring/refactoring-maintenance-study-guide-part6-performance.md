# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 6: Performance y Optimización
### Para Entrevista Senior Software Engineer - Zillow

---

## 6. Performance y Optimización - Haciendo el Código Eficiente

La optimización de performance es crucial para crear aplicaciones escalables y responsivas. Sin embargo, es importante recordar la famosa cita de Donald Knuth: "La optimización prematura es la raíz de todo mal". Primero hacer que funcione, luego hacer que sea correcto, y finalmente hacer que sea rápido.

### Principios de Optimización

1. **Medir antes de optimizar**: Nunca optimices basándote en suposiciones
2. **Optimizar los cuellos de botella**: El 80% del tiempo se gasta en el 20% del código
3. **Complejidad algorítmica primero**: O(n) vs O(n²) importa más que micro-optimizaciones
4. **Caché estratégicamente**: Memoria vs cómputo es un trade-off común
5. **Lazy loading**: No hagas trabajo hasta que sea necesario

### Métricas Importantes

- **Tiempo de respuesta**: Cuánto tarda una operación
- **Throughput**: Cuántas operaciones por segundo
- **Uso de memoria**: Cuánta RAM consume
- **Uso de CPU**: Porcentaje de procesador utilizado
- **I/O operations**: Lecturas/escrituras a disco o red

### Herramientas de Profiling

- **Chrome DevTools**: Para aplicaciones web
- **Node.js Profiler**: Para backend JavaScript
- **Memory Profiler**: Para detectar memory leaks
- **APM Tools**: Application Performance Monitoring (New Relic, DataDog)

---

## 6.1 Memoization

### Explicación Detallada

Memoization es una técnica de optimización que almacena los resultados de funciones costosas y retorna el resultado cacheado cuando se vuelve a llamar con los mismos argumentos. Es especialmente útil para funciones puras que realizan cálculos intensivos.

#### Cuándo usar Memoization:

1. **Funciones puras**: La misma entrada siempre produce la misma salida
2. **Cálculos costosos**: Operaciones que toman tiempo significativo
3. **Llamadas repetidas**: La función se llama múltiples veces con los mismos argumentos
4. **Resultados limitados**: El espacio de posibles resultados es manejable
5. **Trade-off memoria/velocidad favorable**: Tienes memoria disponible para ganar velocidad

#### Cuándo NO usar Memoization:

- **Funciones con efectos secundarios**: No son puras
- **Argumentos únicos**: Si nunca se repiten los argumentos
- **Resultados muy grandes**: Si cachear consume demasiada memoria
- **Datos que cambian frecuentemente**: El cache se vuelve inválido rápidamente

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Memoization Básica
// ============================================

// ❌ SIN MEMOIZATION: Función costosa que se recalcula cada vez
class FibonacciCalculator {
  // Esta implementación tiene complejidad O(2^n) - MUY INEFICIENTE
  calculate(n: number): number {
    console.log(`Calculating fibonacci(${n})`);
    
    if (n <= 1) return n;
    
    // Problema: Recalcula los mismos valores múltiples veces
    return this.calculate(n - 1) + this.calculate(n - 2);
  }
}

// Demostración del problema
function demonstrateProblem() {
  const calc = new FibonacciCalculator();
  
  console.time('fibonacci(40)');
  const result = calc.calculate(40); // Toma MUCHO tiempo
  console.timeEnd('fibonacci(40)');
  
  // fibonacci(40) hace 2^40 llamadas recursivas!
  // Eso es más de 1 TRILLÓN de llamadas
}

// ✅ CON MEMOIZATION: Cachea resultados para evitar recálculos
class MemoizedFibonacciCalculator {
  private cache = new Map<number, number>();
  private hitCount = 0;
  private missCount = 0;
  
  calculate(n: number): number {
    // Verificar cache primero
    if (this.cache.has(n)) {
      this.hitCount++;
      console.log(`Cache hit for fibonacci(${n})`);
      return this.cache.get(n)!;
    }
    
    this.missCount++;
    console.log(`Cache miss - calculating fibonacci(${n})`);
    
    // Calcular si no está en cache
    let result: number;
    
    if (n <= 1) {
      result = n;
    } else {
      result = this.calculate(n - 1) + this.calculate(n - 2);
    }
    
    // Guardar en cache antes de retornar
    this.cache.set(n, result);
    
    return result;
  }
  
  getStats(): CacheStats {
    const total = this.hitCount + this.missCount;
    const hitRate = total > 0 ? (this.hitCount / total) * 100 : 0;
    
    return {
      hits: this.hitCount,
      misses: this.missCount,
      hitRate: `${hitRate.toFixed(2)}%`,
      cacheSize: this.cache.size
    };
  }
  
  clearCache(): void {
    this.cache.clear();
    this.hitCount = 0;
    this.missCount = 0;
  }
}

// ============================================
// EJEMPLO 2: Memoization Genérica Reutilizable
// ============================================

// Implementación de memoization genérica
function memoize<TArgs extends any[], TResult>(
  fn: (...args: TArgs) => TResult,
  options?: MemoizeOptions
): (...args: TArgs) => TResult {
  const cache = new Map<string, CachedValue<TResult>>();
  const maxSize = options?.maxSize || Infinity;
  const ttl = options?.ttl || Infinity;
  const keyGenerator = options?.keyGenerator || JSON.stringify;
  
  return (...args: TArgs): TResult => {
    const key = keyGenerator(args);
    const now = Date.now();
    
    // Verificar si existe en cache y no ha expirado
    if (cache.has(key)) {
      const cached = cache.get(key)!;
      
      if (now - cached.timestamp < ttl) {
        console.log(`Memoize hit for key: ${key}`);
        return cached.value;
      } else {
        // Expiró, eliminar del cache
        cache.delete(key);
      }
    }
    
    console.log(`Memoize miss - computing for key: ${key}`);
    
    // Calcular y cachear
    const result = fn(...args);
    
    // Implementar LRU si hay límite de tamaño
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }
    
    cache.set(key, {
      value: result,
      timestamp: now
    });
    
    return result;
  };
}

// Decorador para memoization
function Memoized(options?: MemoizeOptions) {
  return function(
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    descriptor.value = memoize(originalMethod, options);
    return descriptor;
  };
}

// Uso del memoization genérico
class ExpensiveCalculations {
  // Memoization con decorador
  @Memoized({ maxSize: 100, ttl: 60000 }) // Cache máx 100 items por 1 minuto
  calculatePrimeFactors(n: number): number[] {
    console.log(`Calculating prime factors of ${n}`);
    const factors: number[] = [];
    let divisor = 2;
    
    while (n > 1) {
      while (n % divisor === 0) {
        factors.push(divisor);
        n /= divisor;
      }
      divisor++;
    }
    
    return factors;
  }
  
  // Memoization manual
  calculateExpensiveHash = memoize(
    (input: string): string => {
      console.log(`Calculating hash for: ${input}`);
      
      // Simulación de operación costosa
      let hash = 0;
      for (let i = 0; i < input.length; i++) {
        const char = input.charCodeAt(i);
        hash = ((hash << 5) - hash) + char;
        hash = hash & hash; // Convert to 32-bit integer
        
        // Simulación de trabajo pesado
        for (let j = 0; j < 10000; j++) {
          hash = (hash * 31) % 1000000007;
        }
      }
      
      return hash.toString(36);
    },
    { maxSize: 50 }
  );
}

// ============================================
// EJEMPLO 3: Memoization con Múltiples Argumentos
// ============================================

class DataProcessor {
  private cache = new Map<string, any>();
  
  // Problema: Función con múltiples argumentos y objetos complejos
  processData(
    data: number[],
    options: ProcessingOptions
  ): ProcessingResult {
    // Generar clave única para la combinación de argumentos
    const cacheKey = this.generateCacheKey(data, options);
    
    // Verificar cache
    if (this.cache.has(cacheKey)) {
      console.log('Cache hit for processData');
      return this.cache.get(cacheKey);
    }
    
    console.log('Cache miss - processing data');
    
    // Procesamiento costoso
    const result = this.performExpensiveProcessing(data, options);
    
    // Guardar en cache
    this.cache.set(cacheKey, result);
    
    return result;
  }
  
  private generateCacheKey(data: number[], options: ProcessingOptions): string {
    // Estrategia 1: JSON.stringify (simple pero puede ser lento)
    // return JSON.stringify({ data, options });
    
    // Estrategia 2: Hash personalizado (más rápido)
    const dataHash = this.hashArray(data);
    const optionsHash = this.hashObject(options);
    return `${dataHash}-${optionsHash}`;
  }
  
  private hashArray(arr: number[]): string {
    // Hash rápido para arrays
    let hash = arr.length;
    for (let i = 0; i < Math.min(arr.length, 10); i++) {
      hash = ((hash << 5) - hash) + arr[i];
    }
    return hash.toString(36);
  }
  
  private hashObject(obj: any): string {
    // Hash para objetos
    const str = Object.keys(obj).sort().map(k => `${k}:${obj[k]}`).join(',');
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
    }
    return hash.toString(36);
  }
  
  private performExpensiveProcessing(
    data: number[],
    options: ProcessingOptions
  ): ProcessingResult {
    // Simulación de procesamiento costoso
    const sorted = [...data].sort((a, b) => 
      options.sortOrder === 'asc' ? a - b : b - a
    );
    
    const filtered = sorted.filter(n => 
      n >= options.minValue && n <= options.maxValue
    );
    
    const stats = {
      mean: filtered.reduce((a, b) => a + b, 0) / filtered.length,
      median: filtered[Math.floor(filtered.length / 2)],
      min: Math.min(...filtered),
      max: Math.max(...filtered),
      count: filtered.length
    };
    
    return {
      processedData: filtered,
      statistics: stats,
      options: options
    };
  }
}

// ============================================
// EJEMPLO 4: Memoization con WeakMap para Objetos
// ============================================

// WeakMap permite que los objetos sean garbage collected
class ObjectMemoizer {
  private cache = new WeakMap<object, Map<string, any>>();
  
  // Memoization que usa el objeto como parte de la clave
  memoizeForObject<T>(
    obj: object,
    key: string,
    compute: () => T
  ): T {
    // Obtener o crear cache para este objeto
    if (!this.cache.has(obj)) {
      this.cache.set(obj, new Map());
    }
    
    const objCache = this.cache.get(obj)!;
    
    // Verificar si ya calculamos este valor
    if (objCache.has(key)) {
      console.log(`Cache hit for object method: ${key}`);
      return objCache.get(key);
    }
    
    console.log(`Cache miss - computing ${key} for object`);
    
    // Calcular y cachear
    const result = compute();
    objCache.set(key, result);
    
    return result;
  }
}

// Uso con clases
class User {
  constructor(
    public id: string,
    public name: string,
    public email: string,
    private memoizer: ObjectMemoizer = new ObjectMemoizer()
  ) {}
  
  // Propiedad computada costosa, memoizada por instancia
  get permissions(): string[] {
    return this.memoizer.memoizeForObject(
      this,
      'permissions',
      () => this.calculatePermissions()
    );
  }
  
  get fullProfile(): UserProfile {
    return this.memoizer.memoizeForObject(
      this,
      'fullProfile',
      () => this.fetchFullProfile()
    );
  }
  
  private calculatePermissions(): string[] {
    console.log(`Calculating permissions for user ${this.id}`);
    // Simulación de cálculo costoso
    return ['read', 'write', 'delete'];
  }
  
  private fetchFullProfile(): UserProfile {
    console.log(`Fetching full profile for user ${this.id}`);
    // Simulación de operación costosa
    return {
      id: this.id,
      name: this.name,
      email: this.email,
      avatar: 'avatar.jpg',
      settings: {},
      history: []
    };
  }
}

// ============================================
// EJEMPLO 5: LRU Cache Implementation
// ============================================

// Least Recently Used cache - elimina items menos usados cuando se llena
class LRUCache<K, V> {
  private cache = new Map<K, V>();
  private readonly maxSize: number;
  
  constructor(maxSize: number) {
    this.maxSize = maxSize;
  }
  
  get(key: K): V | undefined {
    if (!this.cache.has(key)) {
      return undefined;
    }
    
    // Mover al final (más reciente)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    
    return value;
  }
  
  set(key: K, value: V): void {
    // Si ya existe, eliminar para re-insertar al final
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }
    
    // Si llegamos al límite, eliminar el más antiguo (primero)
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
      console.log(`LRU Cache evicted: ${firstKey}`);
    }
    
    // Agregar al final (más reciente)
    this.cache.set(key, value);
  }
  
  has(key: K): boolean {
    return this.cache.has(key);
  }
  
  clear(): void {
    this.cache.clear();
  }
  
  get size(): number {
    return this.cache.size;
  }
  
  get stats(): CacheStats {
    return {
      size: this.cache.size,
      maxSize: this.maxSize,
      utilization: `${(this.cache.size / this.maxSize * 100).toFixed(2)}%`
    };
  }
}

// Función memoize con LRU
function memoizeWithLRU<T extends (...args: any[]) => any>(
  fn: T,
  maxSize: number = 100
): T {
  const cache = new LRUCache<string, ReturnType<T>>(maxSize);
  
  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    
    // Intentar obtener del cache
    const cached = cache.get(key);
    if (cached !== undefined) {
      return cached;
    }
    
    // Calcular y cachear
    const result = fn(...args);
    cache.set(key, result);
    
    return result;
  }) as T;
}

// Uso
const memoizedFetch = memoizeWithLRU(
  async (url: string): Promise<any> => {
    console.log(`Fetching: ${url}`);
    const response = await fetch(url);
    return response.json();
  },
  50 // Máximo 50 URLs en cache
);

---

## 6.2 Lazy Loading

### Explicación Detallada

Lazy Loading es una técnica de optimización que retrasa la inicialización de un objeto hasta el momento en que se necesita. Esto mejora el tiempo de inicio, reduce el uso de memoria y evita trabajo innecesario.

#### Tipos de Lazy Loading:

1. **Lazy Initialization**: Crear objetos solo cuando se acceden
2. **Lazy Evaluation**: Calcular valores solo cuando se necesitan
3. **Code Splitting**: Cargar código JavaScript solo cuando se requiere
4. **Virtual Scrolling**: Renderizar solo elementos visibles en listas largas
5. **Progressive Loading**: Cargar recursos en orden de prioridad

#### Beneficios:

- **Menor tiempo de inicio**: La aplicación arranca más rápido
- **Menor uso de memoria**: Solo se carga lo necesario
- **Mejor UX**: El usuario ve contenido más rápido
- **Ahorro de ancho de banda**: No se descargan recursos no usados

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Lazy Initialization
// ============================================

// ❌ SIN LAZY LOADING: Todo se inicializa al crear la instancia
class EagerDatabaseService {
  private primaryConnection: DatabaseConnection;
  private readReplicaConnection: DatabaseConnection;
  private analyticsConnection: DatabaseConnection;
  private cacheConnection: CacheConnection;
  
  constructor() {
    // Problema: TODAS las conexiones se crean inmediatamente
    // aunque puede que no todas se usen
    console.log('Initializing all connections...');
    
    this.primaryConnection = new DatabaseConnection('primary');
    this.readReplicaConnection = new DatabaseConnection('replica');
    this.analyticsConnection = new DatabaseConnection('analytics');
    this.cacheConnection = new CacheConnection('redis');
    
    // Esto puede tomar varios segundos y usar mucha memoria
  }
  
  query(sql: string): any {
    return this.primaryConnection.execute(sql);
  }
}

// ✅ CON LAZY LOADING: Las conexiones se crean solo cuando se necesitan
class LazyDatabaseService {
  private _primaryConnection?: DatabaseConnection;
  private _readReplicaConnection?: DatabaseConnection;
  private _analyticsConnection?: DatabaseConnection;
  private _cacheConnection?: CacheConnection;
  
  // Getters con lazy initialization
  private get primaryConnection(): DatabaseConnection {
    if (!this._primaryConnection) {
      console.log('Lazy loading primary connection...');
      this._primaryConnection = new DatabaseConnection('primary');
    }
    return this._primaryConnection;
  }
  
  private get readReplicaConnection(): DatabaseConnection {
    if (!this._readReplicaConnection) {
      console.log('Lazy loading read replica connection...');
      this._readReplicaConnection = new DatabaseConnection('replica');
    }
    return this._readReplicaConnection;
  }
  
  private get analyticsConnection(): DatabaseConnection {
    if (!this._analyticsConnection) {
      console.log('Lazy loading analytics connection...');
      this._analyticsConnection = new DatabaseConnection('analytics');
    }
    return this._analyticsConnection;
  }
  
  private get cacheConnection(): CacheConnection {
    if (!this._cacheConnection) {
      console.log('Lazy loading cache connection...');
      this._cacheConnection = new CacheConnection('redis');
    }
    return this._cacheConnection;
  }
  
  // Los métodos usan los getters lazy
  query(sql: string): any {
    return this.primaryConnection.execute(sql);
  }
  
  queryReadOnly(sql: string): any {
    return this.readReplicaConnection.execute(sql);
  }
  
  queryAnalytics(sql: string): any {
    return this.analyticsConnection.execute(sql);
  }
  
  getCached(key: string): any {
    return this.cacheConnection.get(key);
  }
  
  // Método para liberar recursos
  dispose(): void {
    this._primaryConnection?.close();
    this._readReplicaConnection?.close();
    this._analyticsConnection?.close();
    this._cacheConnection?.close();
  }
  
  // Método para ver qué conexiones están activas
  getActiveConnections(): string[] {
    const active: string[] = [];
    
    if (this._primaryConnection) active.push('primary');
    if (this._readReplicaConnection) active.push('replica');
    if (this._analyticsConnection) active.push('analytics');
    if (this._cacheConnection) active.push('cache');
    
    return active;
  }
}

// ============================================
// EJEMPLO 2: Lazy Loading con Proxy
// ============================================

// Proxy permite interceptar accesos a propiedades
function createLazyProxy<T extends object>(
  factory: () => T,
  onAccess?: (prop: string | symbol) => void
): T {
  let instance: T | null = null;
  
  return new Proxy({} as T, {
    get(target, prop, receiver) {
      // Crear instancia en el primer acceso
      if (!instance) {
        console.log('Lazy loading via proxy...');
        instance = factory();
      }
      
      // Log de acceso opcional
      if (onAccess) {
        onAccess(prop);
      }
      
      // Retornar la propiedad de la instancia real
      return Reflect.get(instance, prop, receiver);
    },
    
    set(target, prop, value, receiver) {
      if (!instance) {
        instance = factory();
      }
      
      return Reflect.set(instance, prop, value, receiver);
    }
  });
}

// Uso del proxy lazy
class ExpensiveService {
  constructor() {
    console.log('ExpensiveService: Heavy initialization...');
    // Simulación de inicialización costosa
    this.loadConfiguration();
    this.connectToServices();
  }
  
  private loadConfiguration(): void {
    console.log('Loading configuration files...');
  }
  
  private connectToServices(): void {
    console.log('Connecting to external services...');
  }
  
  doWork(): void {
    console.log('Doing actual work');
  }
}

// Crear servicio lazy
const lazyService = createLazyProxy(
  () => new ExpensiveService(),
  (prop) => console.log(`Accessing property: ${String(prop)}`)
);

// El servicio NO se crea hasta que se usa
console.log('Service created but not initialized');
// ... más código ...
lazyService.doWork(); // AQUÍ se inicializa

// ============================================
// EJEMPLO 3: Lazy Loading de Módulos/Componentes
// ============================================

// Sistema de módulos con carga lazy
class ModuleLoader {
  private modules = new Map<string, any>();
  private loaders = new Map<string, () => Promise<any>>();
  
  // Registrar un módulo para carga lazy
  register(name: string, loader: () => Promise<any>): void {
    this.loaders.set(name, loader);
  }
  
  // Cargar módulo cuando se necesite
  async load<T>(name: string): Promise<T> {
    // Si ya está cargado, retornarlo
    if (this.modules.has(name)) {
      console.log(`Module ${name} already loaded`);
      return this.modules.get(name);
    }
    
    // Si no hay loader registrado, error
    const loader = this.loaders.get(name);
    if (!loader) {
      throw new Error(`No loader registered for module: ${name}`);
    }
    
    console.log(`Lazy loading module: ${name}`);
    
    try {
      // Cargar el módulo
      const module = await loader();
      
      // Cachear para futuros usos
      this.modules.set(name, module);
      
      return module;
    } catch (error) {
      console.error(`Failed to load module ${name}:`, error);
      throw error;
    }
  }
  
  // Pre-cargar módulos en background
  async preload(names: string[]): Promise<void> {
    console.log(`Preloading modules: ${names.join(', ')}`);
    
    const promises = names.map(name => 
      this.load(name).catch(err => 
        console.error(`Failed to preload ${name}:`, err)
      )
    );
    
    await Promise.all(promises);
  }
  
  // Verificar si un módulo está cargado
  isLoaded(name: string): boolean {
    return this.modules.has(name);
  }
  
  // Descargar módulo para liberar memoria
  unload(name: string): void {
    if (this.modules.has(name)) {
      console.log(`Unloading module: ${name}`);
      const module = this.modules.get(name);
      
      // Si el módulo tiene método de limpieza, llamarlo
      if (module && typeof module.dispose === 'function') {
        module.dispose();
      }
      
      this.modules.delete(name);
    }
  }
}

// Ejemplo de uso
const moduleLoader = new ModuleLoader();

// Registrar módulos
moduleLoader.register('charts', async () => {
  console.log('Loading charts library...');
  // Simular carga de librería pesada
  await new Promise(resolve => setTimeout(resolve, 1000));
  return {
    renderChart: (data: any) => console.log('Rendering chart', data),
    dispose: () => console.log('Disposing chart resources')
  };
});

moduleLoader.register('pdf', async () => {
  console.log('Loading PDF library...');
  await new Promise(resolve => setTimeout(resolve, 1000));
  return {
    generatePDF: (content: string) => console.log('Generating PDF', content)
  };
});

// Uso lazy - solo se carga cuando se necesita
async function generateReport(includeCharts: boolean) {
  const pdfModule = await moduleLoader.load('pdf');
  
  if (includeCharts) {
    const chartsModule = await moduleLoader.load('charts');
    chartsModule.renderChart({ /* data */ });
  }
  
  pdfModule.generatePDF('Report content');
}

// ============================================
// EJEMPLO 4: Virtual Scrolling / Windowing
// ============================================

// Renderizar solo elementos visibles en listas largas
class VirtualList<T> {
  private items: T[];
  private itemHeight: number;
  private containerHeight: number;
  private scrollTop: number = 0;
  private renderBuffer: number = 3; // Items extra para smooth scrolling
  
  constructor(
    items: T[],
    itemHeight: number,
    containerHeight: number
  ) {
    this.items = items;
    this.itemHeight = itemHeight;
    this.containerHeight = containerHeight;
  }
  
  // Calcular qué items son visibles
  getVisibleItems(): VisibleItem<T>[] {
    const startIndex = Math.max(
      0,
      Math.floor(this.scrollTop / this.itemHeight) - this.renderBuffer
    );
    
    const endIndex = Math.min(
      this.items.length - 1,
      Math.ceil((this.scrollTop + this.containerHeight) / this.itemHeight) + this.renderBuffer
    );
    
    console.log(`Rendering items ${startIndex} to ${endIndex} of ${this.items.length}`);
    
    const visibleItems: VisibleItem<T>[] = [];
    
    for (let i = startIndex; i <= endIndex; i++) {
      visibleItems.push({
        index: i,
        item: this.items[i],
        top: i * this.itemHeight,
        height: this.itemHeight
      });
    }
    
    return visibleItems;
  }
  
  // Actualizar posición de scroll
  setScrollTop(scrollTop: number): void {
    this.scrollTop = scrollTop;
  }
  
  // Obtener altura total de la lista
  getTotalHeight(): number {
    return this.items.length * this.itemHeight;
  }
  
  // Obtener estadísticas de renderizado
  getRenderStats(): RenderStats {
    const visible = this.getVisibleItems();
    
    return {
      totalItems: this.items.length,
      visibleItems: visible.length,
      startIndex: visible[0]?.index || 0,
      endIndex: visible[visible.length - 1]?.index || 0,
      memoryUsage: `${((visible.length / this.items.length) * 100).toFixed(2)}%`
    };
  }
}

// ============================================
// EJEMPLO 5: Progressive Data Loading
// ============================================

// Cargar datos en chunks para mejorar perceived performance
class ProgressiveDataLoader<T> {
  private data: T[] = [];
  private loadedChunks = new Set<number>();
  private chunkSize: number;
  private totalItems: number;
  private loadChunk: (offset: number, limit: number) => Promise<T[]>;
  
  constructor(options: {
    totalItems: number;
    chunkSize: number;
    loadChunk: (offset: number, limit: number) => Promise<T[]>;
  }) {
    this.totalItems = options.totalItems;
    this.chunkSize = options.chunkSize;
    this.loadChunk = options.loadChunk;
    
    // Pre-allocar array
    this.data = new Array(this.totalItems);
  }
  
  // Obtener item con carga lazy
  async getItem(index: number): Promise<T | undefined> {
    if (index < 0 || index >= this.totalItems) {
      return undefined;
    }
    
    // Si ya está cargado, retornarlo
    if (this.data[index] !== undefined) {
      return this.data[index];
    }
    
    // Cargar el chunk que contiene este item
    await this.ensureChunkLoaded(index);
    
    return this.data[index];
  }
  
  // Obtener rango de items
  async getRange(start: number, end: number): Promise<T[]> {
    const promises: Promise<void>[] = [];
    
    // Determinar qué chunks necesitamos
    const startChunk = Math.floor(start / this.chunkSize);
    const endChunk = Math.floor(end / this.chunkSize);
    
    for (let chunk = startChunk; chunk <= endChunk; chunk++) {
      if (!this.loadedChunks.has(chunk)) {
        promises.push(this.loadChunkData(chunk));
      }
    }
    
    // Cargar chunks en paralelo
    await Promise.all(promises);
    
    // Retornar el rango solicitado
    return this.data.slice(start, end + 1).filter(item => item !== undefined);
  }
  
  // Cargar un chunk específico
  private async loadChunkData(chunkIndex: number): Promise<void> {
    if (this.loadedChunks.has(chunkIndex)) {
      return;
    }
    
    const offset = chunkIndex * this.chunkSize;
    const limit = Math.min(this.chunkSize, this.totalItems - offset);
    
    console.log(`Loading chunk ${chunkIndex} (items ${offset}-${offset + limit - 1})`);
    
    try {
      const chunkData = await this.loadChunk(offset, limit);
      
      // Insertar datos en las posiciones correctas
      for (let i = 0; i < chunkData.length; i++) {
        this.data[offset + i] = chunkData[i];
      }
      
      this.loadedChunks.add(chunkIndex);
    } catch (error) {
      console.error(`Failed to load chunk ${chunkIndex}:`, error);
      throw error;
    }
  }
  
  // Asegurar que el chunk está cargado
  private async ensureChunkLoaded(index: number): Promise<void> {
    const chunkIndex = Math.floor(index / this.chunkSize);
    await this.loadChunkData(chunkIndex);
  }
  
  // Pre-cargar chunks adyacentes
  async prefetchAround(index: number, radius: number = 1): Promise<void> {
    const centerChunk = Math.floor(index / this.chunkSize);
    const promises: Promise<void>[] = [];
    
    for (let i = -radius; i <= radius; i++) {
      const chunkIndex = centerChunk + i;
      
      if (chunkIndex >= 0 && chunkIndex < Math.ceil(this.totalItems / this.chunkSize)) {
        if (!this.loadedChunks.has(chunkIndex)) {
          promises.push(this.loadChunkData(chunkIndex));
        }
      }
    }
    
    await Promise.all(promises);
  }
  
  // Obtener estadísticas de carga
  getLoadStats(): LoadStats {
    const totalChunks = Math.ceil(this.totalItems / this.chunkSize);
    const loadedItems = Array.from(this.loadedChunks).reduce(
      (sum, chunk) => sum + Math.min(
        this.chunkSize,
        this.totalItems - (chunk * this.chunkSize)
      ),
      0
    );
    
    return {
      totalChunks,
      loadedChunks: this.loadedChunks.size,
      loadProgress: `${((this.loadedChunks.size / totalChunks) * 100).toFixed(2)}%`,
      loadedItems,
      totalItems: this.totalItems,
      itemsProgress: `${((loadedItems / this.totalItems) * 100).toFixed(2)}%`
    };
  }
}

// Interfaces auxiliares
interface MemoizeOptions {
  maxSize?: number;
  ttl?: number;
  keyGenerator?: (args: any[]) => string;
}

interface CachedValue<T> {
  value: T;
  timestamp: number;
}

interface CacheStats {
  hits?: number;
  misses?: number;
  hitRate?: string;
  cacheSize?: number;
  size?: number;
  maxSize?: number;
  utilization?: string;
}

interface ProcessingOptions {
  sortOrder: 'asc' | 'desc';
  minValue: number;
  maxValue: number;
}

interface ProcessingResult {
  processedData: number[];
  statistics: any;
  options: ProcessingOptions;
}

interface UserProfile {
  id: string;
  name: string;
  email: string;
  avatar: string;
  settings: any;
  history: any[];
}

interface VisibleItem<T> {
  index: number;
  item: T;
  top: number;
  height: number;
}

interface RenderStats {
  totalItems: number;
  visibleItems: number;
  startIndex: number;
  endIndex: number;
  memoryUsage: string;
}

interface LoadStats {
  totalChunks: number;
  loadedChunks: number;
  loadProgress: string;
  loadedItems: number;
  totalItems: number;
  itemsProgress: string;
}

// Clases simuladas para los ejemplos
class DatabaseConnection {
  constructor(private type: string) {
    console.log(`Creating ${type} database connection...`);
  }
  
  execute(sql: string): any {
    console.log(`Executing on ${this.type}: ${sql}`);
  }
  
  close(): void {
    console.log(`Closing ${this.type} connection`);
  }
}

class CacheConnection {
  constructor(private type: string) {
    console.log(`Creating ${type} cache connection...`);
  }
  
  get(key: string): any {
    console.log(`Getting ${key} from ${this.type}`);
  }
  
  close(): void {
    console.log(`Closing ${this.type} connection`);
  }
}
```

### Mejores prácticas de optimización:

1. **Medir primero**: Usa profilers para identificar bottlenecks reales
2. **Optimizar algoritmos antes que código**: O(n) vs O(n²) importa más
3. **Cache inteligentemente**: Balance entre memoria y cómputo
4. **Lazy por defecto**: No hagas trabajo hasta que sea necesario
5. **Monitorear en producción**: El comportamiento real puede ser diferente
6. **Documentar optimizaciones**: Explica por qué se hizo la optimización
7. **Evitar optimización prematura**: Primero correcto, luego rápido

La optimización bien aplicada puede hacer una diferencia dramática en la performance de tu aplicación.