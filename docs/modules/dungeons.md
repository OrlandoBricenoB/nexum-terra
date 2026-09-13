# Módulo: dungeons

- **Estado:** planned (Etapa B / Fase 5; spawn de portales = Etapa C)
- **Contexto DDD:** Session Orchestration + Simulation (`pve_dungeon`); Inventory al cerrar/looteo. Party en azules/rojas.
- **Código:** (pendiente; `nexum-terra/` dungeons vacío hasta Fase 5)
- **Última actualización:** 2026-09-13 — diseño volcado en GDD §16; este archivo se rellena con la primera implementación.

Copia las secciones de `_TEMPLATE.md` al implementar. Hasta entonces no inventar flujos: contrato de sistema en `PLAN.md` (Party §4.6, `SessionKind.pve_dungeon`, ruleset `DungeonRun`, drops → Inventory).

Diseño de tipos, recompensas, muerte y globales: `docs/GDD.md` §16. KO dentro de la instancia: §12.5; wipe/reentrada: §16.4.

**No implementar en Etapa A.** En B: rooms desde el lobby. **No** ciclos de spawn en mapa, portales de zona ni “el que llega primero” hasta el ADR de Etapa C (`PLAN.md` §18). El interior de instancia se reutiliza en C.
