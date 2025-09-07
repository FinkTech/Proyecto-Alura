# Tutorial Completo de n8n: Desde Cero hasta Workflows Avanzados

## Introducción

n8n es una plataforma de automatización de workflows de código abierto que permite conectar diferentes aplicaciones y servicios para automatizar procesos empresariales. En este tutorial aprenderás a crear workflows desde cero, usar expresiones avanzadas y gestionar la exportación/importación de workflows.

---

## 1. Creación de un Workflow desde Cero

### 1.1 Configuración Inicial

**Paso 1: Crear un nuevo workflow**

1. **Accede a n8n**: Utiliza n8n Cloud (recomendado para nuevos usuarios con prueba gratuita disponible) o tu instalación local
2. **Crear workflow**: 
   - Si ves la pantalla de bienvenida: selecciona "Start from Scratch"
   - Si estás en la lista de workflows: selecciona "Create Workflow"

**Paso 2: Renombrar el workflow**

Puedes renombrar un workflow haciendo clic en el nombre del workflow en la parte superior de la interfaz del Editor. Una vez que hayas renombrado el workflow, asegúrate de guardarlo.

### 1.2 Elección y Configuración de un Trigger

**¿Qué es un trigger?**

n8n proporciona dos formas de iniciar un workflow:
- Manualmente, seleccionando Execute Workflow
- Automáticamente, usando un nodo trigger como primer nodo. El nodo trigger ejecuta el workflow en respuesta a un evento externo, o basándose en tu configuración.

**Configurando un Schedule Trigger (ejemplo práctico)**

1. **Agregar el trigger**:
   - Selecciona "Add first step"
   - Busca "Schedule"
   - Selecciona "Schedule Trigger"

2. **Configurar el horario**:
   - **Trigger Interval**: Selecciona "Weeks"
   - **Weeks Between Triggers**: Introduce `1`
   - **Trigger on Weekdays**: Selecciona "Monday"
   - **Trigger at Hour**: Selecciona "9am"
   - **Trigger at Minute**: Introduce `0`

**Otros tipos de triggers comunes**:
- **Webhook**: Para APIs REST y webhooks externos
- **Manual Trigger**: Para ejecución manual
- **Cron**: Para horarios más complejos
- **Email Trigger**: Para procesar emails entrantes

### 1.3 Conexión de Nodos Básicos

#### A. Nodo HTTP Request

**Configuración básica**:

1. **Agregar nodo**:
   - Selecciona el conector "Add node" del nodo anterior
   - Busca "HTTP Request"
   - Selecciona el nodo

2. **Configuración**:
   - **Method**: GET, POST, PUT, DELETE según necesites
   - **URL**: La URL de la API que quieres llamar
   - **Headers**: Si necesitas headers personalizados
   - **Body**: Para métodos POST/PUT

**Ejemplo de configuración**:
```
Method: GET
URL: https://api.github.com/users/octocat
Headers: 
  User-Agent: n8n-workflow
  Accept: application/json
```

#### B. Nodo Function

**Propósito**: Ejecutar código JavaScript personalizado para manipular datos.

**Configuración**:

1. **Agregar nodo Function**:
   - Busca "Function" y selecciona el nodo
   - Se abre el editor de código

2. **Estructura básica**:
```javascript
// Acceder a los datos de entrada
const inputData = $input.all();

// Procesar datos
const outputData = inputData.map(item => {
  return {
    json: {
      // Tus transformaciones aquí
      originalId: item.json.id,
      processedName: item.json.name.toUpperCase(),
      timestamp: new Date().toISOString()
    }
  };
});

// Retornar los datos procesados
return outputData;
```

#### C. Nodo Google Sheets

**Configuración de credenciales**:

1. **Crear credenciales**:
   - En el dropdown "Credential", selecciona "Create new credential"
   - Sigue el proceso de OAuth con Google
   - Autoriza el acceso a Google Sheets

2. **Configurar operación**:
   - **Operation**: Read, Append, Update, Delete
   - **Spreadsheet ID**: ID de tu hoja de cálculo
   - **Sheet**: Nombre de la hoja
   - **Range**: Rango de celdas (ej: A1:C10)

**Ejemplo - Leer datos**:
```
Operation: Read
Spreadsheet ID: 1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms
Sheet: Sheet1
Range: A2:E
```

**Ejemplo - Escribir datos**:
```
Operation: Append
Spreadsheet ID: 1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms
Sheet: Sheet1
Range: A:E
Values: [Usar expresiones para mapear datos]
```

#### D. Nodo Email (Send Email)

**Configuración SMTP**:

1. **Credenciales**:
   - **Host**: smtp.gmail.com (para Gmail)
   - **Port**: 587 (TLS) o 465 (SSL)
   - **Username**: tu email
   - **Password**: contraseña de aplicación

2. **Configurar email**:
   - **From Email**: remitente@dominio.com
   - **To Email**: destinatario@dominio.com
   - **Subject**: Asunto del email
   - **Text/HTML**: Contenido del mensaje

**Ejemplo de configuración**:
```
From: notificaciones@miempresa.com
To: {{ $json.userEmail }}
Subject: Reporte semanal - {{ $now.format('DD/MM/YYYY') }}
HTML: 
<h1>Reporte Semanal</h1>
<p>Estimado {{ $json.userName }},</p>
<p>Tu reporte está listo con {{ $json.totalItems }} elementos procesados.</p>
```

---

## 2. Uso de Expresiones y Nodos Function

### 2.1 Conceptos Fundamentales de Expresiones

**¿Qué son las expresiones?**

Las expresiones en n8n permiten acceder y manipular datos dinámicamente usando JavaScript. Se escriben entre llaves dobles: `{{ expresión }}`

### 2.2 Sintaxis Básica de Expresiones

#### Acceso a Datos

**Datos del nodo actual**:
```javascript
{{ $json.nombreCampo }}          // Campo específico
{{ $json["campo con espacios"] }} // Campo con espacios
{{ $json.usuario.nombre }}        // Objeto anidado
```

**Datos de nodos anteriores**:
```javascript
{{ $('nombre_nodo').first().json.campo }}     // Primer elemento
{{ $('nombre_nodo').all() }}                  // Todos los elementos
{{ $('nombre_nodo').item.json.campo }}        // Elemento actual
```

**Variables de fecha y tiempo**:
```javascript
{{ $now }}                    // Fecha/hora actual
{{ $today }}                  // Fecha actual (sin hora)
{{ $now.format('YYYY-MM-DD') }} // Formato personalizado
{{ $today.minus(7, 'days') }}   // Hace 7 días
```

#### Manipulación de Strings

```javascript
{{ $json.nombre.toUpperCase() }}           // Mayúsculas
{{ $json.email.toLowerCase() }}            // Minúsculas
{{ $json.texto.trim() }}                   // Eliminar espacios
{{ $json.nombre + ' ' + $json.apellido }}  // Concatenar
{{ $json.descripcion.substring(0, 100) }}  // Substring
{{ $json.tags.split(',') }}                // Dividir string
```

#### Operaciones Matemáticas

```javascript
{{ $json.precio * 1.21 }}              // Multiplicación (IVA)
{{ Math.round($json.promedio) }}        // Redondear
{{ Math.max($json.valor1, $json.valor2) }} // Máximo
{{ ($json.total / $json.cantidad).toFixed(2) }} // División con decimales
```

#### Operaciones con Arrays

```javascript
{{ $json.items.length }}                    // Longitud del array
{{ $json.productos.map(item => item.nombre) }} // Mapear valores
{{ $json.numeros.filter(n => n > 10) }}     // Filtrar elementos
{{ $json.precios.reduce((sum, price) => sum + price, 0) }} // Sumar elementos
```

### 2.3 Casos Prácticos con el Nodo Function

#### Ejemplo 1: Procesamiento de Datos de Ventas

```javascript
// Procesar datos de ventas y calcular métricas
const salesData = $input.all();
const processedSales = [];

salesData.forEach((sale, index) => {
  const saleData = sale.json;
  
  // Calcular totales
  const subtotal = saleData.quantity * saleData.unitPrice;
  const tax = subtotal * 0.21; // 21% IVA
  const total = subtotal + tax;
  
  // Categorizar por monto
  let category;
  if (total > 1000) {
    category = 'Alto';
  } else if (total > 500) {
    category = 'Medio';
  } else {
    category = 'Bajo';
  }
  
  processedSales.push({
    json: {
      id: saleData.id,
      customer: saleData.customerName,
      date: new Date(saleData.saleDate).toLocaleDateString('es-ES'),
      subtotal: subtotal.toFixed(2),
      tax: tax.toFixed(2),
      total: total.toFixed(2),
      category: category,
      quarter: `Q${Math.ceil(new Date(saleData.saleDate).getMonth() / 3)}`
    }
  });
});

return processedSales;
```

#### Ejemplo 2: Validación y Limpieza de Datos

```javascript
// Validar y limpiar datos de contactos
const contacts = $input.all();
const validContacts = [];
const errors = [];

contacts.forEach((contact, index) => {
  const data = contact.json;
  const cleanContact = {};
  
  // Validar email
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(data.email)) {
    errors.push({
      row: index + 1,
      error: 'Email inválido',
      email: data.email
    });
    return; // Saltar este contacto
  }
  
  // Limpiar y formatear datos
  cleanContact.name = data.name ? data.name.trim().replace(/\s+/g, ' ') : '';
  cleanContact.email = data.email.toLowerCase().trim();
  cleanContact.phone = data.phone ? data.phone.replace(/\D/g, '') : '';
  cleanContact.company = data.company ? data.company.trim() : '';
  
  // Validar campos requeridos
  if (!cleanContact.name || !cleanContact.email) {
    errors.push({
      row: index + 1,
      error: 'Campos requeridos faltantes',
      data: cleanContact
    });
    return;
  }
  
  // Agregar metadatos
  cleanContact.processedAt = new Date().toISOString();
  cleanContact.source = 'n8n-import';
  
  validContacts.push({
    json: cleanContact
  });
});

// Si hay errores, puedes acceder a ellos en el siguiente nodo
// usando $('Function').first().json.errors
return [
  ...validContacts,
  {
    json: {
      summary: {
        total: contacts.length,
        valid: validContacts.length,
        errors: errors.length,
        errorDetails: errors
      }
    }
  }
];
```

#### Ejemplo 3: Integración de APIs Múltiples

```javascript
// Combinar datos de múltiples fuentes
const userData = $('HTTP Request - Users').first().json;
const orderData = $('HTTP Request - Orders').all();
const productData = $('HTTP Request - Products').all();

// Crear un mapa de productos para búsqueda rápida
const productMap = {};
productData.forEach(item => {
  productMap[item.json.id] = item.json;
});

// Procesar órdenes del usuario
const userOrders = orderData
  .filter(order => order.json.userId === userData.id)
  .map(order => {
    const orderJson = order.json;
    
    // Enriquecer con datos de productos
    const enrichedItems = orderJson.items.map(item => ({
      ...item,
      productName: productMap[item.productId]?.name || 'Producto desconocido',
      category: productMap[item.productId]?.category || 'Sin categoría',
      totalPrice: item.quantity * item.unitPrice
    }));
    
    const orderTotal = enrichedItems.reduce((sum, item) => sum + item.totalPrice, 0);
    
    return {
      json: {
        orderId: orderJson.id,
        orderDate: orderJson.createdAt,
        status: orderJson.status,
        items: enrichedItems,
        itemCount: enrichedItems.length,
        total: orderTotal.toFixed(2),
        customer: {
          id: userData.id,
          name: userData.name,
          email: userData.email
        }
      }
    };
  });

return userOrders;
```

### 2.4 Expresiones Avanzadas

#### Condicionales en Expresiones

```javascript
{{ $json.status === 'active' ? 'Activo' : 'Inactivo' }}
{{ $json.score > 80 ? 'Excelente' : $json.score > 60 ? 'Bueno' : 'Mejorable' }}
```

#### Trabajar con Fechas (Luxon)

```javascript
{{ $now.setZone('America/Argentina/Buenos_Aires').format('DD/MM/YYYY HH:mm') }}
{{ $json.fechaVencimiento.diff($now, 'days').days }} // Días hasta vencimiento
{{ $now.startOf('month').format('YYYY-MM-DD') }}     // Primer día del mes
```

#### Formateo de Números

```javascript
{{ parseFloat($json.precio).toLocaleString('es-AR', {
  style: 'currency', 
  currency: 'ARS' 
}) }}
```

---

## 3. Exportar e Importar Workflows

### 3.1 Métodos de Exportación

#### A. Desde la Interfaz Web

Desde la barra de navegación superior, selecciona los tres puntos en la esquina superior derecha para ver las siguientes opciones:
- Download: Descarga tu workflow actual como archivo JSON a tu computadora
- Import from URL: Importa workflow JSON desde una URL
- Import from File: Importa un workflow como archivo JSON desde tu computadora

**Paso a paso para exportar**:

1. **Abrir menú de exportación**:
   - En tu workflow, haz clic en los tres puntos (⋯) en la esquina superior derecha
   - Selecciona "Download"

2. **Guardar archivo**:
   - El archivo se descargará como `workflow-name.json`
   - Guárdalo en una ubicación segura

#### B. Copy-Paste (Exportación Parcial)

Puedes copiar y pegar un workflow o partes del mismo seleccionando los nodos que quieras copiar al portapapeles (Ctrl + c o cmd + c) y pegándolo (Ctrl + v o cmd + v) en la Interfaz del Editor.

**Para seleccionar múltiples nodos**:
1. Haz clic y arrastra para crear un área de selección
2. O mantén Ctrl/Cmd y haz clic en nodos individuales
3. Copia con Ctrl+C (Windows) o Cmd+C (Mac)

### 3.2 Formato JSON de Workflows

#### Estructura Básica del JSON

```json
{
  "meta": {
    "instanceId": "unique-instance-id"
  },
  "nodes": [
    {
      "parameters": {},
      "id": "unique-node-id",
      "name": "Nombre del Nodo",
      "type": "n8n-nodes-base.nodeName",
      "typeVersion": 1,
      "position": [x, y],
      "credentials": {
        "api": {
          "id": "credential-id",
          "name": "Nombre de la credencial"
        }
      }
    }
  ],
  "connections": {
    "Nombre del Nodo": {
      "main": [
        [
          {
            "node": "Siguiente Nodo",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "version-uuid",
  "id": "workflow-id",
  "name": "Nombre del Workflow",
  "tags": [],
  "triggerCount": 1,
  "updatedAt": "2024-01-15T10:30:00.000Z",
  "createdAt": "2024-01-15T10:00:00.000Z"
}
```

#### Ejemplo Completo de Workflow JSON

```json
{
  "meta": {
    "instanceId": "0123456789abcdef"
  },
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "weeks"
            }
          ]
        }
      },
      "id": "schedule-trigger-id",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [200, 300]
    },
    {
      "parameters": {
        "url": "https://api.github.com/users/n8n-io",
        "options": {}
      },
      "id": "http-request-id",
      "name": "HTTP Request",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [400, 300]
    },
    {
      "parameters": {
        "jsCode": "const userData = $input.all();\n\nreturn userData.map(item => ({\n  json: {\n    login: item.json.login,\n    name: item.json.name,\n    followers: item.json.followers,\n    processedAt: new Date().toISOString()\n  }\n}));"
      },
      "id": "function-id",
      "name": "Process Data",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [600, 300]
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [
        [
          {
            "node": "HTTP Request",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request": {
      "main": [
        [
          {
            "node": "Process Data",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": true,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "workflow-version-uuid",
  "id": "my-workflow-id",
  "name": "GitHub User Data Workflow",
  "tags": ["api", "github", "data-processing"],
  "triggerCount": 1,
  "updatedAt": "2024-01-15T10:30:00.000Z",
  "createdAt": "2024-01-15T10:00:00.000Z"
}
```

### 3.3 Importación de Workflows

#### A. Desde la Interfaz Web

**Importar desde archivo**:

1. **Acceder al menú de importación**:
   - Haz clic en los tres puntos (⋯) en la esquina superior derecha
   - Selecciona "Import from File"

2. **Seleccionar archivo**:
   - Navega hasta tu archivo `.json`
   - Selecciona y confirma la importación

3. **Verificar importación**:
   - El workflow aparecerá en tu canvas
   - Revisa que todos los nodos estén conectados correctamente
   - Configura las credenciales necesarias

**Importar desde URL**:

1. **Usar Import from URL**:
   - Selecciona "Import from URL" del menú
   - Pega la URL del archivo JSON (ej: GitHub raw file)
   - Confirma la importación

**Ejemplo de URL válida**:
```
https://raw.githubusercontent.com/n8n-io/self-hosted-ai-starter-kit/refs/heads/main/n8n/demo-data/workflows/srOnR8PAY3u4RSwb.json
```

#### B. Copy-Paste

**Para partes específicas del workflow**:

1. **Copiar JSON de nodos**:
   ```json
   [
     {
       "parameters": {
         "url": "https://api.ejemplo.com/data"
       },
       "name": "HTTP Request",
       "type": "n8n-nodes-base.httpRequest",
       "position": [400, 300]
     }
   ]
   ```

2. **Pegar en canvas**:
   - Haz clic en el canvas donde quieras los nodos
   - Presiona Ctrl+V (Windows) o Cmd+V (Mac)

### 3.4 Uso en Otro Entorno o VPS

#### Preparación del Workflow para Migración

**1. Exportar con preparación**:

```bash
# Si usas n8n CLI
n8n export:workflow --id=tu-workflow-id --output=./workflows/
n8n export:credentials --output=./credentials/
```

**2. Limpieza del JSON para migración**:

```javascript
// Script para limpiar credenciales del JSON
function cleanWorkflowForMigration(workflowJson) {
  // Crear copia del workflow
  const cleanWorkflow = JSON.parse(JSON.stringify(workflowJson));
  
  // Limpiar metadata específica de instancia
  delete cleanWorkflow.meta.instanceId;
  delete cleanWorkflow.id;
  delete cleanWorkflow.versionId;
  
  // Limpiar credenciales (mantener solo nombres)
  cleanWorkflow.nodes.forEach(node => {
    if (node.credentials) {
      Object.keys(node.credentials).forEach(credType => {
        // Mantener solo el nombre, eliminar ID específico
        node.credentials[credType] = {
          name: node.credentials[credType].name
        };
      });
    }
  });
  
  // Resetear fechas
  cleanWorkflow.createdAt = new Date().toISOString();
  cleanWorkflow.updatedAt = new Date().toISOString();
  
  return cleanWorkflow;
}
```

#### Configuración en Nuevo Entorno

**1. Preparar el entorno de destino**:

```bash
# Instalar n8n en el VPS
npm install -g n8n

# O con Docker
docker pull n8nio/n8n:latest
```

**2. Importar workflow y configurar credenciales**:

```bash
# Importar workflow
n8n import:workflow --input=./mi-workflow.json

# Las credenciales deben recrearse manualmente
# O importar si tienes backup seguro
n8n import:credentials --input=./credentials-backup.json
```

**3. Verificación post-importación**:

- **Revisar conexiones**: Verifica que todos los nodos estén conectados
- **Reconfigurar credenciales**: Recrea las credenciales necesarias
- **Probar ejecución**: Ejecuta el workflow manualmente
- **Activar triggers**: Activa los triggers si es necesario

#### Script de Migración Automática

```bash
#!/bin/bash
# migrate-workflow.sh

# Variables
SOURCE_WORKFLOW_ID="workflow-id"
DEST_N8N_URL="http://tu-nuevo-servidor:5678"

echo "Iniciando migración de workflow..."

# Exportar workflow del origen
n8n export:workflow --id=$SOURCE_WORKFLOW_ID --output=./temp/

# Limpiar archivo para migración
node clean-workflow.js ./temp/workflow.json

# Importar en destino (requiere configurar URL base)
export N8N_BASE_URL=$DEST_N8N_URL
n8n import:workflow --input=./temp/workflow-clean.json

echo "Migración completada. Verifica credenciales manualmente."
```

### 3.5 Buenas Prácticas

#### Seguridad

- Los archivos JSON de workflows exportados incluyen nombres e IDs de credenciales. Aunque los IDs no son sensibles, los nombres podrían serlo, dependiendo de cómo nombres tus credenciales. Elimina o anonimiza esta información del archivo JSON antes de compartir para proteger tus credenciales.

- Nunca incluyas credenciales reales en workflows compartidos
- Usa nombres genéricos para credenciales ("API_KEY" en lugar de "mi-api-super-secreta")

#### Versionado

```bash
# Estructura recomendada para versionado
workflows/
├── v1.0/
│   ├── data-processing-workflow.json
│   └── email-notifications-workflow.json
├── v1.1/
│   ├── data-processing-workflow.json
│   └── email-notifications-workflow.json
└── README.md
```

#### Documentación

```json
{
  "name": "Data Processing Workflow v2.1",
  "tags": ["production", "data-processing", "v2.1"],
  "description": "Procesa datos de ventas cada lunes a las 9:00 AM",
  "notes": [
    "Requiere credencial: google-sheets-prod",
    "Requiere credencial: smtp-notifications", 
    "Timeout configurado a 30 minutos",
    "Última actualización: 2024-01-15"
  ]
}
```

---

## Conclusión

Este tutorial te ha proporcionado una base sólida para trabajar con n8n:

1. **Creación de workflows**: Desde la configuración inicial hasta la conexión de nodos básicos
2. **Expresiones avanzadas**: Manipulación de datos con JavaScript y casos prácticos
3. **Gestión de workflows**: Exportación, importación y migración entre entornos

### Próximos Pasos

- Explora la [galería de workflows](https://n8n.io/workflows/) para inspirarte
- Practica con diferentes APIs y servicios
- Experimenta con nodos de IA para workflows más avanzados
- Considera la implementación en producción con Docker o VPS

### Recursos Adicionales

- [Documentación oficial de n8n](https://docs.n8n.io/)
- [Curso de n8n nivel 1](https://docs.n8n.io/courses/level-one/)
- [Comunidad de n8n](https://community.n8n.io/)
- [Ejemplos de expresiones](https://docs.n8n.io/code/expressions/)

¡Ahora tienes todas las herramientas necesarias para crear workflows poderosos y automatizar tus procesos con n8n!