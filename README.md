# Backend Ventas — API REST Spring Boot

**Innovatech Chile | ISY1101 EP2**

API REST para la gestión de ventas y órdenes de compra, desarrollada con Spring Boot 3.4.4 y Java 17. Containerizada con Docker y desplegada automáticamente en AWS EC2 mediante GitHub Actions.

---

## Stack tecnológico

| Componente | Tecnología |
|---|---|
| Framework | Spring Boot 3.4.4 |
| Lenguaje | Java 17 |
| Base de datos | MySQL 8.0 |
| Contenedor | Docker (multi-stage build) |
| Registry | Docker Hub |
| CI/CD | GitHub Actions |
| Infraestructura | AWS EC2 (Amazon Linux 2023) |

---

## Estructura del repositorio

```
innovatech-backend-ventas/
├── Springboot-API-REST/          # Código fuente Spring Boot
│   ├── src/
│   │   ├── main/java/com/citt/
│   │   │   ├── controller/       # VentaController (endpoints REST)
│   │   │   ├── persistence/
│   │   │   │   ├── entity/       # Entidad Venta (JPA)
│   │   │   │   ├── repository/   # VentaRepository
│   │   │   │   └── services/     # VentaService + VentaServiceImpl
│   │   │   └── exceptions/       # Manejo global de errores
│   │   └── resources/
│   │       └── application.properties
│   ├── Dockerfile                # Multi-stage build
│   └── pom.xml
├── docker-compose.yml            # Stack local: backend + MySQL
├── .env.example                  # Variables de entorno de ejemplo
└── .github/
    └── workflows/
        └── deploy.yml            # Pipeline CI/CD
```

---

## Endpoints disponibles

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/api/v1/ventas` | Listar todas las ventas |
| GET | `/api/v1/ventas/{id}` | Obtener venta por ID |
| POST | `/api/v1/ventas` | Crear nueva venta |
| PUT | `/api/v1/ventas/{id}` | Actualizar venta |
| DELETE | `/api/v1/ventas/{id}` | Eliminar venta |

Documentación interactiva: `http://<host>:8080/swagger-ui.html`

---

## Ejecución local con Docker Compose

### 1. Configurar variables de entorno

```bash
cp .env.example .env
# Editar .env con valores reales
```

### 2. Levantar el stack

```bash
docker compose up --build -d
```

Esto levanta:
- **db-ventas**: MySQL 8.0 en puerto `3308` del host
- **backend-ventas**: Spring Boot en puerto `8080`

### 3. Verificar

```bash
curl http://localhost:8080/api/v1/ventas
```

---

## Pipeline CI/CD

El pipeline en `.github/workflows/deploy.yml` se activa con cada push a la rama `deploy`:

```
push → deploy
       ↓
  [Job 1] Build y Push
       ├── Checkout código
       ├── Set up JDK 17
       ├── Login Docker Hub
       └── Build + Push imagen (tomescobc/backend-ventas:latest)
       ↓
  [Job 2] Deploy en EC2
       ├── SSH a instancia EC2 Backend
       ├── docker pull latest
       ├── docker stop/rm anterior
       ├── docker run con variables de entorno
       └── Verificación docker ps
```

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario Docker Hub |
| `DOCKERHUB_TOKEN` | Access token Docker Hub |
| `EC2_BACKEND_HOST` | IP pública EC2 Backend |
| `EC2_USERNAME` | Usuario SSH (`ec2-user`) |
| `EC2_SSH_PRIVATE_KEY` | Clave privada `.pem` |
| `DB_NAME` | Nombre de la base de datos |
| `DB_USERNAME` | Usuario MySQL |
| `DB_PASSWORD` | Contraseña MySQL |

---

## Decisiones técnicas

### Dockerfile multi-stage
- **Stage 1 (builder):** `eclipse-temurin:17-jdk` — compila el JAR con Maven
- **Stage 2 (runtime):** `eclipse-temurin:17-jre` — ejecuta solo el JAR (sin JDK)
- **Beneficio:** imagen final ~60% más liviana, sin herramientas de compilación en producción

### Usuario no-root
```dockerfile
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
```
Principio de mínimo privilegio: el proceso no puede modificar el sistema de archivos del contenedor.

### Named volume para MySQL
```yaml
volumes:
  ventas-mysql-data:
    driver: local
```
Se eligió **named volume** sobre bind mount porque:
1. Docker gestiona la ubicación → portable entre sistemas operativos y EC2
2. Permisos seguros sin exponer rutas del host
3. Los datos persisten aunque se destruya y recree el contenedor

### Red interna
```yaml
networks:
  ventas-net:
    driver: bridge
```
El backend y MySQL se comunican por nombre de servicio (`db-ventas`) en una red privada. MySQL no es accesible desde el exterior.
