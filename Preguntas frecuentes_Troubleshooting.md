# Guía de Troubleshooting para n8n

## 1. Errores Comunes en n8n

### "Could not find property option"
Este error suele ocurrir cuando:
- Una expresión intenta acceder a una propiedad que no existe en los datos de entrada
- Se referencia un nodo anterior que no se ha ejecutado correctamente
- Hay problemas con el mapeo de datos entre nodos

**Causas comunes:**
- Nodos anteriores no completaron su ejecución
- Estructura de datos incorrecta o inesperada
- Referencias a propiedades que no existen en el JSON de entrada

### Timeouts en Nodos
Los timeouts son frecuentes en:
- **HTTP Request nodes**: Cuando las API externas responden lentamente
- **Database nodes**: Consultas que tardan mucho tiempo
- **Code nodes**: Cuando el código tiene bucles infinitos o procesamiento intensivo

**Síntomas típicos:**
- Error "Request timeout" después de 30-60 segundos
- Nodos que se quedan "ejecutando" indefinidamente
- Workflows que fallan sin mensaje de error claro

### Problemas de Credenciales (Incorrectas o Caducadas)
Los errores de autenticación más frecuentes incluyen:
- **Error 401 (Unauthorized)**: Credenciales incorrectas o tokens expirados
- **Error 403 (Forbidden)**: Permisos insuficientes o scopes incorrectos
- **Error 429**: Límites de rate limiting excedidos

**Mensajes de error típicos:**
- "Forbidden - perhaps check your credentials"
- "Authentication failed"
- "Invalid API key or token"

### Errores en Expresiones o Nodos Function
Los nodos de código (Code/Function) presentan errores como:
- **"Code doesn't return items properly"**: El código no devuelve datos en el formato esperado
- **"A 'json' property isn't an object"**: La clave `json` no apunta a un objeto válido
- **"Cannot find module"**: Intentar importar módulos no disponibles
- **"'import' and 'export' may only appear at the top level"**: Uso incorrecto de ES6 modules

## 2. Soluciones y Pasos de Depuración

### Métodos para Identificar la Causa del Error

#### 1. Análisis Secuencial de Nodos
- Ejecutar el workflow nodo por nodo para identificar dónde falla
- Usar el botón "Execute Node" para probar nodos individuales
- Verificar los datos de salida de cada nodo antes de continuar

#### 2. Revisión de Datos de Entrada
- Verificar que los datos de entrada tengan la estructura esperada
- Usar el inspector de datos para examinar el JSON completo
- Comprobar que las propiedades referenciadas existan

#### 3. Validación de Expresiones
- Probar expresiones en el editor integrado de n8n
- Usar `console.log()` en nodos Code para debug
- Verificar la sintaxis de las expresiones JavaScript

### Uso de Herramientas de Depuración en n8n

#### Execution History
- **Acceso**: Settings > Execution History
- **Funcionalidad**: 
  - Ver historial completo de ejecuciones
  - Analizar datos de entrada y salida de cada nodo
  - Identificar patrones en los fallos
  - Revisar logs de errores detallados

#### Logs del Sistema
- **Activación**: Configurar nivel de log en variables de entorno
- **Variables importantes**:
  - `N8N_LOG_LEVEL=debug` para logs detallados
  - `N8N_LOG_OUTPUT=console` para ver logs en tiempo real

#### Debug Mode
- Activar desde Settings > General > Debug mode
- Proporciona información más detallada en los errores
- Muestra stack traces completos

### Ejemplos de Correcciones Aplicadas

#### Corrección de Error de Credenciales
```
Error original: "401 - Unauthorized"
Solución aplicada:
1. Verificar que las credenciales no hayan expirado
2. Regenerar API key si es necesario
3. Comprobar scopes y permisos
4. Usar el botón "Test" en la configuración de credenciales
```

#### Corrección de Timeout
```
Error original: "Request timeout after 30000ms"
Solución aplicada:
1. Aumentar timeout en Settings del nodo (Add Option > Timeout)
2. Implementar batching para reducir carga
3. Añadir Retry on Fail con intervalos adecuados
```

#### Corrección de Expresión Incorrecta
```
Error original: "Could not find property 'data' of undefined"
Solución aplicada:
1. Verificar existencia de la propiedad: {{ $json.data ? $json.data : 'default' }}
2. Usar optional chaining: {{ $json.data?.subproperty }}
3. Añadir validaciones: {{ Object.keys($json).includes('data') ? $json.data : null }}
```

## 3. Ejemplos de Código Correcto

### Fragmentos de Nodos Function Bien Estructurados

#### Nodo Code - Procesamiento de Arrays
```javascript
// Estructura correcta para procesar múltiples items
const items = $input.all();
const processedItems = [];

for (const item of items) {
  const processedData = {
    id: item.json.id,
    name: item.json.name?.toUpperCase() || 'Unknown',
    timestamp: new Date().toISOString()
  };
  
  processedItems.push({
    json: processedData
  });
}

return processedItems;
```

#### Nodo Code - Manejo de Errores
```javascript
// Manejo seguro de datos con validaciones
try {
  const inputData = $json;
  
  // Validar estructura de datos
  if (!inputData || typeof inputData !== 'object') {
    throw new Error('Input data is not a valid object');
  }
  
  // Procesar datos con validaciones
  const result = {
    processedAt: new Date().toISOString(),
    originalId: inputData.id || null,
    status: inputData.status || 'pending',
    data: {
      ...inputData,
      processed: true
    }
  };
  
  return [{
    json: result
  }];
  
} catch (error) {
  // Retornar error estructurado
  return [{
    json: {
      error: true,
      message: error.message,
      timestamp: new Date().toISOString()
    }
  }];
}
```

#### Nodo Code - Trabajar con Static Data
```javascript
// Usar static data para mantener estado entre ejecuciones
const staticData = getWorkflowStaticData('global');

// Inicializar contador si no existe
if (!staticData.counter) {
  staticData.counter = 0;
}

// Incrementar y usar
staticData.counter++;

return [{
  json: {
    executionNumber: staticData.counter,
    data: $json,
    timestamp: new Date().toISOString()
  }
}];
```

### Configuraciones Válidas de Nodos

#### Webhook Node
```json
{
  "httpMethod": "POST",
  "path": "webhook-endpoint",
  "responseMode": "responseNode",
  "options": {
    "rawBody": false,
    "responseData": "noData"
  }
}
```

**Configuración recomendada:**
- Usar HTTPS cuando sea posible
- Configurar autenticación si maneja datos sensibles
- Validar estructura de datos entrantes
- Configurar respuesta apropiada (200, 201, etc.)

#### HTTP Request Node
```json
{
  "url": "https://api.example.com/data",
  "method": "GET",
  "sendHeaders": true,
  "headerParameters": {
    "parameters": [
      {
        "name": "Authorization",
        "value": "Bearer {{$credentials.api_key}}"
      },
      {
        "name": "Content-Type",
        "value": "application/json"
      }
    ]
  },
  "options": {
    "timeout": 30000,
    "retry": {
      "enabled": true,
      "maxTries": 3,
      "waitBetween": 1000
    }
  }
}
```

**Mejores prácticas:**
- Siempre configurar timeout apropiado
- Usar retry para APIs inestables
- Implementar batching para grandes volúmenes
- Manejar códigos de error específicos

#### Cron Node
```json
{
  "triggerTimes": {
    "cronExpression": "0 9 * * 1-5"
  },
  "timezone": "Europe/Madrid"
}
```

**Configuraciones comunes:**
- `0 */6 * * *` - Cada 6 horas
- `0 9 * * 1-5` - Días laborables a las 9 AM
- `0 0 1 * *` - Primer día de cada mes
- `*/15 * * * *` - Cada 15 minutos

#### Error Handling Workflow
```javascript
// En el nodo Error Trigger
const errorData = $json;

const errorReport = {
  workflow: errorData.workflow.name,
  execution: errorData.execution.id,
  error: errorData.error.message,
  node: errorData.lastNodeExecuted,
  timestamp: new Date().toISOString(),
  data: errorData.data
};

// Enviar notificación o log
return [{ json: errorReport }];
```

## Recursos Adicionales

### Variables de Entorno Útiles
- `N8N_LOG_LEVEL=debug` - Logs detallados
- `NODE_FUNCTION_ALLOW_BUILTIN=*` - Permitir módulos built-in
- `N8N_REINSTALL_MISSING_PACKAGES=true` - Reinstalar paquetes faltantes

### Herramientas de Testing
- Usar el botón "Test" en configuraciones de credenciales
- Ejecutar nodos individualmente con "Execute Node"
- Utilizar Postman o curl para probar webhooks externamente

### Monitoreo y Alertas
- Configurar Error Workflows para notificaciones automáticas
- Usar Health Check endpoints para monitoreo
- Implementar logs estructurados para análisis posterior