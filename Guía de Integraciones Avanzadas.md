# Guía de Integraciones Avanzadas de n8n con Herramientas Populares

## 1. Introducción a las Integraciones Avanzadas

n8n ofrece soporte nativo para una amplia gama de aplicaciones populares, permitiendo crear workflows complejos y automatizaciones sofisticadas. Esta guía cubre las mejores prácticas para integrar Google Sheets, Slack, Shopify, HubSpot y Stripe en workflows de producción.

### Arquitectura General de Integraciones n8n

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Trigger Node  │───▶│  Processing     │───▶│   Action Node   │
│   (Webhook,     │    │  (Transform,    │    │   (Google,      │
│    Cron, etc.)  │    │   Filter, Set)  │    │    Slack, etc.) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       ▼                       │
         │              ┌─────────────────┐              │
         │              │   Credentials   │              │
         │              │   Management    │              │
         │              └─────────────────┘              │
         │                                                │
         └────────────────▶ Error Handling ◀─────────────┘
```

## 2. Google Sheets - Gestión Avanzada de Datos

### 2.1 Configuración de Credenciales Google

Las credenciales de Google Sheets requieren configuración OAuth2 para autenticación segura. Para configurar:

1. **Crear proyecto en Google Cloud Console**
2. **Habilitar Google Sheets API**
3. **Configurar OAuth2 credentials**
4. **Configurar scopes necesarios**: `https://www.googleapis.com/auth/spreadsheets`

### 2.2 Operaciones Disponibles

n8n soporta operaciones de documento y hoja de cálculo, incluyendo crear, actualizar, eliminar, agregar, remover y obtener documentos:

#### **Document Operations:**
- Create/Update spreadsheets
- Manage document properties

#### **Sheet Operations:**
- Append or Update Row, Append Row, Clear, Create, Delete, Delete Rows or Columns, Get Row(s), Update Row

### 2.3 Workflow Ejemplo: Sistema CRM con Google Sheets

```
Arquitectura del Workflow:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Webhook Trigger│───▶│   Data Mapper   │───▶│ Google Sheets   │
│  (Lead Capture) │    │   (Set Fields)  │    │  (Append Row)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                                               │
         ▼                                               ▼
┌─────────────────┐                          ┌─────────────────┐
│  Input Validator│                          │   Notification  │
│  (Filter Node)  │                          │   (Slack Node)  │
└─────────────────┘                          └─────────────────┘
```

#### **Configuración del Set Node (Data Mapper):**

```javascript
// Mapeo de campos para Google Sheets
{
  "timestamp": "{{ $now }}",
  "name": "{{ $json.form_data.name }}",
  "email": "{{ $json.form_data.email }}",
  "phone": "{{ $json.form_data.phone }}",
  "source": "{{ $json.utm_source || 'direct' }}",
  "status": "new_lead"
}
```

#### **Configuración del Google Sheets Node:**

```json
{
  "resource": "sheet",
  "operation": "appendRow",
  "documentId": "{{ $vars.SHEET_ID }}",
  "sheetName": "Leads",
  "values": [
    "{{ $json.timestamp }}",
    "{{ $json.name }}",
    "{{ $json.email }}",
    "{{ $json.phone }}",
    "{{ $json.source }}",
    "{{ $json.status }}"
  ]
}
```

### 2.4 Patrones Avanzados Google Sheets

#### **Pattern 1: Batch Processing con Rate Limiting**

```javascript
// En el Code node antes de Google Sheets
// Rate limiting para evitar límites de API
const delay = 1000; // 1 segundo entre requests
await new Promise(resolve => setTimeout(resolve, delay));

// Procesar en lotes
const batchSize = 100;
const batches = [];
for (let i = 0; i < items.length; i += batchSize) {
  batches.push(items.slice(i, i + batchSize));
}

return batches.map(batch => ({ json: { batch } }));
```

#### **Pattern 2: Deduplicación de Datos**

```javascript
// Verificar duplicados antes de insertar
{
  "resource": "sheet",
  "operation": "getRows",
  "documentId": "{{ $vars.SHEET_ID }}",
  "sheetName": "Leads",
  "filters": {
    "email": "{{ $json.email }}"
  }
}
```

## 3. Slack - Comunicación y Notificaciones Avanzadas

### 3.1 Configuración de Credenciales Slack

Slack requiere scopes específicos según las operaciones que vayas a usar. Los scopes mínimos incluyen:
- `chat:write` - Para enviar mensajes
- `channels:read` - Para leer información de canales
- `users:read` - Para obtener información de usuarios

### 3.2 Operaciones Disponibles

n8n soporta operaciones completas de Channel, File, Message, Reaction, Star, User, y User Group.

### 3.3 Workflow Ejemplo: Sistema de Alertas Inteligente

```
Sistema de Notificación Multi-nivel:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Error Trigger │───▶│  Severity Check │───▶│  Route Message  │
│   (Webhook)     │    │  (Switch Node)  │    │  (Based on Lvl) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                       │
                    ┌───────────┼───────────┐          │
                    ▼           ▼           ▼          ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │ Low: Public │ │Med: Private │ │High: @here  │
            │   Channel   │ │   Channel   │ │   + Email   │
            └─────────────┘ └─────────────┘ └─────────────┘
```

#### **Configuración del Switch Node para Severidad:**

```javascript
// Lógica de routing basada en severidad
{
  "rules": [
    {
      "output": 0,
      "condition": "{{ $json.severity === 'low' }}"
    },
    {
      "output": 1,
      "condition": "{{ $json.severity === 'medium' }}"
    },
    {
      "output": 2,
      "condition": "{{ $json.severity === 'high' }}"
    }
  ]
}
```

#### **Mensaje Dinámico Slack:**

```javascript
// Template para mensaje de alerta
{
  "channel": "{{ $json.severity === 'high' ? $vars.ALERTS_CHANNEL : $vars.GENERAL_CHANNEL }}",
  "text": "{{ $json.severity === 'high' ? '🚨 CRITICAL ALERT 🚨' : '⚠️ System Notice' }}",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Service:* {{ $json.service }}\n*Error:* {{ $json.message }}\n*Time:* {{ $now.toISO() }}"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "View Details" },
          "url": "{{ $json.dashboard_url }}"
        }
      ]
    }
  ]
}
```

### 3.4 Patrón Avanzado: Slack Bot Interactivo

```javascript
// Send and Wait for Approval pattern
{
  "resource": "message",
  "operation": "sendAndWaitForApproval",
  "channel": "#approvals",
  "message": "Request approval for deployment to production",
  "buttons": [
    {
      "text": "Approve ✅",
      "value": "approved"
    },
    {
      "text": "Reject ❌", 
      "value": "rejected"
    }
  ],
  "timeout": 3600 // 1 hora
}
```

## 4. Shopify - E-commerce Automation

### 4.1 Configuración Shopify

Shopify node soporta operaciones con Orders y Products, incluyendo crear, actualizar, eliminar y obtener órdenes y productos.

### 4.2 Workflow Ejemplo: Procesamiento de Órdenes Automático

```
Pipeline de Procesamiento de Órdenes:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│Shopify Webhook  │───▶│ Order Validator │───▶│  Inventory      │
│ (New Order)     │    │  (Filter Node)  │    │  Check (Code)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│Customer Segment │    │   Payment       │    │  Fulfillment    │
│  (HubSpot CRM)  │    │  Verification   │    │   (3PL API)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

#### **Validación de Órdenes:**

```javascript
// Código en Filter Node para validar órdenes
const order = $input.first().json;

// Validaciones
const isValidAmount = order.total_price > 0;
const hasProducts = order.line_items && order.line_items.length > 0;
const isValidEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(order.email);
const isPaid = order.financial_status === 'paid';

return isValidAmount && hasProducts && isValidEmail && isPaid;
```

#### **Actualización de Inventario:**

```javascript
// Code node para gestión de inventario
const lineItems = $input.first().json.line_items;
const inventory = [];

for (const item of lineItems) {
  // Verificar stock disponible
  const variant = await $request({
    method: 'GET',
    url: `https://${$vars.SHOPIFY_STORE}.myshopify.com/admin/api/2023-10/variants/${item.variant_id}.json`,
    headers: {
      'X-Shopify-Access-Token': $credentials.shopify.accessToken
    }
  });
  
  if (variant.variant.inventory_quantity < item.quantity) {
    // Triggear alerta de stock bajo
    inventory.push({
      product: item.title,
      requested: item.quantity,
      available: variant.variant.inventory_quantity,
      status: 'insufficient_stock'
    });
  }
}

return inventory;
```

### 4.3 Integración con Stripe para Pagos

```javascript
// Webhook Stripe -> Actualizar Order Shopify
{
  "resource": "order",
  "operation": "update",
  "orderId": "{{ $json.metadata.shopify_order_id }}",
  "updateFields": {
    "financial_status": "{{ $json.status === 'succeeded' ? 'paid' : 'pending' }}",
    "note": "Payment processed via Stripe: {{ $json.id }}"
  }
}
```

## 5. HubSpot - CRM y Marketing Automation

### 5.1 Configuración HubSpot

HubSpot node puede ser usado como herramienta de AI y soporta operaciones con Contact, Contact List, Company, Deal, Engagement, Form, y Ticket.

### 5.2 Workflow Ejemplo: Lead Scoring Automático

```
Sistema de Lead Scoring:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Form Submit   │───▶│  Lead Enricher  │───▶│  Scoring Logic  │
│   (HubSpot)     │    │   (External)    │    │   (Code Node)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Contact Update │    │   List Assign   │    │ Sales Notify    │
│   (HubSpot)     │    │   (HubSpot)     │    │   (Slack)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

#### **Lógica de Lead Scoring:**

```javascript
// Algoritmo de scoring en Code Node
const contact = $input.first().json;
let score = 0;

// Scoring por empresa
if (contact.company_size) {
  if (contact.company_size > 500) score += 20;
  else if (contact.company_size > 100) score += 15;
  else if (contact.company_size > 50) score += 10;
}

// Scoring por industria
const highValueIndustries = ['technology', 'finance', 'healthcare'];
if (highValueIndustries.includes(contact.industry?.toLowerCase())) {
  score += 15;
}

// Scoring por engagement
if (contact.email_engagement_score > 80) score += 10;
if (contact.website_visits > 5) score += 5;

// Scoring por urgencia (campos del formulario)
if (contact.timeline === 'immediate') score += 25;
else if (contact.timeline === '1-3_months') score += 15;

return [{
  json: {
    contactId: contact.id,
    score: score,
    grade: score >= 70 ? 'A' : score >= 50 ? 'B' : score >= 30 ? 'C' : 'D',
    assignTo: score >= 70 ? 'senior_sales' : 'junior_sales'
  }
}];
```

#### **Actualización de Contacto en HubSpot:**

```javascript
// Configuración del HubSpot node para actualizar
{
  "resource": "contact",
  "operation": "upsert",
  "email": "{{ $('Form Submit').item.json.email }}",
  "properties": {
    "lead_score": "{{ $json.score }}",
    "lead_grade": "{{ $json.grade }}",
    "last_scored_date": "{{ $now.toISO() }}",
    "assigned_sales_rep": "{{ $json.assignTo }}"
  }
}
```

### 5.3 Sincronización Bidireccional HubSpot-Shopify

```javascript
// Code node para sincronizar customer data
const hubspotContact = $input.first().json;
const shopifyCustomer = {
  first_name: hubspotContact.firstname,
  last_name: hubspotContact.lastname,
  email: hubspotContact.email,
  phone: hubspotContact.phone,
  tags: `lead_score_${hubspotContact.lead_score},lifecycle_${hubspotContact.lifecyclestage}`,
  metafields: [
    {
      namespace: "hubspot",
      key: "contact_id",
      value: hubspotContact.id.toString(),
      type: "single_line_text_field"
    }
  ]
};

return [{ json: shopifyCustomer }];
```

## 6. Stripe - Procesamiento de Pagos Avanzado

### 6.1 Configuración Stripe Trigger

Stripe es una suite de APIs de pago que potencia el comercio para negocios online. El trigger node permite capturar webhooks de eventos de Stripe.

### 6.2 Workflow Ejemplo: Gestión Completa de Pagos

```
Pipeline de Gestión de Pagos:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│Stripe Webhook   │───▶│  Event Router   │───▶│  Success Path   │
│ (All Events)    │    │  (Switch Node)  │    │  (Order Update) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       │                       ▼
┌─────────────────┐              │            ┌─────────────────┐
│  Failed Path    │◀─────────────┘            │   Refund Path   │
│ (Customer Notif)│                           │  (Process Return│
└─────────────────┘                           └─────────────────┘
```

#### **Event Router Configuration:**

```javascript
// Switch node para routing de eventos Stripe
{
  "rules": [
    {
      "output": 0, // Success path
      "condition": "{{ $json.type === 'payment_intent.succeeded' }}"
    },
    {
      "output": 1, // Failed path  
      "condition": "{{ $json.type === 'payment_intent.payment_failed' }}"
    },
    {
      "output": 2, // Refund path
      "condition": "{{ $json.type === 'charge.dispute.created' }}"
    }
  ]
}
```

#### **Procesamiento de Pago Exitoso:**

```javascript
// Code node para procesar pago exitoso
const stripeEvent = $input.first().json;
const paymentIntent = stripeEvent.data.object;

// Extraer metadata del customer
const customerData = {
  stripeCustomerId: paymentIntent.customer,
  orderId: paymentIntent.metadata.order_id,
  amount: paymentIntent.amount / 100, // Stripe usa centavos
  currency: paymentIntent.currency,
  paymentMethod: paymentIntent.payment_method,
  receiptUrl: paymentIntent.charges.data[0].receipt_url
};

// Preparar datos para actualizar sistemas downstream
return [{
  json: {
    ...customerData,
    status: 'paid',
    processedAt: new Date().toISOString(),
    nextActions: ['send_receipt', 'update_crm', 'fulfill_order']
  }
}];
```

### 6.3 Manejo de Disputas y Chargebacks

```javascript
// Workflow para manejo de disputas
{
  "resource": "webhook",
  "events": ["charge.dispute.created"],
  "processing": {
    "disputeAmount": "{{ $json.data.object.amount / 100 }}",
    "reason": "{{ $json.data.object.reason }}",
    "status": "{{ $json.data.object.status }}",
    "orderId": "{{ $json.data.object.metadata.order_id }}"
  },
  "actions": [
    "notify_finance_team",
    "flag_customer_account", 
    "gather_supporting_evidence"
  ]
}
```

## 7. Manejo de Credenciales y Variables de Entorno

### 7.1 Mejores Prácticas para Credenciales

#### **Uso de Variables de Entorno:**

```bash
# Configuración segura de variables
N8N_ENCRYPTION_KEY=your-32-character-encryption-key
GOOGLE_OAUTH_CLIENT_ID=your-client-id
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret
SLACK_BOT_TOKEN=xoxb-your-bot-token
SHOPIFY_STORE_URL=your-store.myshopify.com
HUBSPOT_API_KEY=your-hubspot-api-key
STRIPE_SECRET_KEY=sk_test_your-secret-key
```

#### **Acceso Seguro en Workflows:**

```javascript
// Usar variables en lugar de hardcodear
const apiKey = $vars.HUBSPOT_API_KEY;
const storeUrl = $vars.SHOPIFY_STORE_URL;

// Nunca exponer credenciales en logs
console.log('Processing request for store:', storeUrl.replace(/\/\/.*@/, '//***@'));
```

### 7.2 Rotación de Credenciales

```javascript
// Code node para validar expiración de tokens
const tokenExpiry = new Date($credentials.oauth.expires_at * 1000);
const now = new Date();
const hoursUntilExpiry = (tokenExpiry - now) / (1000 * 60 * 60);

if (hoursUntilExpiry < 24) {
  // Trigger workflow de renovación
  return [{
    json: {
      action: 'refresh_token',
      service: 'google_sheets',
      urgency: hoursUntilExpiry < 1 ? 'critical' : 'warning'
    }
  }];
}
```

## 8. Patrones de Workflows Complejos

### 8.1 Patrón: Fan-out/Fan-in

```
Procesamiento Paralelo con Agregación:
          ┌─────────────────┐
          │   Input Data    │
          └─────────────────┘
                   │
           ┌───────┼───────┐
           ▼       ▼       ▼
   ┌─────────────┐ │ ┌─────────────┐
   │  Process A  │ │ │  Process B  │
   │ (Google)    │ │ │  (Slack)    │
   └─────────────┘ │ └─────────────┘
           │       │       │
           └───────▼───────┘
          ┌─────────────────┐
          │   Merge Node    │
          └─────────────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ Final Processing│
          └─────────────────┘
```

#### **Configuración del Merge Node:**

```javascript
// Merge configurado para esperar todas las ramas
{
  "mode": "waitForAll",
  "mergeByIndex": true,
  "timeout": 300 // 5 minutos
}
```

### 8.2 Patrón: Error Handling y Retry Logic

```javascript
// Code node con retry logic
async function executeWithRetry(operation, maxRetries = 3) {
  let attempt = 0;
  
  while (attempt < maxRetries) {
    try {
      const result = await operation();
      return result;
    } catch (error) {
      attempt++;
      
      if (attempt >= maxRetries) {
        // Log error y enviar a dead letter queue
        console.error(`Max retries reached: ${error.message}`);
        throw error;
      }
      
      // Exponential backoff
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Uso en workflow
const result = await executeWithRetry(async () => {
  return await $request({
    method: 'POST',
    url: 'https://api.external-service.com/data',
    json: $input.first().json
  });
});
```

### 8.3 Patrón: Circuit Breaker

```javascript
// Implementación de circuit breaker
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.threshold = threshold;
    this.timeout = timeout;
    this.failureCount = 0;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }
  
  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

## 9. Optimización de Performance

### 9.1 Técnicas de Optimización

#### **Batch Processing:**

```javascript
// Procesar en lotes para APIs con límites
const batchSize = 50;
const items = $input.all();
const batches = [];

for (let i = 0; i < items.length; i += batchSize) {
  const batch = items.slice(i, i + batchSize);
  batches.push({
    json: {
      batch: batch.map(item => item.json),
      batchNumber: Math.floor(i / batchSize) + 1,
      totalBatches: Math.ceil(items.length / batchSize)
    }
  });
}

return batches;
```

#### **Caching Strategy:**

```javascript
// Implementar caching con Workflow Static Data
const cacheKey = `user_${$json.userId}`;
const cache = $getWorkflowStaticData('global');
const cacheExpiry = 5 * 60 * 1000; // 5 minutos

// Verificar cache
if (cache[cacheKey] && (Date.now() - cache[cacheKey].timestamp < cacheExpiry)) {
  return [{ json: cache[cacheKey].data }];
}

// Si no hay cache, obtener datos
const userData = await $request({
  method: 'GET',
  url: `https://api.example.com/users/${$json.userId}`
});

// Guardar en cache
cache[cacheKey] = {
  data: userData,
  timestamp: Date.now()
};

return [{ json: userData }];
```

### 9.2 Rate Limiting

```javascript
// Rate limiting personalizado
class RateLimiter {
  constructor(maxRequests, timeWindow) {
    this.maxRequests = maxRequests;
    this.timeWindow = timeWindow;
    this.requests = [];
  }
  
  async waitForSlot() {
    const now = Date.now();
    
    // Limpiar requests antiguos
    this.requests = this.requests.filter(time => now - time < this.timeWindow);
    
    if (this.requests.length >= this.maxRequests) {
      const oldestRequest = Math.min(...this.requests);
      const waitTime = this.timeWindow - (now - oldestRequest);
      
      if (waitTime > 0) {
        await new Promise(resolve => setTimeout(resolve, waitTime));
      }
    }
    
    this.requests.push(now);
  }
}

// Uso en workflow
const limiter = new RateLimiter(100, 60000); // 100 requests por minuto
await limiter.waitForSlot();

// Proceder con la request
const response = await $request({
  method: 'POST',
  url: 'https://api.external-service.com/data',
  json: $input.first().json
});
```

### 9.3 Optimización de Memoria

```javascript
// Gestión eficiente de memoria para datasets grandes
function* processLargeDataset(data, chunkSize = 1000) {
  for (let i = 0; i < data.length; i += chunkSize) {
    yield data.slice(i, i + chunkSize);
  }
}

// Procesar en chunks
const allItems = $input.all();
const results = [];

for (const chunk of processLargeDataset(allItems, 500)) {
  const processed = chunk.map(item => ({
    ...item.json,
    processed: true,
    timestamp: Date.now()
  }));
  
  results.push(...processed);
  
  // Opcional: liberar memoria explícitamente
  if (global.gc) {
    global.gc();
  }
}

return results.map(item => ({ json: item }));
```

## 10. Prevención de Errores en Producción

### 10.1 Validación de Entrada

```javascript
// Schema de validación robusto
function validateInput(data, schema) {
  const errors = [];
  
  for (const [field, rules] of Object.entries(schema)) {
    const value = data[field];
    
    // Validar required
    if (rules.required && (value === undefined || value === null || value === '')) {
      errors.push(`Field '${field}' is required`);
      continue;
    }
    
    // Validar tipo
    if (value !== undefined && rules.type && typeof value !== rules.type) {
      errors.push(`Field '${field}' must be of type ${rules.type}`);
    }
    
    // Validar longitud
    if (rules.maxLength && value && value.length > rules.maxLength) {
      errors.push(`Field '${field}' exceeds maximum length of ${rules.maxLength}`);
    }
    
    // Validar formato email
    if (rules.format === 'email' && value && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      errors.push(`Field '${field}' must be a valid email address`);
    }
    
    // Validar expresiones regulares
    if (rules.pattern && value && !new RegExp(rules.pattern).test(value)) {
      errors.push(`Field '${field}' does not match required pattern`);
    }
  }
  
  return errors;
}

// Uso en workflow
const schema = {
  email: { required: true, type: 'string', format: 'email' },
  name: { required: true, type: 'string', maxLength: 100 },
  phone: { type: 'string', pattern: '^[+]?[1-9][\d\s\-()]+ },
  amount: { required: true, type: 'number' }
};

const inputData = $input.first().json;
const validationErrors = validateInput(inputData, schema);

if (validationErrors.length > 0) {
  throw new Error(`Validation failed: ${validationErrors.join(', ')}`);
}
```

### 10.2 Manejo de Timeouts

```javascript
// Wrapper para requests con timeout
async function requestWithTimeout(options, timeoutMs = 30000) {
  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => {
      reject(new Error(`Request timed out after ${timeoutMs}ms`));
    }, timeoutMs);
    
    $request(options)
      .then(response => {
        clearTimeout(timeout);
        resolve(response);
      })
      .catch(error => {
        clearTimeout(timeout);
        reject(error);
      });
  });
}

// Uso con diferentes timeouts según el servicio
const serviceTimeouts = {
  'google': 10000,   // 10 segundos
  'slack': 5000,     // 5 segundos  
  'shopify': 15000,  // 15 segundos
  'hubspot': 20000,  // 20 segundos
  'stripe': 30000    // 30 segundos
};

const service = $json.service || 'default';
const timeout = serviceTimeouts[service] || 30000;

const response = await requestWithTimeout({
  method: 'POST',
  url: $json.apiUrl,
  json: $json.payload
}, timeout);
```

### 10.3 Dead Letter Queue Pattern

```javascript
// Implementación de Dead Letter Queue
const DLQ_WEBHOOK_URL = $vars.DEAD_LETTER_QUEUE_URL;
const MAX_RETRIES = 3;

async function sendToDeadLetterQueue(originalData, error, retryCount) {
  const dlqPayload = {
    originalData,
    error: {
      message: error.message,
      stack: error.stack,
      timestamp: new Date().toISOString()
    },
    retryCount,
    workflow: {
      id: $workflow.id,
      name: $workflow.name
    },
    execution: {
      id: $execution.id,
      mode: $execution.mode
    }
  };
  
  try {
    await $request({
      method: 'POST',
      url: DLQ_WEBHOOK_URL,
      json: dlqPayload,
      headers: {
        'Content-Type': 'application/json',
        'X-Source': 'n8n-workflow'
      }
    });
  } catch (dlqError) {
    console.error('Failed to send to DLQ:', dlqError.message);
  }
}

// Uso en workflow con retry logic
let retryCount = 0;
const maxRetries = MAX_RETRIES;

while (retryCount < maxRetries) {
  try {
    // Intentar procesar
    const result = await processData($input.first().json);
    return [{ json: result }];
    
  } catch (error) {
    retryCount++;
    console.log(`Attempt ${retryCount} failed:`, error.message);
    
    if (retryCount >= maxRetries) {
      // Enviar a DLQ
      await sendToDeadLetterQueue($input.first().json, error, retryCount);
      throw new Error(`Failed after ${maxRetries} attempts. Sent to DLQ.`);
    }
    
    // Exponential backoff
    const delay = Math.pow(2, retryCount) * 1000;
    await new Promise(resolve => setTimeout(resolve, delay));
  }
}
```

## 11. Monitoreo y Observabilidad

### 11.1 Logging Estructurado

```javascript
// Función de logging estructurado
function logEvent(level, message, metadata = {}) {
  const logEntry = {
    timestamp: new Date().toISOString(),
    level: level.toUpperCase(),
    message,
    workflow: {
      id: $workflow.id,
      name: $workflow.name
    },
    execution: {
      id: $execution.id,
      mode: $execution.mode
    },
    node: {
      name: $node.name,
      type: $node.type
    },
    ...metadata
  };
  
  console.log(JSON.stringify(logEntry));
  
  // Opcional: enviar a servicio de logging externo
  if (level === 'error' || level === 'warn') {
    // Enviar a Slack, email, o servicio de monitoreo
  }
}

// Uso en workflow
logEvent('info', 'Processing customer order', {
  customerId: $json.customerId,
  orderValue: $json.totalAmount,
  products: $json.lineItems.length
});

try {
  // Procesar orden
  const result = await processOrder($json);
  
  logEvent('info', 'Order processed successfully', {
    orderId: result.id,
    processingTime: Date.now() - startTime
  });
  
} catch (error) {
  logEvent('error', 'Order processing failed', {
    error: error.message,
    customerId: $json.customerId,
    orderData: $json
  });
  throw error;
}
```

### 11.2 Métricas y Alerting

```javascript
// Recolección de métricas personalizadas
class MetricsCollector {
  constructor() {
    this.metrics = $getWorkflowStaticData('global').metrics || {
      counters: {},
      timers: {},
      gauges: {}
    };
  }
  
  incrementCounter(name, value = 1, tags = {}) {
    const key = `${name}_${JSON.stringify(tags)}`;
    this.metrics.counters[key] = (this.metrics.counters[key] || 0) + value;
    this.save();
  }
  
  recordTimer(name, duration, tags = {}) {
    const key = `${name}_${JSON.stringify(tags)}`;
    if (!this.metrics.timers[key]) {
      this.metrics.timers[key] = [];
    }
    this.metrics.timers[key].push(duration);
    this.save();
  }
  
  setGauge(name, value, tags = {}) {
    const key = `${name}_${JSON.stringify(tags)}`;
    this.metrics.gauges[key] = value;
    this.save();
  }
  
  save() {
    $getWorkflowStaticData('global').metrics = this.metrics;
  }
  
  getMetrics() {
    return this.metrics;
  }
}

// Uso en workflow
const metrics = new MetricsCollector();
const startTime = Date.now();

try {
  // Procesar datos
  const result = await processData($input.first().json);
  
  // Registrar métricas de éxito
  metrics.incrementCounter('workflow.success', 1, { 
    workflow: $workflow.name,
    node: $node.name 
  });
  
  metrics.recordTimer('workflow.duration', Date.now() - startTime, {
    workflow: $workflow.name
  });
  
} catch (error) {
  // Registrar métricas de error
  metrics.incrementCounter('workflow.error', 1, { 
    workflow: $workflow.name,
    error: error.constructor.name
  });
  
  throw error;
}
```

## 12. Casos de Uso Completos e Integrados

### 12.1 E-commerce Customer Journey Completo

```
Customer Journey Automation:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Lead Capture    │───▶│ Lead Nurturing  │───▶│ Purchase Event  │
│ (Google Forms)  │    │ (HubSpot + Slack│    │ (Shopify)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ CRM Update      │    │ Email Campaigns │    │ Payment Process │
│ (HubSpot)       │    │ (Automated)     │    │ (Stripe)        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Data Analytics  │    │ Retargeting     │    │ Post-Purchase   │
│ (Google Sheets) │    │ (Custom API)    │    │ (Support Flow)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

#### **Workflow Principal:**

```javascript
// Master workflow para customer journey
const customerJourney = {
  async processLead(leadData) {
    // 1. Validar y enriquecer datos del lead
    const enrichedLead = await this.enrichLeadData(leadData);
    
    // 2. Crear/actualizar contacto en HubSpot
    const hubspotContact = await this.createHubSpotContact(enrichedLead);
    
    // 3. Calcular lead score
    const leadScore = await this.calculateLeadScore(enrichedLead);
    
    // 4. Asignar a lista apropiada
    await this.assignToList(hubspotContact.id, leadScore);
    
    // 5. Notificar equipo de ventas si es lead calificado
    if (leadScore >= 70) {
      await this.notifySalesTeam(hubspotContact, leadScore);
    }
    
    return { hubspotContact, leadScore };
  },
  
  async processPurchase(orderData) {
    // 1. Validar orden
    const validatedOrder = await this.validateOrder(orderData);
    
    // 2. Procesar pago
    const paymentResult = await this.processPayment(validatedOrder);
    
    // 3. Actualizar inventario
    await this.updateInventory(validatedOrder.lineItems);
    
    // 4. Crear customer en HubSpot si no existe
    const customer = await this.createOrUpdateCustomer(validatedOrder);
    
    // 5. Generar factura
    const invoice = await this.generateInvoice(validatedOrder);
    
    // 6. Enviar confirmación
    await this.sendOrderConfirmation(customer, validatedOrder, invoice);
    
    return { order: validatedOrder, payment: paymentResult, customer };
  }
};
```

### 12.2 Automated Customer Support System

```
Support Ticket Automation:
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Slack Message   │───▶│ Intent Analysis │───▶│ Auto-Response   │
│ (Customer Req)  │    │ (AI/NLP)        │    │ (Contextual)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ HubSpot Ticket  │    │ Knowledge Base  │    │ Escalation      │
│ (Auto-Create)   │    │ Search          │    │ (If Needed)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

```javascript
// Sistema automatizado de soporte
const supportSystem = {
  async analyzeIntent(message) {
    // Análisis básico de intent (puede integrarse con AI)
    const intents = {
      'billing': ['bill', 'charge', 'payment', 'invoice', 'refund'],
      'technical': ['bug', 'error', 'not working', 'broken', 'issue'],
      'account': ['login', 'password', 'access', 'account', 'profile'],
      'general': ['help', 'question', 'info', 'how to', 'support']
    };
    
    const messageText = message.toLowerCase();
    
    for (const [intent, keywords] of Object.entries(intents)) {
      if (keywords.some(keyword => messageText.includes(keyword))) {
        return intent;
      }
    }
    
    return 'general';
  },
  
  async generateResponse(intent, customerData) {
    const responses = {
      billing: `Hi ${customerData.name}! I can help with billing questions. Let me pull up your account details...`,
      technical: `Thanks for reporting this ${customerData.name}. I'm creating a technical ticket and our team will investigate.`,
      account: `Hello ${customerData.name}! For account-related questions, I can help you right away.`,
      general: `Hi ${customerData.name}! How can I assist you today?`
    };
    
    return responses[intent] || responses.general;
  },
  
  async createTicket(customerData, message, intent) {
    const ticketData = {
      subject: `${intent.toUpperCase()}: ${message.substring(0, 50)}...`,
      content: message,
      pipeline: intent === 'technical' ? 'tech_support' : 'general_support',
      priority: intent === 'billing' ? 'high' : 'medium',
      source: 'slack_automation',
      customer_email: customerData.email
    };
    
    return await this.createHubSpotTicket(ticketData);
  }
};
```

## 13. Troubleshooting y Debugging

### 13.1 Debugging Avanzado

```javascript
// Sistema de debugging avanzado
class WorkflowDebugger {
  constructor() {
    this.debug = $vars.DEBUG_MODE === 'true';
    this.traces = [];
  }
  
  trace(checkpoint, data, metadata = {}) {
    if (!this.debug) return;
    
    const trace = {
      checkpoint,
      timestamp: new Date().toISOString(),
      node: $node.name,
      execution: $execution.id,
      data: this.sanitizeData(data),
      metadata
    };
    
    this.traces.push(trace);
    console.log(`[TRACE] ${checkpoint}:`, JSON.stringify(trace, null, 2));
  }
  
  sanitizeData(data) {
    // Remover datos sensibles para logging
    const sensitiveFields = ['password', 'token', 'key', 'secret', 'credential'];
    
    if (typeof data === 'object' && data !== null) {
      const sanitized = {};
      for (const [key, value] of Object.entries(data)) {
        if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
          sanitized[key] = '***REDACTED***';
        } else if (typeof value === 'object') {
          sanitized[key] = this.sanitizeData(value);
        } else {
          sanitized[key] = value;
        }
      }
      return sanitized;
    }
    
    return data;
  }
  
  dumpTraces() {
    return this.traces;
  }
}

// Uso en workflow
const debugger = new WorkflowDebugger();

debugger.trace('workflow_start', $input.first().json);

try {
  const processedData = await processData($input.first().json);
  debugger.trace('processing_complete', processedData);
  
  const result = await saveToDatabase(processedData);
  debugger.trace('database_save', { recordId: result.id });
  
} catch (error) {
  debugger.trace('error_occurred', { error: error.message });
  
  // En caso de error, enviar traces para análisis
  if (debugger.debug) {
    await sendDebugReport(debugger.dumpTraces(), error);
  }
  
  throw error;
}
```

### 13.2 Health Checks Automatizados

```javascript
// Sistema de health checks
const healthChecks = {
  async checkDatabaseConnection() {
    try {
      await $request({
        method: 'GET',
        url: `${$vars.DB_URL}/health`,
        timeout: 5000
      });
      return { status: 'healthy', service: 'database' };
    } catch (error) {
      return { status: 'unhealthy', service: 'database', error: error.message };
    }
  },
  
  async checkExternalAPIs() {
    const apis = [
      { name: 'shopify', url: `https://${$vars.SHOPIFY_STORE}.myshopify.com/admin/api/2023-10/shop.json` },
      { name: 'hubspot', url: 'https://api.hubapi.com/crm/v3/owners' },
      { name: 'slack', url: 'https://slack.com/api/auth.test' }
    ];
    
    const results = [];
    
    for (const api of apis) {
      try {
        await $request({
          method: 'GET',
          url: api.url,
          timeout: 10000,
          headers: this.getAuthHeaders(api.name)
        });
        results.push({ status: 'healthy', service: api.name });
      } catch (error) {
        results.push({ status: 'unhealthy', service: api.name, error: error.message });
      }
    }
    
    return results;
  },
  
  getAuthHeaders(service) {
    switch (service) {
      case 'shopify':
        return { 'X-Shopify-Access-Token': $credentials.shopify.accessToken };
      case 'hubspot':
        return { 'Authorization': `Bearer ${$credentials.hubspot.apiKey}` };
      case 'slack':
        return { 'Authorization': `Bearer ${$credentials.slack.botToken}` };
      default:
        return {};
    }
  },
  
  async runAllChecks() {
    const checks = await Promise.allSettled([
      this.checkDatabaseConnection(),
      this.checkExternalAPIs()
    ]);
    
    const results = checks.map(check => 
      check.status === 'fulfilled' ? check.value : { status: 'error', error: check.reason }
    ).flat();
    
    const unhealthyServices = results.filter(result => result.status !== 'healthy');
    
    if (unhealthyServices.length > 0) {
      // Enviar alerta
      await this.sendHealthAlert(unhealthyServices);
    }
    
    return {
      overall: unhealthyServices.length === 0 ? 'healthy' : 'degraded',
      services: results,
      timestamp: new Date().toISOString()
    };
  },
  
  async sendHealthAlert(unhealthyServices) {
    const message = `🚨 Health Check Alert\n\nUnhealthy services:\n${
      unhealthyServices.map(service => `• ${service.service}: ${service.error || 'Unknown error'}`).join('\n')
    }\n\nTime: ${new Date().toISOString()}`;
    
    // Enviar a Slack
    await $request({
      method: 'POST',
      url: 'https://slack.com/api/chat.postMessage',
      headers: {
        'Authorization': `Bearer ${$credentials.slack.botToken}`,
        'Content-Type': 'application/json'
      },
      json: {
        channel: $vars.ALERTS_CHANNEL,
        text: message
      }
    });
  }
};

// Ejecutar health check
return [{ json: await healthChecks.runAllChecks() }];
```

## 14. Conclusiones y Mejores Prácticas

### 14.1 Resumen de Mejores Prácticas

1. **Seguridad Primero**
   - Nunca hardcodear credenciales
   - Usar OAuth cuando esté disponible
   - Implementar rotación de tokens
   - Validar todas las entradas

2. **Reliability y Resilience**
   - Implementar retry logic con exponential backoff
   - Usar circuit breakers para servicios externos
   - Configurar timeouts apropiados
   - Implementar dead letter queues

3. **Performance y Escalabilidad**
   - Procesar en lotes cuando sea posible
   - Implementar caching inteligente
   - Usar rate limiting
   - Optimizar uso de memoria

4. **Observabilidad**
   - Logging estructurado
   - Métricas personalizadas
   - Health checks automatizados
   - Alerting proactivo

5. **Mantenibilidad**
   - Documentar workflows complejos
   - Usar naming conventions consistentes
   - Modularizar lógica compleja
   - Versionar workflows

### 14.2 Checklist de Pre-Producción

#### **Seguridad:**
- [ ] Credenciales configuradas correctamente
- [ ] Variables de entorno seguras
- [ ] Validación de entrada implementada
- [ ] Datos sensibles no en logs

#### **Performance:**
- [ ] Rate limiting configurado
- [ ] Timeouts establecidos
- [ ] Batch processing implementado
- [ ] Caching strategy definida

#### **Reliability:**
- [ ] Error handling completo
- [ ] Retry logic implementado
- [ ] Circuit breakers configurados
- [ ] Dead letter queue preparada

#### **Monitoring:**
- [ ] Logging estructurado activo
- [ ] Métricas recolectándose
- [ ] Alertas configuradas
- [ ] Health checks funcionando

#### **Testing:**
- [ ] Casos de prueba definidos
- [ ] Testing con datos reales
- [ ] Load testing completado
- [ ] Disaster recovery probado

### 14.3 Recursos Adicionales

#### **Documentación Oficial n8n:**
- [n8n Nodes Documentation](https://docs.n8n.io/integrations/)
- [n8n Expressions Guide](https://docs.n8n.io/code-examples/expressions/)
- [n8n GitHub Repository](https://github.com/n8n-io/n8n)

#### **APIs de Terceros:**
- [Google Sheets API](https://developers.google.com/sheets/api)
- [Slack API](https://api.slack.com/)
- [Shopify API](https://shopify.dev/api)
- [HubSpot API](https://developers.hubspot.com/docs/api/overview)
- [Stripe API](https://stripe.com/docs/api)

---

**Nota**: Esta guía se basa en la documentación oficial de n8n y las APIs de los servicios integrados. Mantén esta documentación actualizada con nuevas funcionalidades y cambios en las APIs de terceros.

**Última actualización**: Septiembre 2025  
**Versión del documento**: 2.0