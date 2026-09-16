# Guía técnica — Javier: repositorios, identidad, BFF, mensajería y despliegue

Esta guía cubre lo que te falta: gobernanza de los 9 repositorios del caso, completar el BFF, dockerizar todo, levantar RabbitMQ y Kafka con su topología completa, los microservicios administradores, y el despliegue final en AWS.

---

## 1. Organización de los repositorios en GitHub

### 1.1 Los 9 repositorios, bajo tu cuenta

Todos se crean en **tu cuenta** (tú quedas como Owner), y agregas a Jassack y Valentina como **colaboradores** (Settings → Collaborators → Add people) en cada uno según lo que les toque tocar. Esto mantiene todo centralizado sin dispersar repos entre las tres cuentas.

1. `frontend-rutaexpress` (React + MSAL) — colaborador: Valentina
2. `ms-rutaexpress-bff` — tú
3. `ms-rutaexpress-shipments` — colaborador: Jassack
4. `ms-rutaexpress-catalog` — colaborador: Jassack
5. `ms-rutaexpress-notify` (consumidor RabbitMQ) — colaborador: Jassack
6. `ms-rutaexpress-report` (consumidor Kafka) — colaborador: Valentina
7. `ms-rutaexpress-audit` (consumidor Kafka) — colaborador: Valentina
8. `infra` (compose de apps, mq y kafka) — tú
9. `docs` (documentación del proyecto) — los tres

Crea cada uno vacío, con README inicial, desde github.com/new.

### 1.2 Estructura y protección de rama

En cada repositorio de código (los 7 primeros):

Configura protección de la rama `main` (Settings → Branches → Add rule):
- Require pull request before merging
- Require at least 1 approval

Flujo sugerido: cada quien trabaja en `feature/<nombre-corto>`, abre PR contra `main`, tú (o quien corresponda) revisa y aprueba.

### 1.3 `.gitignore`

Para los repos Spring Boot (Maven) — `bff`, `shipments`, `catalog`, `notify`, `report`, `audit`:
```
target/
*.class
.idea/
*.iml
application-local.yml
application-local.properties
.env
```

Para `frontend-rutaexpress` (React):
```
node_modules/
dist/
build/
.env
```

Para `infra`:
```
.env
*.log
```

No subas nunca `.env` ni ningún archivo con secretos (client secrets, contraseñas de DB, connection strings).

---

## 2. Azure AD — configurar la identidad

Vas a crear **dos** App Registrations: una para el frontend (SPA) y una para el backend (API). *(Si ya la hiciste, sáltate esta sección — solo revisa 2.5.)*

### 2.1 Registro de la SPA (React)

1. Entra a [portal.azure.com](https://portal.azure.com) → **Azure Active Directory** → **App registrations** → **New registration**.
2. Nombre: `RutaExpress-SPA`.
3. Tipo de cuenta: "Accounts in this organizational directory only".
4. Platform: **Single-page application (SPA)**, Redirect URI: `http://localhost:5173` (Vite) o `http://localhost:3000` (Create React App) — confirma con Valentina cuál usará.
5. Anota **Application (client) ID** (`spaClientId`) y **Directory (tenant) ID** (`tenantId`).

### 2.2 Registro de la API (backend)

1. **New registration**, nombre: `RutaExpress-API`.
2. **Expose an API** → Application ID URI: `api://<API_CLIENT_ID>` (se autogenera) → **Add a scope**: `access_as_user`, habilitado.
3. Anota el **Application (client) ID** → este es tu `apiClientId`.
4. **Manifest** → confirma `"accessTokenAcceptedVersion": 2` (tokens v2.0).

### 2.3 App Roles

En `RutaExpress-API` → **App roles** → crea `Admin`, `Operador`, `Cliente`, `Auditor` (Display name = Value en los cuatro casos).

Asigna usuarios en **Enterprise applications** → `RutaExpress-API` → **Users and groups**. Si Jassack/Valentina no aparecen, créalos primero en **Users** → **New user**.

### 2.4 Conectar la SPA con la API

`RutaExpress-SPA` → **API permissions** → **Add a permission** → **My APIs** → `RutaExpress-API` → scope `access_as_user` → **Add permissions**.

### 2.5 Datos a compartir con el equipo

```
TENANT_ID=<tenantId>
SPA_CLIENT_ID=<spaClientId>
API_CLIENT_ID=<apiClientId>
ISSUER=https://login.microsoftonline.com/<TENANT_ID>/v2.0
AUDIENCE=<apiClientId>   (SIN el prefijo api:// — así viene el claim "aud" en tokens v2.0)
```

Valentina necesita `TENANT_ID` y `SPA_CLIENT_ID`. Tú, Jassack y Valentina (para `audit`/`report`) necesitan `ISSUER` y `AUDIENCE` para validar el JWT en cada backend.

---

## 3. BFF (`ms-rutaexpress-bff`) — lo que falta

Ya tienes el proyecto corriendo local con Spring Security validando issuer/audience/rol, y el proxy de `/api/shipments/*` probado con Postman. Falta:

### 3.1 Corregir el `audience` en `application.yml`

Por el detalle de tokens v2.0 que ya descubriste al probar con Postman, debe quedar **sin** el prefijo `api://`:

```yaml
spring:
  config:
    import: optional:file:.env[.properties]

server:
  port: 8080

security:
  jwt:
    issuer-uri: https://login.microsoftonline.com/${TENANT_ID}/v2.0
    audience: ${API_CLIENT_ID}

services:
  shipments-url: ${SHIPMENTS_URL:http://shipments-svc:8081}
  catalog-url: ${CATALOG_URL:http://catalog-svc:8082}
  audit-url: ${AUDIT_URL:http://audit-svc:8083}
  report-url: ${REPORT_URL:http://report-svc:8084}
```

Nota que las URLs por defecto ahora usan nombres de servicio (`shipments-svc`, etc.) en vez de `localhost` — así van a funcionar cuando todo corra dentro de la misma red de Docker Compose (sección 5). Para tus pruebas locales sigue usando `.env` con `SHIPMENTS_URL=http://localhost:8081` mientras Jassack no tenga Docker corriendo.

### 3.2 Agregar los proxies restantes

Mismo patrón que `ShipmentsProxyController`, un controlador por dominio:

```
src/main/java/cl/rutaexpress/bff/controller/CatalogProxyController.java   → /api/catalog/**
src/main/java/cl/rutaexpress/bff/controller/AuditProxyController.java    → /api/audit/**   (solo GET)
src/main/java/cl/rutaexpress/bff/controller/ReportProxyController.java   → /api/report/**  (solo GET)
```

### 3.3 Reforzar la autorización por ruta en `SecurityConfig`

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/catalog/**").hasAnyAuthority("Admin", "Operador")
    .requestMatchers("/api/shipments/**").hasAnyAuthority("Admin", "Operador", "Cliente")
    .requestMatchers("/api/audit/**").hasAnyAuthority("Admin", "Auditor")
    .requestMatchers("/api/report/**").hasAuthority("Admin")
    .anyRequest().authenticated()
)
```

---

## 4. Dockerizar los servicios

Cada microservicio Spring Boot (`bff`, `shipments`, `catalog`, `notify`, `audit`, `report`) necesita su propio `Dockerfile` en la raíz de su repo. Comparte esta plantilla con Jassack y Valentina para que la repliquen en los suyos:

```dockerfile
# Dockerfile (multi-stage)
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Ajusta el `EXPOSE` al puerto real de cada servicio (`8080` bff, `8081` shipments, `8082` catalog, `8083` audit, `8084` report; `notify` no expone puerto público, solo consume colas).

Prueba localmente antes de subir a EC2:
```bash
docker build -t rutaexpress/bff:latest .
docker run --env-file .env -p 8080:8080 rutaexpress/bff:latest
```

---

## 5. Repositorio `infra` — Docker Compose

Estructura del repo `infra`:
```
infra/
├── apps/compose.yml
├── mq/compose.yml
├── kafka/compose.yml
└── docs/
```

### 5.1 `apps/compose.yml` (para `ec2-apps`)

```yaml
services:
  bff:
    image: rutaexpress/bff:latest
    ports: ["8080:8080"]
    env_file: .env
    depends_on: [shipments-svc, catalog-svc]

  shipments-svc:
    image: rutaexpress/shipments:latest
    ports: ["8081:8081"]
    env_file: .env

  catalog-svc:
    image: rutaexpress/catalog:latest
    ports: ["8082:8082"]
    env_file: .env

  notify-svc:
    image: rutaexpress/notify:latest
    env_file: .env

  audit-svc:
    image: rutaexpress/audit:latest
    ports: ["8083:8083"]
    env_file: .env

  report-svc:
    image: rutaexpress/report:latest
    ports: ["8084:8084"]
    env_file: .env

networks:
  default:
    name: rutaexpress-net
```

Todos en la misma red por defecto de Compose — por eso el BFF puede llamarlos por nombre de servicio (`http://shipments-svc:8081`, ver sección 3.1).

### 5.2 `mq/compose.yml` (para `ec2-mq`) — clúster RabbitMQ de 2 nodos

```yaml
services:
  rabbit1:
    image: rabbitmq:3-management
    hostname: rabbit1
    environment:
      RABBITMQ_ERLANG_COOKIE: "rutaexpress-cookie"
    ports: ["5672:5672", "15672:15672"]

  rabbit2:
    image: rabbitmq:3-management
    hostname: rabbit2
    environment:
      RABBITMQ_ERLANG_COOKIE: "rutaexpress-cookie"
    depends_on: [rabbit1]
```

Después de levantarlo, entra al contenedor de `rabbit2` y ejecuta el join al clúster:
```bash
docker exec -it <container_rabbit2> bash
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@rabbit1
rabbitmqctl start_app
```

Management UI queda en `http://<IP_ec2-mq>:15672` (usuario/clave por defecto `guest`/`guest`, cámbialos antes de exponerlo).

### 5.3 `kafka/compose.yml` (para `ec2-kafka`) — Zookeeper (3) + Kafka (3 brokers) + UI

```yaml
services:
  zk1:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_INIT_LIMIT: 5
      ZOOKEEPER_SYNC_LIMIT: 2
      ZOOKEEPER_SERVERS: zk1:2888:3888;zk2:2888:3888;zk3:2888:3888
  zk2:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_SERVER_ID: 2
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_INIT_LIMIT: 5
      ZOOKEEPER_SYNC_LIMIT: 2
      ZOOKEEPER_SERVERS: zk1:2888:3888;zk2:2888:3888;zk3:2888:3888
  zk3:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_SERVER_ID: 3
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_INIT_LIMIT: 5
      ZOOKEEPER_SYNC_LIMIT: 2
      ZOOKEEPER_SERVERS: zk1:2888:3888;zk2:2888:3888;zk3:2888:3888

  kafka1:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zk1, zk2, zk3]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: "zk1:2181,zk2:2181,zk3:2181"
      KAFKA_LISTENERS: "PLAINTEXT://0.0.0.0:9092"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka1:9092"
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
  kafka2:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zk1, zk2, zk3]
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: "zk1:2181,zk2:2181,zk3:2181"
      KAFKA_LISTENERS: "PLAINTEXT://0.0.0.0:9092"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka2:9092"
  kafka3:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zk1, zk2, zk3]
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: "zk1:2181,zk2:2181,zk3:2181"
      KAFKA_LISTENERS: "PLAINTEXT://0.0.0.0:9092"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka3:9092"

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8090:8080"]
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: "kafka1:9092,kafka2:9092,kafka3:9092"
```

**Importante:** los tres nodos de Zookeeper deben tener el mismo valor en `ZOOKEEPER_SERVERS` (la lista completa de los 3, no solo "los otros dos") — es lo que les permite replicarse entre sí y formar un ensamble real. Sin esta variable, cada nodo arranca en modo standalone, aislado de los demás, sin avisar ningún error — simplemente cada broker de Kafka termina registrado en un Zookeeper distinto sin que ninguno vea a los otros.

*(Simplifiqué las variables de listeners internos/externos — cuando lo pruebes en la EC2 real, confirma que `KAFKA_ADVERTISED_LISTENERS` apunte a la IP pública si Jassack/Valentina se conectarán desde fuera.)*

---

## 6. Levantar las 3 instancias EC2

### 6.1 Crear (o reutilizar) un par de llaves SSH

1. Consola de AWS → busca **EC2** → menú lateral → **Key Pairs** (Pares de claves) → **Create key pair**.
2. Nombre: `rutaexpress-key`. Tipo: RSA. Formato: `.pem` (funciona con el cliente SSH nativo de Windows/PowerShell).
3. Al crearla, se descarga automáticamente `rutaexpress-key.pem` — guárdala en una carpeta que recuerdes, ej. `C:\Users\Javier Orellana\.ssh\`. **Esta es la única vez que AWS te la entrega** — si la pierdes, tienes que crear una llave nueva.
4. Vas a reutilizar esta misma llave para las 3 instancias, no necesitas una por instancia.

### 6.2 Crear los 3 Security Groups

Consola de AWS → **EC2** → menú lateral → **Security Groups** → **Create security group**, uno por uno:

**`sg-ec2-apps`**

| Tipo | Protocolo | Puerto | Origen |
|---|---|---|---|
| SSH | TCP | 22 | Tu IP (elige "My IP" en el desplegable) |
| Custom TCP | TCP | 8080 | Anywhere (0.0.0.0/0) — lo vas a restringir más adelante al rango de API Gateway, por ahora déjalo abierto para probar con Postman |

**`sg-ec2-mq`**

| Tipo | Protocolo | Puerto | Origen |
|---|---|---|---|
| SSH | TCP | 22 | Tu IP |
| Custom TCP | TCP | 5672 | El **Security Group ID** de `sg-ec2-apps` (no una IP — puedes seleccionar otro SG como origen) |
| Custom TCP | TCP | 15672 | Tu IP (para que tú entres a la Management UI a administrar) |

**`sg-ec2-kafka`**

| Tipo | Protocolo | Puerto | Origen |
|---|---|---|---|
| SSH | TCP | 22 | Tu IP |
| Custom TCP | TCP | 9092 | El SG de `sg-ec2-apps` |
| Custom TCP | TCP | 8090 | Tu IP |

Usar el SG de origen (en vez de una IP) para 5672/9092 es justamente lo que asegura que **solo tus propios contenedores en `ec2-apps` puedan hablarle a RabbitMQ/Kafka**, no cualquiera en internet.

### 6.3 Lanzar cada instancia

Repite esto 3 veces (una por `ec2-apps`, `ec2-mq`, `ec2-kafka`):

1. **EC2** → **Launch instance**.
2. **Name**: `ec2-apps` (o `ec2-mq` / `ec2-kafka`).
3. **AMI**: Amazon Linux 2023.
4. **Instance type**: `t3.micro` para `ec2-apps` y `ec2-mq` alcanza. Para `ec2-kafka`, sube a **`t3.medium`** — vas a correr 6 contenedores pesados (3 Zookeeper + 3 Kafka) al mismo tiempo, y `t3.micro` (1 GB de RAM) se va a quedar corto y los contenedores van a empezar a caerse solos por falta de memoria.
5. **Key pair**: `rutaexpress-key` (la que ya creaste).
6. **Network settings** → **Edit** → **Select existing security group** → elige el que corresponda (`sg-ec2-apps`, `sg-ec2-mq` o `sg-ec2-kafka`).
7. **Configure storage**: sube el volumen a **20 GiB** (el default de 8 GiB es apretado una vez que tienes varias imágenes de Docker descargadas — especialmente en `ec2-kafka`).
8. **Launch instance**.

### 6.4 (Recomendado) Asignar una IP elástica

Por defecto, la IP pública de una instancia EC2 **cambia** cada vez que la detienes y la vuelves a iniciar — lo cual rompería todas las URLs que configuraste (`ISSUER`, el redirect de API Gateway, lo que le diste a Valentina/Jassack). Para evitar eso:

1. **EC2** → **Elastic IPs** → **Allocate Elastic IP address** → **Allocate**.
2. Selecciona la IP recién creada → **Actions** → **Associate Elastic IP address** → elige la instancia (repite para las 3).

Ahora cada instancia tiene una IP pública fija mientras dure el proyecto.

### 6.5 Conectarte por SSH

Desde PowerShell:

```powershell
ssh -i "C:\Users\Javier Orellana\.ssh\rutaexpress-key.pem" ec2-user@<IP_PUBLICA>
```

Si PowerShell se queja de permisos del archivo `.pem` (`Permissions for '...' are too open`), corrige con:
```powershell
icacls "C:\Users\Javier Orellana\.ssh\rutaexpress-key.pem" /inheritance:r
icacls "C:\Users\Javier Orellana\.ssh\rutaexpress-key.pem" /grant:r "$($env:USERNAME):(R)"
```

**Alternativa sin SSH ni llave, si prefieres algo más simple:** en la consola de AWS, selecciona la instancia → botón **Connect** → pestaña **EC2 Instance Connect** → **Connect**. Te abre una terminal directamente en el navegador, sin necesidad de manejar el `.pem` desde tu PC. Funciona igual de bien para todo lo que sigue.

### 6.6 Instalar Docker en cada instancia

Ya conectado por SSH (o por Instance Connect), en **cada una de las 3**:

```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
exit
```

El `exit` es importante — el cambio de grupo (`usermod -aG docker`) no aplica a la sesión actual, necesitas salir y volver a entrar (`ssh` de nuevo, o reconectar Instance Connect) para poder correr `docker` sin `sudo`.

Después de reconectar:
```bash
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
docker --version
docker compose version
```

Amazon Linux 2023 no trae `docker-compose-plugin` en sus repos por defecto (a diferencia de Ubuntu/Debian), por eso se instala el binario directo desde el repositorio oficial de Compose en GitHub, en vez de usar `dnf install`. `x86_64` corresponde a instancias `t3`/`t3.medium` como las que lanzaste — si alguna vez usan instancias `t4g` (ARM/Graviton), el binario sería `docker-compose-linux-aarch64`.

### 6.7 Llevar el código y las imágenes a cada instancia

Aquí está el detalle que faltaba: tu `compose.yml` de `ec2-apps` dice `image: rutaexpress/bff:latest`, pero esa imagen **solo existe en tu notebook** ahora mismo — Docker en la EC2 no tiene forma de saber qué es "rutaexpress/bff" a menos que se la lleves.

Amazon Linux 2023 tampoco trae **Git** preinstalado, así que instálalo primero (una sola vez por instancia):
```bash
sudo dnf install -y git
git --version
```

**Enfoque recomendado (sin necesidad de una cuenta de Docker Hub): construir directo en cada EC2**

En `ec2-apps`:
```bash
git clone https://github.com/<tu-usuario>/ms-rutaexpress-bff.git
cd ms-rutaexpress-bff
docker build -t rutaexpress/bff:latest .
cd ..
```
Repite el `git clone` + `docker build` para `ms-rutaexpress-shipments`, `ms-rutaexpress-catalog`, `ms-rutaexpress-notify`, `ms-rutaexpress-audit`, `ms-rutaexpress-report` (a medida que Jassack y Valentina los vayan entregando) — cada uno con su propio `docker build -t rutaexpress/<nombre>:latest .`, usando exactamente el mismo nombre de imagen que aparece en `infra/apps/compose.yml`.

Después, clona también `infra`:
```bash
git clone https://github.com/<tu-usuario>/infra.git
cd infra/apps
```
Crea el `.env` ahí mismo con `nano .env` (pega tus variables `TENANT_ID`, `API_CLIENT_ID`, etc.) y levanta:
```bash
docker compose up -d
docker ps
```
Como las imágenes ya existen localmente en esa misma EC2 (las acabas de construir con `docker build`), Compose las usa directo, sin necesitar ningún registry externo.

**Nota para `ec2-mq` y `ec2-kafka`:** estas dos no necesitan este paso — sus `compose.yml` usan imágenes públicas (`rabbitmq:3-management`, `confluentinc/cp-kafka`, etc.) que Docker descarga solo desde Docker Hub la primera vez que corres `docker compose up -d`, igual que hiciste en tu notebook. Solo necesitas clonar `infra` y correr el compose correspondiente:
```bash
git clone https://github.com/<tu-usuario>/infra.git
cd infra/mq        # o infra/kafka
docker compose up -d
```

**Si más adelante quieres evitar repetir el `git clone` + `build` cada vez que alguien actualice su código:** la alternativa es crear una cuenta gratuita en Docker Hub, y que cada quien haga `docker push` de su imagen después de construirla localmente; en la EC2 entonces sería `docker pull` en vez de clonar y construir. Es más práctico para el mediano plazo, pero no es necesario para dejar esto funcionando ahora.

### 6.8 Verificar que todo llegó bien

Repite en cada instancia:
```bash
docker ps
```

Y desde tu notebook, confirma que puedes llegar al BFF a través de la IP pública de `ec2-apps` (todavía sin pasar por API Gateway, para aislar problemas):
```bash
curl http://<IP_ec2-apps>:8080/api/shipments/1
```
Debería darte 401 (sin token) — igual que en tus pruebas locales, confirmando que el contenedor está sano y el Security Group permite el tráfico.

---

## 7. Topología RabbitMQ

Desde la Management UI (`http://<IP_ec2-mq>:15672`) o vía `rabbitmqadmin`:

**Exchanges:** `cmd.direct` (direct), `cmd.topic` (topic), `cmd.dead.dlx` (direct)

**Colas + bindings:**

| Cola | DLQ | Binding direct (`cmd.direct`) | Binding topic (`cmd.topic`) |
|---|---|---|---|
| `q.cmd.email` | `q.cmd.email.dlq` | `email.send` | `email.*` |
| `q.cmd.warehouse` | `q.cmd.warehouse.dlq` | `warehouse.ticket` | `warehouse.#` |
| `q.cmd.label` | `q.cmd.label.dlq` | `label.gen` | `label.*` |

Cada cola principal se declara con el argumento `x-dead-letter-exchange: cmd.dead.dlx` apuntando a su DLQ correspondiente. Documenta esto en `infra/docs/` para que Jassack (productor/consumidor) lo siga al pie de la letra.

---

## 8. Topología Kafka

Crea los tópicos (desde `kafka-ui` o `kafka-topics.sh` dentro de un contenedor):

| Tópico | Particiones | Réplicas | Política | Retención |
|---|---|---|---|---|
| `shipments.events` | 3 | 3 | delete | 3–7 días |
| `audit.timeline` | 3 | 3 | compact,delete | 14–30 días |
| `notify.DLT`, `audit.DLT`, `report.DLT` | 3 c/u | 3 | delete | 7–14 días |

```bash
docker exec -it <container_kafka1> kafka-topics --create \
  --topic shipments.events --partitions 3 --replication-factor 3 \
  --bootstrap-server localhost:9092
```

---

## 9. Microservicios administradores

### 9.1 — `ms-rutaexpress-rabbitmq-admin`

**Proyecto en Spring Initializr:**

| Campo | Valor |
|---|---|
| Group | `cl.rutaexpress` |
| Artifact | `rabbitmqadmin` |
| Package name | `cl.rutaexpress.rabbitmqadmin` |
| Java | 17 |

**Dependencies:** `Spring Web`, `Spring Security`, `OAuth2 Resource Server`.

**`.env`** (en la raíz del proyecto, junto a `pom.xml`):
```
TENANT_ID=1392dab6-2ace-4798-8a92-0cb9a2fb0b99
API_CLIENT_ID=5af865ec-0324-4f82-a1e9-bbba4937c75e
```

**`application.yaml`:**
```yaml
spring:
  config:
    import: optional:file:.env[.properties]

server:
  port: 8085

security:
  jwt:
    issuer-uri: https://login.microsoftonline.com/${TENANT_ID}/v2.0
    audience: ${API_CLIENT_ID}

rabbitmq:
  management-url: ${RABBITMQ_MGMT_URL:http://ec2-mq-ip:15672}
  username: ${RABBITMQ_MGMT_USER:guest}
  password: ${RABBITMQ_MGMT_PASSWORD:guest}
```
Reemplaza `ec2-mq-ip` por la IP real de tu instancia `ec2-mq`, y cambia `guest`/`guest` por credenciales reales antes de dejarlo expuesto.

**`config/SecurityConfig.java`** — idéntico al del BFF (mismo `jwtDecoder` con validador de audience), pero con autorización exclusiva para Admin:
```java
.authorizeHttpRequests(auth -> auth
    .anyRequest().hasAuthority("Admin")
)
```

**`config/RestTemplateConfig.java`** — la API de RabbitMQ usa autenticación HTTP Basic, no JWT, por eso este bean le agrega las credenciales automáticamente a cada request saliente:
```java
package cl.rutaexpress.rabbitmqadmin.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.support.BasicAuthenticationInterceptor;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    @Value("${rabbitmq.username}")
    private String username;

    @Value("${rabbitmq.password}")
    private String password;

    @Bean
    public RestTemplate rabbitRestTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add(new BasicAuthenticationInterceptor(username, password));
        return restTemplate;
    }
}
```
El bean crea su propio `new RestTemplate()` — no depende de que exista otro bean del mismo tipo en el contexto.

**`controller/RabbitAdminController.java`:**
```java
package cl.rutaexpress.rabbitmqadmin.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

@RestController
@RequestMapping("/api/rabbitmq")
public class RabbitAdminController {

    private final RestTemplate restTemplate;

    @Value("${rabbitmq.management-url}")
    private String managementUrl;

    public RabbitAdminController(@Qualifier("rabbitRestTemplate") RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping("/queues")
    public ResponseEntity<Object> listQueues() {
        return restTemplate.getForEntity(managementUrl + "/api/queues", Object.class);
    }

    @GetMapping("/queues/{name}")
    public ResponseEntity<Object> getQueue(@PathVariable String name) {
        return restTemplate.getForEntity(managementUrl + "/api/queues/%2F/" + name, Object.class);
    }

    @GetMapping("/overview")
    public ResponseEntity<Object> getOverview() {
        return restTemplate.getForEntity(managementUrl + "/api/overview", Object.class);
    }
}
```
`/api/queues` (de RabbitMQ) devuelve todas las colas con su profundidad — con eso ves de un vistazo si las 3 DLQ tienen mensajes acumulados. El `%2F` es el encoding del vhost por defecto (`/`), requisito de la API de RabbitMQ.

### 9.2 — `ms-rutaexpress-kafka-admin`

**Proyecto:** Artifact `kafkaadmin`, package `cl.rutaexpress.kafkaadmin`, mismas dependencias base. Agrega a mano en `pom.xml` (Spring Initializr no la ofrece):
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.6.0</version>
</dependency>
```

Kafka no tiene una API REST propia — se administra con la clase `AdminClient`, conectándose directo a los brokers por el puerto 9092, igual que hace `kafka-topics` por línea de comandos.

**`.env`:**
```
TENANT_ID=1392dab6-2ace-4798-8a92-0cb9a2fb0b99
API_CLIENT_ID=5af865ec-0324-4f82-a1e9-bbba4937c75e
```

**`application.yaml`:**
```yaml
spring:
  config:
    import: optional:file:.env[.properties]

server:
  port: 8086

security:
  jwt:
    issuer-uri: https://login.microsoftonline.com/${TENANT_ID}/v2.0
    audience: ${API_CLIENT_ID}

kafka:
  bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:ec2-kafka-ip:9092}
```

**`config/KafkaAdminConfig.java`:**
```java
package cl.rutaexpress.kafkaadmin.config;

import org.apache.kafka.clients.admin.AdminClient;
import org.apache.kafka.clients.admin.AdminClientConfig;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Properties;

@Configuration
public class KafkaAdminConfig {

    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public AdminClient adminClient() {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return AdminClient.create(props);
    }
}
```

**`controller/KafkaAdminController.java`:**
```java
package cl.rutaexpress.kafkaadmin.controller;

import org.apache.kafka.clients.admin.*;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.common.TopicPartition;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.*;
import java.util.concurrent.ExecutionException;

@RestController
@RequestMapping("/api/kafka")
public class KafkaAdminController {

    private final AdminClient adminClient;

    public KafkaAdminController(AdminClient adminClient) {
        this.adminClient = adminClient;
    }

    @GetMapping("/topics")
    public ResponseEntity<Object> listTopics() throws ExecutionException, InterruptedException {
        Set<String> topics = adminClient.listTopics().names().get();
        return ResponseEntity.ok(topics);
    }

    @GetMapping("/topics/{name}")
    public ResponseEntity<Object> describeTopic(@PathVariable String name) throws ExecutionException, InterruptedException {
        TopicDescription description = adminClient.describeTopics(List.of(name))
            .topicNameValues().get(name).get();
        return ResponseEntity.ok(description);
    }

    @GetMapping("/consumer-groups")
    public ResponseEntity<Object> listConsumerGroups() throws ExecutionException, InterruptedException {
        Collection<ConsumerGroupListing> groups = adminClient.listConsumerGroups().all().get();
        return ResponseEntity.ok(groups);
    }

    @GetMapping("/consumer-groups/{groupId}/lag")
    public ResponseEntity<Object> getConsumerGroupLag(@PathVariable String groupId) throws ExecutionException, InterruptedException {
        Map<TopicPartition, OffsetAndMetadata> offsets =
            adminClient.listConsumerGroupOffsets(groupId).partitionsToOffsetAndMetadata().get();
        return ResponseEntity.ok(offsets);
    }
}
```
`AdminClient` es asíncrono (devuelve `KafkaFuture`), por eso cada método usa `.get()` para esperar el resultado y declara `throws ExecutionException, InterruptedException`. Mismo `SecurityConfig.java` que `rabbitmq-admin` (todo exclusivo para `Admin`).

### 9.3 — Integrar ambos en el BFF

**`controller/RabbitMqProxyController.java`:**
```java
package cl.rutaexpress.bff.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

@RestController
@RequestMapping("/api/rabbitmq")
public class RabbitMqProxyController {

    private final RestTemplate restTemplate;

    @Value("${services.rabbitmq-admin-url}")
    private String rabbitmqAdminUrl;

    public RabbitMqProxyController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping("/queues")
    public ResponseEntity<Object> listQueues() {
        return restTemplate.getForEntity(rabbitmqAdminUrl + "/api/rabbitmq/queues", Object.class);
    }

    @GetMapping("/queues/{name}")
    public ResponseEntity<Object> getQueue(@PathVariable String name) {
        return restTemplate.getForEntity(rabbitmqAdminUrl + "/api/rabbitmq/queues/" + name, Object.class);
    }

    @GetMapping("/overview")
    public ResponseEntity<Object> getOverview() {
        return restTemplate.getForEntity(rabbitmqAdminUrl + "/api/rabbitmq/overview", Object.class);
    }
}
```

**`controller/KafkaAdminProxyController.java`:**
```java
package cl.rutaexpress.bff.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

@RestController
@RequestMapping("/api/kafka")
public class KafkaAdminProxyController {

    private final RestTemplate restTemplate;

    @Value("${services.kafka-admin-url}")
    private String kafkaAdminUrl;

    public KafkaAdminProxyController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping("/topics")
    public ResponseEntity<Object> listTopics() {
        return restTemplate.getForEntity(kafkaAdminUrl + "/api/kafka/topics", Object.class);
    }

    @GetMapping("/topics/{name}")
    public ResponseEntity<Object> describeTopic(@PathVariable String name) {
        return restTemplate.getForEntity(kafkaAdminUrl + "/api/kafka/topics/" + name, Object.class);
    }

    @GetMapping("/consumer-groups")
    public ResponseEntity<Object> listConsumerGroups() {
        return restTemplate.getForEntity(kafkaAdminUrl + "/api/kafka/consumer-groups", Object.class);
    }

    @GetMapping("/consumer-groups/{groupId}/lag")
    public ResponseEntity<Object> getConsumerGroupLag(@PathVariable String groupId) {
        return restTemplate.getForEntity(kafkaAdminUrl + "/api/kafka/consumer-groups/" + groupId + "/lag", Object.class);
    }
}
```

**Agregar al `application.yml` del BFF**, dentro del bloque `services:` que ya tienes:
```yaml
services:
  shipments-url: ${SHIPMENTS_URL:http://shipments-svc:8081}
  catalog-url: ${CATALOG_URL:http://catalog-svc:8082}
  audit-url: ${AUDIT_URL:http://audit-svc:8083}
  report-url: ${REPORT_URL:http://report-svc:8084}
  rabbitmq-admin-url: ${RABBITMQ_ADMIN_URL:http://rabbitmq-admin-svc:8085}
  kafka-admin-url: ${KAFKA_ADMIN_URL:http://kafka-admin-svc:8086}
```
Los nombres `rabbitmq-admin-svc`/`kafka-admin-svc` deben coincidir con los que uses en `infra/apps/compose.yml` (sección 9.4). Para pruebas locales sin Docker, agrega a tu `.env` del BFF:
```
RABBITMQ_ADMIN_URL=http://localhost:8085
KAFKA_ADMIN_URL=http://localhost:8086
```

**Agregar a `SecurityConfig.java` del BFF**, junto a las demás reglas de `authorizeHttpRequests`:
```java
.requestMatchers("/api/rabbitmq/**", "/api/kafka/**").hasAuthority("Admin")
```

### 9.4 — Sumarlos a `infra/apps/compose.yml`

```yaml
  rabbitmq-admin-svc:
    image: rutaexpress/rabbitmq-admin:latest
    ports: ["8085:8085"]
    env_file: .env

  kafka-admin-svc:
    image: rutaexpress/kafka-admin:latest
    ports: ["8086:8086"]
    env_file: .env
```

Mismo flujo de siempre en `ec2-apps`: `git clone` + `docker build -t rutaexpress/rabbitmq-admin:latest .` (y equivalente para `kafka-admin`), luego `docker compose up -d` desde `infra/apps`.

---

## 10. API Gateway

### 10.1 — Crear la API

1. Consola de AWS → busca **API Gateway** → **Create API**.
2. En la tarjeta **HTTP API** → **Build**.
3. **API name**: `RutaExpress-API-Gateway`.
4. En **Integrations**, por ahora clic en **Next** sin agregar nada todavía (la agregas manualmente en el paso siguiente, para tener más control).
5. En **Configure routes**, tampoco agregues nada aún — clic en **Next**.
6. **Configure stages**: deja el stage `$default` con **Auto-deploy** activado (es la opción por defecto) — así cualquier cambio que hagas después se publica solo, sin tener que hacer un "deploy" manual cada vez.
7. **Create**.

### 10.2 — Crear la integración hacia el BFF

1. Ya dentro de tu API, menú lateral izquierdo → **Integrations** → **Manage integrations** → **Create**.
2. **Integration type**: `HTTP`.
3. **HTTP method**: `ANY` (deja que cualquier verbo — GET, POST, PUT — pase igual; el filtrado real de qué métodos acepta cada endpoint ya lo hace cada controlador del BFF).
4. **Endpoint URL**:
```
http://<IP_publica_ec2-apps>:8080/{proxy}
```
5. **Create**.

### 10.3 — Crear la ruta

1. Menú lateral → **Routes** → **Create**.
2. **Method**: `ANY`.
3. **Path**: `/{proxy+}`.
4. **Create**.
5. Selecciona la ruta recién creada → **Attach integration** → elige la integración HTTP que armaste en 10.2.

El patrón `{proxy+}` es un comodín que captura **cualquier** ruta (`/api/shipments/1`, `/api/kafka/topics`, lo que sea) y lo reenvía tal cual al BFF — por eso no necesitas crear una ruta distinta por cada endpoint que ya programaste.

### 10.4 — Crear el JWT Authorizer

1. Menú lateral → **Authorization** → **Manage authorizers** → **Create**.
2. **Authorizer type**: `JWT`.
3. **Name**: `RutaExpressJwtAuthorizer`.
4. **Identity source**: `$request.header.Authorization` (ya viene sugerido por defecto).
5. **Issuer URL**:
```
https://login.microsoftonline.com/<TENANT_ID>/v2.0
```
6. **Audience**: agrega tu `<API_CLIENT_ID>` **sin** el prefijo `api://` — recuerda que así es como realmente viene el claim `aud` en los tokens v2.0, según lo que ya corregiste en el BFF.
7. **Create**.

### 10.5 — Adjuntar el authorizer a la ruta

1. **Routes** → selecciona `ANY /{proxy+}`.
2. Pestaña/sección **Attach authorization**.
3. Elige `RutaExpressJwtAuthorizer`.
4. **Attach authorizer**.

### 10.6 — Obtener la URL de invocación

1. Menú lateral → **Stages** → `$default`.
2. Copia el valor de **Invoke URL** (algo como `https://abc123xyz.execute-api.us-east-1.amazonaws.com`).

Esta es la URL que le vas a compartir a Valentina para que la use como base en el frontend React (en vez de apuntar directo a la IP de `ec2-apps`), y la que usarás tú mismo para las pruebas finales end-to-end.

### 10.7 — Probar con Postman

1. Con el mismo token de Admin que ya generaste antes, cambia la URL de tu request de `http://localhost:8080/api/rabbitmq/queues` a:
```
https://<tu-invoke-url>/api/rabbitmq/queues
```
2. Deberías obtener la misma respuesta que cuando le pegabas directo al BFF — la diferencia es que ahora la petición pasó primero por el JWT Authorizer de API Gateway antes de llegar al BFF.
3. **Prueba sin token**: deberías recibir `401` directamente desde API Gateway (ni siquiera llega a tocar tu BFF) — así confirmas que el Authorizer realmente está filtrando antes de reenviar.
4. **Prueba con un token de rol distinto a Admin** contra `/api/rabbitmq/queues`: debería pasar el filtro de API Gateway (el token es válido) pero recibir `403` del BFF (por la regla de autorización que agregaste en 9.3) — confirmando que ambas capas de seguridad están funcionando en conjunto.

---

## 11. Checklist final

- [ ] 9 repositorios creados bajo tu cuenta, colaboradores agregados, `.gitignore` correctos.
- [ ] Azure AD: audience corregido sin prefijo `api://` en BFF y en API Gateway.
- [ ] BFF con los 6 proxies (`shipments`, `catalog`, `audit`, `report`, `rabbitmq`, `kafka`) y autorización por rol y ruta.
- [ ] Dockerfile probado localmente para el BFF (y compartido como plantilla).
- [ ] `infra/apps`, `infra/mq`, `infra/kafka` con sus `compose.yml` corriendo en las 3 EC2.
- [ ] RabbitMQ: 3 colas + 3 DLQ + exchanges configurados.
- [ ] Kafka: `shipments.events`, `audit.timeline` y los `*.DLT` creados, con `Cluster ID` consistente en los 3 brokers.
- [ ] Microservicios administradores (`rabbitmq-admin`, `kafka-admin`) respondiendo, protegidos solo para rol Admin.
- [ ] API Gateway con JWT Authorizer probado end-to-end con Postman (con y sin token, con rol correcto e incorrecto).
- [ ] Prueba end-to-end desde React funcionando.
