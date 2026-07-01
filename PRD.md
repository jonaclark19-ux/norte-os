# NORTE AI — Product Requirements Document (PRD)

**Versión:** 1.0
**Optimizado para:** uso directo con Claude Code por un desarrollador solo.
**Documentos fuente:** Brief del Proyecto (original), `Vision.md`, `NORTE_AI_Arquitectura.md`
**Alcance de este PRD:** el MVP (Fases 0-3 del roadmap de Arquitectura) está detallado por completo. Las fases posteriores se listan a alto nivel en la sección "Funcionalidades futuras" — *esta fue una decisión tomada por defecto ante una pregunta sin respuesta clara; ver nota al final del documento.*

---

## 1. Objetivos

### Objetivo del producto en esta fase
Llevar a NORTE AI de brief a un sistema funcional de uso diario real: captura por voz en tiempo real, clasificación automática, y los 5 modos operando (con la profundidad definida en Arquitectura), construido sobre una base multi-tenant desde el día 1.

### Objetivos de este documento
- Traducir la visión de producto (`Vision.md`) y la arquitectura técnica (`NORTE_AI_Arquitectura.md`) en requerimientos accionables, sin ambigüedad, listos para convertirse en tickets de trabajo.
- Servir como única fuente de verdad de "qué construir" durante el desarrollo del MVP.
- Dejar explícito qué queda fuera del MVP para evitar scope creep durante la construcción.

---

## 2. Features

Agrupadas por módulo. "MVP" indica que están dentro del alcance detallado de este documento.

| Feature | Módulo | MVP |
|---|---|---|
| Conversación por voz en tiempo real (tipo llamada, con interrupciones) | Capa de Voz | Sí |
| Clasificación automática de mensajes en categorías | Motor de Clasificación | Sí |
| Guardado automático en base de datos estructurada | Backend / Datos | Sí |
| Modo Captura Rápida | Modos | Sí |
| Modo Proyecto | Modos | Sí |
| Modo Enfoque (reglas simples) | Modos | Sí |
| Modo Terapia + piso de seguridad | Modos | Sí |
| Modo Revisión Semanal (basado en filtros) | Modos | Sí |
| Dashboard "Hoy" | UI | Sí |
| Pantalla de Proyectos | UI | Sí |
| Diario / espacio de Modo Terapia | UI | Sí |
| Recordatorios básicos | Backend / Datos | Sí |
| Captura de contexto (mixta: automática + manual) | Motor de Contexto | Sí |
| PWA instalable | Plataforma | Sí |
| Uso manos libres (incluye manejando) | Plataforma | Sí |
| Autenticación y aislamiento de datos (RLS) | Backend | Sí |
| Integración Google Calendar | Integraciones | No — Fase 2 |
| Búsqueda semántica (pgvector) | Motor de Clasificación | No — Fase 2 |
| Notificaciones push proactivas | Plataforma | No — Fase 2 |
| Integración WhatsApp / Notion / Sheets | Integraciones | No — Fase 2+ |
| Detección automática de patrones cruzados | Motor de Contexto | No — Fase 2+ |
| Agente autónomo | Sistema | No — Fase 3+ |

---

## 3. Casos de uso

Heredados y expandidos de `Vision.md`, ahora con el módulo responsable:

1. **Captura instantánea de una idea** mientras se maneja, sin abrir ninguna app. → Voz + Captura Rápida
2. **"Tengo 10 minutos, ¿qué avanzo?"** en una pausa. → Modo Enfoque + Motor de Contexto
3. **Desahogo guiado** que termina en claridad o una acción pequeña. → Modo Terapia
4. **"Organiza mi día"** al despertar. → Dashboard Hoy + Motor de Contexto
5. **Revisión semanal** de avances, pendientes y patrones repetidos. → Modo Revisión Semanal
6. **Seguimiento de proyectos en paralelo.** → Modo Proyecto + Pantalla de Proyectos
7. **Consulta directa de pendientes:** "¿Qué tengo pendiente?", "Muéstrame ideas de [proyecto]". → Modo Proyecto / Captura Rápida

---

## 4. User Stories

Formato: *Como [rol], quiero [acción], para [beneficio].* Agrupadas por epic. Cada historia lleva un ID para trazabilidad con Functional Requirements y Criterios de Aceptación.

### Epic A — Captura por voz
- **US-A1:** Como usuario, quiero hablarle a NORTE AI en cualquier momento y que la conversación fluya en tiempo real, para no perder ideas por fricción de captura.
- **US-A2:** Como usuario, quiero poder interrumpir a NORTE AI mientras habla, para que la conversación se sienta natural, como una llamada.
- **US-A3:** Como usuario, quiero que el sistema funcione manos libres mientras manejo, para poder usarlo sin distraerme.

### Epic B — Clasificación automática
- **US-B1:** Como usuario, quiero que el sistema decida automáticamente si lo que dije es tarea, nota, idea, emoción, proyecto, hábito, bloqueo o decisión, para no clasificar manualmente.
- **US-B2:** Como usuario, quiero poder corregir una clasificación equivocada después de guardada, para mantener mis datos correctos.

### Epic C — Modo Proyecto
- **US-C1:** Como usuario, quiero preguntar por el estado de un proyecto y recibir próxima acción, tareas pendientes y prioridad, para saber exactamente qué sigue.
- **US-C2:** Como usuario, quiero registrar un bloqueo en un proyecto, para no perder de vista qué me está deteniendo.

### Epic D — Modo Enfoque
- **US-D1:** Como usuario, quiero decir cuánto tiempo y energía tengo, para recibir una sugerencia de acción que realmente pueda ejecutar ahora.

### Epic E — Modo Terapia
- **US-E1:** Como usuario, quiero un espacio de desahogo hablado que termine en una reflexión clara o una acción pequeña.
- **US-E2:** Como usuario, quiero que el sistema responda con cuidado, de forma natural (no robótica), si detecta que estoy pasando por algo emocionalmente serio.

### Epic F — Modo Revisión Semanal
- **US-F1:** Como usuario, quiero un resumen semanal de qué avancé, qué quedó pendiente y qué se repitió, para tomar decisiones informadas sobre la próxima semana.

### Epic G — Motor de Contexto
- **US-G1:** Como usuario, quiero declarar rápidamente mi energía y ubicación (o que el sistema la infiera cuando sea posible), para recibir recomendaciones realistas y no genéricas.

### Epic H — Dashboard y navegación
- **US-H1:** Como usuario, quiero ver de un vistazo mi prioridad del día, próxima acción, energía actual, tareas rápidas, recordatorios y proyecto principal al abrir la app.

### Epic I — Cuenta y privacidad
- **US-I1:** Como usuario, quiero que mis datos estén aislados de forma segura, para confiar en el sistema con información personal y emocional.
- **US-I2:** Como usuario, quiero poder instalar la app en mi teléfono como una PWA, para acceder a ella como si fuera una app nativa.

---

## 5. User Flows

Flujos textuales, paso a paso, referenciando la arquitectura de dos capas (Capa de Voz / Capa de Cerebro) definida en `NORTE_AI_Arquitectura.md`.

### Flow 1 — Captura Rápida por voz
1. Usuario abre la app o activa la sesión de voz.
2. Capa de Voz establece conexión en tiempo real (streaming).
3. Usuario habla: "Guarda esto: vender paquetes de branding para negocios latinos."
4. Capa de Voz detecta intención de guardado → invoca tool `classify_and_save`.
5. Capa de Cerebro recibe el contenido crudo, lo clasifica con Claude (categoría: idea comercial), y lo guarda en `notes` con `related_project_id` si aplica.
6. Capa de Cerebro devuelve confirmación corta a la Capa de Voz.
7. Capa de Voz responde hablado: "Guardado en [Proyecto] como idea comercial."
8. **Salida del flujo:** dato guardado + confirmación verbal. (Regla de oro cumplida: guardar.)

### Flow 2 — Modo Proyecto: consulta de estado
1. Usuario dice: "Modo proyecto. ¿Cómo va Cloud Studio?"
2. Capa de Voz invoca `get_pending_items` con el proyecto como parámetro.
3. Capa de Cerebro consulta `tasks` y `notes` filtradas por `project_id`, calcula próxima acción según prioridad y estado.
4. Capa de Cerebro devuelve: próxima acción, tareas pendientes, bloqueo actual (si existe), avance registrado.
5. Capa de Voz comunica la respuesta de forma natural y conversacional.
6. **Salida del flujo:** información + posible acción de seguimiento (ej. usuario decide marcar una tarea como hecha).

### Flow 3 — Modo Enfoque: recomendación de microacción
1. Usuario dice: "Tengo 10 minutos, dime qué puedo avanzar."
2. Capa de Voz invoca `get_context_suggestion`, pasando tiempo disponible declarado.
3. Capa de Cerebro consulta `context_logs` (si hay datos recientes de energía/ubicación) o pide confirmación rápida si no los tiene.
4. Capa de Cerebro aplica las reglas del Modo Enfoque: tiempo < 15 min + energía baja/media → sugiere una nota/idea pendiente de bajo esfuerzo, no una tarea de ejecución real.
5. Capa de Voz comunica la sugerencia.
6. **Salida del flujo:** planificar/ejecutar (según la regla de oro).

### Flow 4 — Modo Terapia con activación de piso de seguridad
1. Usuario dice: "Modo terapia. Hoy me siento drenado."
2. Capa de Voz mantiene conversación reflexiva, guiada por el prompt de Modo Terapia.
3. En paralelo, cada mensaje del usuario pasa por clasificación ligera de riesgo (vía Claude).
4. **Caso A — sin señales de riesgo:** la conversación continúa normalmente y se guarda en `journal_entries` al cierre.
5. **Caso B — señal de riesgo serio detectada:** se invoca `flag_emotional_signal`. La Capa de Cerebro entrega a la Capa de Voz información de recursos de apoyo verificados. La Capa de Voz lo comunica con su tono natural, no como un mensaje legal pegado. El evento se registra para trazabilidad.
6. Al cierre de la conversación, se guarda un resumen en `journal_entries` (mood, energy_level, emotional_state, content).
7. **Salida del flujo:** reflexión guardada + (si aplica) acción de cuidado inmediato comunicada.

### Flow 5 — Revisión Semanal
1. Usuario dice: "Modo revisión semanal" (o el sistema lo sugiere proactivamente si corresponde).
2. Capa de Cerebro ejecuta consultas de agregación sobre la última semana: tareas completadas vs. pendientes, notas por categoría, mood promedio de `journal_entries`, proyectos con más actividad.
3. Claude genera una narrativa breve a partir de los resultados (sin búsqueda semántica en el MVP — solo filtros).
4. Capa de Voz comunica el resumen; también queda visible en la pantalla de Revisión.
5. **Salida del flujo:** aclarar (visión clara de la semana) + posible planificación para la próxima.

### Flow 6 — Onboarding inicial
1. Usuario crea cuenta (Supabase Auth).
2. Se crea su `profile` (nombre, timezone, main_goals — opcional en el primer uso).
3. Se le presenta brevemente cómo hablarle a la app (sin tutorial extenso — la app debe explicarse hablando).
4. Primera interacción de prueba: el sistema invita a decir algo simple ("cuéntame una idea que tengas ahora") para demostrar el loop de captura → clasificación → guardado en la primera sesión.
5. **Salida del flujo:** primer dato real guardado, confianza inicial establecida.

---

## 6. Functional Requirements

Numerados por módulo para trazabilidad. Lenguaje directo, sin ambigüedad.

### Capa de Voz (VOICE)
- **VOICE-001:** El sistema debe sostener una conversación de voz en tiempo real, con manejo de interrupciones del usuario mientras el sistema habla.
- **VOICE-002:** El sistema debe funcionar completamente manos libres, sin requerir que el usuario mire o toque la pantalla durante la conversación.
- **VOICE-003:** La Capa de Voz debe poder invocar tools/funciones definidas (`classify_and_save`, `get_context_suggestion`, `get_pending_items`, `create_reminder`, `flag_emotional_signal`) cuando la conversación lo requiera.
- **VOICE-004:** La Capa de Voz no debe contener lógica de negocio ni acceso directo a la base de datos — toda acción real pasa por la Capa de Cerebro.
- **VOICE-005:** El sistema debe soportar conversación en español, inglés, y cambio de idioma dentro de la misma sesión (spanglish).

### Motor de Clasificación (CLASS)
- **CLASS-001:** El sistema debe clasificar cada mensaje relevante del usuario en una de las categorías definidas: tarea, recordatorio, idea, nota emocional, proyecto, diario personal, hábito, meta, decisión, pensamiento recurrente, bloqueo, planificación.
- **CLASS-002:** El sistema debe guardar el mensaje clasificado en la tabla correspondiente según la categoría detectada.
- **CLASS-003:** El usuario debe poder corregir una clasificación después de guardada (ej. "eso en realidad era una tarea, no una nota").
- **CLASS-004:** Toda clasificación debe resultar en una de las cinco salidas definidas por la regla de oro del producto: guardar, aclarar, recordar, planificar o ejecutar. El sistema no debe terminar una interacción sin una de estas salidas.

### Módulo de Proyectos (PROJ)
- **PROJ-001:** El usuario debe poder crear, consultar y actualizar proyectos (`name`, `description`, `status`, `priority`).
- **PROJ-002:** El sistema debe poder responder, para un proyecto dado: próxima acción recomendada, tareas pendientes, bloqueo actual, avance registrado.
- **PROJ-003:** El sistema debe permitir asociar notas, tareas e ideas a un proyecto específico.

### Módulo de Tareas (TASK)
- **TASK-001:** El usuario debe poder crear tareas asociadas o no a un proyecto, con `estimated_minutes`, `energy_required` y `context_required`.
- **TASK-002:** El sistema debe permitir marcar tareas como completadas, en progreso o bloqueadas.
- **TASK-003:** El sistema debe poder filtrar tareas por energía requerida y contexto disponible, para uso en Modo Enfoque.

### Módulo de Notas (NOTE)
- **NOTE-001:** El sistema debe guardar notas con `category`, `content`, y opcionalmente `related_project_id`.
- **NOTE-002:** Las categorías "hábito", "pensamiento recurrente", "bloqueo" y "decisión" se guardan dentro de `notes` usando el campo `category` (no como tablas separadas en el MVP).

### Módulo de Diario / Modo Terapia (JOURNAL)
- **JOURNAL-001:** El sistema debe guardar entradas de diario con `mood`, `energy_level`, `emotional_state` y `content`.
- **JOURNAL-002:** El contenido de Modo Terapia debe recibir el mismo tratamiento de privacidad que el resto de los datos del usuario (sin aislamiento especial adicional).
- **JOURNAL-003:** El sistema debe evaluar cada mensaje en Modo Terapia para detectar señales de riesgo emocional serio, de forma adicional a la clasificación general.
- **JOURNAL-004:** Si se detecta una señal de riesgo, el sistema debe activar el flujo de piso de seguridad (ver Flow 4) sin interrumpir el tono natural de la conversación.

### Módulo de Recordatorios (REM)
- **REM-001:** El usuario debe poder crear recordatorios por voz, con título, descripción opcional y fecha/hora.
- **REM-002:** El sistema debe listar recordatorios pendientes al ser consultado (ej. desde el Dashboard).

### Motor de Contexto (CTX)
- **CTX-001:** El sistema debe capturar contexto de forma mixta: automática cuando sea posible (hora del sistema) y manual/rápida cuando no (energía, ubicación, dispositivo disponible).
- **CTX-002:** El sistema debe usar el contexto capturado para ajustar las recomendaciones de Modo Enfoque y del Dashboard.
- **CTX-003:** La captura manual de contexto debe requerir un mínimo de fricción (selección rápida, no formularios largos).

### Módulo de Revisión Semanal (REVIEW)
- **REVIEW-001:** El sistema debe generar un resumen semanal basado en conteos y filtros: tareas completadas vs. pendientes, notas por categoría, mood promedio.
- **REVIEW-002:** El resumen debe incluir una narrativa breve generada por IA a partir de los datos agregados (sin búsqueda semántica en el MVP).

### Dashboard (DASH)
- **DASH-001:** El Dashboard "Hoy" debe mostrar: prioridad del día, próxima acción recomendada, energía actual, tareas rápidas, recordatorios pendientes, proyecto principal.

### Autenticación y Multi-tenancy (AUTH)
- **AUTH-001:** El sistema debe usar Supabase Auth para autenticación de usuarios.
- **AUTH-002:** Todas las tablas con datos de usuario deben tener Row Level Security (RLS) activado, restringiendo el acceso exclusivamente al propietario de los datos (`auth.uid() = user_id`).
- **AUTH-003:** El sistema debe funcionar correctamente con un solo usuario activo sin requerir cambios estructurales si se agregan más usuarios en el futuro.

### Plataforma (PLAT)
- **PLAT-001:** El sistema debe ser instalable como PWA (manifest + service worker).
- **PLAT-002:** El sistema debe ser utilizable en navegadores móviles, incluyendo soporte de audio en segundo plano cuando el navegador lo permita.
- **PLAT-003:** El sistema debe degradar de forma clara (mensaje explícito al usuario) cuando el navegador no soporte audio en segundo plano o manos libres completo, en vez de fallar silenciosamente.

---

## 7. Non-Functional Requirements

Sin cifras específicas (por decisión explícita) — descritos en términos cualitativos claros.

- **NFR-PERF-001 (Latencia percibida):** La conversación por voz debe sentirse fluida, sin pausas que rompan la sensación de estar hablando con alguien en tiempo real.
- **NFR-SEC-001 (Aislamiento de datos):** Ningún usuario debe poder acceder, directa o indirectamente, a datos de otro usuario, aunque hoy exista un solo usuario real.
- **NFR-SEC-002 (Datos sensibles):** El contenido de Modo Terapia y `journal_entries` debe transmitirse y almacenarse con el mismo estándar de seguridad que el resto de los datos — sin excepciones que lo hagan menos seguro.
- **NFR-REL-001 (Confiabilidad de guardado):** Si la Capa de Voz falla o se corta la conexión, cualquier dato ya capturado antes del corte debe intentar guardarse; el sistema no debe perder silenciosamente información que el usuario ya proporcionó.
- **NFR-USE-001 (Usabilidad):** El sistema debe poder usarse sin manual ni tutorial extenso — la primera interacción debe ser autoexplicativa.
- **NFR-COMPAT-001 (Compatibilidad):** El sistema debe funcionar en los navegadores móviles más usados (Chrome/Android, Safari/iOS), reconociendo que las limitaciones de audio en segundo plano varían entre ellos.
- **NFR-MAINT-001 (Mantenibilidad / desacoplamiento):** La Capa de Voz y la Capa de Cerebro deben poder evolucionar o cambiar de proveedor de IA de forma independiente, sin rediseño estructural del sistema.
- **NFR-COST-001 (Conciencia de costo):** El sistema debe evitar llamadas innecesarias o redundantes a los modelos de IA (voz o razonamiento), dado que es un proyecto autofinanciado en etapa temprana.
- **NFR-I18N-001 (Idioma):** El sistema debe operar de forma fluida en español e inglés, incluyendo cambio de idioma dentro de una misma conversación.
- **NFR-ACC-001 (Accesibilidad):** La interacción por voz debe funcionar como una alternativa completa a la interacción por texto/pantalla, no como un complemento secundario.

---

## 8. Edge Cases

Organizados por módulo. Cada uno debe tener un comportamiento definido antes de construirse, no resolverse "sobre la marcha".

### Voz y conectividad
- **EC-001:** Se pierde la conexión a internet a mitad de una conversación de voz. → El sistema debe notificar claramente y preservar localmente lo que alcanzó a capturar antes del corte, si es técnicamente posible.
- **EC-002:** Ruido de fondo significativo (tráfico, manejando) degrada la transcripción. → El sistema debe poder pedir confirmación o repetición en vez de guardar información incorrecta con falsa confianza.
- **EC-003:** Silencio prolongado durante una sesión de voz activa. → El sistema debe definir un tiempo de espera razonable antes de cerrar la sesión automáticamente, evitando sesiones "colgadas" que consuman recursos.
- **EC-004:** El usuario cambia de idioma a mitad de frase (spanglish). → El sistema debe seguir procesando sin pedir aclaración innecesaria, salvo ambigüedad real de contenido.

### Clasificación
- **EC-005:** Un mensaje no encaja claramente en ninguna categoría. → El sistema debe guardar como nota general en vez de forzar una categoría incorrecta, y puede pedir una aclaración breve si el contexto lo amerita.
- **EC-006:** El usuario corrige una clasificación después de guardada. → El sistema debe mover el registro a la tabla/categoría correcta sin duplicar el dato.

### Modo Terapia
- **EC-007:** Falso positivo de señal de riesgo (el sistema activa el piso de seguridad sin necesidad real, ej. el usuario usa la palabra "drenado" de forma casual). → La respuesta debe seguir sintiéndose natural y no alarmante, evitando que el usuario sienta que fue "malinterpretado" de forma incómoda.
- **EC-008:** Falso negativo (no se detecta una señal real). → Este es el riesgo más serio del sistema; el diseño de detección debe priorizar sensibilidad razonable sobre precisión excesiva en este módulo específico, dado el costo asimétrico del error.
- **EC-009:** El usuario usa Modo Terapia de forma recurrente para el mismo tema sin resolución aparente. → No corresponde a Modo Terapia diagnosticar ni resolver esto en el MVP; el sistema debe seguir ofreciendo el mismo acompañamiento reflexivo sin fingir capacidad clínica.

### Contexto
- **EC-010:** El contexto declarado manualmente contradice una señal automática disponible (ej. usuario dice "estoy en casa" pero es mediodía de un día laboral típico). → El sistema debe priorizar siempre lo que el usuario declara explícitamente sobre cualquier inferencia automática.
- **EC-011:** No hay datos de contexto recientes disponibles al momento de una recomendación. → El sistema debe pedir el dato mínimo necesario en vez de asumir un contexto por defecto incorrecto.

### Datos y proyectos
- **EC-012:** Existen dos proyectos con nombres muy similares. → El sistema debe pedir aclaración antes de asociar una nota o tarea a uno de ellos por error.
- **EC-013:** Se crea un recordatorio con fecha/hora ya pasada (por transcripción incorrecta o ambigüedad del usuario). → El sistema debe confirmar la fecha/hora antes de guardar si detecta que ya pasó.
- **EC-014:** El mismo usuario usa la app desde dos dispositivos simultáneamente. → Los datos deben mantenerse consistentes entre sesiones; no debe haber pérdida de información por condiciones de carrera simples.

### Plataforma
- **EC-015:** El navegador del usuario no soporta audio en segundo plano (limitación conocida de algunos navegadores móviles). → El sistema debe comunicarlo explícitamente en vez de fallar sin explicación.
- **EC-016:** Batería baja o cierre inesperado de la app a mitad de una captura. → Debe minimizarse la pérdida de datos ya hablados antes del cierre, dentro de lo técnicamente razonable para una PWA.

---

## 9. Criterios de aceptación

Formato Given/When/Then, cualitativos (sin cifras), organizados por epic. Sirven como base directa para pruebas de aceptación.

### Epic A — Captura por voz
- **Dado** que el usuario tiene una sesión de voz activa, **cuando** empieza a hablar mientras el sistema está respondiendo, **entonces** el sistema debe detenerse y escuchar, sin ignorar la interrupción.
- **Dado** que el usuario está manejando, **cuando** usa el sistema sin tocar la pantalla, **entonces** debe poder completar una captura completa de forma manos libres.

### Epic B — Clasificación
- **Dado** un mensaje del usuario con intención clara, **cuando** el sistema lo procesa, **entonces** debe guardarse en la categoría correcta sin requerir confirmación adicional en la mayoría de los casos.
- **Dado** que el usuario corrige una clasificación, **cuando** confirma la corrección, **entonces** el dato debe reflejar la categoría correcta sin duplicarse.

### Epic C — Modo Proyecto
- **Dado** un proyecto con tareas y notas asociadas, **cuando** el usuario pregunta por su estado, **entonces** el sistema debe responder con próxima acción, pendientes y bloqueo actual (si existe), sin obligar al usuario a especificar cada dato por separado.

### Epic D — Modo Enfoque
- **Dado** que el usuario declara tiempo y energía disponibles, **cuando** pide una recomendación, **entonces** la sugerencia debe ser ejecutable dentro de ese tiempo y coherente con esa energía (no una tarea pesada si el tiempo es corto).

### Epic E — Modo Terapia
- **Dado** que el usuario está en una conversación de Modo Terapia, **cuando** la conversación termina, **entonces** debe haber quedado registrada una reflexión o una acción pequeña — nunca terminar sin ninguna salida.
- **Dado** que se detecta una señal de riesgo serio, **cuando** el sistema responde, **entonces** debe entregar información de apoyo de forma natural, sin sonar como un mensaje automático desconectado de la conversación.

### Epic F — Revisión Semanal
- **Dado** que ha pasado al menos una semana de uso, **cuando** el usuario pide su revisión semanal, **entonces** el sistema debe presentar un resumen coherente de avances, pendientes y repeticiones basado en datos reales, no genérico.

### Epic G — Motor de Contexto
- **Dado** que no hay contexto reciente disponible, **cuando** el sistema necesita recomendar algo, **entonces** debe pedir el dato mínimo antes de asumir un contexto incorrecto.

### Epic H — Dashboard
- **Dado** que el usuario abre la app, **cuando** llega al Dashboard "Hoy", **entonces** debe ver de inmediato su prioridad, próxima acción y tareas rápidas sin necesidad de navegar a otra pantalla.

### Epic I — Cuenta y privacidad
- **Dado** que existe más de un usuario en el sistema (incluso en pruebas), **cuando** cualquiera de ellos consulta sus datos, **entonces** nunca debe poder ver datos de otro usuario, bajo ninguna condición.

---

## 10. Priorización MoSCoW

Alineada con las fases 0-3 del roadmap definido en `NORTE_AI_Arquitectura.md`. Todo lo listado aquí (Must, Should, Could) constituye el MVP; "Won't" queda fuera de esta etapa de construcción.

### Must have
- Conexión de voz en tiempo real con manejo de interrupciones (VOICE-001 a VOICE-004)
- Motor de clasificación automática (CLASS-001 a CLASS-004)
- Modelo de datos completo con RLS y multi-tenancy (AUTH-001 a AUTH-003)
- Modo Captura Rápida
- Modo Proyecto (versión simple: próxima acción, pendientes, prioridad)
- Dashboard "Hoy"
- PWA instalable (PLAT-001)

### Should have
- Modo Enfoque (reglas simples de energía/tiempo)
- Modo Terapia + piso de seguridad (JOURNAL-003, JOURNAL-004)
- Diario (pantalla dedicada)
- Uso manos libres completo, incluyendo mientras se maneja (VOICE-002)
- Captura de contexto mixta (CTX-001 a CTX-003)

### Could have
- Modo Revisión Semanal (versión basada en filtros, sin búsqueda semántica)
- Corrección manual de clasificaciones desde la interfaz (además de por voz)

### Won't have (en esta fase)
- Integración con Google Calendar
- Búsqueda semántica / pgvector
- Notificaciones push proactivas
- Integraciones con WhatsApp, Notion o Google Sheets
- Detección automática de patrones cruzados entre proyectos
- Agente autónomo

---

## 11. MVP

**Definición explícita:** el MVP de NORTE AI está completo cuando el usuario puede, de principio a fin y por voz:

1. Hablarle al sistema en tiempo real, incluyendo manos libres.
2. Que el sistema clasifique automáticamente lo dicho y lo guarde en la tabla correcta.
3. Usar los 5 modos (Captura Rápida, Proyecto, Enfoque, Terapia, Revisión Semanal), cada uno con la profundidad definida en este documento — no todos tienen que tener inteligencia avanzada, pero todos deben funcionar de extremo a extremo.
4. Ver su día resumido en el Dashboard "Hoy".
5. Confiar en que sus datos están aislados y seguros, incluso siendo el único usuario.

**Fuera del MVP por definición (no por omisión):** todo lo listado como "Won't have" en la sección 10, y cualquier inteligencia basada en búsqueda semántica o aprendizaje de patrones históricos — el MVP usa datos limpios y reglas simples, no memoria semántica.

---

## 12. Funcionalidades futuras

A alto nivel, sin el mismo detalle de user stories/flows que el MVP (ver nota de alcance al inicio del documento). Corresponde a la Fase 4 de `NORTE_AI_Arquitectura.md` y a los objetivos de Etapa 2-3 de `Vision.md`.

- **Google Calendar:** primera integración externa, para cruzar contexto de agenda real con las recomendaciones del sistema.
- **Búsqueda semántica (pgvector):** activar memoria semántica una vez exista volumen de datos suficiente, habilitando detección real de pensamientos recurrentes y patrones cruzados entre proyectos.
- **Notificaciones push proactivas:** el sistema empieza a anticiparse en vez de solo responder (ej. recordar la revisión semanal sin que el usuario la pida).
- **Integraciones adicionales:** WhatsApp, Notion o Google Sheets, según cuál demuestre más valor una vez validado el uso diario (Google Calendar fue la prioridad definida).
- **Agente autónomo:** automatizaciones más profundas (vía n8n/Make u orquestación propia), condicionadas a que el sistema haya demostrado confiabilidad consistente en las fases anteriores.
- **Apertura a usuarios externos (beta privada):** una vez validado el uso personal, sobre la misma arquitectura multi-tenant ya construida desde el MVP.

---

## Nota sobre decisiones tomadas por defecto

Al construir este documento, la pregunta "¿qué alcance debe tener el PRD?" quedó sin respuesta clara ("no lo sé"). Se optó por detallar el MVP por completo y dejar las fases futuras a alto nivel, por ser la opción consistente con el resto de la documentación existente (`NORTE_AI_Arquitectura.md`) y con el timeline de "pocas semanas" ya acordado. Si se prefiere el mismo nivel de detalle para fases futuras, se puede solicitar como una extensión de este documento más adelante, cuando haya evidencia real de uso que justifique planificar esas fases con precisión.
