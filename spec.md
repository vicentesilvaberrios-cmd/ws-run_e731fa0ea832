# Solicitud de cambios (modo fix)

Momo (FIX incremental). NO rompas lo existente: reserva publica, agenda, fichas de cliente, recordatorio (endpoint y reminder_sent_at), email de confirmacion, RLS/multi-tenant por org_id, el constraint de no-solape, zona horaria America/Santiago y precios en CLP. Agrega MULTIPLES PROFESIONALES:

1. Migracion NUEVA incremental 0006 en supabase/migrations (NO modifiques 0001-0005):
   - Tabla professionals (id uuid pk, org_id uuid not null, name text not null, active boolean not null default true, created_at). RLS por org_id con el mismo patron (funciones is_member/is_admin).
   - Agrega professional_id a business_hours, breaks y appointments.
   - BACKFILL seguro para no romper datos: por cada organizacion crea un profesional por defecto y asigna a el las filas existentes de business_hours, breaks y appointments; luego deja professional_id NOT NULL donde corresponda.
   - Cambia el constraint de no-solape para que aplique por professional_id (dos citas del MISMO profesional no se pueden solapar; distintos profesionales si pueden a la misma hora).

2. Panel del duenno: CRUD de profesionales; asignar horarios de atencion y descansos POR profesional.

3. Pagina publica de reserva: el cliente elige el profesional (o una opcion "cualquiera disponible"); la disponibilidad se calcula por profesional. Ajusta la funcion public_availability para considerar el profesional (parametro professional_id) manteniendo la correccion de zona horaria.

4. Crear cita manual y agenda: permitir elegir/mostrar el profesional de cada cita.

Respeta RLS, el no-solape por profesional, la zona horaria Chile y CLP. Alcance acotado a profesionales; no agregues otras features.