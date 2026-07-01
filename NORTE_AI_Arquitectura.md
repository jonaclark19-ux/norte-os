# NORTE AI — Arquitectura Técnica y Plan de Construcción

**Versión:** 1.0
**Rol de este documento:** brief técnico definitivo, listo para pasar a Claude Code. Contiene las decisiones de producto, la arquitectura de sistema, el modelo de datos y el alcance quirúrgico del MVP.

---

## 1. Resumen ejecutivo

NORTE AI es un sistema operativo personal por voz, en tiempo real, que escucha a Jonathan, clasifica automáticamente lo que dice, lo guarda en una base de datos estructurada, y devuelve la siguiente mejor acción según su contexto (hora, energía, ubicación, tiempo disponible).

Se construye desde el día 1 con arquitectura **lista para multi-tenant** (aunque el primer y único usuario real al inicio sea Jonathan), porque existe intención de convertirlo en producto a futuro.

**Regla de oro del producto (ya definida en el brief original):**
> Cada interacción debe terminar en una de estas salidas: guardar, aclarar, recordar, planificar o ejecutar.

---

## 2. Arquitectura del sistema — dos capas

La decisión clave de este documento es separar el sistema en dos capas independientes. Esto es lo que permite tener voz en tiempo real (tipo llamada) sin construir un pipeline de audio desde cero, y sin atarse a un solo proveedor de IA.

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENTE (PWA)                          │
│   Next.js + React · Micrófono · UI de dashboard/proyectos     │
└───────────────┬─────────────────────────────┬─────────────────┘
                │ WebRTC/WebSocket              │ HTTPS (REST)
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────────┐
│   CAPA DE VOZ (tiempo real) │   │   CAPA DE CEREBRO (backend)     │
│   Realtime Voice API        │──▶│   Next.js API Routes / Edge Fn  │
│   (oído + boca del sistema) │   │   Orquestación + clasificación  │
│                              │   │   Claude API para razonamiento  │
│   - STT streaming            │◀──│   Lee/escribe en Supabase       │
│   - TTS streaming             │   │   Motor de contexto             │
│   - Maneja interrupciones     │   │                                  │
│   - Function calling ──────────▶│   Ejecuta "tools" que la voz llama│
└───────────────────────────┘   └───────────────┬───────────────────┘
                                                    ▼
                                    ┌───────────────────────────┐
                                    │   Supabase                 │
                                    │   Postgres + Auth + RLS    │
                                    │   (pgvector en Fase 2)      │
                                    └───────────────────────────┘
```

### Cómo interactúan las dos capas

1. Jonathan habla → la **Capa de Voz** (Realtime API) transcribe y mantiene la conversación fluida en tiempo real, con interrupciones naturales, como una llamada.
2. Cuando la conversación requiere una acción real (guardar algo, consultar tareas, crear recordatorio), la Capa de Voz **llama una función/tool** — no decide ella misma qué hacer con el dato, solo detecta que hay que actuar.
3. Esa llamada golpea la **Capa de Cerebro** (tu backend en Next.js), que:
   - Usa Claude para clasificar el mensaje en la categoría correcta (tarea, nota, idea, emoción, etc.)
   - Aplica las reglas del modo activo (Proyecto, Enfoque, Terapia, etc.)
   - Lee/escribe en Supabase
   - Devuelve un resultado corto a la Capa de Voz, que lo comunica de forma natural
4. Este resultado se lo lleva la Capa de Voz de vuelta a la conversación hablada.

**Por qué esta separación importa:** la Capa de Voz nunca necesita "saber" cómo funciona tu base de datos ni tu lógica de negocio — solo sabe hablar y cuándo pedir ayuda. Esto significa que puedes cambiar de proveedor de voz en el futuro (o de modelo de razonamiento) sin rediseñar todo el sistema.

### Tools que la Capa de Voz puede invocar (v1)

| Tool | Qué hace |
|---|---|
| `classify_and_save` | Envía el contenido crudo al backend, que usa Claude para clasificar y guardar en la tabla correcta |
| `get_context_suggestion` | Pide al motor de contexto la siguiente mejor acción |
| `get_pending_items` | Devuelve tareas/recordatorios pendientes relevantes |
| `create_reminder` | Crea un recordatorio con fecha/hora |
| `flag_emotional_signal` | Se dispara en Modo Terapia si el contenido sugiere angustia significativa — activa el piso de seguridad (sección 7) |

---

## 3. Stack tecnológico definitivo

| Capa | Tecnología | Notas |
|---|---|---|
| Frontend | Next.js 14+ (App Router) + Tailwind CSS | PWA (manifest + service worker) |
| Voz en tiempo real | Realtime Voice API (ej. OpenAI Realtime) | Managed — resuelve streaming, interrupciones y latencia |
| Razonamiento / clasificación | Claude API (Sonnet) | Ya es tu stack en BrandingStudio — reutilizas conocimiento |
| Backend | Next.js API Routes / Vercel Edge Functions | Orquesta entre voz, Claude y Supabase |
| Base de datos | Supabase (Postgres) | Auth + RLS desde el día 1 |
| Búsqueda semántica | pgvector (Supabase) | **Fase 2**, no en MVP |
| Hosting | Vercel | Frontend + backend |
| Notificaciones | Web Push (PWA) | Fase 2, cuando haya Calendar integrado |

**Nota sobre "IA híbrida":** Claude maneja clasificación, lógica de modos y generación de texto de reflexión (Modo Terapia, Revisión Semanal). El modelo de voz en tiempo real maneja únicamente la conversación hablada. No hay conflicto entre ambos — cada uno hace lo que mejor sabe hacer.

---

## 4. Modelo de datos (multi-tenant desde el día 1)

Todas las tablas usan `user_id uuid references auth.users(id)` y tienen **Row Level Security (RLS)** activado desde el inicio, aunque hoy solo exista tu usuario. Esto evita una migración dolorosa si el producto se abre a más usuarios.

```sql
-- Perfil extendido (auth.users ya lo maneja Supabase Auth)
create table profiles (
  id uuid primary key references auth.users(id),
  name text,
  timezone text default 'America/Denver',
  main_goals text[],
  created_at timestamptz default now()
);

create table projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  name text not null,
  description text,
  status text check (status in ('active','paused','done','idea')) default 'active',
  priority int default 3,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table tasks (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  project_id uuid references projects(id),
  title text not null,
  description text,
  status text check (status in ('pending','in_progress','done','blocked')) default 'pending',
  priority int default 3,
  estimated_minutes int,
  due_date timestamptz,
  energy_required text check (energy_required in ('low','medium','high')),
  context_required text, -- 'computer', 'phone', 'anywhere'
  created_at timestamptz default now()
);

create table notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  category text not null, -- 'idea','habito','pensamiento_recurrente','bloqueo','decision', etc.
  content text not null,
  related_project_id uuid references projects(id),
  embedding vector(1536), -- NULL en MVP, se llena en Fase 2
  created_at timestamptz default now()
);

create table journal_entries (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  mood text,
  energy_level int, -- 1-5
  emotional_state text,
  content text,
  source_mode text default 'terapia', -- para diferenciar de notas normales
  created_at timestamptz default now()
);

create table reminders (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  title text not null,
  description text,
  remind_at timestamptz not null,
  status text check (status in ('pending','sent','done','cancelled')) default 'pending'
);

create table goals (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  title text not null,
  timeframe text, -- 'semana','mes','trimestre','año'
  status text default 'active',
  progress int default 0 -- 0-100
);

create table context_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  time_of_day text,
  location_context text, -- 'trabajo','casa','manejando','otro'
  available_time_minutes int,
  energy_level int,
  device_available text, -- 'phone','computer'
  source text default 'manual', -- 'auto' | 'manual' | 'mixed'
  created_at timestamptz default now()
);
```

**Sobre "hábito" y "pensamiento recurrente" (mencionados en el brief original):** en el MVP viven dentro de `notes` usando el campo `category`, no como tablas propias. Cuando actives búsqueda semántica en Fase 2, vas a poder agrupar automáticamente notas de la misma categoría que se repiten — en ese punto evaluamos si conviene separarlas en tablas dedicadas (`habits` con streak tracking, por ejemplo). No lo hacemos ahora porque agregaría complejidad sin datos todavía que la justifiquen.

**RLS (ejemplo, se repite por tabla):**
```sql
alter table notes enable row level security;

create policy "users_manage_own_notes"
on notes for all
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

---

## 5. Alcance quirúrgico del MVP — los 5 modos, con profundidad distinta

Dado el timeline ("pocas semanas, solo tú + Claude Code"), **los 5 modos sí están en el MVP**, pero no todos con la misma profundidad. Esta es la forma de cumplir "se siente como un asistente real" sin comprometer el plazo.

| Modo | Profundidad en MVP | Qué se recorta para después |
|---|---|---|
| **Captura Rápida** | Completa — es el corazón del sistema | — |
| **Modo Proyecto** | Completa pero simple: próxima acción, tareas pendientes, prioridad — basado en queries directas | Recomendación "inteligente" de priorización basada en patrones históricos → Fase 2 |
| **Modo Enfoque** | Basado en reglas simples (si energía baja + tiempo <15 min → sugiere una nota/idea pendiente de bajo esfuerzo) | Motor de contexto con aprendizaje de tus patrones reales de uso → Fase 3 |
| **Modo Terapia** | Conversación reflexiva + guardado en `journal_entries` + piso de seguridad (sección 7) | Detección fina de "pensamiento repetitivo" vía semántica → Fase 2 |
| **Revisión Semanal** | Resumen basado en conteos y filtros (tareas completadas, mood promedio, notas por categoría) + narrativa generada por Claude | Detección automática de patrones cruzados entre proyectos → Fase 2 |

**Motor de contexto en MVP:** captura mixta como definiste — hora del sistema automática, energía y ubicación por selección rápida (3 botones: trabajo/casa/manejando + 3 niveles de energía) en vez de GPS real. Esto evita pedir permisos de ubicación desde el día 1 y sigue dando contexto útil.

---

## 6. Modo Terapia — diseño del piso de seguridad

No es un mensaje robótico visible. Es una garantía técnica invisible:

1. Cada mensaje del usuario en Modo Terapia pasa por una clasificación ligera adicional (vía Claude) que evalúa si hay señales de riesgo serio (no solo "mal día").
2. Si se detecta, la Capa de Voz recibe instrucciones específicas del backend (vía el tool `flag_emotional_signal`) con información de recursos de apoyo ya verificados — pero la entrega sigue siendo con la voz natural del asistente, no un texto legal pegado.
3. El evento se registra (sin exponerlo como una alerta intrusiva) para que, si ocurre, tengas trazabilidad después.
4. El resto del contenido de Modo Terapia se guarda con el mismo nivel de privacidad que cualquier otro dato (como definiste), sin aislamiento especial.

---

## 7. Roadmap por fases

**Fase 0 — Cimientos (semana 1)**
- Next.js + Supabase (schema + Auth + RLS) desplegado en Vercel
- PWA básica instalable
- Conexión de prueba con la Capa de Voz: hablar y recibir respuesta en tiempo real

**Fase 1 — Loop central (semanas 2-3)**
- Clasificación automática funcionando end-to-end (voz → clasifica → guarda)
- Captura Rápida + Modo Proyecto
- Dashboard Hoy (prioridad, próxima acción, tareas rápidas)

**Fase 2 — Modos restantes (semana 4)**
- Modo Enfoque (reglas simples)
- Modo Terapia + piso de seguridad
- Diario

**Fase 3 — Cierre de MVP**
- Revisión Semanal (versión con filtros)
- Pulido de manos libres mientras manejas

**Fase 4 — Post-MVP**
- Google Calendar
- Búsqueda semántica (pgvector)
- Notificaciones push
- WhatsApp / Notion
- Detección de patrones y agente autónomo

---

## 8. Próximos pasos técnicos (para Claude Code)

1. Elegir y confirmar proveedor concreto de Realtime Voice API (evaluar límites de uso en PWA/iOS Safari).
2. Scaffolding del proyecto Next.js + Supabase con el schema de la sección 4.
3. Implementar RLS y probar aislamiento de datos aunque haya un solo usuario.
4. Construir el primer tool (`classify_and_save`) end-to-end como prueba de concepto de las dos capas hablando entre sí.
5. Definir el prompt de sistema de Claude para clasificación (categorías exactas, formato de salida JSON).

---

*Este documento reemplaza al brief original como fuente de verdad técnica. El brief original (`Brief del Proyecto`) sigue siendo la referencia de visión de producto.*
