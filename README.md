# Agenda Repostería
## Tabla de contenidos

* [Visión](#visión)
* [Características](#características)
* [Arquitectura y stack](#arquitectura-y-stack)
* [Estructura del repo](#estructura-del-repo)
* [Modelado de datos (MVP)](#modelado-de-datos-mvp)
* [Reglas de negocio (MVP)](#reglas-de-negocio-mvp)
* [API (borrador)](#api-borrador)
* [Requisitos](#requisitos)
* [Configuración de entorno](#configuración-de-entorno)
* [Puesta en marcha](#puesta-en-marcha)
* [Base de datos: migraciones y seed](#base-de-datos-migraciones-y-seed)
* [Scripts útiles](#scripts-útiles)
* [Tests y calidad](#tests-y-calidad)
* [Despliegue](#despliegue)
* [Roadmap](#roadmap)
* [Contribuir](#contribuir)
* [Licencia](#licencia)

---

## Visión

**DulceAgenda** permite a una repostería llevar el control de sus pedidos en un solo lugar: ver el calendario de encargos, gestionar clientes, adjuntar referencias, y recibir recordatorios para que nada se olvide.

* MVP enfocado en **agenda de pedidos** y **recordatorios**.
* Extensible hacia **inventario**, **POS**, **checkout online** y **analytics**.

## Características

* 📆 **Calendario** con vista semanal/mensual y filtros por estado (pendiente, en proceso, listo, entregado).
* 🧾 **Pedidos** con fecha/hora, tipo de entrega (retiro/envío), items y notas.
* 👤 **Clientes** con datos de contacto e historial.
* 🔔 **Recordatorios** por email (y futuro WhatsApp) X días/horas antes.
* 🖼️ **Adjuntos** (imágenes de referencia del diseño) por pedido.
* 🔐 **Autenticación** con email/contraseña (cookies httpOnly) y roles (admin/operador).
* 📱 **PWA** (opcional) para usar en el celular y modo offline básico.

## Arquitectura y stack

**Monorepo** con front + API + worker.

* **Front:** React + Vite, TypeScript, Tailwind, React Router, React Hook Form + Zod.
* **API:** Node.js (Fastify o Express), TypeScript, **Prisma ORM**.
* **DB:** PostgreSQL (SQLite para desarrollo rápido).
* **Auth:** JWT con cookies httpOnly, refresh tokens; Zod para validación.
* **Archivos:** S3 compatible (Cloudflare R2 / MinIO) o almacenamiento local en dev.
* **Email:** Nodemailer (SMTP)
* **Realtime (futuro):** Socket.IO o tRPC + websockets.
* **Infra dev:** Docker Compose para DB/MinIO opcionales.
* **CI/CD (sugerido):** GitHub Actions.

## Estructura del repo

```
/apps
  /web        # React (Vite)
  /api        # API Node (Fastify/Express)
  /worker     # Jobs: recordatorios, emails (BullMQ opcional)
/packages
  /ui         # Componentes compartidos (opcional)
  /db         # Prisma schema y cliente
  /config     # tsconfig, eslint, prettier, commitlint
/infra        # Dockerfile, docker-compose, IaC mínimo
```

## Modelado de datos (MVP)

Entidades principales:

* **User**: id, email, password_hash, role.
* **Customer**: id, nombre, teléfono, email, notas.
* **Order**: id, customerId, fechaEntrega (date/time), tipoEntrega (RETIRO|ENVÍO), estado (PENDIENTE|EN_PROCESO|LISTO|ENTREGADO|CANCELADO), total, notas.
* **OrderItem**: id, orderId, producto, variante, cantidad, precioUnitario.
* **Attachment**: id, orderId, url, nombreArchivo.
* **Reminder**: id, orderId, offset (minutos/horas/días antes), canal (EMAIL), enviadoAt.

Diagrama (ASCII simplificado):

```
Customer 1—* Order 1—* OrderItem
                   |
                   *—* Attachment
                   |
                   1—* Reminder
```

> El esquema vive en `/packages/db/schema.prisma`.

## Reglas de negocio (MVP)

* No se pueden crear **pedidos** con fecha en el pasado.
* **Cupos por franja** (opcional): límite de N pedidos por día/hora configurable.
* **Recordatorios**: se planifican al crear/editar un pedido según `offset`.
* **Estados**: solo ciertas transiciones (p.ej., no pasar de ENTREGADO → EN_PROCESO).

## API (borrador)

*Prefijo* `/:version?` → `v1` por defecto. Ej: `/api/v1/orders`.

### Auth

* `POST /api/v1/auth/register` → { email, password }
* `POST /api/v1/auth/login` → cookies de sesión (httpOnly)
* `POST /api/v1/auth/logout`
* `POST /api/v1/auth/refresh`

### Clientes

* `GET /api/v1/customers` (query `q`)
* `POST /api/v1/customers`
* `GET /api/v1/customers/:id`
* `PUT /api/v1/customers/:id`
* `DELETE /api/v1/customers/:id`

### Pedidos

* `GET /api/v1/orders` (filtros: rango de fechas, estado, cliente)
* `POST /api/v1/orders`
* `GET /api/v1/orders/:id`
* `PUT /api/v1/orders/:id`
* `PATCH /api/v1/orders/:id/status` → { estado }
* `DELETE /api/v1/orders/:id`

### Adjuntos

* `POST /api/v1/orders/:id/attachments` (multipart)
* `DELETE /api/v1/attachments/:id`

### Recordatorios

* `POST /api/v1/orders/:id/reminders`
* `POST /api/v1/reminders/:id/send` (test manual)

**Ejemplo de creación de pedido**

```bash
curl -X POST http://localhost:3000/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "cus_123",
    "fechaEntrega": "2025-01-20T15:00:00.000Z",
    "tipoEntrega": "RETIRO",
    "items": [
      {"producto": "Torta chocotorta", "variante": "20 porciones", "cantidad": 1, "precioUnitario": 25000}
    ],
    "notas": "Sin nueces. Feliz cumpleaños Martina"
  }'
```

## Requisitos

* **Node.js** ≥ 20
* **pnpm** ≥ 9 (o npm/yarn, pero usamos pnpm por defecto)
* **Docker** (opcional para DB/MinIO)

## Configuración de entorno

Crear `.env` en raíz del repo o por app (ver `/apps/api/.env.example`).

```env
# General
NODE_ENV=development
APP_URL=http://localhost:3000
CORS_ORIGIN=http://localhost:5173

# Base de datos (Postgres)
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/dulceagenda?schema=public

# Auth
JWT_SECRET=super-secreto-largo
ACCESS_TOKEN_TTL=15m
REFRESH_TOKEN_TTL=7d
COOKIE_DOMAIN=localhost

# Email (SMTP)
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USER=
SMTP_PASS=
SMTP_FROM="DulceAgenda <no-reply@dulceagenda.test>"

# Almacenamiento de archivos
STORAGE_PROVIDER=local # o s3
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=dulceagenda
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin

# Integraciones opcionales
MERCADOPAGO_ACCESS_TOKEN=
GOOGLE_CALENDAR_JSON_BASE64=
```

> Para desarrollo puedes usar **MailHog** (SMTP local) y **MinIO** (S3 local) vía Docker Compose.

## Puesta en marcha

1. **Clonar e instalar dependencias**

```bash
git clone <repo>
cd <repo>
pnpm i
```

2. **Levantar servicios de dev (opcional)**

```bash
# Postgres + MinIO + MailHog
docker compose up -d
```

3. **Migrar base y generar cliente Prisma**

```bash
pnpm db:migrate
pnpm db:generate
```

4. **Semillas de datos (opcional)**

```bash
pnpm db:seed
```

5. **Correr API y Web**

```bash
# Terminal 1
pnpm dev:api
# Terminal 2
pnpm dev:web
```

* Web: `http://localhost:5173`
* API: `http://localhost:3000/api/v1`

## Base de datos: migraciones y seed

* Esquema Prisma en `/packages/db/schema.prisma`.
* Comandos básicos:

```bash
pnpm db:studio   # Prisma Studio
pnpm db:migrate  # aplica migraciones
pnpm db:reset    # reset + seed
```

**Ejemplo de modelo (extracto)**

```prisma
model Order {
  id           String      @id @default(cuid())
  customer     Customer    @relation(fields: [customerId], references: [id])
  customerId   String
  fechaEntrega DateTime
  tipoEntrega  TipoEntrega
  estado       OrderStatus @default(PENDIENTE)
  total        Int         @default(0)
  notas        String?
  items        OrderItem[]
  attachments  Attachment[]
  reminders    Reminder[]
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt
}

enum TipoEntrega { RETIRO ENVIO }

enum OrderStatus { PENDIENTE EN_PROCESO LISTO ENTREGADO CANCELADO }
```

## Scripts útiles

> Ver `package.json` en raíz y en cada app.

* `pnpm dev:web` → Vite en modo dev
* `pnpm dev:api` → API con ts-node-dev
* `pnpm build` → build de web + api
* `pnpm lint` → ESLint
* `pnpm test` → unit/integration
* `pnpm db:*` → utilidades Prisma

## Tests y calidad

* **Unitarios**: lógica de dominio (validaciones, servicios)
* **Integración**: endpoints API con DB de prueba
* **E2E (opcional)**: Playwright sobre la UI
* Lint/format con **ESLint + Prettier**. Commits con **Conventional Commits**.

## Despliegue

* **Render / Railway / Fly.io**
* API y Web como servicios separados. Variables de entorno desde panel.
* DB: Postgres gestionado. Archivos: R2/S3.
* SSL y dominios desde el proveedor.

**Notas**

* Configurar `CORS_ORIGIN` con el dominio real.
* Generar `JWT_SECRET` fuerte.
* Bloquear rutas de admin detrás de rol.

## Roadmap

* [ ] Vista Kanban por estados
* [ ] Búsqueda/filtrado avanzado
* [ ] Recordatorios por WhatsApp (API Business)
* [ ] Integración Google Calendar (sincronización bidireccional)
* [ ] Exportación a PDF de órdenes del día
* [ ] PWA completa (offline + sync)
* [ ] Multi-usuario por tienda (roles y permisos)

## Contribuir

1. Crea un issue con la propuesta
2. Abre una rama `feature/<nombre-corto>`
3. Agrega tests y actualiza docs
4. Abre PR pequeña con screenshots/Notas

**Checklist de PR**

* [ ] Migraciones Prisma incluidas
* [ ] Tests pasan en CI
* [ ] Sin secretos en commits
* [ ] Screenshots de UI (si aplica)

## Licencia

AGPL © 2025 — Ustedes ✨

---

### Anexos (opcionales)

**Colección de Insomnia/Postman**: `./docs/api-collection.json`

**OpenAPI** (si se habilita): servir en `/api/docs` con Swagger UI.

**Accesibilidad**: componentes con focus visible, contraste AA, soporte de teclado.

**i18n**: preparar traducciones con `react-intl` o `i18next`.
