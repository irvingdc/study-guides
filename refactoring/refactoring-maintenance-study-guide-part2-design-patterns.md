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
// ============================================
// EJEMPLO 1: Simple Factory Pattern
// ============================================

// Problema: Sistema de notificaciones con múltiples canales
// Necesitamos crear diferentes tipos de notificaciones basadas en preferencias del usuario

// 1. Definir la interfaz común para todos los productos
interface Notification {
  send(message: string, recipient: string): Promise<boolean>;
  validateRecipient(recipient: string): boolean;
  getType(): string;
  getCost(): number; // Costo por notificación
  getDeliveryTime(): string; // Tiempo estimado de entrega
}

// 2. Implementaciones concretas de diferentes tipos de notificaciones

class EmailNotification implements Notification {
  private readonly COST_PER_EMAIL = 0.001; // $0.001 por email
  
  async send(message: string, recipient: string): Promise<boolean> {
    if (!this.validateRecipient(recipient)) {
      console.error(`Invalid email address: ${recipient}`);
      return false;
    }
    
    // Simular envío de email
    console.log(`📧 Sending email to ${recipient}`);
    console.log(`Subject: Notification`);
    console.log(`Body: ${message}`);
    
    // Simular latencia de red
    await new Promise(resolve => setTimeout(resolve, 100));
    
    return true;
  }
  
  validateRecipient(recipient: string): boolean {
    // Validación básica de email
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(recipient);
  }
  
  getType(): string {
    return 'EMAIL';
  }
  
  getCost(): number {
    return this.COST_PER_EMAIL;
  }
  
  getDeliveryTime(): string {
    return 'Instant to 5 minutes';
  }
}

class SMSNotification implements Notification {
  private readonly COST_PER_SMS = 0.05; // $0.05 por SMS
  private readonly MAX_LENGTH = 160;
  
  async send(message: string, recipient: string): Promise<boolean> {
    if (!this.validateRecipient(recipient)) {
      console.error(`Invalid phone number: ${recipient}`);
      return false;
    }
    
    // Truncar mensaje si es muy largo
    const truncatedMessage = message.length > this.MAX_LENGTH
      ? message.substring(0, this.MAX_LENGTH - 3) + '...'
      : message;
    
    console.log(`📱 Sending SMS to ${recipient}`);
    console.log(`Message (${truncatedMessage.length} chars): ${truncatedMessage}`);
    
    // Simular latencia de red
    await new Promise(resolve => setTimeout(resolve, 200));
    
    return true;
  }
  
  validateRecipient(recipient: string): boolean {
    // Validación básica de número de teléfono (formato US)
    const phoneRegex = /^\+?1?\d{10,14}$/;
    return phoneRegex.test(recipient.replace(/[\s-()]/g, ''));
  }
  
  getType(): string {
    return 'SMS';
  }
  
  getCost(): number {
    return this.COST_PER_SMS;
  }
  
  getDeliveryTime(): string {
    return 'Usually instant';
  }
}

class PushNotification implements Notification {
  private readonly COST_PER_PUSH = 0.0001; // $0.0001 por push
  
  async send(message: string, recipient: string): Promise<boolean> {
    if (!this.validateRecipient(recipient)) {
      console.error(`Invalid device token: ${recipient}`);
      return false;
    }
    
    console.log(`🔔 Sending push notification to device ${recipient}`);
    console.log(`Message: ${message}`);
    
    // Simular latencia de red
    await new Promise(resolve => setTimeout(resolve, 50));
    
    return true;
  }
  
  validateRecipient(recipient: string): boolean {
    // Validación básica de token de dispositivo
    return recipient.length >= 32 && /^[a-zA-Z0-9]+$/.test(recipient);
  }
  
  getType(): string {
    return 'PUSH';
  }
  
  getCost(): number {
    return this.COST_PER_PUSH;
  }
  
  getDeliveryTime(): string {
    return 'Instant';
  }
}

class SlackNotification implements Notification {
  private readonly COST_PER_MESSAGE = 0.01; // $0.01 por mensaje
  private webhookUrl: string;
  
  constructor(webhookUrl?: string) {
    this.webhookUrl = webhookUrl || process.env.SLACK_WEBHOOK_URL || '';
  }
  
  async send(message: string, recipient: string): Promise<boolean> {
    if (!this.validateRecipient(recipient)) {
      console.error(`Invalid Slack channel: ${recipient}`);
      return false;
    }
    
    console.log(`💬 Sending Slack message to ${recipient}`);
    console.log(`Webhook: ${this.webhookUrl}`);
    console.log(`Message: ${message}`);
    
    // Simular llamada a API de Slack
    await new Promise(resolve => setTimeout(resolve, 150));
    
    return true;
  }
  
  validateRecipient(recipient: string): boolean {
    // Validación de canal de Slack (debe empezar con # o @)
    return /^[#@].+/.test(recipient);
  }
  
  getType(): string {
    return 'SLACK';
  }
  
  getCost(): number {
    return this.COST_PER_MESSAGE;
  }
  
  getDeliveryTime(): string {
    return 'Usually instant';
  }
}

// 3. Simple Factory - Centraliza la lógica de creación
class NotificationFactory {
  // Mapa de tipos registrados para hacer la factory extensible
  private static notificationTypes = new Map<string, () => Notification>();
  
  // Registrar tipos por defecto
  static {
    this.registerType('email', () => new EmailNotification());
    this.registerType('sms', () => new SMSNotification());
    this.registerType('push', () => new PushNotification());
    this.registerType('slack', () => new SlackNotification());
  }
  
  // Método principal de factory
  static createNotification(type: string): Notification {
    const creator = this.notificationTypes.get(type.toLowerCase());
    
    if (!creator) {
      throw new Error(`Unknown notification type: ${type}. 
        Available types: ${Array.from(this.notificationTypes.keys()).join(', ')}`);
    }
    
    return creator();
  }
  
  // Método para crear basado en configuración del usuario
  static createFromUserPreference(user: User): Notification {
    // Lógica de negocio para determinar el tipo de notificación
    if (user.preferences.pushEnabled && user.deviceToken) {
      return this.createNotification('push');
    } else if (user.preferences.smsEnabled && user.phone) {
      return this.createNotification('sms');
    } else if (user.preferences.emailEnabled && user.email) {
      return this.createNotification('email');
    } else if (user.preferences.slackEnabled && user.slackChannel) {
      return this.createNotification('slack');
    }
    
    // Fallback a email
    return this.createNotification('email');
  }
  
  // Método para crear múltiples notificaciones
  static createMultiChannel(types: string[]): Notification[] {
    return types.map(type => this.createNotification(type));
  }
  
  // Permitir registrar nuevos tipos dinámicamente
  static registerType(type: string, creator: () => Notification): void {
    this.notificationTypes.set(type.toLowerCase(), creator);
  }
  
  // Obtener información sobre tipos disponibles
  static getAvailableTypes(): string[] {
    return Array.from(this.notificationTypes.keys());
  }
  
  // Crear la notificación más económica
  static createMostEconomical(): Notification {
    const allTypes = this.getAvailableTypes();
    let cheapest: Notification | null = null;
    let lowestCost = Infinity;
    
    for (const type of allTypes) {
      const notification = this.createNotification(type);
      if (notification.getCost() < lowestCost) {
        lowestCost = notification.getCost();
        cheapest = notification;
      }
    }
    
    return cheapest || this.createNotification('email');
  }
}

// ============================================
// EJEMPLO 2: Factory Method Pattern
// ============================================

// El Factory Method permite a las subclases decidir qué clase instanciar

// 1. Producto abstracto
abstract class Document {
  abstract getType(): string;
  abstract render(): string;
  abstract save(): void;
  abstract validate(): boolean;
}

// 2. Productos concretos
class PDFDocument extends Document {
  private content: string = '';
  
  getType(): string {
    return 'PDF';
  }
  
  render(): string {
    return `<PDF>${this.content}</PDF>`;
  }
  
  save(): void {
    console.log('Saving PDF document...');
  }
  
  validate(): boolean {
    // Validar estructura PDF
    return true;
  }
  
  setContent(content: string): void {
    this.content = content;
  }
}

class WordDocument extends Document {
  private content: string = '';
  
  getType(): string {
    return 'DOCX';
  }
  
  render(): string {
    return `<DOCX>${this.content}</DOCX>`;
  }
  
  save(): void {
    console.log('Saving Word document...');
  }
  
  validate(): boolean {
    // Validar estructura DOCX
    return true;
  }
  
  setContent(content: string): void {
    this.content = content;
  }
}

class ExcelDocument extends Document {
  private data: any[][] = [];
  
  getType(): string {
    return 'XLSX';
  }
  
  render(): string {
    return `<XLSX>${JSON.stringify(this.data)}</XLSX>`;
  }
  
  save(): void {
    console.log('Saving Excel document...');
  }
  
  validate(): boolean {
    // Validar estructura XLSX
    return true;
  }
  
  setData(data: any[][]): void {
    this.data = data;
  }
}

// 3. Creator abstracto con Factory Method
abstract class Application {
  private documents: Document[] = [];
  
  // Factory Method - las subclases decidirán qué documento crear
  abstract createDocument(): Document;
  
  // Método que usa el factory method
  newDocument(): Document {
    // Llama al factory method que será implementado por subclases
    const doc = this.createDocument();
    this.documents.push(doc);
    
    console.log(`Created new ${doc.getType()} document`);
    
    // Operaciones comunes para todos los documentos
    this.configureDocument(doc);
    this.registerDocumentHandlers(doc);
    
    return doc;
  }
  
  openDocument(path: string): Document {
    // Lógica para abrir documento existente
    const doc = this.createDocument();
    console.log(`Opening ${doc.getType()} from ${path}`);
    return doc;
  }
  
  private configureDocument(doc: Document): void {
    // Configuración común para todos los documentos
    console.log(`Configuring ${doc.getType()} document...`);
  }
  
  private registerDocumentHandlers(doc: Document): void {
    // Registrar event handlers comunes
    console.log(`Registering handlers for ${doc.getType()}...`);
  }
  
  getDocuments(): Document[] {
    return this.documents;
  }
}

// 4. Creators concretos que implementan el Factory Method
class PDFApplication extends Application {
  createDocument(): Document {
    // Esta subclase decide crear PDFDocument
    return new PDFDocument();
  }
}

class WordApplication extends Application {
  createDocument(): Document {
    // Esta subclase decide crear WordDocument
    return new WordDocument();
  }
}

class ExcelApplication extends Application {
  createDocument(): Document {
    // Esta subclase decide crear ExcelDocument
    return new ExcelDocument();
  }
}

// ============================================
// EJEMPLO 3: Abstract Factory Pattern
// ============================================

// Abstract Factory crea familias de objetos relacionados

// 1. Definir interfaces para la familia de productos

// Familia de productos: Componentes UI
interface Button {
  render(): void;
  onClick(handler: () => void): void;
}

interface Input {
  render(): void;
  getValue(): string;
  setValue(value: string): void;
}

interface Checkbox {
  render(): void;
  isChecked(): boolean;
  setChecked(checked: boolean): void;
}

// 2. Implementaciones concretas para diferentes temas/plataformas

// Tema Light
class LightButton implements Button {
  private handler?: () => void;
  
  render(): void {
    console.log('🔲 Rendering light theme button');
  }
  
  onClick(handler: () => void): void {
    this.handler = handler;
  }
}

class LightInput implements Input {
  private value: string = '';
  
  render(): void {
    console.log('⬜ Rendering light theme input');
  }
  
  getValue(): string {
    return this.value;
  }
  
  setValue(value: string): void {
    this.value = value;
  }
}

class LightCheckbox implements Checkbox {
  private checked: boolean = false;
  
  render(): void {
    console.log('☐ Rendering light theme checkbox');
  }
  
  isChecked(): boolean {
    return this.checked;
  }
  
  setChecked(checked: boolean): void {
    this.checked = checked;
  }
}

// Tema Dark
class DarkButton implements Button {
  private handler?: () => void;
  
  render(): void {
    console.log('⬛ Rendering dark theme button');
  }
  
  onClick(handler: () => void): void {
    this.handler = handler;
  }
}

class DarkInput implements Input {
  private value: string = '';
  
  render(): void {
    console.log('⬛ Rendering dark theme input');
  }
  
  getValue(): string {
    return this.value;
  }
  
  setValue(value: string): void {
    this.value = value;
  }
}

class DarkCheckbox implements Checkbox {
  private checked: boolean = false;
  
  render(): void {
    console.log('■ Rendering dark theme checkbox');
  }
  
  isChecked(): boolean {
    return this.checked;
  }
  
  setChecked(checked: boolean): void {
    this.checked = checked;
  }
}

// 3. Abstract Factory interface
interface UIFactory {
  createButton(): Button;
  createInput(): Input;
  createCheckbox(): Checkbox;
  
  // Método para crear un formulario completo
  createForm(): Form;
}

// 4. Concrete Factories
class LightThemeFactory implements UIFactory {
  createButton(): Button {
    return new LightButton();
  }
  
  createInput(): Input {
    return new LightInput();
  }
  
  createCheckbox(): Checkbox {
    return new LightCheckbox();
  }
  
  createForm(): Form {
    // Crear un formulario con componentes del tema light
    return new Form(
      this.createButton(),
      this.createInput(),
      this.createCheckbox()
    );
  }
}

class DarkThemeFactory implements UIFactory {
  createButton(): Button {
    return new DarkButton();
  }
  
  createInput(): Input {
    return new DarkInput();
  }
  
  createCheckbox(): Checkbox {
    return new DarkCheckbox();
  }
  
  createForm(): Form {
    // Crear un formulario con componentes del tema dark
    return new Form(
      this.createButton(),
      this.createInput(),
      this.createCheckbox()
    );
  }
}

// 5. Cliente que usa Abstract Factory
class Form {
  constructor(
    private submitButton: Button,
    private nameInput: Input,
    private agreeCheckbox: Checkbox
  ) {}
  
  render(): void {
    console.log('=== Rendering Form ===');
    this.nameInput.render();
    this.agreeCheckbox.render();
    this.submitButton.render();
    console.log('===================');
  }
  
  setup(): void {
    this.submitButton.onClick(() => {
      if (this.agreeCheckbox.isChecked()) {
        console.log(`Submitting form for: ${this.nameInput.getValue()}`);
      } else {
        console.log('Please agree to terms');
      }
    });
  }
}

class Application {
  private uiFactory: UIFactory;
  
  constructor(theme: 'light' | 'dark') {
    // Decidir qué factory usar basado en el tema
    this.uiFactory = theme === 'dark' 
      ? new DarkThemeFactory() 
      : new LightThemeFactory();
  }
  
  createUI(): void {
    // Usar abstract factory para crear UI
    // No importa qué tema esté activo, el código es el mismo
    const button = this.uiFactory.createButton();
    const input = this.uiFactory.createInput();
    const checkbox = this.uiFactory.createCheckbox();
    
    button.render();
    input.render();
    checkbox.render();
  }
  
  createLoginForm(): Form {
    return this.uiFactory.createForm();
  }
  
  changeTheme(theme: 'light' | 'dark'): void {
    this.uiFactory = theme === 'dark' 
      ? new DarkThemeFactory() 
      : new LightThemeFactory();
    
    console.log(`Theme changed to ${theme}`);
  }
}

// ============================================
// USO DE LOS FACTORIES
// ============================================

// Tipos auxiliares
interface User {
  id: string;
  email?: string;
  phone?: string;
  deviceToken?: string;
  slackChannel?: string;
  preferences: {
    emailEnabled: boolean;
    smsEnabled: boolean;
    pushEnabled: boolean;
    slackEnabled: boolean;
  };
}

// 1. Uso de Simple Factory
async function demonstrateSimpleFactory() {
  console.log('\n=== SIMPLE FACTORY DEMO ===\n');
  
  // Crear notificaciones específicas
  const emailNotif = NotificationFactory.createNotification('email');
  await emailNotif.send('Hello via Email', 'user@example.com');
  
  const smsNotif = NotificationFactory.createNotification('sms');
  await smsNotif.send('Hello via SMS', '+1234567890');
  
  // Crear basado en preferencias del usuario
  const user: User = {
    id: 'user123',
    email: 'user@example.com',
    phone: '+1234567890',
    deviceToken: 'abc123def456',
    preferences: {
      emailEnabled: false,
      smsEnabled: false,
      pushEnabled: true,
      slackEnabled: false
    }
  };
  
  const userNotif = NotificationFactory.createFromUserPreference(user);
  console.log(`\nUser prefers: ${userNotif.getType()}`);
  
  // Crear la opción más económica
  const cheapest = NotificationFactory.createMostEconomical();
  console.log(`\nCheapest option: ${cheapest.getType()} at $${cheapest.getCost()}`);
  
  // Crear múltiples canales
  const multiChannel = NotificationFactory.createMultiChannel(['email', 'sms', 'push']);
  console.log(`\nCreated ${multiChannel.length} notification channels`);
  
  // Registrar un nuevo tipo dinámicamente
  class TeamsNotification implements Notification {
    async send(message: string, recipient: string): Promise<boolean> {
      console.log(`📣 Sending Teams message to ${recipient}: ${message}`);
      return true;
    }
    
    validateRecipient(recipient: string): boolean {
      return recipient.startsWith('@');
    }
    
    getType(): string { return 'TEAMS'; }
    getCost(): number { return 0.01; }
    getDeliveryTime(): string { return 'Instant'; }
  }
  
  NotificationFactory.registerType('teams', () => new TeamsNotification());
  const teamsNotif = NotificationFactory.createNotification('teams');
  await teamsNotif.send('Hello Teams!', '@channel');
}

// 2. Uso de Factory Method
function demonstrateFactoryMethod() {
  console.log('\n=== FACTORY METHOD DEMO ===\n');
  
  // Diferentes aplicaciones crean diferentes tipos de documentos
  const pdfApp = new PDFApplication();
  const wordApp = new WordApplication();
  const excelApp = new ExcelApplication();
  
  // Cada aplicación crea su tipo específico de documento
  const pdfDoc = pdfApp.newDocument();
  const wordDoc = wordApp.newDocument();
  const excelDoc = excelApp.newDocument();
  
  // Pero el código cliente no necesita saber los tipos específicos
  function processDocument(app: Application) {
    const doc = app.newDocument();
    doc.save();
    console.log(`Processed ${doc.getType()} document`);
  }
  
  processDocument(pdfApp);
  processDocument(wordApp);
  processDocument(excelApp);
}

// 3. Uso de Abstract Factory
function demonstrateAbstractFactory() {
  console.log('\n=== ABSTRACT FACTORY DEMO ===\n');
  
  // Crear aplicación con tema light
  let app = new Application('light');
  console.log('Light Theme:');
  app.createUI();
  
  const lightForm = app.createLoginForm();
  lightForm.render();
  
  // Cambiar a tema dark
  app.changeTheme('dark');
  console.log('\nDark Theme:');
  app.createUI();
  
  const darkForm = app.createLoginForm();
  darkForm.render();
  
  // El código cliente no necesita saber sobre temas específicos
  function createUserInterface(factory: UIFactory) {
    const button = factory.createButton();
    const input = factory.createInput();
    const checkbox = factory.createCheckbox();
    
    // Usar los componentes sin saber su tema
    button.render();
    input.render();
    checkbox.render();
  }
  
  createUserInterface(new LightThemeFactory());
  createUserInterface(new DarkThemeFactory());
}

// Ejecutar demos
// demonstrateSimpleFactory();
// demonstrateFactoryMethod();
// demonstrateAbstractFactory();
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