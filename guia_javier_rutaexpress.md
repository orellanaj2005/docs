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

## 12. Base de datos Oracle (no cubierta en este documento originalmente)

Este documento nunca llegó a definir dónde vive la base de datos de `shipments`, `catalog`,
`audit` y `report`. Se resolvió así:

### 12.1 — `ms-rutaexpress-db`: repo nuevo, solo para desarrollo local

Un décimo repo, `ms-rutaexpress-db`, con un `docker-compose.yml` que levanta
[`gvenzl/oracle-free`](https://github.com/gvenzl/oci-oracle-free) (Oracle 23ai Free — mucho más
simple que las imágenes oficiales de Oracle, que exigen aceptar licencia y compilar la imagen
uno mismo). Pensado para correr los microservicios localmente con `mvnw spring-boot:run` (sin
Docker) apuntando a `localhost:1522` (no `1521` — ese puerto ya lo puede estar usando una
instalación nativa de Oracle XE en la máquina de cada uno, revisa con `netstat`/`ss` antes de
asumir que está libre).

Contiene además:
- `initdb/`: scripts SQL que crean un usuario Oracle por microservicio (`rutaexpress` para
  shipments, `catalog`, `audit_user`, `report_user`) — **cada microservicio necesita su propio
  esquema**, no pueden compartir uno. Si comparten esquema, Flyway falla al migrar el segundo
  servicio con un error de "checksum mismatch": los dos servicios numeran su primera migración
  como `V1` pero con contenido distinto, y Flyway lo detecta como si la migración `V1` hubiera
  cambiado de contenido en vez de reconocer que son dos servicios independientes.
- `seed/`: scripts SQL con datos de prueba (4 servicios de catálogo, 6 envíos cubriendo los 6
  estados posibles, y eventos correspondientes en audit/report) para no tener que probar contra
  una base vacía.

### 12.2 — En la EC2 real (`ec2-apps`): Oracle fusionado dentro de `infra/apps`, no un repo aparte

`host.docker.internal` (el hostname que usa `ms-rutaexpress-db` para que los microservicios
lleguen a Oracle desde fuera de su red de Docker) es una magia exclusiva de **Docker Desktop**
(Windows/Mac) — **no existe en Docker nativo de Linux**, es decir, no funciona en ninguna de las
3 EC2. Usarlo ahí revienta con `UnknownHostException: host.docker.internal`.

La solución fue agregar `oracle-db` como un servicio más dentro de
`infra/apps/compose.override.yml` (mismo `gvenzl/oracle-free`, misma config), en vez de vivir en
su propio `docker-compose.yml` aparte. Así los demás contenedores del mismo `compose` lo alcanzan
por nombre de contenedor (`oracle-db`), sin depender de ningún hostname especial de host. El
`.env` de `infra/apps` en la EC2 debe tener:
```
DB_HOST=oracle-db
DB_PORT=1521
DB_SERVICE=XEPDB1

ORACLE_HOST=oracle-db
ORACLE_PORT=1521
ORACLE_SERVICE=XEPDB1
```
(nota el puerto `1521`, no `1522` — acá sí es el puerto interno real del contenedor, no hay
mapeo al host de por medio, es contenedor-a-contenedor directo).

`shipments`/`catalog` usan las variables `ORACLE_*`; `audit`/`report` (repos de Valentina) usan
`DB_*` con nombres de variable distintos — hay que setear ambos juegos en el mismo `.env`. Ojo
también con el nombre del PDB por defecto: `shipments`/`catalog` default a `XEPDB1`, pero
`audit`/`report` default a `FREEPDB1` si no se fija `DB_SERVICE` explícitamente — hay que forzar
`DB_SERVICE=XEPDB1` para que los 4 apunten al mismo PDB que efectivamente existe.

### 12.3 — Bug: "Grant succeeded" en el log, pero el permiso no queda

Los scripts de `initdb/` reportan `Grant succeeded.` en el log del contenedor, pero al levantar
los microservicios fallan con `ORA-01045: Login denied. User X does not have CREATE SESSION
privilege.` (o, si el login sí pasa, Flyway falla más adelante por falta del rol `RESOURCE`
para crear tablas). Consultando `dba_role_privs` directamente se confirma que el usuario
realmente no tiene ningún rol asignado, pese al mensaje de éxito en el log — no se identificó la
causa exacta (probablemente algo específico de cómo `gvenzl/oracle-free` ejecuta los scripts de
`container-entrypoint-initdb.d`), pero el workaround que sí funciona y persiste es volver a
correr el `GRANT` a mano, con `COMMIT` explícito, conectado como `system`:
```bash
docker exec -i apps-oracle-db-1 bash -c "sqlplus -s system/oracle@//localhost:1521/XEPDB1" <<'EOF'
GRANT CONNECT, RESOURCE TO rutaexpress;
GRANT CONNECT, RESOURCE TO catalog;
GRANT CONNECT, RESOURCE TO audit_user;
GRANT CONNECT, RESOURCE TO report_user;
COMMIT;
EOF
```
Si el contenedor `oracle-db` se recrea desde cero (volumen nuevo), hay que estar atento a que
este mismo síntoma puede repetirse — verifica con `SELECT table_name FROM user_tables;`
conectado como cada usuario si los microservicios no levantan.

---

## 13. Bug de Spring Boot 4: Flyway no se activa solo con `flyway-core`

Este fue el bug más difícil de encontrar, y afecta a **cualquier microservicio nuevo que use
Spring Boot 4.1.1 + Flyway** (`shipments` y `catalog`, no `audit`/`report` que están en Boot
3.3.4).

**Síntoma:** el microservicio arranca, conecta bien a Oracle (se ve el `HikariPool` conectando
sin problema), pero falla con `SchemaManagementException: Schema validation: missing table
[nombre_tabla]` — como si Flyway nunca hubiera corrido. Revisando el log completo del arranque,
**no aparece ninguna línea de Flyway** (ni siquiera un intento fallido) — a diferencia de cuando
falla por permisos (ver 12.3), donde Flyway sí deja rastro en el log antes de fallar.

**Causa real:** Spring Boot 4 partió la autoconfiguración de varias integraciones que antes
venían incluidas en el módulo central `spring-boot-autoconfigure`, en artefactos propios y
separados (mismo patrón que ya se ve con `spring-boot-hibernate`, `spring-boot-jpa`,
`spring-boot-data-jpa`, `spring-boot-kafka`). Con solo `flyway-core` +
`flyway-database-oracle` en el `pom.xml` (las librerías de migración en sí, sin el "pegamento"
que las conecta a Spring), Boot 4 ni siquiera sabe que Flyway existe — no hay ningún
`FlywayAutoConfiguration` registrado para que se active.

**Fix:** agregar la dependencia que faltaba:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-flyway</artifactId>
</dependency>
```
(sin versión explícita — la maneja el BOM del `spring-boot-starter-parent`). Ya aplicado en
`ms-rutaexpress-shipments` y `ms-rutaexpress-catalog`. Si alguno de tus otros microservicios en
Boot 4.x usa Flyway en algún momento, va a necesitar exactamente lo mismo.

---

## 14. Otros bugs encontrados probando contra Oracle real (no los detecta H2)

Los tests con H2 (usados en el perfil `test` de cada microservicio) son más permisivos que
Oracle real en un par de puntos. Estos tres aparecieron recién al probar contra la Oracle real
de la EC2, nunca en CI/tests locales:

1. **`ORA-18716: no está en ninguna zona horaria`** al leer `created_at`/`updated_at`. Es un bug
   de compatibilidad entre el driver `ojdbc11` y el tipo JDBC que Hibernate 7 usa por defecto
   para campos `Instant` (llama a `ResultSet.getObject(col, OffsetDateTime.class)`, que el
   driver de Oracle no soporta bien contra una columna `TIMESTAMP` plana). No se arregla
   cambiando la zona horaria de la JVM ni con la propiedad `oracle.jdbc.timezoneAsRegion` — el
   fix es forzar el tipo JDBC `TIMESTAMP` clásico vía `@JdbcTypeCode(SqlTypes.TIMESTAMP)` en el
   campo, sin cambiar el tipo Java (`Instant`) que ya usa el resto del código. Ya corregido en
   `Shipment.java` y `ShippingService.java`.

2. **Mapeo `boolean` incorrecto** (`catalog`, campo `active`): Oracle 23ai tiene un tipo
   `BOOLEAN` nativo, y Hibernate 7 mapea `boolean` de Java a ese tipo por defecto en ese
   dialecto — pero la migración creó la columna como `NUMBER(1)` (portable a versiones más
   viejas de Oracle). Fix: `@Convert(converter = NumericBooleanConverter.class)` en el campo,
   para que Hibernate lo siga tratando como 0/1.

3. **Exchange de RabbitMQ desactualizado** en `shipments`: quedó un `application.yaml` apuntando
   al nombre viejo de exchange (`rutaexpress.cmd`) de antes de que se corrigiera la topología
   para calzar con la sección 7 de este documento — solo se había corregido el bean Java, no el
   valor de la propiedad. Ya corregido a `cmd.direct`.

---

## 15. Frontend: HTTPS obligatorio para MSAL (sin dominio propio)

Sirviendo el frontend por HTTP plano sobre la IP pública (`http://<ip>:5173`), la app queda en
**pantalla en blanco** con este error en la consola del navegador:
```
Uncaught BrowserAuthError: crypto_nonexistent: The crypto object or function is not available.
```
MSAL necesita la Web Crypto API (`window.crypto.subtle`) para PKCE, y los navegadores modernos
solo la exponen en **contextos seguros** (HTTPS o `localhost`) — nunca en HTTP plano sobre una
IP pública.

Como no hay dominio propio para un certificado real (Let's Encrypt necesita uno), se generó un
**certificado autofirmado** dentro del propio build de `nginx` en `frontend-rutaexpress/Dockerfile`:
```dockerfile
RUN apk add --no-cache openssl && \
    mkdir -p /etc/nginx/ssl && \
    openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
      -keyout /etc/nginx/ssl/selfsigned.key \
      -out /etc/nginx/ssl/selfsigned.crt \
      -subj "/CN=rutaexpress-frontend"
```
`nginx.conf` escucha en `443 ssl` con ese certificado. El contenedor expone `443` internamente;
`infra/apps/compose.override.yml` lo mapea al puerto que definas en `FRONTEND_PORT` (ver
sección 16). El navegador va a mostrar una advertencia de "sitio no seguro" la primera vez (hay
que aceptar manualmente, "Avanzado → Continuar") — sigue siendo HTTPS real, solo que el
certificado no lo emitió una autoridad reconocida. Si más adelante consiguen un dominio, lo ideal
es reemplazar esto por Let's Encrypt/Certbot o poner CloudFront delante.

**Otro repo con Dockerfile propio, mismo repo:** Valentina había subido su propio `Dockerfile`
para el frontend (modo desarrollo, `vite dev --host 0.0.0.0`, sin HTTPS) al mismo tiempo — hubo
que resolver un conflicto de merge a favor del build de producción con nginx descrito arriba,
por el tema de HTTPS. Avísale para que no le sorprenda en su próximo `git pull`.

---

## 16. `infra/apps/compose.override.yml`: build local + frontend + Oracle fusionado

Recapitulando los tres agregados a `compose.override.yml` (se carga automáticamente junto con
`compose.yml` al correr `docker compose up`, sin necesidad de indicarlo):

1. **Build local en vez de imagen publicada:** cada servicio pasa de `image:
   rutaexpress/xxx:latest` a `build: ../../ms-rutaexpress-xxx` — asume que todos los repos
   hermanos están clonados en la misma carpeta que `infra`.
2. **`frontend-svc`:** build con `context`/`args` (las variables `VITE_*` se hornean en el
   bundle en tiempo de build, no de runtime — deben venir como build args, no como
   `environment:`), puerto parametrizado con `${FRONTEND_PORT:-5173}` (agregado porque el 5173
   fijo chocaba con otro proceso en `ec2-apps` — pon el que tengas libre en el `.env` de cada
   máquina) mapeado al **443** interno del contenedor (nginx con HTTPS, ver sección 15).
3. **`oracle-db`:** ver sección 12.2.

`.env` de referencia completo para `infra/apps` en una EC2 (ejemplo con los valores reales de
`ec2-apps`, `ec2-mq`, `ec2-kafka` — ajusta si las IPs elásticas cambian):
```
TENANT_ID=1392dab6-2ace-4798-8a92-0cb9a2fb0b99
API_CLIENT_ID=5af865ec-0324-4f82-a1e9-bbba4937c75e

KAFKA_BOOTSTRAP_SERVERS=34.233.170.20:9092

RABBITMQ_HOST=100.62.195.178
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest
RABBITMQ_MGMT_URL=http://100.62.195.178:15672
RABBITMQ_MGMT_USER=guest
RABBITMQ_MGMT_PASSWORD=guest

DB_HOST=oracle-db
DB_PORT=1521
DB_SERVICE=XEPDB1

ORACLE_HOST=oracle-db
ORACLE_PORT=1521
ORACLE_SERVICE=XEPDB1

CORS_ALLOWED_ORIGINS=https://<ip-publica-ec2-apps>:<FRONTEND_PORT>

FRONTEND_PORT=5173
VITE_TENANT_ID=1392dab6-2ace-4798-8a92-0cb9a2fb0b99
VITE_SPA_CLIENT_ID=<spaClientId>
VITE_API_SCOPE=api://5af865ec-0324-4f82-a1e9-bbba4937c75e/access_as_user
VITE_API_BASE_URL=<invoke-url-del-api-gateway>
VITE_REDIRECT_URI=https://<ip-publica-ec2-apps>:<FRONTEND_PORT>
```
Este `.env` **no se sube al repo** (está en `.gitignore`) — cada máquina/EC2 necesita el suyo
propio, con sus propios valores.

Para levantar (o reconstruir después de un cambio):
```bash
cd infra/apps
docker compose down          # solo si hay contenedores viejos en mal estado
docker compose up -d --build
```

---

## 17. CORS: dos capas distintas que hay que configurar por separado

El navegador bloqueaba las llamadas del frontend al API Gateway con errores de CORS, en dos
capas separadas — hay que arreglar ambas, una no reemplaza a la otra:

### 17.1 — CORS en el BFF

`SecurityConfig.java` del BFF ahora lee `cors.allowed-origins` (env var
`CORS_ALLOWED_ORIGINS`, default `http://localhost:5173`) y permite `OPTIONS` sin autenticar
(preflight). Si el origen del frontend cambia (nuevo puerto, nueva IP, nuevo dominio), hay que
actualizar esta variable y reiniciar el BFF — si no, el navegador bloquea la respuesta aunque el
BFF funcione perfecto.

### 17.2 — CORS en API Gateway (la parte que falta en la sección 10 de este documento)

Configurar CORS a nivel de API (API Gateway → tu API → **CORS** en el menú lateral, no una ruta
específica) con:
- **Access-Control-Allow-Origin**: el origen exacto del frontend (`https://<ip>:<puerto>`)
- **Access-Control-Allow-Headers**: `authorization, content-type`
- **Access-Control-Allow-Methods**: `GET, POST, PUT, PATCH, DELETE, OPTIONS`
- **Access-Control-Allow-Credentials**: activado

**Esto solo no basta.** La ruta `ANY /{proxy+}` con el JWT Authorizer atado también captura el
método `OPTIONS`, así que el preflight sigue pidiendo el token (que nunca lo trae) y responde
`401` — con los headers CORS correctos, pero un navegador exige un status `2xx` en la respuesta
del preflight sin importar qué headers traiga, así que sigue fallando. Hay que agregar una
**ruta separada** sin autorizador:

1. **Routes → Create**
2. Method: **`OPTIONS`**, Path: **`/{proxy+}`** (mismo comodín, método específico en vez de `ANY`)
3. Adjuntar la misma integración HTTP hacia el BFF
4. **No** adjuntarle el JWT Authorizer a esta ruta
5. Guardar

Con esto, `OPTIONS` va por la ruta sin auth, y todo lo demás sigue yendo por `ANY /{proxy+}`
con el Authorizer intacto.

---

## 18. Notas sueltas de la sesión de despliegue

- **IPs elásticas:** ya asignadas a las 3 instancias (confirmado) — no deberían cambiar aunque
  se apaguen/prendan las máquinas, salvo que se liberen explícitamente.
- **Client Secret en Azure AD para `RutaExpress-SPA`:** no debería existir ninguno — es una SPA
  (cliente público con PKCE). Si alguien le agrega uno sin querer en Certificates & secrets, hay
  que borrarlo: Azure empieza a exigir que cualquier `client_secret` recibido sea válido, y
  rompe el flujo de Postman/la app.
- **Postman + PKCE + cliente público, gotchas encontrados:**
  - **Client Authentication**: usar `Send client credentials in body`, no `Send as Basic Auth
    header` (con Basic Auth, Azure se queja de que falta `client_id` en el body).
  - El campo **Client Secret** debe quedar **completamente vacío** — cuidado con
    autocompletado del navegador o con pegar el mismo texto en dos campos por error (pasó:
    quedó parte del valor de `Scope` pegado ahí sin querer, y Azure respondía `AADSTS7000215:
    Invalid client secret provided`).
  - Si el popup de login interno de Postman se queda pegado esperando el callback, marcar
    **"Authorize using browser"** (usa el navegador del sistema en vez del popup embebido) suele
    destrabarlo.
  - Al armar requests a mano: el método (`GET`, `POST`, etc.) va en el dropdown aparte, nunca
    escrito dentro del campo de la URL — si se escribe "GET https://..." directo en la barra de
    URL, Postman tira `Error: Invalid protocol: get https:`.
- **Dos puertas de entrada al BFF, ambas activas a la vez:** `http://<ip-ec2-apps>:8080/...`
  (directo, sin pasar por el Gateway) y `https://<invoke-url>/...` (a través del Gateway, con su
  propio Authorizer). Ambas van a seguir funcionando mientras el Security Group tenga el 8080
  abierto al público — si quieren forzar que todo el tráfico externo pase por el Gateway, hay
  que cerrar el 8080 en `sg-ec2-apps` (dejarlo accesible solo internamente o para tu IP) más
  adelante, no es urgente ahora.

---

## 19. Checklist final

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
- [x] Base de datos Oracle: `oracle-db` fusionado en `infra/apps/compose.override.yml`, un
      esquema por microservicio, permisos verificados (sección 12).
- [x] `spring-boot-flyway` agregado a `shipments`/`catalog` (sección 13).
- [x] Bugs de tipos de datos contra Oracle real corregidos: `ORA-18716`, mapeo `boolean`,
      exchange de RabbitMQ desactualizado (sección 14).
- [x] Frontend sirviendo por HTTPS (certificado autofirmado) — MSAL funcionando (sección 15).
- [x] CORS configurado en BFF y en API Gateway (con ruta `OPTIONS` separada sin Authorizer,
      sección 17).
- [x] Datos de ejemplo cargados en las 4 bases (`shipments`, `catalog`, `audit`, `report`) en la
      EC2 real, no solo en local.
- [ ] Cerrar el puerto 8080 en `sg-ec2-apps` cuando quieran forzar que todo pase por el API
      Gateway (sección 18, no urgente).
