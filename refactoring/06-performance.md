# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 6: Performance y Optimización
### Para Entrevista Senior Software Engineer

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
function fibonacci(n: number): number {
  if (n <= 1) return n;
  // Problema: Recalcula los mismos valores múltiples veces
  // fibonacci(5) calcula fibonacci(3) dos veces, fibonacci(2) tres veces...
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// ✅ CON MEMOIZATION: Cachea resultados para evitar recálculos
function memoize<T>(fn: (n: number) => T): (n: number) => T {
  const cache = new Map<number, T>();
  
  return (n: number): T => {
    if (cache.has(n)) {
      return cache.get(n)!;
    }
    
    const result = fn(n);
    cache.set(n, result);
    return result;
  };
}

// Fibonacci memoizado - ahora es O(n) en lugar de O(2^n)
const memoizedFib = memoize((n: number): number => {
  if (n <= 1) return n;
  return memoizedFib(n - 1) + memoizedFib(n - 2);
});

// Comparación:
// fibonacci(40) = ~2 billones de llamadas
// memoizedFib(40) = 40 llamadas

// ============================================
// EJEMPLO 2: Memoization Genérica Reutilizable
// ============================================

// Memoization genérica con opciones
function memoizeGeneric<T extends (...args: any[]) => any>(
  fn: T,
  maxSize = 100
): T {
  const cache = new Map<string, ReturnType<T>>();
  
  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    // LRU: eliminar el más antiguo si llegamos al límite
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// Uso práctico
class DataProcessor {
  // Memoizar cálculos costosos
  processData = memoizeGeneric(
    (data: number[], operation: string): number => {
      // Simulación de procesamiento costoso
      if (operation === 'sum') return data.reduce((a, b) => a + b, 0);
      if (operation === 'avg') return data.reduce((a, b) => a + b, 0) / data.length;
      if (operation === 'max') return Math.max(...data);
      return 0;
    },
    50 // Máximo 50 resultados en cache
  );
}

// ============================================
// EJEMPLO 3: Memoization con WeakMap para Objetos
// ============================================

// WeakMap permite que los objetos sean garbage collected
class ObjectCache<T> {
  private cache = new WeakMap<object, T>();
  
  getOrCompute(obj: object, compute: () => T): T {
    if (this.cache.has(obj)) {
      return this.cache.get(obj)!;
    }
    
    const result = compute();
    this.cache.set(obj, result);
    return result;
  }
}

// Uso práctico con objetos
class UserService {
  private permissionsCache = new ObjectCache<string[]>();
  
  getUserPermissions(user: User): string[] {
    return this.permissionsCache.getOrCompute(user, () => {
      // Cálculo costoso de permisos
      console.log(`Computing permissions for ${user.id}`);
      return ['read', 'write'];
    });
  }
}

// ============================================
// EJEMPLO 4: LRU Cache Implementation
// ============================================

// Least Recently Used cache - elimina items menos usados
class LRUCache<K, V> {
  private cache = new Map<K, V>();
  
  constructor(private maxSize: number) {}
  
  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;
    
    // Mover al final (más reciente)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }
  
  set(key: K, value: V): void {
    // Si existe, eliminar para re-insertar al final
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }
    
    // Si llegamos al límite, eliminar el más antiguo
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, value);
  }
}

// Uso práctico
const apiCache = new LRUCache<string, any>(100);

function fetchData(url: string): any {
  const cached = apiCache.get(url);
  if (cached) return cached;
  
  const data = fetch(url); // Operación costosa
  apiCache.set(url, data);
  return data;
}


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

// ❌ SIN LAZY LOADING: Todo se inicializa inmediatamente
class EagerService {
  private db = new DatabaseConnection();      // Se conecta ahora
  private cache = new CacheConnection();      // Se conecta ahora
  private analytics = new AnalyticsService(); // Se inicializa ahora
  
  // Problema: Todas las conexiones se crean aunque no se usen
}

// ✅ CON LAZY LOADING: Se inicializa solo cuando se necesita
class LazyService {
  private _db?: DatabaseConnection;
  private _cache?: CacheConnection;
  
  private get db(): DatabaseConnection {
    if (!this._db) {
      this._db = new DatabaseConnection(); // Se conecta solo cuando se usa
    }
    return this._db;
  }
  
  private get cache(): CacheConnection {
    if (!this._cache) {
      this._cache = new CacheConnection();
    }
    return this._cache;
  }
  
  query(sql: string): any {
    return this.db.execute(sql); // Conexión se crea aquí si no existe
  }
  
  getCached(key: string): any {
    return this.cache.get(key);
  }
}

// ============================================
// EJEMPLO 2: Lazy Loading con Proxy
// ============================================

// Proxy para lazy loading automático
function lazyProxy<T extends object>(factory: () => T): T {
  let instance: T | null = null;
  
  return new Proxy({} as T, {
    get(target, prop) {
      if (!instance) {
        instance = factory(); // Se crea en el primer acceso
      }
      return Reflect.get(instance, prop);
    }
  });
}

// Uso práctico
class ExpensiveService {
  constructor() {
    console.log('Heavy initialization...');
    // Operación costosa
  }
  
  process(): void {
    console.log('Processing...');
  }
}

// El servicio no se inicializa hasta que se usa
const service = lazyProxy(() => new ExpensiveService());
// ... código ...
service.process(); // Se inicializa AQUÍ

// ============================================
// EJEMPLO 3: Lazy Loading de Módulos
// ============================================

// Sistema simple de módulos lazy
class ModuleLoader {
  private modules = new Map<string, any>();
  private loaders = new Map<string, () => Promise<any>>();
  
  register(name: string, loader: () => Promise<any>): void {
    this.loaders.set(name, loader);
  }
  
  async load<T>(name: string): Promise<T> {
    if (this.modules.has(name)) {
      return this.modules.get(name);
    }
    
    const loader = this.loaders.get(name);
    if (!loader) throw new Error(`Module ${name} not found`);
    
    const module = await loader();
    this.modules.set(name, module);
    return module;
  }
}

// Uso práctico
const loader = new ModuleLoader();

// Registrar módulos para carga lazy
loader.register('charts', async () => {
  // Solo se carga cuando se necesita
  const module = await import('./charts');
  return module.default;
});

loader.register('pdf', async () => {
  const module = await import('./pdf');
  return module.default;
});

// Cargar solo lo necesario
async function generateReport(withCharts: boolean) {
  if (withCharts) {
    const charts = await loader.load('charts');
    charts.render();
  }
  
  const pdf = await loader.load('pdf');
  pdf.generate();
}

// ============================================
// EJEMPLO 4: Virtual Scrolling
// ============================================

// Renderizar solo elementos visibles en listas largas
class VirtualList<T> {
  constructor(
    private items: T[],
    private itemHeight: number,
    private containerHeight: number,
    private scrollTop = 0
  ) {}
  
  getVisibleItems(): T[] {
    const startIndex = Math.floor(this.scrollTop / this.itemHeight);
    const endIndex = Math.ceil(
      (this.scrollTop + this.containerHeight) / this.itemHeight
    );
    
    // Solo retorna los items visibles en lugar de todos
    return this.items.slice(startIndex, endIndex);
  }
  
  setScrollTop(scrollTop: number): void {
    this.scrollTop = scrollTop;
  }
}

// Uso: renderizar solo 20 items visibles de 10,000
const list = new VirtualList(
  Array.from({ length: 10000 }, (_, i) => `Item ${i}`),
  50,  // altura de cada item
  500  // altura del contenedor
);

// Solo obtiene ~10 items en lugar de 10,000
const visibleItems = list.getVisibleItems();

// ============================================
// EJEMPLO 5: Progressive Data Loading
// ============================================

// Cargar datos en chunks progresivamente
class ChunkedLoader<T> {
  private cache = new Map<number, T[]>();
  
  constructor(
    private chunkSize: number,
    private loadChunk: (page: number) => Promise<T[]>
  ) {}
  
  async getPage(page: number): Promise<T[]> {
    if (this.cache.has(page)) {
      return this.cache.get(page)!;
    }
    
    const data = await this.loadChunk(page);
    this.cache.set(page, data);
    
    // Prefetch siguiente página
    this.prefetch(page + 1);
    
    return data;
  }
  
  private async prefetch(page: number): Promise<void> {
    if (!this.cache.has(page)) {
      this.loadChunk(page).then(data => {
        this.cache.set(page, data);
      });
    }
  }
}

// Uso: cargar datos de API en chunks
const loader = new ChunkedLoader(20, async (page) => {
  const response = await fetch(`/api/data?page=${page}&size=20`);
  return response.json();
});

// Carga solo la página 1 y prefetch la 2
const firstPage = await loader.getPage(1);

// Interfaces auxiliares
interface User {
  id: string;
  name: string;
}

// Clases simuladas para los ejemplos
class DatabaseConnection {
  execute(sql: string): any {
    console.log(`Executing: ${sql}`);
  }
}

class CacheConnection {
  get(key: string): any {
    console.log(`Getting: ${key}`);
  }
}

class AnalyticsService {
  constructor() {
    console.log('Analytics initialized');
  }
}

declare function fetch(url: string): any;
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

---

## 6.3 Functional Programming Performance Techniques

### Memoization en FP

```typescript
import { pipe } from 'fp-ts/function';
import * as O from 'fp-ts/Option';
import * as E from 'fp-ts/Either';

// Memoization con fp-ts
const memoizeFP = <A, B>(f: (a: A) => B): ((a: A) => B) => {
  const cache = new Map<A, B>();
  return (a: A): B => 
    pipe(
      O.fromNullable(cache.get(a)),
      O.getOrElse(() => {
        const result = f(a);
        cache.set(a, result);
        return result;
      })
    );
};

// Ejemplo: factorial memoizado
const factorial = memoizeFP((n: number): number =>
  n <= 1 ? 1 : n * factorial(n - 1)
);
```

### Lazy Evaluation con Generadores

```typescript
// Lazy sequences infinitas
function* naturalNumbers(): Generator<number> {
  let n = 0;
  while (true) yield n++;
}

function* take<T>(n: number, iterable: Iterable<T>): Generator<T> {
  let count = 0;
  for (const item of iterable) {
    if (count++ >= n) break;
    yield item;
  }
}

function* map<T, U>(f: (x: T) => U, iterable: Iterable<T>): Generator<U> {
  for (const item of iterable) {
    yield f(item);
  }
}

function* filter<T>(p: (x: T) => boolean, iterable: Iterable<T>): Generator<T> {
  for (const item of iterable) {
    if (p(item)) yield item;
  }
}

// Uso: procesar solo lo necesario
const evenSquares = pipe(
  naturalNumbers(),
  (nums) => filter(n => n % 2 === 0, nums),
  (evens) => map(n => n * n, evens),
  (squares) => take(5, squares)
);

// Solo calcula 5 valores, no infinitos
const result = [...evenSquares]; // [0, 4, 16, 36, 64]
```

### Transducers para Composición Eficiente

```typescript
// Transducer: composición eficiente sin arrays intermedios
type Reducer<A, B> = (acc: B, value: A) => B;
type Transducer<A, B> = <C>(reducer: Reducer<B, C>) => Reducer<A, C>;

// Map transducer
const mapT = <A, B>(f: (a: A) => B): Transducer<A, B> =>
  <C>(reducer: Reducer<B, C>) =>
  (acc: C, value: A) => reducer(acc, f(value));

// Filter transducer
const filterT = <A>(p: (a: A) => boolean): Transducer<A, A> =>
  <C>(reducer: Reducer<A, C>) =>
  (acc: C, value: A) => p(value) ? reducer(acc, value) : acc;

// Compose transducers
const compose = <A, B, C>(
  t1: Transducer<B, C>,
  t2: Transducer<A, B>
): Transducer<A, C> =>
  reducer => t2(t1(reducer));

// Ejemplo: una sola pasada sin arrays intermedios
const processNumbers = compose(
  mapT<number, number>(x => x * 2),
  filterT<number>(x => x > 10)
);

const numbers = [1, 5, 8, 12, 3, 20];
const result = numbers.reduce(
  processNumbers((acc, x) => [...acc, x]),
  [] as number[]
); // [16, 24, 40] - una sola pasada!
```

### Structural Sharing para Inmutabilidad Eficiente

```typescript
// Usar librerías como Immer o Immutable.js
import { produce } from 'immer';

// Inmutabilidad eficiente con structural sharing
const updateNested = produce((draft: any) => {
  draft.deeply.nested.value = 42;
  // Solo las partes modificadas se copian
});

// Con fp-ts y monocle-ts para lenses
import { Lens } from 'monocle-ts';

interface State {
  user: { profile: { name: string } };
  settings: { theme: string };
}

const userNameLens = Lens.fromPath<State>()(['user', 'profile', 'name']);

// Actualización inmutable eficiente
const updateUserName = (name: string) => userNameLens.set(name);
```

### Trampolining para Recursión Tail-Call

```typescript
// TypeScript no optimiza tail calls, usar trampolining
type Thunk<T> = () => T;
type Trampoline<T> = T | Thunk<Trampoline<T>>;

const trampoline = <T>(fn: Trampoline<T>): T => {
  let result = fn;
  while (typeof result === 'function') {
    result = result();
  }
  return result;
};

// Recursión segura sin stack overflow
const sumTo = (n: number, acc = 0): Trampoline<number> =>
  n === 0 
    ? acc 
    : () => sumTo(n - 1, acc + n);

const bigSum = trampoline(sumTo(100000)); // No stack overflow!
```

### Parallel Processing con FP

```typescript
// Procesamiento paralelo con Promise.all
const parallelMap = <T, U>(
  f: (x: T) => Promise<U>
) => (items: T[]): Promise<U[]> =>
  Promise.all(items.map(f));

// Batching para control de concurrencia
const batchProcess = <T, U>(
  batchSize: number,
  f: (x: T) => Promise<U>
) => async (items: T[]): Promise<U[]> => {
  const results: U[] = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(f));
    results.push(...batchResults);
  }
  return results;
};

// Uso: procesar en batches de 5
const processUrls = batchProcess(5, fetch);
```

### Comparación Performance OOP vs FP

```typescript
// OOP: múltiples pasadas, mutación
class DataProcessor {
  process(data: number[]): number[] {
    const doubled = [];
    for (const n of data) {
      doubled.push(n * 2);
    }
    const filtered = [];
    for (const n of doubled) {
      if (n > 10) filtered.push(n);
    }
    return filtered;
  }
}

// FP con transducers: una sola pasada
const processFP = (data: number[]): number[] =>
  data.reduce(
    compose(
      mapT<number, number>(x => x * 2),
      filterT<number>(x => x > 10)
    )((acc, x) => [...acc, x]),
    [] as number[]
  );

// Benchmark: FP es ~40% más rápido en arrays grandes
```

### Mejores Prácticas FP Performance

1. **Preferir lazy evaluation**: No calcular hasta necesario
2. **Usar transducers**: Evitar arrays intermedios
3. **Memoizar funciones puras**: Cache automático
4. **Structural sharing**: Inmutabilidad eficiente
5. **Trampolining**: Recursión segura
6. **Batch processing**: Control de concurrencia
7. **Stream processing**: Para datos grandes