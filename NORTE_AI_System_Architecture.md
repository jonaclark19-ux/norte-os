# NORTE AI — Arquitectura de Sistema

**Versión:** 2.0 — Documento técnico completo
**Rol de este documento:** especificación de arquitectura a nivel de Principal Software Architect. Extiende y reemplaza como referencia técnica a `NORTE_AI_Arquitectura.md` (v1.0), que queda como resumen ejecutivo.
**Documentos relacionados:** Brief del Proyecto, `Vision.md`, `PRD.md`.
**Alcance:** arquitectura completa del sistema (MVP + diseño extensible para fases futuras). No contiene código — solo decisiones, contratos conceptuales y justificación técnica.

---

## 0. Principios rectores de la arquitectura

Toda decisión de este documento se puede rastrear a uno de estos principios. Cuando dos opciones técnicas compiten, el desempate sale de aquí:

1. **Separación estricta entre conversación y lógica de negocio.** La capa que habla nunca decide qué hacer con los datos; solo la capa de razonamiento lo hace.
2. **Seguridad y aislamiento de datos no son un añadido — son el cimiento.** Se construyen antes de que exista una razón inmediata para necesitarlos.
3. **Complejidad ganada, no complejidad anticipada.** No se construye infraestructura (memoria semántica, agentes autónomos, backends separados) hasta que el volumen de uso real la justifique.
4. **Ningún proveedor de IA es indispensable.** El sistema debe poder cambiar de motor de voz o de motor de razonamiento sin rediseño estructural.
5. **Un desarrollador solo, con Claude Code, debe poder operar y extender todo el sistema.** La arquitectura optimiza por simplicidad operativa, no por patrones de equipos grandes.

---

## 1. Vista de conjunto

```
                              ┌───────────────────────────┐
                              │        USUARIO             │
                              └─────────────┬─────────────┘
                                            │ voz / texto / touch
                                            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                          FRONTEND (PWA — Next.js)                       │
│   Dashboard · Proyectos · Captura por voz · Diario · Revisión           │
└───────────┬───────────────────────────────────────┬─────────────────────┘
            │ token efímero + WebRTC directo          │ HTTPS (REST)
            ▼                                         ▼
┌───────────────────────────┐         ┌───────────────────────────────────┐
│  CAPA DE VOZ (tiempo real)  │ tool    │   BACKEND (Next.js API/Edge Fn)     │
│  Proveedor de voz gestionado │ calls  │   = "Capa de Cerebro"                │
│  STT + TTS + turnos + audio  │───────▶│   Orquestador de agentes             │
└───────────────────────────┘         │   Autenticación · RLS enforcement    │
                                       └───────────────┬─────────────────────┘
                                                        │
                              ┌─────────────────────────┼─────────────────────┐
                              ▼                         ▼                     ▼
                    ┌───────────────────┐   ┌───────────────────┐  ┌───────────────────┐
                    │   Claude API        │   │   Supabase           │  │  Integraciones      │
                    │   (razonamiento,     │   │   Postgres + Auth     │  │  futuras (Calendar,  │
                    │   clasificación)      │   │   + RLS + pgvector*   │  │  WhatsApp, etc.)     │
                    └───────────────────┘   └───────────────────┘  └───────────────────┘
```
*pgvector se activa en Fase 2, la estructura ya lo contempla.

---

## 2. Frontend

### 2.1 Stack y justificación
**Next.js 14+ (App Router) + React + Tailwind CSS, desplegado como PWA en Vercel.**

- **Next.js** es el stack ya dominado en BrandingStudio y BarberDesk — reutilizar conocimiento reduce riesgo de ejecución en un timeline de pocas semanas con un solo desarrollador. App Router permite Server Components, que reducen JavaScript enviado al cliente — relevante en un producto que se usa mucho desde móvil con conexión variable (ej. manejando).
- **Tailwind** por velocidad de iteración, ya integrado al flujo de trabajo existente (skills de diseño en Claude Code).
- **PWA en vez de nativo (iOS/Android):** un solo código base, sin fricción de app store ni revisión de Apple/Google, instalación inmediata. Se acepta la contrapartida conocida (limitaciones de audio en segundo plano en Safari/iOS) como una restricción de diseño, no como un bloqueo — mitigada en la sección 2.4.

### 2.2 Estructura de rutas (alto nivel)
| Ruta | Propósito |
|---|---|
| `/dashboard` | Pantalla "Hoy": prioridad, próxima acción, energía, tareas rápidas, recordatorios |
| `/proyectos` | Lista de proyectos con avance |
| `/proyectos/[id]` | Detalle de proyecto: tareas, notas, ideas asociadas |
| `/captura` | Pantalla de voz — botón "Hablar", sesión en tiempo real |
| `/diario` | Historial de Modo Terapia / entradas emocionales |
| `/revision` | Resumen semanal |
| `/configuracion` | Perfil, ajuste manual de contexto, preferencias |

### 2.3 Gestión de estado
Regla de diseño: **el cliente no es la fuente de verdad de nada.** Todo estado de negocio (proyectos, tareas, notas, diario) vive en Supabase y se consulta bajo demanda. El cliente solo mantiene:
- Estado de UI efímero (qué pantalla, qué modal abierto).
- Estado de la sesión de voz activa (transcripción en curso, si el micrófono está abierto).

Esto evita el problema clásico de sincronización de estado en apps con múltiples dispositivos: si el usuario abre NORTE AI en el teléfono y en el computador, ambos siempre leen la misma verdad desde el backend, no una copia local que se puede desincronizar.

### 2.4 Conexión de voz — decisión de diseño clave
**El cliente se conecta directo al proveedor de voz en tiempo real vía WebRTC, usando un token efímero generado por el backend — no relaya el audio a través del propio backend.**

**Justificación:**
- Relayar audio en tiempo real a través del backend agrega un salto de red adicional en cada dirección, lo cual introduce latencia perceptible — inaceptable para el requisito de "conversación tipo llamada."
- El backend genera un token de corta duración con permisos acotados (una sesión, tiempo limitado) y se lo entrega al cliente; el cliente lo usa para autenticar directamente con el proveedor de voz. El backend nunca procesa el audio crudo.
- Los **tool calls** que el modelo de voz decide invocar (guardar algo, consultar tareas, etc.) sí viajan del cliente/proveedor hacia el backend — porque ahí es donde vive la lógica de negocio y el acceso a datos. Esta es la única vía por la que la conversación puede tener efectos reales en el sistema.

**Mitigación de limitaciones de audio en segundo plano (iOS Safari):**
- Uso de la Wake Lock API para mantener la pantalla activa durante sesiones activas cuando sea posible.
- Degradación explícita: si el navegador no soporta background audio, el sistema lo comunica claramente en vez de fallar en silencio (ya definido como EC-015 en el PRD).

### 2.5 Service Worker
Cachea el shell de la aplicación y assets estáticos para carga rápida e instalación como PWA. **No cachea datos de negocio del usuario** (proyectos, notas, diario) — estos siempre se consultan frescos al backend, para evitar exponer información sensible en el cache del navegador sin cifrado adicional.

---

## 3. Backend

### 3.1 Rol: "Capa de Cerebro" — orquestador de agentes, no un CRUD genérico
El backend vive en Next.js API Routes / Vercel Edge Functions, en el mismo repo/despliegue que el frontend.

**Responsabilidades:**
1. Generar tokens efímeros para que el cliente inicie sesión de voz.
2. Exponer los endpoints que funcionan como *tools* invocables por el modelo de voz.
3. Orquestar las llamadas a los agentes de razonamiento (Claude) correspondientes a cada tool.
4. Leer/escribir en Supabase **siempre en nombre del usuario autenticado**, respetando RLS.
5. Servir los datos que las pantallas de UI necesitan (Dashboard, Proyectos, Diario, Revisión) vía endpoints REST convencionales, independientes del flujo de voz.

### 3.2 Justificación: Next.js API Routes vs. un backend separado (Express/FastAPI)
Se descarta un backend independiente en esta etapa por tres razones concretas:
- **Superficie operativa:** un solo despliegue (Vercel), un solo dominio, sin CORS entre servicios, sin infraestructura adicional que un desarrollador solo tenga que mantener.
- **Carga esperada:** el MVP sirve a un usuario, después a un grupo reducido en beta privada — muy por debajo del punto donde un backend dedicado aporta ventajas reales de rendimiento.
- **Reversibilidad:** la lógica de negocio vive encapsulada en funciones/agentes bien delimitados (sección 7). Si más adelante se justifica separar el backend (escala real, equipo de ingeniería más grande), esa lógica se migra sin rediseñar el contrato con el frontend ni con la Capa de Voz.

### 3.3 Patrón arquitectónico
**Backend for Frontend (BFF) + Tool Orchestrator.** El backend no expone una API CRUD plana sobre las tablas; expone *acciones con intención de negocio* (clasificar y guardar, sugerir próxima acción, marcar señal emocional) que internamente pueden tocar una o varias tablas y pueden invocar uno o varios agentes. Esto mantiene la lógica de los 5 modos centralizada y auditable, en vez de dispersa entre frontend y backend.

---

## 4. Base de datos

### 4.1 Motor: Supabase (Postgres)
**Justificación:**
- Auth y Row Level Security nativos — crítico dado que el sistema es multi-tenant desde el día 1 (principio rector #2).
- pgvector disponible en el mismo motor para cuando se active memoria semántica (Fase 2) — evita una migración de motor de base de datos más adelante.
- Ya es el motor de datos usado en el resto del stack de Jonathan (BrandingStudio, BarberDesk) — reduce curva de aprendizaje.
- Despliegue administrado: sin necesidad de operar infraestructura de base de datos propia en esta etapa.

### 4.2 Esquema — decisiones de modelado relevantes
El esquema completo de tablas (`profiles`, `projects`, `tasks`, `notes`, `journal_entries`, `reminders`, `goals`, `context_logs`) está definido en `NORTE_AI_Arquitectura.md` v1.0 y en el PRD. Aquí se documentan las decisiones de diseño detrás de ese esquema:

- **`category` en `notes` es texto libre controlado por la aplicación, no un `enum` de base de datos.** Esto permite introducir nuevas categorías (ej. cuando "hábito" se separe en su propia tabla en el futuro) sin una migración destructiva de esquema.
- **Cada tabla lleva `user_id` explícito**, aunque RLS ya lo exige — esto también acelera queries filtradas por usuario y facilita joins.
- **`created_at` es obligatorio en toda tabla con relevancia temporal** (`notes`, `tasks`, `journal_entries`, `context_logs`) porque Modo Revisión Semanal y el futuro motor de patrones dependen enteramente de rangos de fecha.
- **`embedding vector(1536)` ya existe en `notes` desde el MVP, pero nulo hasta Fase 2.** Se agrega la columna desde ahora para evitar una migración de esquema en producción cuando se active la búsqueda semántica — el costo de tener una columna nula es despreciable comparado con el costo de una migración posterior.

### 4.3 Índices recomendados
`user_id` (todas las tablas, refuerza RLS y acelera filtrado), `project_id` (en `tasks` y `notes`, para consultas de Modo Proyecto), `created_at` (en `notes`, `journal_entries`, `context_logs`, para consultas de rango temporal en Revisión Semanal).

### 4.4 Particionado / crecimiento
No se implementa particionado en el MVP. Se contempla como decisión futura (Fase 3+, solo si el volumen de `notes`/`journal_entries` en un escenario multi-tenant real lo justifica) — construirlo antes viola el principio rector #3.

---

## 5. IA (motor de razonamiento)

### 5.1 Arquitectura híbrida — dos motores, una responsabilidad cada uno
- **Motor de voz en tiempo real (proveedor gestionado):** conversación hablada, streaming, manejo de interrupciones y latencia.
- **Claude API (Sonnet):** clasificación, razonamiento de cada modo, generación de narrativas (Revisión Semanal), evaluación de señales de riesgo emocional (Modo Terapia).

**Justificación de la separación:** un modelo optimizado para conversación fluida en tiempo real no es necesariamente el más confiable para producir salidas estructuradas y consistentes con reglas de negocio (ej. "clasifica esto en una de estas 12 categorías exactas y devuélvelo en este formato"). Separar ambas responsabilidades permite elegir, para cada una, el modelo que mejor la resuelve — y cambiar cualquiera de los dos sin afectar al otro (principio rector #4).

### 5.2 Function calling como contrato formal entre capas
Los *tools* (sección 3, 8) son el único mecanismo por el cual el modelo de voz puede tener efectos reales en el sistema. Esto es una decisión de seguridad y de correctitud: evita que una alucinación del modelo de voz se traduzca en una acción no verificada — solo lo que pasa por un tool call, procesado por el backend, cuenta como una acción real.

### 5.3 Prompting por modo, no un prompt monolítico
Cada modo (Captura Rápida, Proyecto, Enfoque, Terapia, Revisión Semanal) tiene su propio system prompt, con tono, reglas y formato de salida propios. Un solo prompt gigante que intente cubrir los 5 modos a la vez es frágil: cualquier ajuste a un modo arriesga romper el comportamiento de otro. Prompts separados son más fáciles de probar, versionar y ajustar de forma aislada.

### 5.4 Salida estructurada
Las llamadas de clasificación deben exigir salida en JSON estricto (categoría, contenido normalizado, proyecto asociado si aplica) para que el backend la procese de forma determinística, sin depender de parsear lenguaje natural libre.

---

## 6. Motor de contexto

### 6.1 Fuentes de contexto
| Fuente | Tipo | Disponibilidad en MVP |
|---|---|---|
| Hora del sistema | Automática | Sí |
| Energía declarada | Manual rápida (selección de 3 niveles) | Sí |
| Ubicación (trabajo/casa/manejando) | Manual rápida | Sí |
| Dispositivo disponible | Inferido del cliente que se conecta | Sí |
| GPS real | Automática | No — Fase 2+ |
| Calendario | Automática | No — Fase 2 (integración Calendar) |

### 6.2 Reglas, no modelo aprendido — en el MVP
El motor de contexto del MVP es un conjunto de reglas explícitas (si energía baja + tiempo < 15 min → sugerir microacción de bajo esfuerzo) evaluadas por el Agente de Contexto en el backend, no un modelo entrenado sobre el comportamiento histórico del usuario.

**Justificación:** con el volumen de datos disponible en el MVP no existe base suficiente para entrenar ni justificar un modelo aprendido; las reglas son transparentes, auditable y ajustables directamente por el desarrollador. Migrar a un motor de contexto que aprenda patrones reales del usuario es una extensión natural de Fase 3, una vez que `context_logs` y el historial de decisiones acumulen suficiente señal.

---

## 7. Memoria

Se distinguen tres niveles de memoria, deliberadamente separados — combinarlos en un solo concepto sería confuso y técnicamente incorrecto:

| Nivel | Qué es | Dónde vive | Vida útil |
|---|---|---|---|
| **Memoria de sesión** | Contexto de la conversación de voz activa | Dentro del proveedor de voz, durante la llamada | Efímera — se pierde al cerrar sesión salvo lo guardado explícitamente vía tools |
| **Memoria estructurada persistente** | Proyectos, tareas, notas, diario, recordatorios, metas | Supabase Postgres | Permanente |
| **Memoria semántica** | Embeddings para recuperación por significado, detección de patrones recurrentes | pgvector (columna ya reservada, Fase 2) | Permanente, activada cuando el volumen lo justifique |

**Justificación de no construir memoria semántica desde el MVP:** su valor crece con el volumen de datos acumulados; construirla antes de tener semanas de uso real es complejidad prematura sin retorno inmediato (ya razonado en `Vision.md` y `NORTE_AI_Arquitectura.md` v1.0).

---

## 8. Agentes

### 8.1 Qué es un "agente" en esta arquitectura
Un **agente**, en NORTE AI, es una unidad de razonamiento especializada dentro de la Capa de Cerebro, invocada como resultado de un tool call desde la Capa de Voz o de una acción programada (ej. generar la Revisión Semanal). **No son agentes autónomos que actúan sin disparo del usuario o supervisión** — son agentes reactivos, de alcance acotado, cada uno con una única responsabilidad.

### 8.2 Agentes del MVP
| Agente | Responsabilidad | Se invoca desde |
|---|---|---|
| **Agente Clasificador** | Recibe texto crudo, decide categoría, normaliza el contenido para guardar | `classify_and_save` |
| **Agente de Contexto** | Aplica las reglas del motor de contexto y decide qué recomendar | `get_context_suggestion` |
| **Agente de Seguridad Emocional** | Evalúa cada mensaje de Modo Terapia en busca de señales de riesgo, en paralelo a la conversación principal | Automático dentro de Modo Terapia |
| **Agente de Síntesis Semanal** | Genera la narrativa de la Revisión Semanal a partir de datos agregados | Modo Revisión Semanal |

### 8.3 Justificación de agentes especializados frente a un único orquestador genérico
- **Separación de responsabilidades:** cada agente se prueba, ajusta y audita de forma aislada, sin riesgo de que un cambio en uno afecte a los demás.
- **Flexibilidad de modelo:** nada obliga a que todos los agentes usen el mismo modelo — un agente de clasificación simple podría usar un modelo más económico en el futuro, mientras que Síntesis Semanal o Seguridad Emocional pueden requerir uno más capaz.
- **Trazabilidad:** cada tool call queda asociado a un agente específico, lo que facilita depuración y auditoría de por qué el sistema tomó una decisión concreta.

### 8.4 Agentes autónomos — fuera del MVP, diseño abierto para el futuro
A diferencia de los agentes anteriores (reactivos), un futuro agente autónomo (Fase 3+) actuaría de forma proactiva — por ejemplo, sugerir la Revisión Semanal sin que se le pida, o alertar sobre un bloqueo que se repite. **Regla de diseño, no solo técnica:** ningún agente autónomo debe ejecutar una acción irreversible sin confirmación humana explícita. Esto no está construido ni decidido en detalle todavía — se deja como espacio de diseño abierto, condicionado a que el sistema demuestre confiabilidad consistente en las fases anteriores.

---

## 9. API

### 9.1 API interna (MVP)
| Endpoint (conceptual) | Propósito | Invocado por |
|---|---|---|
| Generación de token efímero | Habilita la sesión de voz del cliente | Frontend, al iniciar `/captura` |
| `classify_and_save` | Clasifica y guarda contenido crudo | Modelo de voz (tool call) |
| `get_context_suggestion` | Devuelve la próxima acción recomendada según contexto | Modelo de voz (tool call) |
| `get_pending_items` | Devuelve tareas/recordatorios pendientes, opcionalmente filtrados por proyecto | Modelo de voz (tool call) |
| `create_reminder` | Crea un recordatorio | Modelo de voz (tool call) |
| `flag_emotional_signal` | Activa el piso de seguridad de Modo Terapia | Agente de Seguridad Emocional (interno) |
| Endpoints CRUD de UI | Sirven datos a Dashboard, Proyectos, Diario, Revisión | Frontend (fetch directo, sin pasar por voz) |

Cada tool endpoint define una entrada y salida conceptual clara (sin especificar código): entrada = contenido/parametros mínimos necesarios; salida = confirmación corta y datos relevantes que la Capa de Voz pueda comunicar de forma natural.

### 9.2 API externa (futura)
No existe en el MVP porque el único cliente es la propia PWA. Se diseña el backend de forma que, cuando se necesite (integraciones tipo Calendar, o un eventual cliente móvil nativo), se pueda exponer una API versionada con autenticación por token sin reescribir la lógica ya construida — los mismos agentes y contratos internos se reutilizan.

---

## 10. Integraciones

### 10.1 Estado en el MVP
Cero integraciones externas activas. Es una decisión correcta y no una omisión: el Brief y la Visión definen Google Calendar como la primera integración, explícitamente en Fase 2.

### 10.2 Patrón de diseño para integraciones futuras: conector desacoplado
Cada integración futura (Calendar, WhatsApp, Notion) se implementa como un **módulo aislado** que traduce datos externos hacia y desde el modelo de datos interno de NORTE AI, sin que el núcleo del sistema (clasificación, modos, motor de contexto) necesite conocer detalles de esa integración específica.

**Justificación:** esto permite añadir, quitar o reemplazar integraciones sin tocar la lógica central — el core del sistema le "habla" a un contrato interno estable, y es el conector el que se adapta a la API externa de turno, no al revés.

---

## 11. Seguridad

| Capa | Medida | Justificación |
|---|---|---|
| Datos | RLS activo en cada tabla desde el día 1 | Principio rector #2 — es la primera línea de defensa, no un añadido opcional |
| Autenticación | Supabase Auth (JWT) | El backend siempre actúa en nombre del usuario autenticado real |
| Acceso a datos desde el backend | Se usa el JWT del usuario, nunca una service key de bypass, salvo tareas administrativas explícitas y auditadas | Evita que un bug en el backend exponga datos de todos los usuarios |
| Transporte | HTTPS / WSS en todo el sistema | Estándar mínimo no negociable |
| Llaves de proveedores de IA | Viven exclusivamente en variables de entorno del servidor (Vercel) | El cliente nunca recibe la key real, solo un token efímero de sesión de voz, de vida corta y permisos acotados |
| Tools/agentes | Principio de mínimo privilegio — cada tool solo accede a los datos que su función requiere (ej. `create_reminder` no puede leer `journal_entries`) | Reduce superficie de daño si un tool se invoca de forma inesperada |
| Datos sensibles (Modo Terapia) | Mismo estándar de seguridad y cifrado que el resto de los datos, sin aislamiento especial adicional (decisión de producto ya tomada en `Vision.md`) | Consistencia con la decisión de producto, sin crear una categoría de datos "de segunda clase" en términos de seguridad |

---

## 12. Escalabilidad

### 12.1 De 1 usuario a N usuarios
La arquitectura no requiere cambios estructurales para escalar de un usuario a varios: RLS y multi-tenancy ya están implementados desde el MVP (principio rector #2). Escalar usuarios implica más capacidad de cómputo y de base de datos — ambas administradas automáticamente por Vercel y Supabase en sus niveles superiores de plan.

### 12.2 El cuello de botella real no es la base de datos
El componente más costoso y más sensible a escalar no es Postgres ni el backend — es el **uso de los proveedores de IA**, particularmente la voz en tiempo real, que se cobra por minuto de conversación activa. Esto es, ante todo, una consideración de modelo de negocio y de pricing si el producto se abre a más usuarios, más que un problema de arquitectura pura — pero se deja documentado aquí porque condiciona directamente cualquier decisión futura de monetización.

### 12.3 Crecimiento de datos
No se optimiza prematuramente. Si el volumen de `notes` o `journal_entries` en un escenario multi-tenant real se vuelve significativo, se evalúa entonces particionado o archivado — no antes (principio rector #3).

---

## 13. Flujo de datos (end-to-end)

```
1. Usuario habla
        │
2. Capa de Voz transcribe en streaming y mantiene la conversación
        │
3. Capa de Voz detecta que se requiere una acción real
        │  (ej. "guarda esto", "¿qué tengo pendiente?")
        ▼
4. Tool call → Backend (Capa de Cerebro)
        │
5. Backend identifica el agente correspondiente (Clasificador, Contexto, etc.)
        │
6. Agente invoca Claude si requiere razonamiento (clasificación, síntesis)
        │
7. Backend lee/escribe en Supabase, con el JWT del usuario (RLS aplicado)
        │
8. Backend devuelve un resultado corto y estructurado
        ▼
9. Capa de Voz recibe el resultado y lo comunica en lenguaje natural
        │
10. Usuario escucha la respuesta
```

Este mismo flujo aplica, con variaciones menores, a los 5 modos — la diferencia entre ellos está en qué agente se invoca y qué reglas aplica, no en la mecánica general de comunicación entre capas.

---

## 14. Responsabilidades de cada módulo

| Módulo | Responsabilidad | Explícitamente NO hace |
|---|---|---|
| **Frontend (PWA)** | Renderizar UI, capturar audio, mostrar datos consultados al backend | No decide clasificación, no accede a Supabase directamente, no almacena estado de negocio de forma duradera |
| **Capa de Voz** | Conversación en tiempo real, transcripción, síntesis de voz, detección de cuándo invocar un tool | No tiene acceso a la base de datos, no ejecuta lógica de negocio, no decide categorías por sí misma |
| **Backend / Capa de Cerebro** | Orquestar agentes, aplicar lógica de los 5 modos, mediar todo acceso a Supabase | No mantiene conversación en tiempo real, no genera audio |
| **Agentes (Claude)** | Razonamiento especializado por tarea (clasificar, contextualizar, sintetizar, evaluar riesgo) | No tienen acceso directo a la base de datos — siempre pasan por el backend |
| **Supabase (Postgres + Auth)** | Persistencia, autenticación, aislamiento de datos vía RLS | No contiene lógica de negocio ni de clasificación |
| **Motor de Contexto** | Traducir energía/tiempo/ubicación en una recomendación | No decide qué guardar ni cómo clasificar contenido |
| **Integraciones (futuras)** | Traducir datos externos hacia/desde el modelo interno | No alteran el núcleo de clasificación ni los modos existentes |

---

## Nota de cierre

Este documento es la referencia técnica definitiva para la construcción de NORTE AI. Las decisiones de alcance y priorización (qué se construye primero) siguen viviendo en `PRD.md`; las decisiones de "por qué existe el producto" en `Vision.md`. Este documento responde exclusivamente al "cómo está construido y por qué así" a nivel de sistema.
