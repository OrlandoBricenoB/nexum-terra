# Nexum Terra — contexto para agentes

1. Arquitectura y fases: `PLAN.md` (constitución). No crear `PLAN2.md`.
2. Builds, clases, stats, items, skills: `docs/GDD.md`. Zonas / overworld: GDD §15 (congelado).
3. Color, tipo, Theme, voz de UI: `docs/brand/BRAND.md`. No inventar paletas.
4. Cómo está hecho cada módulo/feature: `docs/modules/` (plantilla `_TEMPLATE.md`). Actualizar negocio, técnica y gameplay en el mismo cambio que el código.
5. Cliente/servidor Godot: carpeta `nexum-terra/`.
6. Código y contratos en inglés; docs de diseño en español.
7. Simulación = EntityID + ruleset. Chat/auth/match/**pagos y NC** = Hono (Workers para comercio), nunca ENet.
8. Producto: Etapa A = rooms; Etapa B = mazmorras + meta; Etapa C = open world **solo** tras decisión explícita (`PLAN.md` §1.1 y §18). No implementar regiones, zonas, conquista ni AOI de mundo antes.
9. Dos monedas: Hesedias (Inventory, personaje) y Nexum Coin (Commerce, cuenta). Mercado de jugadores solo Hesedias. Modelo en `PLAN.md` §29 y `docs/GDD.md` §17; no meter PSP en Fases 1–4.
