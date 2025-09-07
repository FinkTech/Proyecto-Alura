# Guía Avanzada de Seguridad y Buenas Prácticas en n8n

## 1. Manejo Seguro de Credenciales y Variables de Entorno

### 1.1 Almacenamiento de Credenciales

n8n almacena todas las credenciales encriptadas en su base de datos y restringe el acceso a ellas por defecto. Sin embargo, es fundamental seguir las mejores prácticas para su gestión:

#### **Mejores Prácticas para Credenciales:**

- **Usar OAuth cuando sea posible**: n8n recomienda usar OAuth para aplicaciones de terceros que lo soporten. El protocolo OAuth permite a n8n solicitar acceso con alcance específico a recursos en tu cuenta de terceros sin tener que proporcionar credenciales a largo plazo directamente.

- **Limitar alcance de API Keys**: Como mejor práctica, si tu aplicación proporciona tal funcionalidad, n8n recomienda limitar el acceso de esa API key solo a los recursos que necesitas acceder dentro de n8n.

- **Gestión de credenciales externas**: Con la funcionalidad de secretos externos, puedes almacenar información sensible de credenciales en una bóveda externa, y hacer que n8n la cargue cuando sea necesario. Esto proporciona una capa extra de seguridad y permite gestionar credenciales usadas en múltiples entornos de n8n en un lugar central.

#### **Configuración de Secretos Externos:**

```yaml
# Ejemplo de configuración para secretos externos
N8N_EXTERNAL_SECRETS_PROVIDER=vault
N8N_EXTERNAL_SECRETS_VAULT_URL=https://vault.company.com
N8N_EXTERNAL_SECRETS_VAULT_TOKEN_PATH=/path/to/token
```

**⚠️ Nota importante**: Los nombres de secretos no pueden contener espacios, guiones u otros caracteres especiales. n8n soporta nombres de secretos que contienen caracteres alfanuméricos (a-z, A-Z, y 0-9), y guiones bajos.

### 1.2 Variables de Entorno de Seguridad

Para instancias auto-hospedadas, es crucial configurar correctamente las variables de entorno relacionadas con seguridad:

```bash
# Autenticación básica
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure_password_here

# Configuración de sesiones
N8N_SESSION_TIMEOUT=3600  # Tiempo de expiración de sesión en segundos

# Encriptación de datos
N8N_ENCRYPTION_KEY=your-32-character-encryption-key

# Configuración SSL/TLS
N8N_PROTOCOL=https
N8N_SSL_KEY=/path/to/private.key
N8N_SSL_CERT=/path/to/certificate.crt
```

## 2. Autenticación y Autorización en Nodos y APIs

### 2.1 Autenticación de Usuario

Se requiere un nombre de usuario y contraseña para autenticar en la aplicación, con MFA opcional para usos externos. SSO, SAML y LDAP están disponibles con el plan Enterprise de n8n.

#### **Configuración de Autenticación Robusta:**

1. **Autenticación Multifactor (MFA)**:
   - Activar MFA para todos los usuarios
   - Usar aplicaciones de autenticación como Google Authenticator o Authy

2. **Single Sign-On (SSO)**:
   - Configurar SSO para gestión centralizada de usuarios
   - Implementar SAML o LDAP según las necesidades organizacionales

3. **Control de Acceso Basado en Roles (RBAC)**:
   Los permisos RBAC avanzados están disponibles en todos los planes pagos para asegurar la gobernanza, incluyendo la designación de roles de usuario super administrador.

### 2.2 Seguridad en APIs

#### **Configuración Segura de la API Pública:**

```bash
# Deshabilitar API pública si no se usa
N8N_PUBLIC_API_DISABLED=true

# Si se usa, configurar autenticación robusta
N8N_API_KEY_ROTATION_ENABLED=true
N8N_API_RATE_LIMITING_ENABLED=true
```

#### **Autenticación en Nodos HTTP Request:**

```javascript
// Ejemplo de configuración segura para HTTP Request
{
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "httpBasicAuth",
  "headers": {
    "User-Agent": "n8n-workflow",
    "X-API-Version": "v1"
  }
}
```

## 3. Estrategias de Auditoría de Workflows

### 3.1 Logging y Monitoreo

n8n recopila y almacena todos los registros del servidor en una ubicación central. Los usuarios autorizados pueden consultar la información de registros según sea necesario para rastrear acciones a usuarios individuales. Mantenemos el historial de registros de auditoría y registros de actividad histórica durante al menos 12 meses, con al menos los últimos tres meses inmediatamente disponibles para análisis.

#### **Configuración de Auditoría:**

```bash
# Habilitar logging detallado
N8N_LOG_LEVEL=debug
N8N_LOG_OUTPUT=file
N8N_LOG_FILE=/var/log/n8n/audit.log

# Configuración de retención de ejecuciones
EXECUTIONS_DATA_MAX_AGE=336  # 14 días en horas
EXECUTIONS_DATA_PRUNE=true
```

### 3.2 Monitoreo de Ejecuciones

#### **Parámetros de Auditoría Recomendados:**

1. **Tracking de Ejecuciones:**
   - Registrar todas las ejecuciones de workflows
   - Monitorear fallos y errores
   - Identificar patrones de uso anómalos

2. **Auditoría de Accesos:**
   - Registrar accesos a credenciales
   - Monitorear cambios en workflows
   - Rastrear actividad de usuarios

```javascript
// Ejemplo de webhook para auditoría
{
  "workflowId": "{{ $workflow.id }}",
  "executionId": "{{ $execution.id }}",
  "userId": "{{ $user.id }}",
  "timestamp": "{{ $now }}",
  "status": "{{ $execution.status }}"
}
```

## 4. Protección de Datos Sensibles y Cumplimiento de Regulaciones

### 4.1 Encriptación de Datos

#### **En Tránsito:**
Cuando usas la aplicación web de n8n, encripta el tráfico entre tu cliente y los servicios de n8n en tránsito. Lo mismo aplica para el tráfico relacionado con la API pública o nodos trigger de webhook. n8n usa Cloudflare para gestionar y renovar certificados SSL.

#### **En Reposo:**
n8n encripta los datos del cliente en reposo en el volumen montado de tu instancia. n8n usa encriptación del lado del servidor de Azure Storage (usando AES256 y una implementación compatible con FIPS-140-2).

#### **Para Instancias Auto-hospedadas:**

```bash
# Configuración de proxy reverso con SSL
server {
    listen 443 ssl http2;
    server_name n8n.company.com;
    
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 4.2 Cumplimiento GDPR y SOC 2

n8n alinea su programa de seguridad con SOC 2, un marco estándar para el cumplimiento de seguridad. Esto significa que hemos implementado procesos y seguimos procedimientos que mantienen altos estándares de seguridad para los datos de nuestros clientes.

#### **Medidas de Cumplimiento:**

1. **Gestión de Datos Personales:**
   - Implementar políticas de retención de datos
   - Establecer procedimientos de eliminación de datos
   - Documentar flujos de datos personales

2. **Derechos del Usuario:**
   - Implementar mecanismos de acceso a datos
   - Proporcionar capacidades de corrección de datos
   - Facilitar la portabilidad de datos

## 5. Backup, Recuperación y Versionado Seguro

### 5.1 Estrategia de Backup

n8n almacena datos de clientes en una cuenta de producción segura en Azure, usando una combinación de Azure Blob Storage y bases de datos PostgreSQL. n8n automáticamente hace backup de todos los datos de clientes y del sistema diariamente para proteger contra pérdida catastrófica debido a eventos imprevistos que impacten todo el sistema.

#### **Configuración de Backup para Auto-hospedados:**

```bash
#!/bin/bash
# Script de backup automatizado

# Variables
BACKUP_DIR="/backup/n8n"
DB_NAME="n8n"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup de base de datos
pg_dump $DB_NAME | gzip > "$BACKUP_DIR/db_backup_$DATE.sql.gz"

# Backup de archivos de configuración
tar -czf "$BACKUP_DIR/config_backup_$DATE.tar.gz" /root/.n8n/

# Encriptar backups
gpg --cipher-algo AES256 --compress-algo 1 --s2k-mode 3 \
    --s2k-digest-algo SHA512 --s2k-count 65536 --symmetric \
    --output "$BACKUP_DIR/encrypted_backup_$DATE.gpg" \
    "$BACKUP_DIR/db_backup_$DATE.sql.gz"

# Limpieza de archivos temporales
rm "$BACKUP_DIR/db_backup_$DATE.sql.gz"

# Retener solo los últimos 30 días de backups
find $BACKUP_DIR -name "*.gpg" -mtime +30 -delete
```

### 5.2 Plan de Recuperación ante Desastres

n8n ha formulado un plan detallado de recuperación ante desastres que describe los roles, responsabilidades y procedimientos detallados para recuperar sistemas en caso de falla.

#### **Procedimiento de Restauración:**

1. **Identificación del Incidente**
2. **Evaluación del Impacto**
3. **Activación del Plan de Recuperación**
4. **Restauración de Servicios**
5. **Validación y Testing**

### 5.3 Control de Versiones de Workflows

```javascript
// Metadata para versionado de workflows
{
  "version": "1.2.3",
  "author": "usuario@empresa.com",
  "description": "Descripción de cambios",
  "changelog": [
    {
      "version": "1.2.3",
      "date": "2024-01-15",
      "changes": ["Mejoras de seguridad", "Optimización de performance"]
    }
  ]
}
```

## 6. Errores de Seguridad Comunes y Cómo Evitarlos

### 6.1 Errores Frecuentes

#### **1. Exposición de Credenciales en Logs:**

❌ **Error común:**
```javascript
console.log('API Response:', response);  // Puede contener tokens
```

✅ **Solución segura:**
```javascript
console.log('API Response status:', response.status);
// Nunca loguear respuestas completas que puedan contener credenciales
```

#### **2. Uso de HTTP en lugar de HTTPS:**

❌ **Error común:**
```bash
N8N_PROTOCOL=http  # Tráfico sin encriptar
```

✅ **Configuración segura:**
```bash
N8N_PROTOCOL=https
N8N_SSL_KEY=/path/to/ssl.key
N8N_SSL_CERT=/path/to/ssl.crt
```

#### **3. Credenciales Hardcodeadas:**

❌ **Error común:**
```javascript
const apiKey = "sk-1234567890abcdef";  // Hardcodeado
```

✅ **Uso correcto:**
```javascript
const apiKey = $credentials.apiKey;  // Desde credenciales n8n
```

#### **4. Falta de Validación de Entrada:**

❌ **Error común:**
```javascript
const userInput = $input.first().json.data;
// Usar directamente sin validación
```

✅ **Validación segura:**
```javascript
const userInput = $input.first().json.data;
if (typeof userInput !== 'string' || userInput.length > 1000) {
  throw new Error('Invalid input');
}
```

### 6.2 Vulnerabilidades de Configuración

#### **1. API Pública Expuesta Sin Necesidad:**
- Deshabilitar si no se usa: `N8N_PUBLIC_API_DISABLED=true`

#### **2. Logging Excesivo:**
- Usar nivel apropiado: `N8N_LOG_LEVEL=info` en producción

#### **3. Timeouts de Sesión Largos:**
- Configurar timeout apropiado: `N8N_SESSION_TIMEOUT=1800`

## 7. Checklist de Buenas Prácticas para Workflows de Producción

### 7.1 Checklist Pre-Despliegue

#### **Seguridad de Credenciales:**
- [ ] Todas las credenciales usan OAuth cuando es posible
- [ ] API keys tienen permisos mínimos necesarios
- [ ] No hay credenciales hardcodeadas en el código
- [ ] Secretos externos configurados para datos sensibles

#### **Configuración de Seguridad:**
- [ ] HTTPS habilitado y configurado correctamente
- [ ] MFA activado para todos los usuarios
- [ ] Timeouts de sesión configurados apropiadamente
- [ ] API pública deshabilitada si no se usa

#### **Monitoreo y Auditoría:**
- [ ] Logging configurado con nivel apropiado
- [ ] Retención de ejecuciones configurada
- [ ] Alertas configuradas para fallos críticos
- [ ] Auditoría de accesos implementada

#### **Protección de Datos:**
- [ ] Encriptación en tránsito configurada
- [ ] Encriptación en reposo implementada
- [ ] Políticas de retención de datos definidas
- [ ] Procedimientos de eliminación de datos establecidos

#### **Backup y Recuperación:**
- [ ] Backups automáticos configurados
- [ ] Encriptación de backups implementada
- [ ] Procedimientos de restauración probados
- [ ] Plan de recuperación ante desastres documentado

### 7.2 Checklist de Workflow Individual

#### **Validación de Datos:**
- [ ] Todas las entradas validadas y sanitizadas
- [ ] Manejo de errores implementado
- [ ] Timeouts configurados para llamadas externas
- [ ] Rate limiting considerado donde sea apropiado

#### **Gestión de Secretos:**
- [ ] Sin datos sensibles en logs
- [ ] Credenciales correctamente referenciadas
- [ ] Alcance mínimo de permisos utilizado
- [ ] Rotación de credenciales planificada

#### **Performance y Recursos:**
- [ ] Límites de memoria considerados
- [ ] Optimización de consultas implementada
- [ ] Paralelización apropiada configurada
- [ ] Cleanup de recursos temporales implementado

### 7.3 Checklist de Mantenimiento Continuo

#### **Mensual:**
- [ ] Revisión de logs de seguridad
- [ ] Verificación de backups
- [ ] Actualización de credenciales según políticas
- [ ] Review de usuarios y permisos

#### **Trimestral:**
- [ ] Auditoría de workflows activos
- [ ] Revisión de configuraciones de seguridad
- [ ] Testing de procedimientos de recuperación
- [ ] Actualización de documentación de seguridad

#### **Anual:**
- [ ] Revisión completa de políticas de seguridad
- [ ] Penetration testing
- [ ] Renovación de certificados SSL
- [ ] Capacitación de seguridad para el equipo

## 8. Recursos Adicionales y Contactos

### 8.1 Documentación Oficial
- [n8n Security Documentation](https://docs.n8n.io/reference/security/)
- [n8n Privacy and Security](https://docs.n8n.io/privacy-security/)
- [n8n Hosting Documentation](https://docs.n8n.io/hosting/)

### 8.2 Contactos de Seguridad
- **Consultas de Privacidad**: privacy@n8n.io
- **Reporte de Vulnerabilidades**: security@n8n.io
- **Soporte General**: [n8n Support](https://support.anthropic.com)

### 8.3 Herramientas Recomendadas
- **Gestión de Secretos**: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault
- **Monitoreo**: Prometheus, Grafana, ELK Stack
- **Backup**: rsync, pg_dump, cloud storage solutions
- **SSL/TLS**: Let's Encrypt, Certbot

---

**Nota**: Esta guía se basa en la documentación oficial de n8n y debe ser adaptada según las necesidades específicas de tu organización y entorno de despliegue. Mantén esta documentación actualizada con los cambios en las políticas de seguridad y nuevas funcionalidades de n8n.

**Última actualización**: Septiembre 2025  
**Versión del documento**: 1.0