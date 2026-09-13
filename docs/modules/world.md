# Módulo: world

- **Estado:** planned (Etapa C, **congelado**)
- **Contexto DDD:** World (no hay código)
- **Código:** (pendiente; **prohibido** hasta ADR de Etapa C)
- **Última actualización:** 2026-09-13 — candado de producto: rooms → mazmorras+meta → open world solo con decisión explícita.

Este archivo se rellena con `_TEMPLATE.md` **cuando exista implementación**. Hasta entonces no es un backlog de tickets ni un prototipo.

- Constitución y candado: `PLAN.md` §1.1, §4.12, §18, ADR-010.
- Diseño de regiones, colores de zona y conquista: `docs/GDD.md` §15.
- Portales de mazmorra en el mapa (ciclos, carrera al portal, zona de spawn): `docs/GDD.md` §16.3. El interior sigue siendo instancia (`docs/modules/dungeons.md`), no un shard.
- Robo en cadáveres, logout bajo ataque, escoltas, renegados, divisiones: diseño `docs/GDD.md` §12 (C). No implementar.

**No implementar** mapas de región, AOI, `open_world` ruleset, `world_x/world_y`, estatuas, flags de zona en el tick, ni una escena `World` “para adelantar”.
