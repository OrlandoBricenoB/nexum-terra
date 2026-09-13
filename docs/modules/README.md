# Módulos y features — índice

Cada bounded context / feature entregable tiene **un** markdown aquí. Un desarrollador nuevo empieza por esta tabla y abre el doc del módulo.

Norma completa: `PLAN.md` §28. Plantilla: [`_TEMPLATE.md`](./_TEMPLATE.md).

**Estado:** `planned` = aún no hay código; el doc se rellena al implementar (no un wish list). `active` = el doc describe lo que hay en el repo. `deprecated` = no usar; apunta al sucesor.

| Módulo | Estado | Documento | Capa |
| --- | --- | --- | --- |
| identity | planned | [identity.md](./identity.md) | backend |
| character | planned | [character.md](./character.md) | backend |
| inventory | planned | [inventory.md](./inventory.md) | backend + data |
| social | planned | [social.md](./social.md) | backend |
| chat | planned | [chat.md](./chat.md) | backend WS |
| matchmaking | planned | [matchmaking.md](./matchmaking.md) | backend |
| orchestration | planned | [orchestration.md](./orchestration.md) | backend + Godot spawn |
| match-directory | planned | [match-directory.md](./match-directory.md) | backend |
| progression | planned | [progression.md](./progression.md) | backend |
| simulation | planned | [simulation.md](./simulation.md) | Godot `core/` |
| lobby | planned | [lobby.md](./lobby.md) | Godot session |
| spectator | planned | [spectator.md](./spectator.md) | Godot + API |
| client-input-ui | planned | [client-input-ui.md](./client-input-ui.md) | Godot cliente |
| party | planned (Etapa B / Fase 5) | [party.md](./party.md) | backend |
| dungeons | planned (Etapa B instancia; spawn de mapa **C**) | [dungeons.md](./dungeons.md) | Godot + backend |
| world | planned (**Etapa C congelada**) | [world.md](./world.md) | Godot + backend |

Al crear un módulo que no esté en la tabla: añadir fila **y** archivo en el mismo cambio que el código.

Toda modificación de lógica de negocio, técnica o gameplay de un módulo se escribe en su documento en ese mismo cambio.
