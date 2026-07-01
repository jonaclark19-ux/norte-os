# NORTE OS — Hoja de Ruta

**Estado:** vigente · **Versión:** 1.0
**Relacionado:** `docs/DECISIONS.md` (registro ADR). Fases alineadas con `NORTE_AI_Arquitectura.md` §7 y el MVP del `PRD.md`.

---

## Estado actual

- ✅ Análisis técnico completo (informe de CTO).
- ✅ Todas las decisiones de producto ratificadas (ver ADR-008 a ADR-013).
- ✅ Decisiones de ingeniería oficializadas (ADR-001 a ADR-007).
- ⏭️ **Siguiente paso operativo:** arrancar en el repo `norte-os` (esta rama de análisis vive en `tarter-dashboard` por límite de acceso de la sesión; el desarrollo real ocurre en `norte-os`).

---

## Fase 0 — Cimientos (arranca de inmediato en `norte-os`)

Sin bloqueos pendientes. Orden de ejecución:

1. **Scaffold** Next.js 14 (App Router) + TypeScript strict + Tailwind con los tokens del diseño (paleta ámbar `#E8B65C` / rosa terapia `#D98C86`, fondo `#0E0D0C`; tipografías Hanken Grotesk + Instrument Serif).
2. **Migraciones Supabase** (ADR-001, ADR-002): tablas del esquema canónico, RLS en el 100% de las tablas, índices, `audit_log` + `context_logs` particionadas con automatización de particiones, `voice_sessions`, soft delete + triggers.
3. **Prueba de aislamiento RLS** con dos usuarios de test (AUTH-002, NFR-SEC-001).
4. **Contratos Zod** (ADR-004) + esqueleto de **prompts por modo** (ADR-003, ADR-006).
5. **Design system** extraído de `NORTE_OS.dc.html`: primitivos (tokens, tipografía dual, colores) + shells de las 14 pantallas.
6. **PWA base** instalable (manifest + service worker que cachea el shell, no datos de negocio).
7. **Auth Magic Link** (ADR-012) + pantalla de Onboarding (01).

## Fase 1 — Loop central

- `classify_and_save` **end-to-end** como prueba de las dos capas (voz → clasifica → guarda).
- OpenAI Realtime WebRTC: token efímero + sesión de voz + relay de tool calls autenticado con JWT (ADR-009).
- Modo Captura Rápida + Modo Proyecto.
- Dashboard "Hoy".
- **Prueba de humo en iPhone real** de la voz al conducir (ADR-010), con degradación honesta.

## Fase 2 (MVP) — Modos restantes

- Motor de Contexto (reglas) + Modo Enfoque.
- Modo Terapia + piso de seguridad backend-side (ADR-003) + recursos por país (ADR-011).
- Diario.
- Revisión Semanal (filtros + narrativa Claude).
- Pulido de manos libres.

## Fase 3+ — Post-MVP

Ver ADR-C1 a ADR-C6: embeddings/pgvector, Google Calendar, push, integraciones, detección de patrones, agente autónomo, escala.

---

## Camino crítico

`norte-os` creado → sesión de Claude Code apuntada al repo → **Fase 0** (sin consultas: scaffold → migraciones + RLS → aislamiento → design system) → **Fase 1** (`classify_and_save` + voz).
