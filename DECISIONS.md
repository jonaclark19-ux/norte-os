# NORTE OS — Registro Oficial de Decisiones (ADR)

**Estado:** vigente · **Versión:** 1.0
**Rol de este documento:** acta oficial de decisiones del proyecto. Cada decisión queda con su motivo.
**Documentos fuente:** `Vision.md`, `PRD.md`, `NORTE_AI_Arquitectura.md`, `NORTE_AI_System_Architecture.md`, `NORTE_AI_Database_Design.md` y el diseño `NORTE_OS.dc.html`.

> Nomenclatura: **A** = decisión autónoma del CTO (buenas prácticas / coherencia con docs). **B** = ratificada por el fundador (afecta producto, UX, costo, legal/ético o negocio). **C** = diferida a fases futuras sin bloquear.

---

## ADR-000 · Modelo de gobernanza

El CTO decide de forma autónoma toda decisión resoluble por buenas prácticas de ingeniería, experiencia SaaS moderna o coherencia con la documentación. Se escala al fundador **solo** cuando una decisión: (1) afecta la visión del producto, (2) cambia la experiencia del usuario, (3) implica un costo económico importante, (4) tiene implicancias legales o éticas, o (5) modifica el modelo de negocio. Toda decisión autónoma se documenta aquí con su motivo.

---

## Decisiones autónomas (A)

### ADR-001 · Fuente de verdad del esquema
`NORTE_AI_Database_Design.md` es el esquema canónico. El schema simplificado de `NORTE_AI_Arquitectura.md` v1.0 queda obsoleto.
**Motivo:** el propio doc de Database Design se declara reemplazo; elimina ambigüedad para todas las migraciones.

### ADR-002 · Calibración del schema en semana 1
Desde el día 1: schema completo + RLS + `soft delete` + `created_at` + `audit_log` y `context_logs` particionadas + `voice_sessions`, **con automatización de creación de particiones**. Se difiere `version`/optimistic-locking (ver ADR-C3).
**Motivo:** evita migraciones dolorosas (convertir tabla poblada a particionada) sin cargar la semana 1 con complejidad que aún no paga (versionado con un solo dispositivo). Excepción justificada al principio "complejidad ganada, no anticipada".

### ADR-003 · Detección de seguridad emocional (arquitectura)
La evaluación de riesgo en Modo Terapia corre **backend-side en cada turno** (Agente de Seguridad en paralelo a la conversación), **no** como tool discrecional que el modelo de voz decide invocar.
**Motivo:** minimiza el falso negativo (EC-008), el riesgo más serio del sistema. Resuelve la contradicción entre `Arquitectura.md` (§2) y `System_Architecture.md` (§8.2) a favor del diseño más robusto.

### ADR-004 · Taxonomía de clasificación
Mapa canónico cerrado *categoría → tabla destino → JSON validado con Zod*. Las 12 categorías del PRD (CLASS-001) se rutean a su tabla (`tasks`, `reminders`, `goals`, `journal_entries`, o `notes.category`). Fallback a `nota_general` (EC-005). Los labels del diseño ("Idea comercial") son presentación, no valores de BD.
**Motivo:** contrato duro Claude↔backend; prerrequisito del Agente Clasificador. Evita parsear lenguaje libre.

### ADR-005 · Stack e ingeniería
Next.js 14 (App Router) + TypeScript strict + Tailwind (tokens del diseño) + Supabase (`@supabase/ssr`) + `@anthropic-ai/sdk` + Zod + Serwist (PWA) + migraciones versionadas con Supabase CLI. Monorepo único (frontend + API juntos).
**Motivo:** coherente con la documentación y operable por un desarrollador solo (principio rector #5). Un solo despliegue, sin CORS.

### ADR-006 · Modelos de IA y resiliencia de captura
Claude Sonnet para clasificación, evaluación de seguridad y síntesis semanal. Captura con **confirmación optimista** ("Guardado") + buffer local que reintenta ante corte de red.
**Motivo:** mitiga latencia percibida (R3) y pérdida silenciosa de datos (NFR-REL-001, EC-001/016).

### ADR-007 · Navegación
Configuración accesible desde el avatar/encabezado en "Hoy", sin ocupar un 5.º tab.
**Motivo:** patrón estándar; la tab bar del diseño ya tiene 4 destinos + botón de voz central.

---

## Decisiones ratificadas por el fundador (B)

### ADR-008 · Repositorio
Repositorio nuevo `norte-os`. No se reutiliza el proyecto anterior (`tarter-dashboard`).
**Motivo:** separación limpia de un producto distinto.

### ADR-009 · Capa de Voz
OpenAI Realtime vía WebRTC para el MVP. Token efímero generado por el backend; el cliente se conecta directo al proveedor. Los tool calls se relayan al backend **autenticados con el JWT del usuario** (no con el token de voz). Reevaluación de proveedor solo ante limitaciones importantes.
**Motivo:** menor latencia (sin relay de audio por el backend), arquitectura vendor-swappable (principio rector #4).

### ADR-010 · Conducción y degradación de plataforma
El uso mientras se conduce es un **objetivo de primera clase**. Se diseña con Wake Lock + audio en primer plano. Ante límites de la plataforma (p. ej. iOS Safari en segundo plano), **degradación explícita y honesta** (PLAT-003). Nunca se anuncia una capacidad que la plataforma no sostiene.
**Motivo:** instrucción directa del fundador + NFR de compatibilidad. No prometer lo que no se puede cumplir.

### ADR-011 · Modo Terapia
Modo Terapia es **acompañamiento reflexivo, nunca sustituto de ayuda profesional** ni diagnóstico. Los recursos de emergencia se obtienen **dinámicamente según el país del usuario** (locale/geo + timezone del perfil), verificados. Sin números de crisis hardcodeados.
**Motivo:** seguridad del usuario y coherencia con `Vision.md` ("Qué NO hará el producto"). Implicancia ética.

### ADR-012 · Autenticación
Magic Link vía Supabase para el MVP. Google OAuth se pospone.
**Motivo:** mínima fricción, coherente con el onboarding "empezar hablando"; priorizar calidad de producto primero.

### ADR-013 · Política de costos
Instrumentación de costo desde el día 1 (`voice_sessions`, timeout de sesión EC-003, tope de duración), pero **sin optimización agresiva** en el MVP: se prioriza velocidad de desarrollo.
**Motivo:** proyecto autofinanciado en etapa de validación, no de escala. El costo de voz por minuto es el driver a vigilar (System Architecture §12.2).

---

## Decisiones diferidas (C) — no bloquean el MVP

- **ADR-C1** · Proveedor de embeddings + confirmación de dimensión del `vector` (hoy `vector(1536)`). Fase 2.
- **ADR-C2** · Activación de `pgvector` + índice HNSW cuando exista volumen real. Fase 2.
- **ADR-C3** · Versionado / optimistic-locking cuando haya uso multi-dispositivo real (EC-014).
- **ADR-C4** · Integraciones externas: Google Calendar, Web Push, WhatsApp, Notion. Fase 2+.
- **ADR-C5** · Detección de patrones cruzados y agente autónomo. Fase 3+.
- **ADR-C6** · Archivado / réplicas de lectura / particionado a escala. Cuando el volumen multi-tenant lo justifique.
