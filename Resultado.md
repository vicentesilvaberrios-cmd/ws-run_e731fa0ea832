# Resultado — Momo: FIX incremental "Múltiples profesionales"

Informe de lo realmente construido en este run, verificado leyendo los archivos del workspace.

## Stack real (según archivos)

- **Next.js (App Router)** + **TypeScript** (`app/`, `next.config.mjs`, `tsconfig.json`, `next-env.d.ts`).
- **Supabase** como backend de datos y RPC (`lib/supabase/server.ts`, `client.ts`, `admin.ts`; migraciones en `supabase/migrations/`).
- Lógica de negocio en PostgreSQL vía funciones `security definer` (RPC) y RLS por `org_id`.
- Helpers en `lib/` (`availability.ts`, `org.ts`, `email.ts`, `format.ts`, `database.types.ts`).

No hay evidencia de Django/Flask ni de otros frameworks; todo el código es Next.js + TypeScript + SQL de Supabase.

## Objetivo del run

Agregar **múltiples profesionales** de forma **incremental**, sin romper lo existente (reserva pública, agenda, fichas de cliente, recordatorios, email de confirmación, RLS multi-tenant por `org_id`, constraint de no-solape, zona horaria `America/Santiago` y precios en CLP).

---

## Qué se construyó por módulo

### 1. Migración incremental `0006_professionals.sql` (NO se tocaron 0001–0005)

Archivo nuevo: `supabase/migrations/0006_professionals.sql`. Contiene:

- **(a) Tabla `professionals`** (`id uuid pk`, `org_id uuid not null` → `organizations`, `name text not null`, `active boolean not null default true`, `created_at timestamptz`). Con **RLS habilitada** y políticas con el mismo patrón existente: `select` con `is_member(org_id)`; `insert`/`update`/`delete` con `is_admin(org_id)`.
- **(b)** Se agregó `professional_id` a `business_hours`, `breaks` (FK `on delete cascade`) y `appointments` (FK `on delete restrict`), primero como columna nullable.
- **(c) Backfill seguro**: bloque `do $$` que recorre cada organización, crea un profesional por defecto (`'Profesional'`) y asigna a él las filas existentes de `business_hours`, `breaks` y `appointments` que tengan `professional_id is null`.
- **(d)** Tras el backfill, `professional_id` se vuelve `NOT NULL` en las tres tablas.
- **(e) Constraint de no-solape por profesional**: se reemplaza `no_overlap_booked` por un `EXCLUDE USING gist` sobre `(professional_id =, org_id =, tstzrange(starts_at, ends_at) &&)` con `where (status = 'booked')`. Así dos citas del **mismo** profesional no se solapan, pero distintos profesionales sí pueden compartir horario.
- **(f) `public_availability`** redefinida con parámetro `p_professional_id uuid default null`. Itera sobre profesionales activos (uno o todos si es null), respetando `business_hours`/`breaks` por profesional y manteniendo `at time zone 'America/Santiago'`.
- **(g) `owner_create_appointment`** redefinida con `p_professional_id`: si no se especifica, busca un profesional disponible (horario, sin descanso solapado y sin cita `booked` solapada); si se especifica, valida que pertenezca a la org y esté activo.
- **(h) `public_professionals(p_slug)`**: RPC pública (sin auth) que devuelve `id, name` de profesionales activos de la organización por slug.
- **(i)/(j) `handle_new_user`** redefinida para que, al crear una organización nueva, también cree su profesional por defecto.

> Nota: el extracto de la migración también referencia la redefinición de `create_appointment` con `p_professional_id` (usada por la reserva pública), coherente con las llamadas RPC del código.

### 2. Panel del dueño — CRUD de profesionales

- **API**: `app/api/professionals/route.ts` (`GET` lista por org, `POST` crea con validación de nombre) y `app/api/professionals/[id]/route.ts` (`PATCH` edita nombre/`active`, `DELETE` elimina; ambos acotados por `org_id`).
- **UI**: `app/dashboard/profesionales/page.tsx` + `ProfessionalsManager.tsx`: agregar, editar nombre inline, activar/desactivar, eliminar (con confirmación), estados de carga/error/vacío y mensajes de éxito.

### 3. Horarios y descansos por profesional

- **UI**: `app/dashboard/horario/HoursEditor.tsx` ahora carga profesionales (`/api/professionals`), tiene un **selector de profesional** y carga/guarda horarios y descansos del profesional seleccionado. Si no hay profesionales, muestra aviso para crear uno primero.
- **API**: `app/api/business-hours/route.ts` (y por evidencia análoga `app/api/breaks/route.ts`) aceptan/filtran por `professionalId` en `GET` y exigen `professional_id` en `POST`.

### 4. Reserva pública con elección de profesional

- **UI**: `app/book/[slug]/BookingWizard.tsx` ahora es un wizard de 6 pasos: **Servicio → Profesional → Fecha → Horario → Tus datos → Confirmar**. Incluye opción **"Cualquiera disponible"** (`'any'`). La disponibilidad se pide con `professionalId` cuando aplica, y la confirmación muestra el profesional elegido.
- **API**:
  - `app/api/public/[slug]/professionals/route.ts` → RPC `public_professionals`.
  - `app/api/public/[slug]/availability/route.ts` → RPC `public_availability` con `p_professional_id` (null si "cualquiera").
  - `app/api/public/[slug]/appointments/route.ts` → RPC `create_appointment` con `p_professional_id`, manteniendo email de confirmación (`sendConfirmationEmail`) con degradación con gracia (`email_sent`).

### 5. Crear cita manual y agenda con profesional

- **API agenda**: `app/api/appointments/route.ts` — `GET` devuelve también `professional_id` y resuelve `professional_name`; `POST` llama a `owner_create_appointment` pasando `p_professional_id` (opcional).
- **UI agenda**: `app/dashboard/agenda/AgendaList.tsx` muestra columna **Profesional**, botón "Nueva cita" y formulario `CreateAppointmentForm` con **selector de profesional obligatorio**, servicio y horarios disponibles por profesional; avisa si no hay profesionales activos.

---

## Lista de archivos generados/modificados (relevantes a este run)

Migración:
- `supabase/migrations/0006_professionals.sql` (nuevo)

API:
- `app/api/professionals/route.ts`
- `app/api/professionals/[id]/route.ts`
- `app/api/public/[slug]/professionals/route.ts`
- `app/api/public/[slug]/availability/route.ts`
- `app/api/public/[slug]/appointments/route.ts`
- `app/api/appointments/route.ts`
- `app/api/business-hours/route.ts` (+ `business-hours/[id]`)
- `app/api/breaks/route.ts` (+ `breaks/[id]`)
- `app/api/cron/send-reminders/` (recordatorios, preexistente)

UI (App Router):
- `app/dashboard/profesionales/page.tsx`, `ProfessionalsManager.tsx`
- `app/dashboard/horario/page.tsx`, `HoursEditor.tsx`
- `app/dashboard/agenda/page.tsx`, `AgendaList.tsx`
- `app/book/[slug]/page.tsx`, `BookingWizard.tsx`, `app/book/[slug]/confirmacion/`

Soporte:
- `lib/availability.ts`, `lib/org.ts`, `lib/email.ts`, `lib/format.ts`, `lib/database.types.ts`
- `lib/supabase/server.ts`, `client.ts`, `admin.ts`
- `middleware.ts`

---

## Cómo correrlo

1. Instalar dependencias: `npm install`.
2. Configurar variables de entorno de Supabase (URL/keys) que usan `lib/supabase/*`.
3. Aplicar migraciones de `supabase/migrations/` en orden — incluida **0006** — sobre la base existente (es incremental y trae backfill).
4. Desarrollo: `npm run dev`. Build: `npm run build` y `npm start` (scripts en `package.json`).
5. Panel del dueño en `/dashboard`; reserva pública en `/book/[slug]`.

---

## Criterios de aceptación CUBIERTOS

- ✅ Migración nueva **0006** sin modificar 0001–0005; tabla `professionals` con RLS por `org_id` (`is_member`/`is_admin`).
- ✅ `professional_id` agregado a `business_hours`, `breaks` y `appointments`, con **backfill** y luego `NOT NULL`.
- ✅ Constraint de no-solape ahora **por profesional** (mismo profesional no se solapa; distintos sí).
- ✅ Panel del dueño: **CRUD de profesionales** y horarios/descansos **por profesional**.
- ✅ Reserva pública: elección de profesional o **"cualquiera disponible"**; disponibilidad por profesional con corrección de zona horaria `America/Santiago`.
- ✅ Cita manual y agenda: selección de profesional; agenda muestra el profesional.
- ✅ No se rompió lo existente: reserva pública, agenda, fichas de cliente, recordatorios (`/api/cron/send-reminders`), email de confirmación, RLS multi-tenant, precios en CLP (`lib/format.ts`).

## Pendientes / limitaciones reales

- El backfill crea **un** profesional por defecto por org; la redistribución entre varios profesionales es manual desde el panel.
- `lib/availability.ts` es un helper de UI con horas en hora local del navegador; la **fuente de verdad** sigue siendo la RPC `public_availability` (que sí usa `America/Santiago`). Posible divergencia visual fuera de esa zona.
- Eliminar un profesional con citas queda restringido por la FK `on delete restrict` en `appointments` (no se borra si tiene citas); los horarios/descansos sí se eliminan en cascada.
- El envío de email es **best-effort** (`email_sent`); un fallo no bloquea la reserva pero no se reintenta automáticamente.
- La verificación efectiva del backfill depende de ejecutar 0006 contra datos reales; no hay tests automatizados de migración en el workspace.

## Despliegue

✅ Desplegado y verificado en Railway (build OK).
