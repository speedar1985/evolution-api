# GEMINI.md — Evolution API (Motor WhatsApp Personal / Baileys)

## 1. Descripción del Proyecto
Plataforma REST API multi-instancia para conexión con **WhatsApp Personal (WhatsApp Web vía Baileys)** y proveedores adicionales, integrada como motor del canal personal dentro del ecosistema omnicanal de `responde.nquito.com`. Permite la creación de instancias por empresa y por agente, vinculación por código QR, envío/recepción de mensajes multimedia y notas de voz, sincronización de contactos y despacho de eventos en tiempo real vía Webhooks hacia el backend `.NET`.

- **Repositorio Evolution API (este workspace):** `c:\Users\rikar\source\repos\speedar1985\evolution-api` (`https://github.com/speedar1985/evolution-api`)
- **Repositorio Backend asociado (.NET 9):** `c:\Users\rikar\source\repos\FacebookMessengerApi` (`https://webhooks.nquito.com`)
- **Repositorio Frontend asociado (Angular 19):** `C:\Users\rikar\source\repos\responde.nquito.com` (`https://responde.nquito.com`)
- **Despliegue Docker / Endpoint Interno:** `http://10.10.20.4:8085` (definido en `docker-compose-evolution.yml` / `docker-compose.yaml`)

---

## 2. Stack Tecnológico
- **Runtime & Lenguaje:** Node.js + TypeScript 5 (`strict mode`), compilación con `tsup` y ejecución en desarrollo con `tsx`.
- **Framework Web:** Express.js (`4.21+`) con arquitectura modular basada en `RouterBroker`.
- **Motor WhatsApp:** `baileys` (`7.0.0-rc.9`) para sesiones multi-dispositivo WhatsApp Web mediante QR / Pairing Code, además de soporte para Meta Business API.
- **ORM y Bases de Datos (`Prisma` v6.16):**
  - Soporte multi-proveedor mediante esquemas dedicados en `prisma/`:
    - PostgreSQL (`postgresql-schema.prisma` / `psql_bouncer-schema.prisma`) — usado en el contenedor `evolution_db`.
    - SQL Server (`sqlserver-schema.prisma`) — esquema adaptado para integración con infraestructura Microsoft SQL Server.
    - MySQL (`mysql-schema.prisma`).
- **Caché y Sesiones:** Redis (`ioredis` / contenedor `evolution_redis`) con fallback local a `node-cache`.
- **Multimedia y Utilidades:** `fluent-ffmpeg` (procesamiento de audio/notas de voz OGG/Opus), `sharp` / `jimp` (imágenes), `qrcode`, `axios`, `pino` (logs estructurados).

---

## 3. Comandos Frecuentes
- **Servidor de desarrollo (hot reload):** `npm run dev:server`
- **Compilar para producción:** `npm run build`
- **Iniciar en producción:** `npm run start:prod`
- **Generar cliente Prisma (definir `DATABASE_PROVIDER` antes):**
  ```powershell
  $env:DATABASE_PROVIDER="postgresql" # o sqlserver / mysql
  npm run db:generate
  ```
- **Ejecutar migraciones en Windows:**
  ```powershell
  $env:DATABASE_PROVIDER="postgresql"
  npm run db:migrate:dev:win
  # En producción:
  npm run db:deploy:win
  ```
- **Linter y formato:** `npm run lint` / `npm run lint:check`

---

## 4. Arquitectura e Integración con el Ecosistema NQuito

### 1. Estructura Modular (`src/`)
- **`api/routes/` (`RouterBroker`)**: Define rutas Express validando entradas con `JSONSchema7` (`src/validate/`).
- **`api/controllers/`**: Capa delgada que delega operaciones a los servicios.
- **`api/services/`**: Lógica de negocio principal (`WAMonitoringService`, gestión de instancias, chats, mensajes, contactos y webhooks).
- **`api/integrations/channel/`**: Implementación de proveedores de mensajería, destacando `whatsapp/baileys` (`BaileysStartupService`) que mantiene el ciclo de vida de los sockets de WhatsApp Web.
- **`api/integrations/event/`**: Despachadores de eventos hacia sistemas externos (Webhook HTTP, WebSocket, RabbitMQ, SQS, NATS, Pusher).
- **`repository/`**: Capa de acceso a datos mediante `PrismaRepository`.

### 2. Flujo de Integración con `FacebookMessengerApi` y `responde.nquito.com`
1. **Autenticación entre Servicios:** `FacebookMessengerApi` consume los endpoints de Evolution API enviando el encabezado `apikey` configurado por empresa (`Configuracion.EvolutionApiUrl` y `Configuracion.EvolutionApiKey`).
2. **Convención de Nombres de Instancia (Multi-Tenant):**
   - Instancias personales de agentes: `usr_{UsuarioId}_emp_{EmpresaId}`.
   - En `MetaDbContext` (SQL Server del backend `.NET`), las conversaciones provenientes de estas instancias se registran bajo el canal `baileys|usr_{UsuarioId}_emp_{EmpresaId}`, permitiendo aislar la bandeja de WhatsApp Personal de cada asesor en `chat-view`.
3. **Ciclo de Vida de Instancias (`EvolutionController` / `EvolutionApiService`):**
   - Creación: `POST /instance/create` con `integration: "WHATSAPP-BAILEYS"` y `qrcode: true`.
   - Conexión QR: `GET /instance/connect/{instanceName}`.
   - Configuración automática de Webhook: apunta a `https://webhooks.nquito.com/webhook/evolution/{empresaId}`.
   - Cierre/Eliminación: `DELETE /instance/delete/{instanceName}`.
4. **Recepción de Eventos (`WebhookController.Evolution.cs`):**
   - Evolution API emite eventos `messages.upsert` al webhook del backend `.NET`.
   - El backend descarga audios/adjuntos mediante `/chat/getBase64FromMediaMessage/{instanceName}`, transcribe notas de voz con `GoogleCloudSttService` y enruta el mensaje hacia el **Bot de Flujos**, el **Enjambre de Agentes IA (`SwarmOrchestratorService`)** o la bandeja del **Agente Humano**.

---

## 5. Convenciones de Código y Reglas Operativas
1. **Aislamiento por Instancia:** Toda operación y consulta a base de datos o a `waMonitor.waInstances[instanceName]` debe estar estrictamente acotada por `instanceName` o `instanceId`.
2. **Validación y DTOs:** Usar clases simples en `src/dto/` (sin decoradores de `class-validator`) y validar los payloads en los routers mediante esquemas `JSONSchema7` (`this.dataValidate`).
3. **Compatibilidad Multi-Base de Datos:** Al modificar modelos en Prisma, mantener sincronizados los esquemas en `prisma/postgresql-schema.prisma`, `prisma/sqlserver-schema.prisma` y `prisma/mysql-schema.prisma`.
4. **Seguridad:** Nunca versionar archivos `.env` con claves reales (`AUTHENTICATION_API_KEY`, cadenas de conexión de producción) ni exponer el puerto de Evolution API sin validación de `apikey`.
