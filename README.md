# SRE Agent Demo - Reaction Commerce Platform

Meta-repositorio que orquesta el despliegue local de **Reaction Commerce** (Mailchimp Open Commerce), una plataforma de e-commerce basada en microservicios Docker.

## Arquitectura

```
                        +-------------------+
                        |  example-storefront|  (Next.js)
                        |   localhost:4000   |
                        +---------+---------+
                                  |
                                  | GraphQL
                                  v
+-------------------+   +-------------------+   +-------------------+
|  reaction-admin   |-->|    reaction API    |<->|     MongoDB       |
|  localhost:4080   |   |   localhost:3000   |   |  localhost:27017  |
+-------------------+   +-------------------+   +-------------------+
        Panel Admin         API GraphQL            Base de Datos
                                  |
                        +---------+---------+
                        | Red Docker:       |
                        | reaction.localhost |
                        +-------------------+
```

### Servicios

| Servicio | Descripcion | Imagen Docker | Puerto |
|---|---|---|---|
| **reaction** | API backend GraphQL | `reactioncommerce/reaction:4.2.0` | 3000 |
| **reaction-admin** | Panel de administracion (Meteor) | `reactioncommerce/admin:4.0.0-beta.20` | 4080 |
| **example-storefront** | Tienda para clientes (Next.js) | `reactioncommerce/example-storefront:5.2.1` | 4000 |
| **MongoDB** | Base de datos (replica set) | `mongo:4.2.0` | 27017 |

## Requisitos Previos

Instalar antes de continuar:

- **Docker** >= 20.x y **Docker Compose** v2+
- **Git**
- **Node.js** >= 12.x
- **Yarn** (`npm install -g yarn`)
- **GNU Make** (opcional, para usar el Makefile)

Verificar instalacion:

```bash
docker --version
docker compose version
git --version
node --version
yarn --version
```

## Inicio Rapido (Sin Make)

Si no tienes `make` instalado o prefieres control manual, sigue estos pasos:

### 1. Clonar este repositorio

```bash
git clone https://github.com/Texelbit/sre-agent-demo.git
cd sre-agent-demo
```

### 2. Clonar los subproyectos

```bash
git clone https://github.com/reactioncommerce/reaction.git reaction
cd reaction && git checkout v4.2.0 && cd ..

git clone https://github.com/reactioncommerce/reaction-admin.git reaction-admin
cd reaction-admin && git checkout v4.0.0-beta.20 && cd ..

git clone https://github.com/reactioncommerce/example-storefront.git example-storefront
cd example-storefront && git checkout v5.2.1 && cd ..
```

### 3. Crear archivos de entorno

```bash
cp reaction/.env.example reaction/.env
cp reaction-admin/.env.example reaction-admin/.env
cp example-storefront/.env.example example-storefront/.env
```

### 4. Crear la red Docker

```bash
docker network create reaction.localhost
```

### 5. Levantar los servicios (en orden)

**Primero: Reaction API + MongoDB** (la base del sistema)

```bash
cd reaction
docker compose up -d
cd ..
```

Esperar ~30 segundos para que MongoDB inicialice el replica set (ver healthcheck con `docker ps`).

**Segundo: Reaction Admin**

```bash
cd reaction-admin
docker compose up -d
cd ..
```

**Tercero: Storefront**

```bash
cd example-storefront
docker compose up -d
cd ..
```

### 6. Verificar que todo esta corriendo

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Debes ver 4 contenedores (`reaction-api-1`, `reaction-mongo-1`, `reaction-admin-reaction-admin-1`, `example-storefront-web-1`) con estado `Up`.

### 7. Acceder a las aplicaciones

| Aplicacion | URL |
|---|---|
| Storefront (tienda) | http://localhost:4000 |
| Admin Panel | http://localhost:4080 |
| GraphQL Playground | http://localhost:3000/graphql |

## Inicio Rapido (Con Make)

Si tienes `make` instalado:

```bash
# Produccion (imagenes pre-construidas)
make init

# Desarrollo (con hot-reload)
make init-dev
```

### Comandos Make disponibles

| Comando | Descripcion |
|---|---|
| `make init` | Setup completo en modo produccion |
| `make init-dev` | Setup completo en modo desarrollo |
| `make start` | Iniciar todos los servicios |
| `make stop` | Detener todos los servicios |
| `make clean` | Eliminar contenedores, volumenes e imagenes locales |
| `make destroy` | Eliminar **todo**, incluyendo repositorios clonados |
| `make clone` | Solo clonar los subproyectos |
| `make build` | Reconstruir imagenes Docker |
| `make list` | Listar todos los targets disponibles |

## Conflicto de Puertos

Si algun puerto ya esta en uso (ej. el 3000), debes remapear los puertos en los archivos correspondientes:

### Ejemplo: cambiar API de puerto 3000 a 3001

**1. `reaction/docker-compose.yml`** - Cambiar el mapeo de puertos:

```yaml
ports:
  - "3001:3000"   # era "3000:3000"
```

**2. `reaction/.env`** - Actualizar ROOT_URL:

```
ROOT_URL=http://localhost:3001
```

**3. `reaction-admin/.env`** - Actualizar las URLs del API:

```
PUBLIC_GRAPHQL_API_URL_HTTP=http://localhost:3001/graphql
PUBLIC_GRAPHQL_API_URL_WS=ws://localhost:3001/graphql
PUBLIC_FILES_BASE_URL=http://localhost:3001
PUBLIC_I18N_BASE_URL=http://localhost:3001
```

**4. `example-storefront/.env`** - Actualizar URLs del API:

```
BUILD_GRAPHQL_URL=http://localhost:3001/graphql
EXTERNAL_GRAPHQL_URL=http://localhost:3001/graphql
```

**5. Rebuild del storefront** (obligatorio, las URLs se bakeean en el build de Next.js):

Editar `example-storefront/.env.prod` con las mismas URLs y luego:

```bash
cd example-storefront
# Cambiar docker-compose.yml para build local:
#   build: .
#   image: example-storefront:local
docker compose build
docker compose up -d
```

## Problemas Comunes

### reaction-admin se cae inmediatamente

**Causa:** MongoDB aun no termina de inicializar el replica set.

**Solucion:** Esperar a que `reaction-mongo-1` tenga status `(healthy)` y reiniciar:

```bash
cd reaction-admin
docker compose restart
```

### Storefront muestra "Sorry! We couldn't find what you're looking for"

**Causa 1:** El API no esta corriendo o no es accesible.

```bash
# Verificar que el API responde
curl http://localhost:3000/graphql
```

**Causa 2:** Si cambiaste el puerto del API, el storefront tiene las URLs de GraphQL hardcodeadas en el build de Next.js. Necesitas rebuild (ver seccion "Conflicto de Puertos").

### Error "port is already allocated"

Otro proceso usa el puerto. Identificar cual:

```bash
# Linux/Mac
lsof -i :3000

# Windows
netstat -ano | findstr :3000
```

Opciones: detener ese proceso o remapear el puerto (ver seccion anterior).

### Errores de line endings en Windows

Si al hacer build del storefront ves errores como `Invalid port input: "4000"`, los archivos `.env` tienen line endings de Windows (`\r\n`). Convertir a Unix:

```bash
# Git Bash / WSL
sed -i 's/\r$//' example-storefront/.env.prod
```

### Error "no route to host" o conexiones rechazadas entre servicios

Los servicios se comunican via la red Docker `reaction.localhost`. Verificar:

```bash
docker network inspect reaction.localhost
```

Todos los contenedores deben estar conectados a esta red.

## Estructura del Proyecto

```
sre-agent-demo/
├── config.mk              # Configuracion: repos, versiones, dependencias
├── config.local.mk        # Overrides locales (no versionado)
├── Makefile                # Orquestacion de todo el ciclo de vida
├── config/                 # Configs por version (v3.x, v4.x)
├── docs/                   # Guias de migracion y troubleshooting
├── release.py              # Script de release con versionado semantico
├── reaction/               # [clonado] API backend GraphQL
├── reaction-admin/         # [clonado] Panel de administracion
└── example-storefront/     # [clonado] Tienda Next.js
```

## Configuracion Personalizada

Crear un archivo `config.local.mk` para sobreescribir valores de `config.mk` sin modificar el archivo original:

```makefile
# Ejemplo: usar una version diferente del API
define SUBPROJECT_REPOS
https://github.com/reactioncommerce/reaction.git,reaction,v4.1.0 \
https://github.com/reactioncommerce/reaction-admin.git,reaction-admin,v4.0.0-beta.18 \
https://github.com/reactioncommerce/example-storefront.git,example-storefront,v5.2.0
endef
```

## Detener y Limpiar

```bash
# Detener servicios (conserva datos)
cd reaction && docker compose stop && cd ..
cd reaction-admin && docker compose stop && cd ..
cd example-storefront && docker compose stop && cd ..

# Eliminar contenedores y volumenes (pierde datos de MongoDB)
cd reaction && docker compose down -v && cd ..
cd reaction-admin && docker compose down -v && cd ..
cd example-storefront && docker compose down -v && cd ..

# Eliminar la red Docker
docker network rm reaction.localhost
```

O con Make:

```bash
make stop      # Solo detener
make clean     # Eliminar contenedores + volumenes
make destroy   # Eliminar todo, incluyendo repos clonados
```
