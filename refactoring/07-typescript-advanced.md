# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 7: TypeScript Avanzado
### Para Entrevista Senior Software Engineer

---

## 7. TypeScript Avanzado - Dominando el Sistema de Tipos

TypeScript es mucho más que "JavaScript con tipos". Su sistema de tipos es increíblemente poderoso y permite escribir código más seguro, expresivo y mantenible. Dominar TypeScript avanzado es esencial para un Senior Software Engineer.

### ¿Por qué TypeScript Avanzado es importante?

1. **Type Safety**: Detecta errores en tiempo de compilación, no en producción
2. **Mejor IntelliSense**: El IDE puede proveer mejor autocompletado y refactoring
3. **Documentación viva**: Los tipos documentan la intención del código
4. **Refactoring seguro**: Cambiar tipos detecta todos los lugares afectados
5. **Menos bugs**: Muchos errores comunes son imposibles con buenos tipos

### Conceptos clave de TypeScript:

- **Type Inference**: TypeScript deduce tipos cuando es posible
- **Structural Typing**: Los tipos son compatibles por estructura, no por nombre
- **Type Narrowing**: El compilador entiende cuando los tipos se vuelven más específicos
- **Generics**: Escribir código reutilizable que funciona con múltiples tipos
- **Union Types**: Un valor puede ser uno de varios tipos
- **Intersection Types**: Combinar múltiples tipos en uno

---

## 7.1 Utility Types

### Explicación Detallada

Los Utility Types son tipos predefinidos en TypeScript que transforman otros tipos de maneras útiles. Son fundamentales para escribir código DRY (Don't Repeat Yourself) y mantener los tipos sincronizados con el código.

#### Utility Types más importantes:

1. **Partial<T>**: Hace todas las propiedades opcionales
2. **Required<T>**: Hace todas las propiedades requeridas
3. **Readonly<T>**: Hace todas las propiedades de solo lectura
4. **Pick<T, K>**: Selecciona un subconjunto de propiedades
5. **Omit<T, K>**: Omite propiedades específicas
6. **Record<K, T>**: Crea un objeto con claves K y valores T
7. **ReturnType<T>**: Obtiene el tipo de retorno de una función
8. **Parameters<T>**: Obtiene los tipos de parámetros de una función

#### Cuándo usar Utility Types:

- **Actualizaciones parciales**: Partial para updates de objetos
- **Configuración con defaults**: Required vs Partial
- **Inmutabilidad**: Readonly para prevenir mutaciones
- **DTOs y APIs**: Pick/Omit para diferentes vistas de datos
- **Type-safe maps**: Record para objetos como diccionarios

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Partial, Required, y Readonly
// ============================================

// Tipo base
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  settings?: { theme: 'light' | 'dark' };
}

// PARTIAL<T> - Hace todas las propiedades opcionales
function updateUser(user: User, updates: Partial<User>): User {
  return { ...user, ...updates };
}

// Uso: puedes actualizar solo algunos campos
const user: User = { id: '1', name: 'John', email: 'j@j.com', role: 'user' };
const updated = updateUser(user, { name: 'Jane' }); // Solo actualiza name

// REQUIRED<T> - Hace todas las propiedades requeridas
type CompleteUser = Required<User>; // settings ya no es opcional

// READONLY<T> - Hace propiedades de solo lectura
type ImmutableUser = Readonly<User>;
const immutable: ImmutableUser = user;
// immutable.name = 'New'; // Error! Cannot assign to 'name'

// ============================================
// EJEMPLO 2: Pick, Omit, y Exclude
// ============================================

// PICK<T, K> - Selecciona propiedades específicas
type UserBasic = Pick<User, 'id' | 'name'>;
// UserBasic = { id: string; name: string }

// OMIT<T, K> - Excluye propiedades específicas  
type UserPublic = Omit<User, 'email' | 'settings'>;
// UserPublic = { id: string; name: string; role: 'admin' | 'user' }

// Uso práctico en APIs
function getPublicUser(user: User): UserPublic {
  const { email, settings, ...publicData } = user;
  return publicData;
}

// EXCLUDE<T, U> - Excluye tipos de una unión
type Role = 'admin' | 'user' | 'guest';
type NonAdmin = Exclude<Role, 'admin'>; // 'user' | 'guest'

// EXTRACT<T, U> - Extrae tipos de una unión
type AdminOnly = Extract<Role, 'admin'>; // 'admin'

// ============================================
// EJEMPLO 3: Record y Mapped Types
// ============================================

// RECORD<K, T> - Crea objetos con claves K y valores T
type Permissions = Record<'admin' | 'user', string[]>;
const perms: Permissions = {
  admin: ['read', 'write', 'delete'],
  user: ['read']
};

// Mapped Types - Transforma propiedades
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

type NullableUser = Nullable<User>;
// Todas las propiedades ahora pueden ser null

// Template literal types con mapped types
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getId: () => string; getName: () => string; ... }

// ============================================
// EJEMPLO 4: ReturnType y Parameters
// ============================================

// Función con tipo de retorno complejo
function processData(id: string, options: { format: boolean }) {
  return {
    id,
    data: [1, 2, 3],
    formatted: options.format
  };
}

// RETURNTYPE<T> - Extrae el tipo de retorno
type Result = ReturnType<typeof processData>;
// Result = { id: string; data: number[]; formatted: boolean }

// PARAMETERS<T> - Extrae los tipos de parámetros
type Params = Parameters<typeof processData>;
// Params = [string, { format: boolean }]

// Uso práctico: wrapper genérico
function cached<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) {
      cache.set(key, fn(...args));
    }
    return cache.get(key);
  };
}

// ============================================
// EJEMPLO 5: Conditional Types
// ============================================

// Tipos condicionales - T extends U ? X : Y
type IsString<T> = T extends string ? true : false;
type Test1 = IsString<string>;  // true
type Test2 = IsString<number>;  // false

// Inferencia con 'infer'
type Unpacked<T> = T extends (infer U)[] ? U : T;
type Elem = Unpacked<string[]>;  // string
type Same = Unpacked<number>;    // number

// Template Literal Types
type Method = 'GET' | 'POST';
type Path = '/users' | '/posts';
type Route = `${Method} ${Path}`;
// Route = 'GET /users' | 'GET /posts' | 'POST /users' | 'POST /posts'

// Uso práctico
type ApiResponse<T> = T extends { error: any }
  ? { success: false; error: string }
  : { success: true; data: T };

---

## 7.2 Generics Avanzados

### Explicación Detallada

Los generics permiten escribir código que funciona con múltiples tipos manteniendo type safety. Son fundamentales para crear librerías y componentes reutilizables.

#### Conceptos de Generics:

1. **Type Parameters**: Variables de tipo que se definen al usar el código
2. **Generic Constraints**: Limitar qué tipos pueden usarse
3. **Default Type Parameters**: Valores por defecto para generics
4. **Conditional Type Distribution**: Cómo se comportan los generics con unions
5. **Higher-Order Generics**: Generics que toman otros generics

#### Beneficios de Generics:

- **Reutilización**: Un código funciona con múltiples tipos
- **Type Safety**: Mantiene información de tipos
- **IntelliSense**: El IDE conoce los tipos exactos
- **Menos duplicación**: No necesitas escribir versiones para cada tipo

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Generic Constraints
// ============================================

// Constraint básico - T debe tener propiedad 'id'
interface HasId {
  id: string;
}

class Store<T extends HasId> {
  private items = new Map<string, T>();
  
  add(item: T): void {
    this.items.set(item.id, item); // TypeScript sabe que item.id existe
  }
  
  find(id: string): T | undefined {
    return this.items.get(id);
  }
}

// Uso con constraint
interface Product {
  id: string;
  name: string;
}

const store = new Store<Product>();
store.add({ id: '1', name: 'Laptop' }); // OK
// store.add({ name: 'Mouse' }); // Error! Falta 'id'

// ============================================
// EJEMPLO 2: Multiple Type Parameters
// ============================================

// Múltiples parámetros de tipo
class Pair<T, U> {
  constructor(
    public first: T,
    public second: U
  ) {}
  
  swap(): Pair<U, T> {
    return new Pair(this.second, this.first);
  }
}

const pair = new Pair<string, number>('hello', 42);
const swapped = pair.swap(); // Pair<number, string>

// Función genérica con múltiples parámetros
function map<T, U>(
  array: T[],
  fn: (item: T) => U
): U[] {
  return array.map(fn);
}

const numbers = [1, 2, 3];
const strings = map(numbers, n => n.toString()); // string[]

// ============================================
// EJEMPLO 3: Generic Functions con Type Inference
// ============================================

// TypeScript infiere los tipos automáticamente
function identity<T>(value: T): T {
  return value;
}

const str = identity('hello');  // T se infiere como string
const num = identity(42);       // T se infiere como number

// Constraint con keyof
function getProperty<T, K extends keyof T>(
  obj: T,
  key: K
): T[K] {
  return obj[key];
}

const person = { name: 'Alice', age: 30 };
const name = getProperty(person, 'name'); // string
const age = getProperty(person, 'age');   // number
// getProperty(person, 'invalid'); // Error!

// ============================================
// EJEMPLO 4: Utility Type Patterns
// ============================================

// DeepPartial - Hace todo parcial recursivamente
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface Config {
  server: {
    port: number;
    host: string;
  };
  database: {
    url: string;
  };
}

type PartialConfig = DeepPartial<Config>;
// Ahora puedes actualizar solo partes anidadas:
const update: PartialConfig = {
  server: { port: 3000 } // No necesitas 'host'
};

// ============================================
// EJEMPLO 5: Branded Types
// ============================================

// Branded types previenen mezclar tipos similares
type Brand<K, T> = K & { __brand: T };

type UserId = Brand<string, 'UserId'>;
type PostId = Brand<string, 'PostId'>;

function toUserId(id: string): UserId {
  return id as UserId;
}

function toPostId(id: string): PostId {
  return id as PostId;
}

// Ahora no puedes mezclar los tipos
function getUser(id: UserId) { /* ... */ }
function getPost(id: PostId) { /* ... */ }

const userId = toUserId('123');
const postId = toPostId('456');

getUser(userId);  // OK
// getUser(postId); // Error! PostId != UserId

---

## 7.3 Type Guards

### Explicación Detallada

Los Type Guards son funciones o expresiones que permiten a TypeScript refinar el tipo de una variable en un scope específico. Son esenciales para trabajar con union types y unknown types de manera segura.

#### Tipos de Type Guards:

1. **typeof guards**: Para tipos primitivos
2. **instanceof guards**: Para instancias de clases
3. **in guards**: Para verificar propiedades
4. **User-defined guards**: Funciones con predicado de tipo
5. **Assertion functions**: Funciones que aseguran un tipo

#### Por qué son importantes:

- **Type narrowing**: Refinan tipos en bloques de código
- **Eliminan casting**: No necesitas usar 'as' inseguro
- **Runtime safety**: Verifican tipos en tiempo de ejecución
- **Better IntelliSense**: El IDE conoce el tipo exacto después del guard

### Ejemplo Práctico con Explicación

```typescript
// ============================================
// EJEMPLO 1: Type Guards Básicos
// ============================================

// Union type que necesita type guards
type Response<T> = 
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

// Type guard con predicado de tipo
function isSuccess<T>(res: Response<T>): res is { status: 'success'; data: T } {
  return res.status === 'success';
}

// Uso del type guard
function handle<T>(response: Response<T>) {
  if (isSuccess(response)) {
    console.log(response.data); // TypeScript sabe que 'data' existe
  } else {
    console.log(response.error); // TypeScript sabe que 'error' existe
  }
}

// Guards nativos
function process(value: string | number) {
  if (typeof value === 'string') {
    console.log(value.toUpperCase()); // value es string
  } else {
    console.log(value.toFixed(2)); // value es number
  }
}

// ============================================
// EJEMPLO 2: Discriminated Unions
// ============================================

// Union con discriminador común
type Payment = 
  | { type: 'card'; number: string }
  | { type: 'paypal'; email: string }
  | { type: 'bank'; account: string };

// Pattern matching con switch
function processPayment(payment: Payment) {
  switch (payment.type) {
    case 'card':
      console.log(payment.number); // TypeScript sabe que existe
      break;
    case 'paypal':
      console.log(payment.email);
      break;
    case 'bank':
      console.log(payment.account);
      break;
    default:
      // Exhaustive check
      const _: never = payment;
      throw new Error('Unknown payment');
  }
}

// ============================================
// EJEMPLO 3: Assertion Functions
// ============================================

// Assertion function - asegura un tipo o lanza error
function assert(condition: any, msg?: string): asserts condition {
  if (!condition) {
    throw new Error(msg || 'Assertion failed');
  }
}

function assertIsString(value: unknown): asserts value is string {
  assert(typeof value === 'string', 'Not a string');
}

// Uso de assertions
function process(data: unknown) {
  assertIsString(data);
  // Ahora TypeScript sabe que data es string
  console.log(data.toUpperCase());
}

// Assertion con narrowing
function assertDefined<T>(val: T | undefined): asserts val is T {
  if (val === undefined) {
    throw new Error('Value is undefined');
  }
}

const maybeString: string | undefined = getValue();
assertDefined(maybeString);
// Ahora maybeString es string, no string | undefined

declare function getValue(): string | undefined;

// ============================================
// EJEMPLO 4: Narrowing con 'in' operator
// ============================================

// Narrowing con 'in'
type Bird = { fly(): void; layEggs(): void };
type Fish = { swim(): void; layEggs(): void };

function move(animal: Bird | Fish) {
  if ('fly' in animal) {
    animal.fly(); // TypeScript sabe que es Bird
  } else {
    animal.swim(); // TypeScript sabe que es Fish
  }
}

// instanceof para clases
class Car { drive() {} }
class Boat { sail() {} }

function operate(vehicle: Car | Boat) {
  if (vehicle instanceof Car) {
    vehicle.drive();
  } else {
    vehicle.sail();
  }
}

// ============================================
// EJEMPLO 5: Type Guards para Unknown
// ============================================

// Validación de datos externos
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value &&
    typeof (value as any).id === 'string' &&
    typeof (value as any).name === 'string'
  );
}

// Uso seguro con datos externos
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();
  
  if (!isUser(data)) {
    throw new Error('Invalid user data');
  }
  
  // Ahora data es User
  return data;
}

// Funciones auxiliares
declare function fetch(url: string): Promise<{ json(): Promise<unknown> }>;
```

### Mejores prácticas de TypeScript Avanzado:

1. **Prefiere type inference**: Deja que TypeScript infiera tipos cuando sea posible
2. **Usa utility types**: No reinventes la rueda, usa los tipos built-in
3. **Crea type guards reutilizables**: Centraliza la lógica de validación
4. **Evita 'any'**: Usa 'unknown' cuando no sepas el tipo
5. **Documenta tipos complejos**: Los tipos complejos necesitan explicación
6. **Usa branded types**: Para prevenir mezclar tipos similares
7. **Exhaustive checking**: Usa 'never' para garantizar que manejas todos los casos

TypeScript avanzado te permite escribir código más seguro y expresivo, detectando errores en tiempo de compilación que de otra manera aparecerían en producción.