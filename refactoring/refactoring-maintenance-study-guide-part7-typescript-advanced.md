# Guía de Estudio: Refactorización y Mantenimiento de Software
## Parte 7: TypeScript Avanzado
### Para Entrevista Senior Software Engineer - Zillow

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

// Tipo base de usuario
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
  role: 'admin' | 'user' | 'guest';
  settings?: UserSettings;
  createdAt: Date;
  updatedAt: Date;
}

interface UserSettings {
  theme: 'light' | 'dark';
  notifications: boolean;
  language: string;
}

// PARTIAL<T> - Útil para actualizaciones parciales
class UserService {
  private users = new Map<string, User>();
  
  // Partial permite actualizar solo algunos campos
  async updateUser(id: string, updates: Partial<User>): Promise<User> {
    const user = this.users.get(id);
    if (!user) {
      throw new Error('User not found');
    }
    
    // TypeScript sabe que todas las propiedades en 'updates' son opcionales
    const updatedUser: User = {
      ...user,
      ...updates,
      updatedAt: new Date() // Siempre actualizar timestamp
    };
    
    // Validación: No permitir cambiar ID
    if (updates.id && updates.id !== id) {
      throw new Error('Cannot change user ID');
    }
    
    this.users.set(id, updatedUser);
    return updatedUser;
  }
  
  // Ejemplo de uso con Partial anidado
  async updateUserSettings(
    id: string, 
    settings: Partial<UserSettings>
  ): Promise<User> {
    const user = this.users.get(id);
    if (!user) {
      throw new Error('User not found');
    }
    
    return this.updateUser(id, {
      settings: {
        ...user.settings,
        ...settings
      }
    });
  }
}

// REQUIRED<T> - Fuerza que todas las propiedades estén presentes
type CompleteUser = Required<User>;

// Útil para validación de datos completos
function validateCompleteProfile(user: any): user is CompleteUser {
  return (
    user.id !== undefined &&
    user.name !== undefined &&
    user.email !== undefined &&
    user.age !== undefined &&
    user.role !== undefined &&
    user.settings !== undefined && // settings ahora es requerido
    user.createdAt !== undefined &&
    user.updatedAt !== undefined
  );
}

// READONLY<T> - Previene mutaciones accidentales
type ImmutableUser = Readonly<User>;

class UserRepository {
  // Retorna usuarios inmutables para prevenir modificaciones externas
  async findById(id: string): Promise<ImmutableUser | null> {
    const user = await this.fetchUser(id);
    return user ? Object.freeze(user) : null;
  }
  
  private async fetchUser(id: string): Promise<User | null> {
    // Simulación de fetch desde DB
    return null;
  }
}

// Deep Readonly - Readonly recursivo
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

type DeeplyImmutableUser = DeepReadonly<User>;

// ============================================
// EJEMPLO 2: Pick, Omit, y Exclude
// ============================================

// PICK<T, K> - Seleccionar propiedades específicas
type UserBasicInfo = Pick<User, 'id' | 'name' | 'email'>;
type UserPublicProfile = Pick<User, 'name' | 'role' | 'createdAt'>;

// Útil para diferentes vistas de los mismos datos
class UserController {
  // Endpoint público - solo información básica
  async getPublicProfile(id: string): Promise<UserPublicProfile> {
    const user = await this.userService.findById(id);
    
    // Pick asegura que solo retornamos los campos permitidos
    const publicProfile: UserPublicProfile = {
      name: user.name,
      role: user.role,
      createdAt: user.createdAt
    };
    
    return publicProfile;
  }
  
  // Endpoint autenticado - más información
  async getUserInfo(id: string, requesterId: string): Promise<UserBasicInfo> {
    await this.checkPermission(requesterId, id);
    
    const user = await this.userService.findById(id);
    
    const userInfo: UserBasicInfo = {
      id: user.id,
      name: user.name,
      email: user.email
    };
    
    return userInfo;
  }
  
  private userService = new UserService();
  
  private async checkPermission(requesterId: string, targetId: string): Promise<void> {
    // Verificar permisos
  }
}

// OMIT<T, K> - Excluir propiedades específicas
type UserWithoutSensitiveData = Omit<User, 'email' | 'settings'>;
type UserCreateDTO = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;

// Útil para DTOs y transferencia de datos
class UserMapper {
  // Mapear a DTO sin campos autogenerados
  static toCreateDTO(input: any): UserCreateDTO {
    return {
      name: input.name,
      email: input.email,
      age: input.age,
      role: input.role || 'user',
      settings: input.settings
    };
  }
  
  // Mapear a respuesta sin datos sensibles
  static toPublicResponse(user: User): UserWithoutSensitiveData {
    const { email, settings, ...publicData } = user;
    return publicData;
  }
}

// EXCLUDE<T, U> - Excluir tipos de una unión
type UserRole = User['role']; // 'admin' | 'user' | 'guest'
type NonAdminRole = Exclude<UserRole, 'admin'>; // 'user' | 'guest'

// Útil para restricciones de tipos
function createRestrictedUser(data: {
  name: string;
  email: string;
  role: NonAdminRole; // No puede ser admin
}): User {
  return {
    id: generateId(),
    ...data,
    age: 0,
    createdAt: new Date(),
    updatedAt: new Date()
  };
}

// ============================================
// EJEMPLO 3: Record y Mapped Types
// ============================================

// RECORD<K, T> - Crear objetos con tipos específicos
type UserPermissions = Record<UserRole, string[]>;

const permissions: UserPermissions = {
  admin: ['read', 'write', 'delete', 'manage_users'],
  user: ['read', 'write'],
  guest: ['read']
};

// Record con tipos más complejos
type UserCache = Record<string, {
  user: User;
  cachedAt: Date;
  expiresAt: Date;
}>;

class CacheManager {
  private cache: UserCache = {};
  
  set(user: User, ttl: number = 3600000): void {
    const now = new Date();
    this.cache[user.id] = {
      user,
      cachedAt: now,
      expiresAt: new Date(now.getTime() + ttl)
    };
  }
  
  get(id: string): User | null {
    const cached = this.cache[id];
    
    if (!cached) return null;
    
    if (cached.expiresAt < new Date()) {
      delete this.cache[id];
      return null;
    }
    
    return cached.user;
  }
}

// Mapped Types personalizados
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

type NullableUser = Nullable<User>;

// Mapped type con transformación de propiedades
type Getters<T> = {
  [P in keyof T as `get${Capitalize<string & P>}`]: () => T[P];
};

type UserGetters = Getters<User>;
// Resultado:
// {
//   getId: () => string;
//   getName: () => string;
//   getEmail: () => string;
//   // etc...
// }

// ============================================
// EJEMPLO 4: ReturnType y Parameters
// ============================================

// Función compleja con tipo de retorno inferido
async function fetchUserWithPosts(userId: string) {
  const user = await fetchUser(userId);
  const posts = await fetchPosts(userId);
  
  return {
    ...user,
    posts,
    postCount: posts.length,
    lastPostDate: posts[0]?.createdAt || null
  };
}

// RETURNTYPE<T> - Capturar el tipo de retorno
type UserWithPosts = Awaited<ReturnType<typeof fetchUserWithPosts>>;

// Ahora podemos usar este tipo en otros lugares
class UserProfileService {
  async getCompleteProfile(userId: string): Promise<UserWithPosts> {
    return fetchUserWithPosts(userId);
  }
  
  formatProfile(profile: UserWithPosts): string {
    return `${profile.name} has ${profile.postCount} posts`;
  }
}

// PARAMETERS<T> - Capturar tipos de parámetros
type FetchUserParams = Parameters<typeof fetchUserWithPosts>;
// FetchUserParams es [string]

// Útil para crear wrappers o decoradores
function withCache<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  const cache = new Map<string, ReturnType<T>>();
  
  return (...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// ============================================
// EJEMPLO 5: Conditional Types
// ============================================

// Tipos condicionales básicos
type IsString<T> = T extends string ? true : false;

type Test1 = IsString<string>;  // true
type Test2 = IsString<number>;  // false

// Tipos condicionales más complejos
type ExtractArrayType<T> = T extends (infer U)[] ? U : never;

type StringArray = ExtractArrayType<string[]>;  // string
type NotArray = ExtractArrayType<number>;       // never

// Conditional types para APIs
type APIResponse<T> = T extends { error: string }
  ? { success: false; error: string }
  : { success: true; data: T };

function handleResponse<T>(response: T): APIResponse<T> {
  if (hasError(response)) {
    return { success: false, error: (response as any).error } as APIResponse<T>;
  }
  return { success: true, data: response } as APIResponse<T>;
}

function hasError(response: any): response is { error: string } {
  return response && typeof response.error === 'string';
}

// Template Literal Types
type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type APIEndpoint = '/users' | '/posts' | '/comments';

type APIRoute = `${HTTPMethod} ${APIEndpoint}`;
// Genera: 'GET /users' | 'GET /posts' | ... | 'DELETE /comments'

// Útil para type-safe routing
const routes: Record<APIRoute, Function> = {
  'GET /users': () => { /* handler */ },
  'POST /users': () => { /* handler */ },
  'GET /posts': () => { /* handler */ },
  // TypeScript fuerza que implementemos todas las combinaciones
  'POST /posts': () => { /* handler */ },
  'PUT /users': () => { /* handler */ },
  'PUT /posts': () => { /* handler */ },
  'PUT /comments': () => { /* handler */ },
  'DELETE /users': () => { /* handler */ },
  'DELETE /posts': () => { /* handler */ },
  'DELETE /comments': () => { /* handler */ },
  'GET /comments': () => { /* handler */ },
  'POST /comments': () => { /* handler */ }
};

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

// Constraint básico - T debe tener una propiedad 'id'
interface Identifiable {
  id: string | number;
}

class Repository<T extends Identifiable> {
  private items = new Map<string | number, T>();
  
  add(item: T): void {
    // TypeScript sabe que item tiene 'id' por el constraint
    this.items.set(item.id, item);
  }
  
  findById(id: T['id']): T | undefined {
    return this.items.get(id);
  }
  
  update(id: T['id'], updates: Partial<T>): T | undefined {
    const item = this.items.get(id);
    if (!item) return undefined;
    
    const updated = { ...item, ...updates };
    this.items.set(id, updated);
    return updated;
  }
  
  // Método que requiere constraint adicional
  findByProperty<K extends keyof T>(
    property: K,
    value: T[K]
  ): T[] {
    const results: T[] = [];
    
    for (const item of this.items.values()) {
      if (item[property] === value) {
        results.push(item);
      }
    }
    
    return results;
  }
}

// Uso con diferentes tipos que cumplen el constraint
interface Product extends Identifiable {
  id: string;
  name: string;
  price: number;
}

interface Order extends Identifiable {
  id: number;
  customerId: string;
  total: number;
}

const productRepo = new Repository<Product>();
const orderRepo = new Repository<Order>();

// TypeScript infiere los tipos correctamente
productRepo.add({ id: 'P1', name: 'Laptop', price: 999 });
orderRepo.add({ id: 1, customerId: 'C1', total: 999 });

// ============================================
// EJEMPLO 2: Multiple Type Parameters
// ============================================

// Mapa bidireccional con dos tipos genéricos
class BiMap<K, V> {
  private keyToValue = new Map<K, V>();
  private valueToKey = new Map<V, K>();
  
  set(key: K, value: V): void {
    // Limpiar mapeos anteriores si existen
    const oldValue = this.keyToValue.get(key);
    if (oldValue !== undefined) {
      this.valueToKey.delete(oldValue);
    }
    
    const oldKey = this.valueToKey.get(value);
    if (oldKey !== undefined) {
      this.keyToValue.delete(oldKey);
    }
    
    this.keyToValue.set(key, value);
    this.valueToKey.set(value, key);
  }
  
  getByKey(key: K): V | undefined {
    return this.keyToValue.get(key);
  }
  
  getByValue(value: V): K | undefined {
    return this.valueToKey.get(value);
  }
  
  deleteByKey(key: K): boolean {
    const value = this.keyToValue.get(key);
    if (value === undefined) return false;
    
    this.keyToValue.delete(key);
    this.valueToKey.delete(value);
    return true;
  }
  
  deleteByValue(value: V): boolean {
    const key = this.valueToKey.get(value);
    if (key === undefined) return false;
    
    this.valueToKey.delete(value);
    this.keyToValue.delete(key);
    return true;
  }
}

// Uso: mapeo entre IDs y UUIDs
const idMap = new BiMap<number, string>();
idMap.set(1, 'abc-123');
idMap.set(2, 'def-456');

console.log(idMap.getByKey(1));        // 'abc-123'
console.log(idMap.getByValue('abc-123')); // 1

// ============================================
// EJEMPLO 3: Generic Functions con Type Inference
// ============================================

// Función genérica que infiere tipos de los argumentos
function groupBy<T, K extends keyof T>(
  items: T[],
  key: K
): Record<string, T[]> {
  const groups: Record<string, T[]> = {};
  
  for (const item of items) {
    const groupKey = String(item[key]);
    
    if (!groups[groupKey]) {
      groups[groupKey] = [];
    }
    
    groups[groupKey].push(item);
  }
  
  return groups;
}

// TypeScript infiere los tipos automáticamente
const users: User[] = [
  { id: '1', name: 'Alice', role: 'admin', age: 30 } as User,
  { id: '2', name: 'Bob', role: 'user', age: 25 } as User,
  { id: '3', name: 'Charlie', role: 'user', age: 35 } as User
];

const usersByRole = groupBy(users, 'role');
// TypeScript sabe que usersByRole es Record<string, User[]>

// Función genérica más compleja con múltiples constraints
function mergeObjects<
  T extends object,
  U extends object,
  K extends keyof T & keyof U
>(
  obj1: T,
  obj2: U,
  conflictResolver?: (key: K, val1: T[K], val2: U[K]) => any
): T & U {
  const result = { ...obj1 } as T & U;
  
  for (const key in obj2) {
    if (key in obj1 && conflictResolver) {
      // Resolver conflictos si hay una función proporcionada
      (result as any)[key] = conflictResolver(
        key as any,
        (obj1 as any)[key],
        (obj2 as any)[key]
      );
    } else {
      (result as any)[key] = obj2[key];
    }
  }
  
  return result;
}

// ============================================
// EJEMPLO 4: Higher-Order Types
// ============================================

// Tipo que transforma las propiedades de otro tipo
type AsyncifyMethods<T> = {
  [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K] extends (...args: infer A) => infer R
      ? (...args: A) => Promise<R>
      : never
    : T[K];
};

// Clase original con métodos síncronos
class SyncService {
  getName(): string {
    return 'Service';
  }
  
  calculate(a: number, b: number): number {
    return a + b;
  }
  
  data: string = 'some data';
}

// Versión async de la clase
type AsyncService = AsyncifyMethods<SyncService>;
// AsyncService tendrá:
// {
//   getName(): Promise<string>;
//   calculate(a: number, b: number): Promise<number>;
//   data: string; // Las propiedades no-función permanecen igual
// }

// Factory genérico que crea proxies async
function makeAsync<T extends object>(service: T): AsyncifyMethods<T> {
  return new Proxy(service, {
    get(target, prop) {
      const value = target[prop as keyof T];
      
      if (typeof value === 'function') {
        return async (...args: any[]) => {
          // Simular operación async
          await new Promise(resolve => setTimeout(resolve, 100));
          return value.apply(target, args);
        };
      }
      
      return value;
    }
  }) as AsyncifyMethods<T>;
}

// ============================================
// EJEMPLO 5: Branded Types
// ============================================

// Branded types para prevenir mezclar tipos similares
type Brand<K, T> = K & { __brand: T };

type UserId = Brand<string, 'UserId'>;
type PostId = Brand<string, 'PostId'>;
type Email = Brand<string, 'Email'>;

// Funciones para crear branded types
function createUserId(id: string): UserId {
  return id as UserId;
}

function createPostId(id: string): PostId {
  return id as PostId;
}

function createEmail(email: string): Email {
  if (!email.includes('@')) {
    throw new Error('Invalid email');
  }
  return email as Email;
}

// Ahora no podemos mezclar accidentalmente los tipos
class BlogService {
  getPost(postId: PostId): Post | null {
    // TypeScript garantiza que recibimos un PostId, no un UserId
    return null;
  }
  
  getPostsByUser(userId: UserId): Post[] {
    // TypeScript garantiza que recibimos un UserId, no un PostId
    return [];
  }
  
  sendEmail(to: Email, subject: string): void {
    // TypeScript garantiza que 'to' es un Email válido
  }
}

// Esto causará error de compilación:
// const userId = createUserId('123');
// blogService.getPost(userId); // Error! UserId no es asignable a PostId

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
type ApiResponse<T> = 
  | { status: 'success'; data: T }
  | { status: 'error'; error: string; code: number }
  | { status: 'loading' };

// Type guard para cada variante
function isSuccess<T>(response: ApiResponse<T>): response is { status: 'success'; data: T } {
  return response.status === 'success';
}

function isError<T>(response: ApiResponse<T>): response is { status: 'error'; error: string; code: number } {
  return response.status === 'error';
}

function isLoading<T>(response: ApiResponse<T>): response is { status: 'loading' } {
  return response.status === 'loading';
}

// Uso de type guards
function handleApiResponse<T>(response: ApiResponse<T>): void {
  if (isSuccess(response)) {
    // TypeScript sabe que response tiene 'data'
    console.log('Success:', response.data);
  } else if (isError(response)) {
    // TypeScript sabe que response tiene 'error' y 'code'
    console.error(`Error ${response.code}: ${response.error}`);
  } else if (isLoading(response)) {
    // TypeScript sabe que response solo tiene 'status'
    console.log('Loading...');
  }
}

// ============================================
// EJEMPLO 2: Type Guards Complejos
// ============================================

// Tipos de dominio complejos
interface CreditCardPayment {
  type: 'credit_card';
  cardNumber: string;
  cvv: string;
  expiryDate: string;
}

interface PayPalPayment {
  type: 'paypal';
  email: string;
  password: string;
}

interface BankTransferPayment {
  type: 'bank_transfer';
  accountNumber: string;
  routingNumber: string;
  accountHolder: string;
}

type PaymentMethod = CreditCardPayment | PayPalPayment | BankTransferPayment;

// Type guards más sofisticados
class PaymentValidator {
  // Guard que también valida la estructura
  static isCreditCard(payment: PaymentMethod): payment is CreditCardPayment {
    return (
      payment.type === 'credit_card' &&
      'cardNumber' in payment &&
      'cvv' in payment &&
      'expiryDate' in payment &&
      this.isValidCardNumber(payment.cardNumber) &&
      this.isValidCVV(payment.cvv)
    );
  }
  
  static isPayPal(payment: PaymentMethod): payment is PayPalPayment {
    return (
      payment.type === 'paypal' &&
      'email' in payment &&
      'password' in payment &&
      this.isValidEmail(payment.email)
    );
  }
  
  static isBankTransfer(payment: PaymentMethod): payment is BankTransferPayment {
    return (
      payment.type === 'bank_transfer' &&
      'accountNumber' in payment &&
      'routingNumber' in payment &&
      'accountHolder' in payment
    );
  }
  
  private static isValidCardNumber(cardNumber: string): boolean {
    return /^\d{16}$/.test(cardNumber.replace(/\s/g, ''));
  }
  
  private static isValidCVV(cvv: string): boolean {
    return /^\d{3,4}$/.test(cvv);
  }
  
  private static isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

// Uso con type guards
function processPayment(payment: PaymentMethod): void {
  if (PaymentValidator.isCreditCard(payment)) {
    // TypeScript sabe que es CreditCardPayment y que es válido
    processCreditCardPayment(payment.cardNumber, payment.cvv, payment.expiryDate);
  } else if (PaymentValidator.isPayPal(payment)) {
    // TypeScript sabe que es PayPalPayment y que el email es válido
    processPayPalPayment(payment.email, payment.password);
  } else if (PaymentValidator.isBankTransfer(payment)) {
    // TypeScript sabe que es BankTransferPayment
    processBankTransfer(payment.accountNumber, payment.routingNumber);
  } else {
    // TypeScript sabe que esto nunca debería ejecutarse
    const _exhaustive: never = payment;
    throw new Error(`Unknown payment type: ${_exhaustive}`);
  }
}

// ============================================
// EJEMPLO 3: Assertion Functions
// ============================================

// Assertion function - lanza error si no se cumple la condición
function assertIsDefined<T>(value: T | null | undefined, message?: string): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(message || 'Value is null or undefined');
  }
}

function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== 'string') {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

function assertIsNumber(value: unknown): asserts value is number {
  if (typeof value !== 'number' || isNaN(value)) {
    throw new Error(`Expected valid number, got ${value}`);
  }
}

// Assertion function para arrays
function assertIsArray<T>(
  value: unknown,
  itemGuard?: (item: unknown) => item is T
): asserts value is T[] {
  if (!Array.isArray(value)) {
    throw new Error('Value is not an array');
  }
  
  if (itemGuard) {
    for (let i = 0; i < value.length; i++) {
      if (!itemGuard(value[i])) {
        throw new Error(`Invalid item at index ${i}`);
      }
    }
  }
}

// Uso de assertion functions
function processUserData(data: unknown): void {
  // Antes de la assertion, data es 'unknown'
  assertIsDefined(data, 'User data is required');
  // Después de la assertion, data es no-null
  
  assertIsObject(data);
  // Ahora data es un object
  
  assertHasProperty(data, 'name');
  assertIsString(data.name);
  // Ahora TypeScript sabe que data.name es string
  
  assertHasProperty(data, 'age');
  assertIsNumber(data.age);
  // Ahora TypeScript sabe que data.age es number
  
  // Podemos usar data con seguridad
  console.log(`User: ${data.name}, Age: ${data.age}`);
}

function assertIsObject(value: unknown): asserts value is object {
  if (typeof value !== 'object' || value === null) {
    throw new Error('Value is not an object');
  }
}

function assertHasProperty<K extends string>(
  obj: object,
  key: K
): asserts obj is object & Record<K, unknown> {
  if (!(key in obj)) {
    throw new Error(`Object does not have property '${key}'`);
  }
}

// ============================================
// EJEMPLO 4: Discriminated Unions
// ============================================

// Union type con discriminador común
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

// Type guards usando el discriminador
function isCircle(shape: Shape): shape is { kind: 'circle'; radius: number } {
  return shape.kind === 'circle';
}

function isRectangle(shape: Shape): shape is { kind: 'rectangle'; width: number; height: number } {
  return shape.kind === 'rectangle';
}

function isTriangle(shape: Shape): shape is { kind: 'triangle'; base: number; height: number } {
  return shape.kind === 'triangle';
}

// Función que usa exhaustive checking
function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      // TypeScript sabe que shape tiene 'radius'
      return Math.PI * shape.radius ** 2;
    
    case 'rectangle':
      // TypeScript sabe que shape tiene 'width' y 'height'
      return shape.width * shape.height;
    
    case 'triangle':
      // TypeScript sabe que shape tiene 'base' y 'height'
      return (shape.base * shape.height) / 2;
    
    default:
      // Exhaustive check - si agregamos un nuevo tipo, TypeScript avisará
      const _exhaustive: never = shape;
      throw new Error(`Unknown shape: ${_exhaustive}`);
  }
}

// ============================================
// EJEMPLO 5: Type Guards para Unknown
// ============================================

// Guards para validar datos externos (APIs, user input, etc.)
interface ValidationResult<T> {
  success: boolean;
  data?: T;
  errors?: string[];
}

class DataValidator {
  // Validar y transformar unknown a tipo específico
  static validateUser(input: unknown): ValidationResult<User> {
    const errors: string[] = [];
    
    if (!this.isObject(input)) {
      return { success: false, errors: ['Input must be an object'] };
    }
    
    // Validar cada campo
    if (!this.hasStringProperty(input, 'id')) {
      errors.push('Missing or invalid id');
    }
    
    if (!this.hasStringProperty(input, 'name')) {
      errors.push('Missing or invalid name');
    }
    
    if (!this.hasStringProperty(input, 'email')) {
      errors.push('Missing or invalid email');
    } else if (!this.isValidEmail(input.email)) {
      errors.push('Invalid email format');
    }
    
    if (!this.hasNumberProperty(input, 'age')) {
      errors.push('Missing or invalid age');
    } else if (input.age < 0 || input.age > 150) {
      errors.push('Age out of valid range');
    }
    
    if (!this.hasEnumProperty(input, 'role', ['admin', 'user', 'guest'])) {
      errors.push('Invalid role');
    }
    
    if (errors.length > 0) {
      return { success: false, errors };
    }
    
    // Si llegamos aquí, input es válido
    return {
      success: true,
      data: input as User
    };
  }
  
  private static isObject(value: unknown): value is object {
    return typeof value === 'object' && value !== null;
  }
  
  private static hasStringProperty<K extends string>(
    obj: object,
    key: K
  ): obj is object & Record<K, string> {
    return key in obj && typeof (obj as any)[key] === 'string';
  }
  
  private static hasNumberProperty<K extends string>(
    obj: object,
    key: K
  ): obj is object & Record<K, number> {
    return key in obj && typeof (obj as any)[key] === 'number';
  }
  
  private static hasEnumProperty<K extends string, T extends string>(
    obj: object,
    key: K,
    values: T[]
  ): obj is object & Record<K, T> {
    return key in obj && values.includes((obj as any)[key]);
  }
  
  private static isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

// Uso seguro con datos externos
async function fetchAndProcessUser(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();
  
  const validation = DataValidator.validateUser(data);
  
  if (!validation.success) {
    throw new Error(`Invalid user data: ${validation.errors?.join(', ')}`);
  }
  
  return validation.data;
}

// Funciones auxiliares para los ejemplos
declare function fetchUser(id: string): Promise<User>;
declare function fetchPosts(userId: string): Promise<Post[]>;
declare function generateId(): string;
declare function processCreditCardPayment(card: string, cvv: string, expiry: string): void;
declare function processPayPalPayment(email: string, password: string): void;
declare function processBankTransfer(account: string, routing: string): void;

interface Post {
  id: string;
  title: string;
  content: string;
  createdAt: Date;
}
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