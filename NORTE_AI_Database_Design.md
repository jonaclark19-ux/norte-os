# NORTE AI — Diseño de Base de Datos (PostgreSQL / Supabase)

**Versión:** 1.0
**Rol de este documento:** especificación completa de base de datos, lista para ejecutarse en el SQL Editor de Supabase por secciones, en el orden en que aparecen aquí.
**Documentos relacionados:** `NORTE_AI_System_Architecture.md`, `PRD.md`.
**Nota de alcance:** este documento refina y reemplaza el esquema simplificado presentado en `NORTE_AI_Arquitectura.md` v1.0 — añade auditoría, soft delete, versionado y particionado, que no estaban en esa primera versión.

---

## 0. Principios de diseño para escalar durante años

1. **No todas las tablas necesitan el mismo nivel de protección.** Auditoría, soft delete y versionado se aplican donde el contenido es editable y valioso para el usuario (proyectos, tareas, notas, diario, recordatorios, metas) — no a tablas de telemetría que ya son inmutables por diseño (`context_logs`, `voice_sessions`, el propio `audit_log`).
2. **Las decisiones que son costosas de revertir se toman temprano.** Convertir una tabla no particionada en particionada más adelante requiere migrar datos; por eso `audit_log` (la tabla de mayor crecimiento esperado) nace particionada desde el día 1, aunque el volumen actual no lo requiera todavía.
3. **Un solo mecanismo de historial, no tres redundantes.** "Auditoría", "versionado" e "historial" se piden como requisitos separados, pero construir tres sistemas que almacenan lo mismo tres veces es un error de diseño. Aquí se construye **un** `audit_log` genérico que sirve a los tres propósitos, cada uno con su propia forma de consultarlo (sección 9-10 explica la distinción).
4. **RLS asegura, las queries filtran.** La seguridad (quién puede ver un registro) vive en las políticas de RLS. La visibilidad de negocio (si se muestra un registro borrado o no) vive en cómo se consulta, no en la política de seguridad — mezclar ambas cosas hace las políticas más difíciles de razonar con el tiempo.
5. **Tipos de datos precisos desde el inicio.** `smallint` en vez de `integer` donde el rango es acotado (prioridades, progreso, energía), `timestamptz` siempre (nunca `timestamp` sin zona horaria), `text` en vez de `varchar(n)` (no hay ventaja de rendimiento en Postgres y `varchar(n)` solo agrega límites arbitrarios que eventualmente hay que migrar).

---

## 1. Extensiones requeridas

```sql
create extension if not exists pgcrypto;   -- gen_random_uuid()
create extension if not exists vector;     -- pgvector, columnas embedding (activas en Fase 2)
```

**Justificación:** ambas extensiones están disponibles de forma nativa en Supabase. Se activa `vector` desde ahora, aunque no se use hasta Fase 2, para evitar una migración de esquema (agregar una columna `vector` a una tabla ya poblada) más adelante — el costo de tenerla ya lista es cero.

---

## 2. Convenciones generales

| Convención | Regla | Justificación |
|---|---|---|
| Primary keys | `uuid default gen_random_uuid()` | No enumerable (a diferencia de `serial`), seguro de exponer en URLs, sin coordinación necesaria si en el futuro hay múltiples regiones/réplicas de escritura |
| Timestamps | Siempre `timestamptz`, nunca `timestamp` | El sistema depende de zona horaria correcta para recordatorios y motor de contexto — un `timestamp` sin zona es una fuente silenciosa de bugs a largo plazo |
| Texto | `text`, no `varchar(n)` | Sin diferencia de rendimiento en Postgres; `varchar(n)` solo impone un límite arbitrario que eventualmente requiere migración |
| Enumeraciones | `CHECK` sobre `text`, no `ENUM` nativo de Postgres | Añadir un valor a un `CHECK` es una migración trivial (`ALTER TABLE ... DROP CONSTRAINT / ADD CONSTRAINT`); añadir un valor a un `ENUM` nativo tiene más fricción operativa (no se puede hacer dentro de una transacción en versiones antiguas de Postgres, y no se puede remover un valor sin recrear el tipo) |
| Campos numéricos acotados | `smallint` con `CHECK` de rango (prioridad, progreso, energía) | Ahorro de almacenamiento real a escala de años y millones de filas (2 bytes vs. 4), sin ningún costo funcional |
| Zona horaria del usuario | `text` con nombre IANA (ej. `'America/Denver'`), no un offset numérico | Los nombres IANA manejan horario de verano correctamente; un offset fijo se desincroniza dos veces al año |

---

## 3. Esquema completo

### 3.1 `profiles`

```sql
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  name text,
  timezone text not null default 'America/Denver',
  main_goals text[] not null default '{}',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  version integer not null default 1
);
```

**Justificación de diseño:**
- `id` referencia directamente a `auth.users(id)` en vez de tener su propio UUID independiente — un perfil no existe sin una cuenta de autenticación, y esta relación 1:1 es la forma natural de expresarlo en Supabase.
- Sin `deleted_at`: un perfil no se "borra suavemente" de forma independiente — borrar la cuenta es un evento distinto (ver sección 8.3, retención de cuenta), no una operación de contenido como borrar una nota.
- `main_goals text[]` es deliberadamente distinto de la tabla `goals` (sección 3.7): aquí es una lista breve de temas/intenciones usada como contexto para los prompts de IA (ej. "libertad financiera", "enfoque"), no una entidad rastreable con progreso. `goals` sí lo es. Mantenerlos separados evita forzar una sola tabla a cumplir dos propósitos distintos.

### 3.2 `projects`

```sql
create table projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  description text,
  status text not null default 'active'
    check (status in ('active','paused','done','idea')),
  priority smallint not null default 3
    check (priority between 1 and 5),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1
);
```

### 3.3 `tasks`

```sql
create table tasks (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  project_id uuid references projects(id) on delete set null,
  title text not null,
  description text,
  status text not null default 'pending'
    check (status in ('pending','in_progress','done','blocked')),
  priority smallint not null default 3
    check (priority between 1 and 5),
  estimated_minutes integer
    check (estimated_minutes is null or estimated_minutes > 0),
  due_date timestamptz,
  energy_required text check (energy_required in ('low','medium','high')),
  context_required text check (context_required in ('computer','phone','anywhere')),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1
);
```

**Justificación de `on delete set null` en `project_id`:** si un proyecto llegara a eliminarse permanentemente (fuera del flujo normal de soft delete, ej. una purga administrativa), sus tareas no deben desaparecer con él — quedan como tareas sin proyecto asignado en vez de perderse. Perder una tarea porque su proyecto fue purgado sería un daño silencioso e innecesario.

### 3.4 `notes`

```sql
create table notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  category text not null check (category in (
    'idea','habito','pensamiento_recurrente','bloqueo',
    'decision','nota_general','planificacion'
  )),
  content text not null,
  related_project_id uuid references projects(id) on delete set null,
  embedding vector(1536),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1
);
```

**Justificación de `embedding vector(1536)` presente pero sin usar en el MVP:** agregar esta columna ahora cuesta cero (una columna nula no ocupa espacio significativo); agregarla después, sobre una tabla ya poblada en producción, es una migración con posible bloqueo de tabla en volúmenes grandes. Se paga el costo mínimo hoy para evitar el costo mayor después — ejemplo directo del principio de diseño #2.

### 3.5 `journal_entries`

```sql
create table journal_entries (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  mood text,
  energy_level smallint check (energy_level between 1 and 5),
  emotional_state text,
  content text,
  source_mode text not null default 'terapia',
  had_safety_flag boolean not null default false,
  embedding vector(1536),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1
);
```

**Justificación de `had_safety_flag`:** el PRD (JOURNAL-004) exige que, si se activa el piso de seguridad de Modo Terapia, el evento quede registrado para trazabilidad. Este campo es esa trazabilidad a nivel de dato — permite consultar después "¿cuándo se activó esto?" sin tener que inferirlo del contenido en lenguaje natural.

### 3.6 `reminders`

```sql
create table reminders (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  title text not null,
  description text,
  remind_at timestamptz not null,
  status text not null default 'pending'
    check (status in ('pending','sent','done','cancelled')),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1
);
```

### 3.7 `goals`

```sql
create table goals (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  title text not null,
  timeframe text check (timeframe in ('semana','mes','trimestre','anio')),
  status text not null default 'active'
    check (status in ('active','achieved','abandoned')),
  progress smallint not null default 0
    check (progress between 0 and 100),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1
);
```

### 3.8 `context_logs`

```sql
create table context_logs (
  id uuid default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  time_of_day text,
  location_context text check (location_context in ('trabajo','casa','manejando','otro')),
  available_time_minutes integer
    check (available_time_minutes is null or available_time_minutes >= 0),
  energy_level smallint check (energy_level between 1 and 5),
  device_available text check (device_available in ('phone','computer')),
  source text not null default 'manual' check (source in ('auto','manual','mixed')),
  created_at timestamptz not null default now(),
  primary key (id, created_at)
) partition by range (created_at);
```

**Justificación de que esta tabla NO tenga `deleted_at`, `version` ni auditoría:** un registro de contexto es, por naturaleza, un evento inmutable ("así estaba el usuario en este momento"). Nadie edita un `context_log` pasado — se crea uno nuevo. Aplicarle soft delete o versionado sería proteger algo que nunca cambia. Ya nace particionada por mes (ver sección 11) porque es la tabla de mayor frecuencia de escritura de todo el sistema.

### 3.9 `voice_sessions` *(tabla nueva, propuesta en este documento)*

```sql
create table voice_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  mode text check (mode in
    ('captura_rapida','proyecto','enfoque','terapia','revision_semanal')),
  started_at timestamptz not null default now(),
  ended_at timestamptz,
  tool_calls_count integer not null default 0,
  had_safety_flag boolean not null default false,
  created_at timestamptz not null default now()
);
```

**Por qué se agrega esta tabla, que no estaba en versiones anteriores del esquema:** `NORTE_AI_System_Architecture.md` identifica el costo de la Capa de Voz como el cuello de botella real de escalabilidad del sistema (sección 12.2 de ese documento). Sin una tabla que registre duración y modo de cada sesión, no hay forma de medir ese costo, ni de saber qué modos se usan más para priorizar desarrollo futuro. Es una tabla de telemetría — por eso, igual que `context_logs`, no lleva soft delete ni versionado.

### 3.10 `audit_log` *(tabla nueva, transversal)*

```sql
create table audit_log (
  id uuid default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  table_name text not null,
  record_id uuid not null,
  action text not null check (action in ('INSERT','UPDATE','DELETE')),
  old_data jsonb,
  new_data jsonb,
  changed_at timestamptz not null default now(),
  primary key (id, changed_at)
) partition by range (changed_at);
```

Ver secciones 9 y 10 para el mecanismo que la alimenta y cómo se consulta.

---

## 4. Relaciones

```
auth.users (Supabase)
    │ 1:1
    ▼
profiles
    │
    │ 1:N (user_id) — a todas las tablas siguientes
    │
    ├── projects ──┐
    │              │ 1:N (project_id, nullable)
    │              ├──▶ tasks
    │              └──▶ notes (related_project_id, nullable)
    │
    ├── tasks
    ├── notes
    ├── journal_entries
    ├── reminders
    ├── goals
    ├── context_logs
    ├── voice_sessions
    └── audit_log (registra cambios de todas las anteriores)
```

| Relación | Cardinalidad | On Delete |
|---|---|---|
| `auth.users` → `profiles` | 1:1 | CASCADE |
| `auth.users` → cada tabla de usuario | 1:N | CASCADE |
| `projects` → `tasks` | 1:N (opcional) | SET NULL |
| `projects` → `notes` | 1:N (opcional) | SET NULL |

---

## 5. Índices

| Tabla | Índice | Propósito |
|---|---|---|
| Todas las tablas de usuario | `(user_id)` | Acelera el filtrado que RLS aplica en cada query |
| `tasks` | `(user_id, status)` | Dashboard y Modo Proyecto: "tareas pendientes" |
| `tasks` | `(project_id)` | Consultas de Modo Proyecto |
| `tasks` | `(user_id, due_date)` | Recordatorios y priorización por fecha |
| `notes` | `(user_id, category)` | Filtrado por categoría (ej. revisar todos los "bloqueo") |
| `notes` | `(related_project_id)` | Notas asociadas a un proyecto |
| `journal_entries` | `(user_id, created_at desc)` | Rango temporal para Revisión Semanal |
| `reminders` | `(user_id, remind_at) where status = 'pending'` | Índice parcial — mucho más pequeño que uno completo porque solo indexa recordatorios activos, que es la única consulta operativa real ("¿qué recordatorios están por vencer?") |
| `context_logs` | BRIN sobre `(created_at)` | Datos append-only, naturalmente ordenados por tiempo — un índice BRIN ocupa una fracción del espacio de un B-tree equivalente y es igual de efectivo en este patrón de acceso |
| `audit_log` | `(table_name, record_id, changed_at)` | Reconstruir el historial completo de un registro específico |
| `audit_log` | `(user_id, changed_at)` | Consultas generales de auditoría por usuario |
| `voice_sessions` | `(user_id, started_at desc)` | Listado de sesiones recientes, cálculo de uso/costo |
| `notes.embedding`, `journal_entries.embedding` | Índice `ivfflat` o `hnsw` (pgvector) | **No se crea en el MVP** — un índice vectorial sobre una columna vacía no aporta nada; se crea en Fase 2 cuando haya datos que indexar |

---

## 6. Row Level Security

**Patrón aplicado de forma idéntica a:** `projects`, `tasks`, `notes`, `journal_entries`, `reminders`, `goals`, `context_logs`, `voice_sessions`, `audit_log`, y `profiles` (con `id` en vez de `user_id`).

```sql
alter table projects enable row level security;

create policy "select_own_rows"
  on projects for select
  using (auth.uid() = user_id);

create policy "insert_own_rows"
  on projects for insert
  with check (auth.uid() = user_id);

create policy "update_own_rows"
  on projects for update
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create policy "delete_own_rows"
  on projects for delete
  using (auth.uid() = user_id);
```

**Decisión explícita: la política de `select` no filtra `deleted_at`.** RLS responde a "¿este registro le pertenece a quien pregunta?", no a "¿debería mostrarse en la pantalla principal?". Filtrar por `deleted_at is null` es responsabilidad de cada query de la aplicación (o de una vista, ver sección 8.2) — mezclar seguridad con lógica de presentación en la misma política hace que, con los años, sea más difícil auditar qué es una regla de seguridad y qué es una regla de negocio.

**`delete` sigue existiendo a nivel de política** como salvaguarda técnica, aunque el flujo normal de la aplicación nunca la invoque directamente (usa soft delete). Sirve para tareas administrativas de purga (sección 8.3).

---

## 7. Auditoría

### 7.1 Función de trigger genérica

```sql
create or replace function fn_audit_log() returns trigger as $$
begin
  if (tg_op = 'DELETE') then
    insert into audit_log (user_id, table_name, record_id, action, old_data)
    values (old.user_id, tg_table_name, old.id, 'DELETE', to_jsonb(old));
    return old;
  elsif (tg_op = 'UPDATE') then
    insert into audit_log (user_id, table_name, record_id, action, old_data, new_data)
    values (new.user_id, tg_table_name, new.id, 'UPDATE', to_jsonb(old), to_jsonb(new));
    return new;
  elsif (tg_op = 'INSERT') then
    insert into audit_log (user_id, table_name, record_id, action, new_data)
    values (new.user_id, tg_table_name, new.id, 'INSERT', to_jsonb(new));
    return new;
  end if;
end;
$$ language plpgsql security definer;
```

Se aplica como trigger `after insert or update or delete` en cada tabla de contenido de usuario (`projects`, `tasks`, `notes`, `journal_entries`, `reminders`, `goals`).

**Justificación de una función genérica en vez de una tabla de auditoría por entidad:** una tabla de auditoría por tabla (`projects_audit`, `tasks_audit`, etc.) duplica seis veces la misma estructura y exige seis veces el mantenimiento cuando cambie el formato de auditoría. Una tabla genérica con `jsonb` se adapta automáticamente a cualquier cambio de esquema en las tablas auditadas, sin requerir su propia migración.

### 7.2 Por qué `security definer`
La función corre con los privilegios de quien la definió (no de quien dispara el trigger), lo que le permite insertar en `audit_log` sin que el usuario final necesite permiso de escritura directo sobre esa tabla — el usuario nunca debería poder insertar manualmente una entrada de auditoría falsa.

---

## 8. Soft Delete

### 8.1 Patrón
Toda tabla de contenido de usuario tiene `deleted_at timestamptz` (nulo por defecto). "Borrar" desde la aplicación significa `update ... set deleted_at = now()`, nunca un `delete` real.

**Justificación:** preserva la integridad referencial de reportes históricos (si se borra un proyecto, la Revisión Semanal de la semana pasada no debe romperse ni mostrar huecos), permite deshacer un borrado accidental, y — dado que este documento ya definió `audit_log` — cada soft delete queda de todas formas registrado como un evento `UPDATE` normal, sin necesidad de lógica especial.

### 8.2 Cascada de soft delete: proyecto → tareas y notas

```sql
create or replace function fn_cascade_soft_delete_project() returns trigger as $$
begin
  if new.deleted_at is not null and old.deleted_at is null then
    update tasks set deleted_at = new.deleted_at
      where project_id = new.id and deleted_at is null;
    update notes set deleted_at = new.deleted_at
      where related_project_id = new.id and deleted_at is null;
  end if;
  return new;
end;
$$ language plpgsql;

create trigger trg_cascade_soft_delete_project
  after update on projects
  for each row execute function fn_cascade_soft_delete_project();
```

**Justificación:** sin esta cascada, borrar un proyecto dejaría sus tareas y notas "vivas" pero huérfanas de contexto visible — el usuario las seguiría viendo sueltas en otras pantallas (ej. una vista general de tareas) después de haber "borrado" el proyecto al que pertenecían, lo cual es una inconsistencia confusa de producto, no solo técnica.

### 8.3 Retención y purga (eliminación de cuenta)
El soft delete no es indefinido por diseño: se recomienda un job programado (fuera del alcance de este documento de esquema) que purgue —con `delete` real— registros con `deleted_at` de más de 90 días. Para eliminación de cuenta completa, el flujo recomendado es: (1) soft delete de todos los datos del usuario con periodo de gracia, (2) purga real después del periodo de gracia o a solicitud explícita del usuario. El `on delete cascade` en cada `user_id` sigue existiendo como salvaguarda técnica de última instancia (si `auth.users` se elimina directamente), pero el flujo normal de producto pasa primero por el soft delete.

---

## 9. Versionado

```sql
create or replace function fn_set_updated_at_and_version() returns trigger as $$
begin
  new.updated_at = now();
  new.version = old.version + 1;
  return new;
end;
$$ language plpgsql;
```

Se aplica como trigger `before update` en cada tabla con columna `version` (todas las de contenido de usuario, sección 3).

**Para qué sirve el número de versión, más allá de saber "cuántas veces cambió":**
- **Bloqueo optimista (optimistic locking):** si el mismo usuario edita el mismo registro desde dos dispositivos casi al mismo tiempo (caso EC-014 del PRD), el cliente puede enviar la versión que tenía al momento de editar; si no coincide con la versión actual en base de datos, la escritura se rechaza en vez de sobrescribir silenciosamente un cambio más reciente.
- **Ancla para reconstruir historial:** cada fila de `audit_log` para un registro dado corresponde a una transición de versión específica — se puede reconstruir exactamente "cómo era este registro en la versión 3" leyendo las entradas de `audit_log` en orden hasta llegar a esa versión, sin necesidad de una tabla de versiones separada que duplique el contenido completo en cada cambio.

---

## 10. Historial

**Historial no es una tabla nueva — es una forma de consultar `audit_log`.** Esta es la aplicación directa del principio de diseño #3.

Ejemplos de lo que el historial permite responder, sin ninguna infraestructura adicional a la ya definida:
- *"¿Cómo evolucionó el estado de esta tarea?"* → filtrar `audit_log` por `table_name = 'tasks'`, `record_id = <id>`, ordenado por `changed_at`, leyendo el campo `status` dentro de `new_data` en cada fila.
- *"¿Qué bloqueos se repitieron este mes?"* → consulta agregada sobre `audit_log` filtrada por `table_name = 'notes'` y `new_data->>'category' = 'bloqueo'`, agrupada por semana — la misma fuente de datos que alimentaría, más adelante, el motor de detección de patrones de Fase 2/3 descrito en `NORTE_AI_Arquitectura.md`.
- *"¿Qué cambió en este proyecto la semana pasada?"* → base directa del Modo Revisión Semanal.

Esto significa que el trabajo de instrumentar `audit_log` correctamente ahora paga dividendos directos en una funcionalidad de producto (detección de patrones) que originalmente estaba planeada como trabajo de Fase 2/3 separado — el esquema de base de datos ya la deja lista.

---

## 11. Escalabilidad futura

### 11.1 Particionado — ya implementado, no solo planeado
`audit_log` y `context_logs` se crearon `partition by range` desde el día 1 (secciones 3.8 y 3.10), aunque el volumen actual de un solo usuario no lo requiera. **Esta es la decisión de escalabilidad más importante de todo el documento:** convertir una tabla existente y poblada en una tabla particionada en Postgres no es un `ALTER TABLE` simple — requiere crear una tabla nueva particionada y migrar los datos, lo cual implica downtime o complejidad operativa considerable en producción. Nacer particionada evita ese problema por completo.

```sql
-- Ejemplo de partición mensual (se repite para cada mes)
create table audit_log_2026_07 partition of audit_log
  for values from ('2026-07-01') to ('2026-08-01');

create table context_logs_2026_07 partition of context_logs
  for values from ('2026-07-01') to ('2026-08-01');
```

**Recomendación operativa:** automatizar la creación de la partición del mes siguiente con una función programada (cron job o Edge Function de Supabase) en vez de crearlas manualmente — verificar disponibilidad de `pg_partman` en el plan de Supabase que se esté usando; si no está disponible, un job simple que ejecute el `create table ... partition of ...` del mes siguiente es suficiente y no depende de esa extensión.

### 11.2 Archivado
Una vez que el sistema tenga suficiente historia (multi-tenant real, no solo el MVP personal), las particiones de `audit_log` de más de 12-24 meses son candidatas a moverse a almacenamiento más económico (ej. exportarlas y quitarlas del clúster activo) — esto es una operación natural sobre una tabla particionada (`detach partition`) y prácticamente inviable sobre una tabla plana del mismo tamaño.

### 11.3 pgvector — activación en Fase 2
Cuando `notes.embedding` y `journal_entries.embedding` empiecen a poblarse, se crea el índice vectorial correspondiente:
```sql
create index on notes using hnsw (embedding vector_cosine_ops);
```
`hnsw` se recomienda sobre `ivfflat` para este caso de uso porque no requiere una fase de entrenamiento sobre un volumen mínimo de datos — importante dado que, incluso en Fase 2, el volumen por usuario individual seguirá siendo relativamente bajo comparado con los casos de uso típicos donde `ivfflat` brilla.

### 11.4 Multi-tenancy a escala
La aislación de datos vía RLS (sección 6) ya es la misma, sin cambios, para un usuario o para cien mil. Lo que cambia con la escala es infraestructura administrada por Supabase (réplicas de lectura, pooling de conexiones vía PgBouncer) — no el esquema. Esto es deliberado: el esquema no debería necesitar rediseño solo porque cambió el número de usuarios, únicamente porque cambiaron los requisitos de producto.

### 11.5 Índices no usados
Con los años, algunos de los índices definidos en la sección 5 pueden dejar de ser útiles a medida que cambian los patrones reales de consulta. Se recomienda revisar periódicamente `pg_stat_user_indexes` (vista nativa de Postgres) para detectar índices con uso cercano a cero y eliminarlos — cada índice no usado sigue consumiendo espacio y ralentizando escrituras sin aportar nada a cambio.

---

## 12. Checklist de mantenimiento a largo plazo

- [ ] Automatizar la creación de particiones futuras de `audit_log` y `context_logs` (sección 11.1) antes de que falte la partición del mes siguiente.
- [ ] Job periódico de purga de registros con `deleted_at` mayor a 90 días (sección 8.3).
- [ ] Revisión trimestral de `pg_stat_user_indexes` para eliminar índices no utilizados (sección 11.5).
- [ ] Activar el índice vectorial (`hnsw`) sobre `embedding` en cuanto arranque Fase 2 — no antes, no después.
- [ ] Monitorear el tamaño de `audit_log` como proxy directo de actividad del sistema — es, por diseño, la tabla que más rápido crece.
- [ ] Antes de abrir el sistema a usuarios externos (beta), auditar manualmente que las políticas RLS de la sección 6 estén activas en el 100% de las tablas — un `enable row level security` olvidado en una sola tabla nueva es el error de seguridad más común y más grave en sistemas multi-tenant.

---

## Nota de cierre

Este esquema es intencionalmente más elaborado que lo estrictamente necesario para un solo usuario en el MVP — es la aplicación directa del principio rector de `NORTE_AI_System_Architecture.md`: *"complejidad ganada, no anticipada"*, con una excepción deliberada y justificada: el particionado de `audit_log` y `context_logs`, donde el costo de anticiparse es mínimo y el costo de no hacerlo es una migración dolorosa en producción.
