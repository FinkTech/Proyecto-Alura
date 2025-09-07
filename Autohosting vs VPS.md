# Guía de Referencia n8n: Autohosting vs VPS

## 1. Autohosting vs VPS Pagado

### Qué significa autohostear n8n en tu PC

Autohostear n8n en tu PC significa ejecutar la aplicación directamente en tu computadora personal usando Docker o npm, convirtiendo tu máquina en el servidor que aloja todos tus workflows, credenciales y datos. Tu PC se convierte en el centro de comando para todas tus automatizaciones.

**Configuración típica de autohosting:**
- Instalación local con Docker: `docker run -it --rm --name n8n -p 5678:5678 n8nio/n8n`
- Acceso través de `localhost:5678` o `http://tu-ip-local:5678`
- Datos almacenados en volúmenes Docker locales

### Diferencias entre VPS con uptime garantizado y autohosting

| Aspecto | Autohosting en PC | VPS Pagado |
|---------|-------------------|------------|
| **Uptime** | Dependiente de que tu PC esté encendida | 99.9% uptime garantizado |
| **Acceso** | Solo cuando tu PC está activa | 24/7 disponible |
| **IP pública** | IP doméstica (puede cambiar) | IP fija dedicada |
| **Mantenimiento** | Tu responsabilidad completa | Infraestructura gestionada |
| **Costos** | Solo electricidad | Mensualidad fija |
| **Rendimiento** | Limitado por tu hardware | Recursos dedicados escalables |

### Ventajas del Autohosting

**Ventajas:**
- **Control total**: Acceso completo al sistema y configuraciones
- **Costos bajos**: Solo pagas electricidad
- **Privacidad máxima**: Datos nunca salen de tu red local
- **Customización ilimitada**: Puedes modificar cualquier configuración
- **Aprendizaje**: Ideal para entender el funcionamiento interno

**Riesgos:**
- **Disponibilidad**: Workflows se detienen si apagas la PC
- **Conectividad**: Dependiente de tu conexión a internet
- **Seguridad**: Tu red doméstica puede ser menos segura
- **Escalabilidad limitada**: Restringido por el hardware de tu PC
- **Sin soporte**: Completamente autogestión

### Ventajas del VPS

**Ventajas:**
- **Alta disponibilidad**: Workflows ejecutándose 24/7
- **Acceso remoto**: Gestión desde cualquier lugar
- **Webhooks**: Fácil configuración de endpoints públicos
- **Escalabilidad**: Recursos ajustables según demanda
- **Respaldos automáticos**: Muchos proveedores incluyen backups

**Riesgos:**
- **Costos recurrentes**: Mensualidad fija independiente del uso
- **Dependencia del proveedor**: Sujeto a políticas externas
- **Complejidad**: Requiere conocimientos de administración de servidores

## 2. Consideraciones Técnicas

### Uso de Docker para instalar n8n

n8n recommends using Docker for most self-hosting needs. It provides a clean, isolated environment, avoids operating system and tooling incompatibilities, and makes database and environment management simpler.

**Instalación básica con Docker:**

```bash
# Instalación simple para desarrollo
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Con persistencia de datos
docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

**Docker Compose para producción:**

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=password
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=n8n
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    depends_on:
      - postgres

  postgres:
    image: postgres:13
    restart: always
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  postgres_data:
```

### Configuración de versiones específicas

Para usar n8n 1.103.2 (current latest) o cualquier versión específica:

```bash
# Versión específica
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n:1.103.2

# Para desarrollo con la versión beta
docker run -d \
  --name n8n-dev \
  -p 5679:5678 \
  -v n8n_dev_data:/home/node/.n8n \
  n8nio/n8n:next
```

**Variables de entorno importantes:**
```env
N8N_VERSION=1.103.2
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure_password
GENERIC_TIMEZONE=America/Argentina/Buenos_Aires
```

### Backup, restore y persistencia de datos

**Estructura de datos de n8n:**
- **Workflows**: Almacenados en base de datos (SQLite por defecto)
- **Credenciales**: Encriptadas en base de datos
- **Configuraciones**: En archivos de configuración
- **Archivos locales**: En directorio `/home/node/.n8n`

**Estrategias de backup:**

1. **Backup de volumen Docker completo:**
```bash
# Parar el contenedor
docker stop n8n

# Crear backup del volumen
docker run --rm -v n8n_data:/source -v $(pwd):/backup alpine tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz -C /source .

# Reiniciar contenedor
docker start n8n
```

2. **Backup mediante API de n8n:**
This workflow will backup your workflows to Github. It uses the public api to export all of the workflow data using the n8n node.

3. **Backup de base de datos PostgreSQL:**
```bash
# Backup de PostgreSQL
docker exec postgres pg_dump -U n8n n8n > n8n-db-backup-$(date +%Y%m%d).sql

# Restaurar desde backup
docker exec -i postgres psql -U n8n -d n8n < n8n-db-backup-20241201.sql
```

**Restauración:**
```bash
# Restaurar volumen Docker
docker run --rm -v n8n_data:/target -v $(pwd):/backup alpine tar xzf /backup/n8n-backup-20241201.tar.gz -C /target
```

### Uptime y disponibilidad de workflows

**En autohosting:**
- **Uptime**: 0% cuando la PC está apagada
- **Dependencias**: Conexión a internet, energía eléctrica
- **Monitoreo**: Configuración manual necesaria

**En VPS:**
- **Uptime**: 99.9%+ típicamente garantizado
- **Redundancia**: Múltiples niveles de respaldo
- **Monitoreo**: Incluido en muchos servicios

**Configuraciones para maximizar uptime:**

```yaml
# Docker Compose con reinicio automático
services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:5678/healthz || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 3. Mejores Prácticas

### Exportar workflows de autohosting a VPS

**Método 1: Exportación manual**
1. Acceder a la interfaz web de n8n
2. Workflows → Seleccionar workflows → Download
3. Subir archivos JSON al VPS
4. Importar en la nueva instancia

**Método 2: Backup completo de datos**
```bash
# En autohosting - crear backup completo
docker exec n8n n8n export:workflow --backup --output=/tmp/workflows-backup.json

# Transferir archivo al VPS
scp workflows-backup.json user@vps-ip:/tmp/

# En VPS - importar workflows
docker exec n8n n8n import:workflow --input=/tmp/workflows-backup.json
```

**Método 3: Via API programática**
This workflow provides a streamlined way to restore all your n8n workflows from backup JSON files stored in a designated Google Drive folder y workflows similares permiten automatizar este proceso.

### Mantenimiento de credenciales y variables de entorno

**Gestión de credenciales:**

1. **Backup de credenciales:**
This template enables you to restore your n8n credentials from a backup file in Google Drive

```bash
# Exportar credenciales (requiere configuración especial)
docker exec n8n n8n export:credentials --decrypt --output=/tmp/credentials-backup.json
```

2. **Variables de entorno críticas:**
```env
# Archivo .env para migración
N8N_ENCRYPTION_KEY=your-encryption-key-here
N8N_USER_FOLDER=/home/node/.n8n
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure_password
WEBHOOK_URL=https://your-domain.com/
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
N8N_PORT=5678
```

**⚠️ Importante**: La clave de encriptación (`N8N_ENCRYPTION_KEY`) debe ser idéntica entre instancias para mantener las credenciales funcionales.

**Proceso de migración completo:**

1. **Preparación en autohosting:**
```bash
# Crear directorio de migración
mkdir n8n-migration
cd n8n-migration

# Backup de datos completo
docker exec n8n n8n export:workflow --backup --output=/tmp/workflows.json
docker exec n8n n8n export:credentials --decrypt --output=/tmp/credentials.json
docker cp n8n:/tmp/workflows.json ./
docker cp n8n:/tmp/credentials.json ./

# Backup de variables de entorno
docker exec n8n env | grep N8N_ > n8n-env.txt
```

2. **Configuración en VPS:**
```bash
# Transferir archivos al VPS
scp -r n8n-migration/ user@vps-ip:/home/user/

# En VPS - configurar mismas variables de entorno
# Especialmente importante: N8N_ENCRYPTION_KEY
```

3. **Restauración en VPS:**
```bash
# Importar workflows
docker exec n8n n8n import:workflow --input=/tmp/workflows.json

# Importar credenciales
docker exec n8n n8n import:credentials --input=/tmp/credentials.json
```

### Checklist de migración

**Antes de migrar:**
- [ ] Backup completo de workflows
- [ ] Backup de credenciales
- [ ] Documentar variables de entorno
- [ ] Verificar versión de n8n
- [ ] Backup de archivos locales si se usan

**Durante la migración:**
- [ ] Configurar misma versión de n8n
- [ ] Aplicar mismas variables de entorno
- [ ] Mantener misma clave de encriptación
- [ ] Configurar base de datos (si aplica)
- [ ] Configurar webhooks y URLs públicas

**Después de migrar:**
- [ ] Probar workflows críticos
- [ ] Verificar webhooks funcionando
- [ ] Configurar monitoreo
- [ ] Establecer backups automáticos
- [ ] Actualizar URLs en servicios externos

---

## Fuentes de Referencia

- [n8n Installation Documentation](https://docs.n8n.io/getting-started/installation/)
- [n8n Self-hosting Guide](https://docs.n8n.io/hosting/)
- [n8n GitHub Repository](https://github.com/n8n-io/n8n)
- [Docker Compose Configuration](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/)
- [n8n Community Discussions](https://community.n8n.io/)

*Documento actualizado: Diciembre 2024*