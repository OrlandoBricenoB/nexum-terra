# Nexum Terra — Plan Director del Proyecto

**Este archivo es la constitución del proyecto.** Toda decisión de arquitectura, dominio, API, red, persistencia y fases de construcción se deriva de aquí. Si una implementación contradice este documento, se actualiza el código o se actualiza este plan con una decisión explícita (ADR). No se improvisan capas, tablas ni protocolos “por el camino”.

- **Nombre:** Nexum Terra
- **Género actual:** Top-down 2D, lobby social + combates instanciados (PvP primero; PvE después)
- **Destino (congelado):** Open world MMORPG por **regiones** (shards), no un mapa único. No se construye hasta una **decisión explícita**.
- **Principio de producto:** crecer por capas y **no mezclar eras**. Primero un juego de **solo rooms** de combate. Después mazmorras **y** el metajuego (build, inventario, mercado, clanes, etc.). El mundo abierto es otra sesión, más grande y persistente, y **no** se adelanta “un poco” en las eras anteriores.

---

## Índice

1. [Visión y norte](#1-visión-y-norte)
2. [Principios no negociables](#2-principios-no-negociables)
3. [Arquitectura elegida](#3-arquitectura-elegida)
4. [Mapa de dominios (bounded contexts)](#4-mapa-de-dominios-bounded-contexts)
5. [Arquitectura general del sistema](#5-arquitectura-general-del-sistema)
6. [Stack tecnológico](#6-stack-tecnológico)
7. [Estructura del monorepo](#7-estructura-del-monorepo)
8. [Contratos compartidos y protocolos](#8-contratos-compartidos-y-protocolos)
9. [Modelo de datos](#9-modelo-de-datos)
10. [API REST](#10-api-rest)
11. [WebSocket (chat y señalización)](#11-websocket-chat-y-señalización)
12. [Red de juego (Godot)](#12-red-de-juego-godot)
13. [Núcleo de simulación y combate](#13-núcleo-de-simulación-y-combate)
14. [Orquestación de sesiones](#14-orquestación-de-sesiones)
15. [Cliente Godot](#15-cliente-godot)
16. [Cómo añadir features sin colisionar](#16-cómo-añadir-features-sin-colisionar)
17. [Fases de construcción](#17-fases-de-construcción)
18. [Camino a open world MMORPG](#18-camino-a-open-world-mmorpg)
19. [Seguridad](#19-seguridad)
20. [Observabilidad y operaciones](#20-observabilidad-y-operaciones)
21. [Convenciones](#21-convenciones)
22. [Decisiones técnicas (ADR)](#22-decisiones-técnicas-adr)
23. [Riesgos y mitigaciones](#23-riesgos-y-mitigaciones)
24. [Definición de hecho (Definition of Done)](#24-definición-de-hecho-definition-of-done)
25. [Mapa de documentos (no hay PLAN2)](#25-mapa-de-documentos-no-hay-plan2)
26. [Despliegue: Cloudflare, AWS y el cluster de juego](#26-despliegue-cloudflare-aws-y-el-cluster-de-juego)
27. [Branding y UI](#27-branding-y-ui)
28. [Documentación por módulo y feature](#28-documentación-por-módulo-y-feature)
29. [Modelo de negocio y comercio](#29-modelo-de-negocio-y-comercio)

---

## 1. Visión y norte

Nexum Terra empieza como un **hub vivo**: una habitación (lobby) donde el jugador se mueve, habla, practica ataques contra dummies y busca partida. Desde ese hub se entra a **rooms de combate** (1v1, 5v5). Cuando ese recorte esté **jugado y estable**, se añaden **rooms de mazmorra** y el metajuego (build, inventario, mercado, clanes, etc.). El destino de fantasía es un **mundo abierto persistente por regiones**, con las mismas reglas de movimiento y combate; **no se implementa** hasta que el equipo lo decida por escrito, meses después de la era de mazmorras.

La trampa a evitar: construir un “juego de salas” rígido y luego romperlo para el MMO. Por eso el núcleo admite un futuro `world_shard`. La trampa inversa es igual de cara: **meter zonas, loot de cadáver, conquista o mapas de región** mientras aún no hay un juego de rooms. Por eso:

- Toda instancia (lobby, PvP, PvE, futuro shard de mundo) es una **Session**.
- Todo actor simulable (jugador, dummy, monstruo, jefe, NPC, proyectil con dueño) es una **Entity** con `EntityID`.
- El combate, el movimiento y el ciclo de tick **no conocen** `PlayerID`. Conocen entidades, componentes y un ruleset.
- El backend de cuentas, inventario, amigos y matchmaking **no conoce** Godot. Habla por puertos.
- **Open world no se “deja a medias”.** Diseño en `docs/GDD.md` §15 y arquitectura en §18 de este PLAN. Código, mapas de región, flags de zona en el tick y AOI de mundo: **prohibidos** hasta el desbloqueo de la Etapa C.

### 1.1 Recorte de producto por etapa

Tres eras. No se pisan. La C no arranca por “quedó tiempo en el sprint”.

| Etapa | Fases | Lo que el jugador ve | Lo que el sistema es | Cuándo |
| --- | --- | --- | --- | --- |
| **A — Rooms** | 1–4 | Login, lobby 2D, chat, práctica, colas, 1v1/5v5, espectador | Modular monolith + sesiones instanciadas | Ahora. Producto publicable al cerrar Fase 4 |
| **B — Mazmorras + meta** | 5.x | Parties, **rooms de mazmorra desde el lobby**, IA; build, inventario, mercado, clanes y el resto del metajuego de `docs/GDD.md` que no sea mundo | Mismo núcleo; ruleset `pve_dungeon`; persistencia de inventario/economía | **Después** de probar el juego de solo rooms. No antes |
| **C — Open world** | 6–8 | Regiones persistentes, sistema de zonas (celeste/verde/amarilla/roja), conquista, loot de cadáver en mundo | Shards **por región** + instancias de mazmorra/PvP reutilizadas | **Solo** tras meses de Etapa B y una **decisión explícita** (ADR en este PLAN). Ver §18 |

Las mazmorras **no** se construyen hasta que el núcleo de rooms (auth, lobby, red, combate PvP, matchmaking, persistencia de resultados) esté sólido. En B se entra **igual que a un duelo**: desde el lobby a un battle node. El spawn aleatorio de portales en zonas del mapa es **Etapa C** (`docs/GDD.md` §16). El open world **no** se construye hasta que mazmorras + meta se hayan jugado y el equipo lo escriba aquí. El diseño de zonas y de esos spawns ya está en el GDD para no perderlo; eso **no** autoriza implementación.

---

## 2. Principios no negociables

1. **Servidor autoritativo.** El cliente predice movimiento y muestra VFX. El servidor decide posición legal, hits, daño, CC, cooldowns y muerte. Nunca se confía en un paquete de “yo hice 50 de daño”.
2. **EntityID desde el día uno.** No existe un sistema de combate acoplado a jugadores. Los dummies del lobby ya son entidades. La Fase 5 añade IA; no refactoriza IDs.
3. **Separación de planos de red.** Chat, auth, invitaciones y matchmaking van por **HTTP/WebSocket (Hono)**. Simulación (posiciones, inputs, hits) va por **red de Godot (ENet/UDP)**. Nunca se mezclan.
4. **Hexágono por dominio.** Cada bounded context tiene núcleo puro, puertos (interfaces) y adaptadores (Hono, Postgres, Mongo, Docker, Godot). El dominio no importa Hono, Neon, Mongo ni nodos de Godot.
5. **Contenido data-driven.** Habilidades, items, mapas, rulesets y spawns viven en datos versionados. Añadir una skill o un mapa no exige reescribir el loop de combate.
6. **Una sesión, muchos rulesets.** Lobby, duelo, arena y mazmorra cambian reglas; el motor de tick no. El ruleset de mundo abierto está **reservado** (Etapa C); no se registra ni se ejecuta antes del ADR de §18.
7. **Contratos versionados.** REST, WS y paquetes de juego tienen esquema. Un cambio incompatible sube versión; no se “rompe y ya”.
8. **Crecimiento vertical primero.** Un modular monolith bien partido. **No hay microservicios** hasta que una réplica del backend no baste. Dos entrypoints de deploy (Workers vs EC2) no son microservicios: son el mismo código con adaptadores distintos.
9. **Fail closed.** Token inválido, instancia caída, puerto ocupado o tick atrasado: se rechaza o se cancela la sesión. No se “sigue igual”.
10. **El PLAN manda.** Features nuevas se enganchan a un dominio existente o declaran uno nuevo. No se cuelan en controladores gordos ni en el `_process` del jugador.
11. **Documentar al mismo tiempo que se construye.** Cada módulo/feature tiene un documento vivo en `docs/modules/`. Todo cambio de lógica de negocio, técnica o de gameplay se refleja ahí en el mismo cambio que el código. Código sin doc del módulo no está terminado.
12. **Eras de producto no se adelantan.** A = rooms. B = mazmorras + meta. C = open world solo con decisión explícita. Diseño de C puede vivir en el GDD; código de C no.

---

## 3. Arquitectura elegida

### 3.1 Por qué hexagonal + DDD (y no “MVC + scripts de Godot”)

Un MMORPG acumula sistemas que se pisan: inventario afecta combate, el lobby afecta presencia, el matchmaker afecta orquestación, el chat afecta parties. Si todo eso vive en rutas Hono o en nodos Godot, cada feature nueva rompe la anterior.

**Arquitectura hexagonal (puertos y adaptadores)** + **Domain-Driven Design por bounded contexts** es la que mejor encaja:

- El **dominio** expresa reglas (quién puede invitar, cómo se calcula un hit, cuándo una cola está lista).
- Los **puertos** son interfaces (`UserRepository`, `ChatBus`, `SessionSpawner`, `Clock`, `SpatialQuery`).
- Los **adaptadores** son el mundo real (Postgres, Mongo, Hono WS, Docker, ENet, escena Godot).

Alternativas descartadas (y por qué):

| Alternativa | Problema para Nexum Terra |
| --- | --- |
| Microservicios desde el día 1 | Complejidad de red, deploys y transacciones antes de tener jugadores |
| Clean Architecture rígida de 20 carpetas por CRUD | Ceremonia; hexagonal por contexto es suficiente |
| ECS puro en backend Node | El backend no simula el mundo; Godot sí. ECS/componentes viven en el núcleo de simulación |
| “Todo en Godot” (auth, chat, DB) | Godot headless no es el sitio de cuentas, amigos ni orquestación de procesos |
| Lobby hardcodeado distinto al combate | Impide el salto a mundo abierto; obliga a reescribir movimiento y combate |

### 3.2 Forma concreta (híbrido)

```
┌─────────────────────────────────────────────────────────────┐
│                     Composition Root                        │
│         (worker.ts | realtime.ts | Godot DedicatedMain)     │
└─────────────┬───────────────────────────────┬───────────────┘
              │                               │
   ┌──────────▼──────────┐         ┌──────────▼──────────┐
   │  Application Layer  │         │  Simulation Loop    │
   │  (use cases)        │         │  (tick, 20–30 Hz)   │
   └──────────┬──────────┘         └──────────┬──────────┘
              │                               │
   ┌──────────▼──────────┐         ┌──────────▼──────────┐
   │  Domain (puro)      │         │  Domain (puro)      │
   │  Identity, Social,  │         │  Entity, Combat,    │
   │  Matchmaking, ...   │         │  Movement, AI       │
   └──────────┬──────────┘         └──────────┬──────────┘
              │                               │
   ┌──────────▼──────────┐         ┌──────────▼──────────┐
   │  Adapters           │         │  Adapters           │
   │  HTTP, WS, SQL,     │         │  Godot Nodes, ENet, │
   │  Mongo, Docker      │         │  TileMap, Animation │
   └─────────────────────┘         └─────────────────────┘
```

- **Backend (Node):** hexagonal clásico por carpeta de dominio.
- **Simulación (Godot headless + cliente compartiendo core):** dominio de entidades **sin** `extends Node`. Los nodos son adaptadores de render, input y transporte.
- **Comunicación entre contextos del backend:** eventos de dominio en proceso cuando corren en el mismo entrypoint. Entre Workers y el proceso realtime de EC2: HTTP interno con service key (no es un tercer servicio). Extraer Redis/cola solo si una réplica ya no basta.
- **Comunicación realtime ↔ Godot dedicado:** el orquestador **en EC2** arranca procesos y les pasa env (`MATCH_UUID`, `MAP_ID`, `RULESET`, `JOIN_SECRET`, puerto). El servidor Godot valida join tokens con HMAC inyectado o contra el proceso realtime local (no contra Workers en el hot path del tick).

### 3.3 Modular monolith (backend)

Un solo codebase Hono. **Dos entrypoints**, no microservicios:

| Entrypoint | Dónde corre | Qué monta |
| --- | --- | --- |
| `app/worker` | **Cloudflare Workers** | REST de servicios + **Neon vía Hyperdrive** (auth, perfil, inventario, amigos, historial, tokens de cuenta) |
| `app/realtime` | **AWS EC2** (mismo host que Godot) | WebSockets (chat, invites, `match:ready`), MongoDB, matchmaker in-memory, orquestador |

Hasta que una réplica de alguno de esos entrypoints no baste, no se parte chat, matchmaker ni orquestación en procesos/servicios extra.

```
backend/src/modules/<contexto>/
  domain/          # entidades, value objects, errores, eventos
  application/     # casos de uso (login, enqueue, invite)
  ports/           # interfaces
  adapters/
    http/          # rutas Hono de este contexto (Workers y/o EC2)
    ws/            # handlers WS (solo entrypoint realtime / EC2)
    persistence/   # Neon (Workers + Hyperdrive) / Mongo (EC2)
    infra/         # spawn Godot, clock
```

Regla: un módulo **no importa** el `adapters` de otro. Si Identity necesita avisar a Social, publica `UserRegistered`; Social escucha. Si hace falta leer datos de otro contexto, se usa un **puerto de consulta** (ACL / facade), no se entra a sus tablas a ciegas.

---

## 4. Mapa de dominios (bounded contexts)

Cada contexto tiene un idioma propio. No se mezclan términos (`Room` de chat ≠ `Room` de match ≠ `Session` de simulación).

### 4.1 Identity & Access

- **Responsabilidad:** cuentas, credenciales, sesiones de API, tokens de unión a servidores de juego.
- **Lenguaje:** User, Credential, AccessToken, RefreshToken, GameJoinToken.
- **No hace:** combate, inventario, chat.

### 4.2 Profile & Character

- **Responsabilidad:** personaje visible en lobby/combate, caps de vitals (maná, stamina, vitalidad), cosméticos de presentación.
- **Lenguaje:** Character, Appearance, BaseStats, Loadout, Honor, KillCount, KingdomRank.
- **Al inicio:** un personaje por cuenta. El modelo admite N personajes sin reescribir auth.
- **No hace:** simular el combate en tiempo real (solo persiste el resultado y el loadout).

### 4.3 Inventory & Economy (persistente; combate consume snapshots)

- **Responsabilidad:** posesión de items, equipo, moneda (**Hesedias**), crafts, mercado de jugadores. Más adelante: drops de sesión.
- **Lenguaje:** ItemDef, ItemInstance, EquipmentSlot, Currency, MarketListing, Snapshot.
- **Regla:** el servidor de batalla recibe un **snapshot inmutable** del loadout al iniciar la sesión (mods **ya resueltos**: def + calidad/nivel/rolls). No escribe inventario a mitad de un 1v1. Al terminar, aplica un **delta** (recompensas, durabilidad futura, Hesedias) vía evento `SessionEnded`.
- **Crafting y mercado** son casos de uso de este contexto (Hono/REST en Workers). No van por ENet. Al **listar** un ítem, sale del inventario (escrow en `market_listings`); el vendedor no lo equipa ni lo duplica. Compra válida con el vendedor **offline**.
- **Moneda de mundo:** solo **Hesedias** (`currency_id` `hesedias`) en `character_wallets`. El mercado de jugadores **no** cotiza Nexum Coin ni dinero real.
- **Por qué:** evita race conditions entre tick de combate y REST de inventario. En mazmorras (Etapa B) y más tarde en shards (Etapa C) el drop es el mismo puerto: evento → Inventory.
- **Diseño de rolls, calidad y orbes de uso:** `docs/GDD.md` §8.10–8.18. No se implementa en Fases 1–4 (Etapa A). Mercado y crafting completos = Etapa B, no open world. Dinero real y NC: contexto **Commerce** (§4.14, §29), no este.

### 4.4 Social

- **Responsabilidad:** amigos, bloqueos, presencia ligera (online/offline/in_match), invitaciones sociales.
- **Lenguaje:** Friendship, Block, Presence, Invite (social, no de match).

### 4.5 Chat

- **Responsabilidad:** canales, historial, mensajes offline, moderación básica.
- **Lenguaje:** Channel (global, whisper, party, system, future guild/zone), ChatMessage.
- **Transporte:** WebSocket del proceso realtime en **EC2**. El servidor Godot **no** reenvía chat.
- **Persistencia:** MongoDB al que se conecta ese proceso (no Hyperdrive, no Workers).

### 4.6 Party

- **Responsabilidad:** grupo de 2–5 jugadores que viaja junto a una sesión PvE (y, solo en Etapa C, al mundo).
- **Lenguaje:** Party, PartyMember, Leader, ReadyState.
- **Reutiliza:** el flujo de sala privada del matchmaking, no un sistema paralelo. En Fase 5, “sala privada → orquestar mapa dungeon” es un caso de uso de Party + Matchmaking + Orchestration.
- **No se implementa hasta Fase 5**, pero el ID `party_id` ya existe como opcional en Session y Chat.

### 4.7 Matchmaking

- **Responsabilidad:** colas, salas privadas PvP, criterios de emparejamiento, ready-check.
- **Lenguaje:** Queue (1v1, 5v5, future modes), Ticket, PrivateMatchRoom, MatchProposal.
- **Estado al inicio:** in-memory en el proceso **realtime (EC2)**. No en Workers.
- **Puerto:** `MatchmakingStore`. Redis solo si una réplica de ese proceso no basta. El dominio no cambia.

### 4.8 Session Orchestration

- **Responsabilidad:** ciclo de vida de procesos Godot headless (lobby persistente y battle nodes efímeros).
- **Lenguaje:** Session, SessionKind, SessionEndpoint (ip, port), JoinToken, Allocation, Health.
- **SessionKind (cerrado, extensible):**
  - `lobby` — persistente, práctica, sin ranking
  - `pvp_duel` — 1v1
  - `pvp_arena` — 5v5
  - `pve_dungeon` — Fase 5 (rooms desde lobby; en C el join sale del `world_shard` al mismo kind)
  - `world_shard` — Etapa C (Fases 6+); **nombre reservado, sin implementación** hasta ADR de desbloqueo
  - `spectator_relay` — no es un proceso extra; es un rol de conexión sobre una sesión existente
- **No simula el juego.** Solo crea, vigila y mata instancias.

### 4.9 Simulation (Godot headless, dominio compartido con el cliente)

- **Responsabilidad:** tick, movimiento, habilidades, hitboxes, CC, muerte, victoria, espectadores.
- **Lenguaje:** Entity, EntityID, Intent, Command, Snapshot, Ruleset, Tick, AreaOfInterest.
- **Ruleset:** plug-in de reglas (`PracticeLobby`, `CompetitivePvp`, `DungeonRun`, `OpenWorld`). Cambia scoring, persistencia de daño, respawn, friendly fire, AI enabled.

### 4.10 Spectator & Match Directory

- **Responsabilidad:** listar sesiones vivas, metadatos para el UI del lobby, unirse como espectador.
- **Lenguaje:** LiveMatch, SpectatorJoin.
- **Datos:** proyección en memoria (o Redis) alimentada por el orquestador. No es la fuente de verdad del combate.

### 4.11 Progression & Battle Log (Postgres)

- **Responsabilidad:** historial de partidas, MMR futuro, XP, unlocks, **clasificación** (Honor + asesinatos → rango D–SSS, `docs/GDD.md` §11.6) y **rango de reino** (`docs/GDD.md` §11.7).
- **Lenguaje:** MatchRecord, ParticipantResult, Rating, Honor, KillCount, ClassificationRank, KingdomRank.
- **El lobby de práctica no escribe aquí.** Daño a dummies es efímero. Honor y kills no suben en `practice_lobby`.
- **Clasificación:** rango derivado de umbrales AND en `nexum-terra/data/classification-ranks.json`. No se elige a mano.
- **Rango de reino:** persistido (`kingdom_rank_id`); promociones de evento (Soldado, Élite) no se infieren solo de Honor. Comandante sí se infiere de Honor 150 + clasificación B cuando esos contadores existan. Órdenes/territorios no se implementan hasta Etapa C.

### 4.12 World (Etapa C, congelado)

- **Responsabilidad (cuando se desbloquee):** regiones del overworld, shards, interés espacial, persistencia de posición, spawns de mundo, reglas de zona (PvP/loot) y conquista de territorios.
- **Lenguaje:** Region, ZoneDanger (celeste/verde/amarilla/roja), Shard, AOI, WorldPosition, Territory, PowerStatue.
- **Hasta el ADR de desbloqueo de Etapa C no hay código** de este contexto: ni mapas de región, ni AOI, ni flags de zona en el tick, ni cadáveres lootables de mundo, ni estatuas de conquista.
- **Gancho permitido (sin implementar OpenWorld):** el enum puede listar `SessionKind.world_shard`; las entidades ya tienen posición. No se añade `ruleset open_world` ejecutable ni data de zonas.
- **Regiones, no un mapa único.** Cada región es un shard (o un conjunto de shards) propio: recorta carga (optimización) y recorta el playground (gameplay: viajes, drops, conquista). Detalle de diseño: `docs/GDD.md` §15.

### 4.13 Mapa de dependencias permitidas

```
Identity ← Profile/Character
Identity ← Social
Identity ← Chat
Character ← Inventory
Character + Inventory → snapshot → Simulation (vía Orchestration)
Matchmaking → Orchestration → Simulation process
Social/Chat ← eventos de Matchmaking (invites, ready)
Simulation → Battle Log / Inventory  (solo al terminar o en checkpoints PvE/MMO)
World (futuro) → mismos puertos que Simulation
Identity ← Commerce
Commerce → Inventory (cumplir SKU: ítem o Hesedias)
Commerce → Progression (p. ej. puntos de habilidad comprados)
```

Prohibido: Simulation importando Hono; Chat importando hitboxes; Matchmaking escribiendo stats de combate; **Godot o ENet cobrando o acreditando NC**.

### 4.14 Commerce & Entitlements

- **Responsabilidad:** dinero real, **Nexum Coin (NC)**, catálogo de SKU, pedidos, webhooks del procesador de pagos, entitlements (beta/Patreon/crowdfunding, suscripción) y cumplimiento hacia Inventory/Progression. Infoproductos (guías, etc.) si se venden con el mismo checkout.
- **Lenguaje:** NexumCoin, Sku, Order, Payment, Entitlement, Fulfillment, Subscription.
- **No hace:** simular combate; listar ítems en el mercado de jugadores; inventar balances de Hesedias.
- **Transporte:** solo **Hono REST en Workers** + webhook del PSP. Nunca ENet. El cliente no “se da” NC ni ítems de tienda.
- **Por qué existe aparte de Inventory:** PCI, idempotencia de pagos y entitlements de cuenta no son posesión de ítems. Mezclarlos en `market_listings` rompe fail-closed y auditorías.
- **Gancho desde Etapa A:** ids de moneda, tablas y este contexto están **declarados**. No hay tienda jugable ni PSP en Fases 1–4. El primer migrate de wallets **ya** distingue `hesedias` y `nexum_coin`; no se añade NC con un parche de última hora.
- Diseño de producto (tasas, canales, pay-to-win): `docs/GDD.md` §17 y este PLAN §29.

---

## 5. Arquitectura general del sistema

### 5.1 Componentes

```
[Jugador]
   │
   ├─ Game Client (Godot 4.7.2)
   │     ├─ Render top-down 2D
   │     ├─ Input (teclado, ratón, mando, táctil)
   │     ├─ Predicción de movimiento + reconciliación
   │     ├─ UI: login, chat flotante, colas, invites, espectador
   │     ├─ HTTP → Cloudflare Workers (Hono REST + Neon/Hyperdrive)
   │     ├─ WebSocket → EC2 (Hono WS + Mongo)
   │     └─ ENet/UDP → Godot headless en el mismo EC2
   │
   ├─ Cloudflare Workers (servicios)
   │     ├─ REST: auth, perfiles, inventario, amigos, historial, comercio (NC / webhooks)
   │     └─ PostgreSQL (Neon) vía Hyperdrive
   │
   └─ AWS EC2
         ├─ Hono realtime: WS (chat, invites, match:ready) + Mongo + matchmaker + orquestador
         └─ Godot 4.7.2 headless: Lobby Server + Battle Nodes
```

### 5.2 Flujo de un jugador (día a día, Fases 2–4)

1. Cliente abre → REST Workers `POST /api/auth/login` → JWT (Neon).
2. REST Workers `GET /api/me` + loadout.
3. WebSocket a **EC2** con el mismo JWT → canal `global` + presencia (Mongo).
4. El proceso realtime (o REST de sesión en EC2) emite endpoint del lobby + `GameJoinToken`.
5. Cliente carga mapa Lobby, conecta ENet al Godot headless **en ese EC2**.
6. Lobby valida token (HMAC local), spawnea entidad.
7. El jugador se mueve, ataca dummies (daño no persistente), chatea por WS (Mongo).
8. Encola Quick Match o crea sala privada contra el proceso realtime (misma máquina que Godot). Invites por WS.
9. Matchmaker cierra el grupo → orquestador en EC2 levanta battle node → WS `match:ready {ip, port, join_token, map_id}`.
10. Cliente **mantiene** el WS de EC2, **desconecta** ENet del lobby, carga mapa de batalla, conecta al nuevo puerto del mismo host (u otro EC2 de flota, más adelante).
11. Tras el combate, el battle node reporta al proceso realtime; este persiste el resultado llamando al REST interno de Workers (Neon). Se apaga el node; el cliente vuelve al lobby (paso 4–5).

Espectador: desde lobby, `GET /api/matches/live` → conecta al mismo battle node con `role=spectator`. Recibe snapshots; cualquier input de movimiento/habilidad se descarta.

### 5.3 Por qué el lobby es una “habitación” y no el mundo (todavía)

El lobby es una **Session persistente de práctica y social**. Misma entidad, mismo movimiento, mismo input. Cambia el ruleset:

- Sin ranking ni match history por hits
- Respawn inmediato
- Dummies estáticos (AI nula o idle)
- Cap de jugadores (si se llena: segundo proceso `lobby_2`, mismo patrón que un shard)

Cuando (y solo cuando) exista decisión de Etapa C, el cliente hará el mismo handshake contra un `world_shard` de **una región**. El lobby puede convertirse en una ciudad (Aurora u otra instancia social) o permanecer como room. **No se tira el código de rooms.** Hasta ese ADR, el lobby **no** es un proto-overworld: no hay peligro de zona, no hay conquista, no hay “salir a un campo”.

---

## 6. Stack tecnológico

| Capa | Tecnología | Dónde | Rol |
| --- | --- | --- | --- |
| Cliente | Godot **4.7.2** | dispositivo | Render 2D top-down, input, predicción, UI |
| Servicios REST | Node.js + TypeScript + **Hono** | **Cloudflare Workers** | Auth, perfiles, inventario, amigos, historial, **comercio/pagos** |
| Postgres | **Neon + Hyperdrive** | desde Workers | Users, characters, friends, inventory, match logs |
| Tiempo real | Node.js + TypeScript + **Hono WS** | **AWS EC2** | Chat, whispers, invites, `match:ready`, colas |
| Documentos | **MongoDB** | desde el proceso WS en EC2 | Chat, mensajes offline, lobby events |
| Servidor de juego | Godot **4.7.2** headless | **el mismo EC2** | Lobby + battle nodes, simulación autoritativa |
| Orquestación | child_process / Docker en EC2 | **EC2** | Spawn/kill de headless |
| Cola futura | Redis | solo si una réplica no basta | Tickets, presencia |
| Contratos | TypeScript types + JSON Schema | repo | REST/WS/paquetes |

### 6.1 Red de Godot

- Transporte de simulación: **ENet** (UDP, integrado en Godot High-level Multiplayer o API ENet directa).
- Tick de servidor objetivo: **20 Hz** lobby, **20–30 Hz** combate. Configurable por ruleset.
- Cliente: interpolación de remotos + predicción local del propio personaje.

### 6.2 Autenticación

- REST (Workers): JWT de acceso corto + refresh token (Neon).
- WS (EC2): el mismo access token en el handshake; el proceso realtime verifica la firma (secreto/JWKS compartido con Workers).
- Godot join: **GameJoinToken** emitido por el proceso realtime en EC2, un solo uso / TTL corto (60–90 s), atado a `user_id`, `character_id`, `session_id`, `role` (`player` \| `spectator`).

---

## 7. Estructura del monorepo

```
NexumTerra/
  PLAN.md                         # constitución de arquitectura (este archivo)
  AGENTS.md                       # índice para el agente
  docs/
    GDD.md                        # diseño de juego (reinos, builds, items, stats)
    brand/BRAND.md                # colores, tipografía, voz visual
    modules/                      # cómo está hecho cada módulo/feature (vivos)
      README.md
      _TEMPLATE.md
  README.md
  docker-compose.yml
  .env.example
  contracts/                     # fuente de verdad de DTOs y eventos
    rest/
    ws/
    game/                        # paquetes de simulación (nombres + payload)
    jsonschema/
  backend/
    src/
      app/
        worker.ts                # entrypoint Cloudflare Workers (REST + Neon/Hyperdrive)
        realtime.ts              # entrypoint EC2 (WS + Mongo + match + orch)
      modules/
        identity/
        character/
        inventory/
        social/
        chat/
        matchmaking/
        orchestration/
        match-directory/
        progression/
      shared/
        domain/                  # Result, Clock, EventBus
        infra/
    tests/
  nexum-terra/                   # proyecto Godot 4.7 (cliente + dedicated)
    project.godot
    core/                        # dominio puro (sin Node)
      entity/
      combat/
      movement/
      rulesets/
      protocol/
    adapters/
      godot/                     # CharacterBody2D, AnimatedSprite, UI
      net/                       # ENet / MultiplayerAPI
      input/
    client/                      # escenas y flujo de cliente
    dedicated/                   # entrypoint headless
    maps/
      lobby/
      pvp/
      dungeons/                  # vacío hasta Fase 5
    data/                        # skills, items, dummy defs (recursos)
  infra/
    postgres/migrations/
    mongo/indexes/
    docker/
    scripts/spawn-godot-session.sh
```

**Un solo proyecto Godot.** Cliente y dedicado comparten `core/` y `maps/`. El preset headless arranca `dedicated/DedicatedMain.tscn` con argumentos `--session-kind`, `--map`, `--port`, `--match-uuid`.

No se duplica el juego en dos repos. El cliente **no** contiene la autoridad del daño; contiene predicción y presentación.

---

## 8. Contratos compartidos y protocolos

Toda comunicación cruza un contrato en `contracts/`. Godot replica los nombres de evento/paquete 1:1.

### 8.1 Convenciones

- REST: JSON, `camelCase`, errores `{ "error": { "code", "message" } }`.
- WS: `{ "v": 1, "type": "chat:send", "id": "uuid", "payload": { } }`.
- Game packets: enteros de tipo estables; payload binario o JSON compacto al inicio (JSON es válido hasta que el profiler lo contradiga). Prefijo `v1/`.
- IDs públicos: UUID v7 (texto). EntityID en simulación: `uint32` de sesión (más barato en tick) mapeado a `character_id` UUID solo en el handshake y en el reporte final.

### 8.2 Versionado

- `v` en WS y cabecera `X-API-Version` en REST.
- Cambios backward-compatible (campo nuevo opcional): misma versión.
- Cambios breaking: `v2` y periodo de solape.

---

## 9. Modelo de datos

### 9.1 PostgreSQL (Neon) — relacional

Migraciones en `infra/postgres/migrations/`. Nombres en `snake_case`.

#### `users`

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | |
| email | citext unique | |
| password_hash | text | argon2id |
| display_name | text unique | |
| created_at | timestamptz | |
| banned_at | timestamptz null | |

#### `refresh_tokens`

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | |
| user_id | uuid FK | |
| token_hash | text | |
| expires_at | timestamptz | |
| revoked_at | timestamptz null | |

#### `characters`

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | |
| user_id | uuid FK unique (fase 1: 1:1) | |
| name | text unique | |
| kingdom_id | text | catálogo `nexum-terra/data/kingdoms.json`. Jugable: `fontaine` `terrara` `spectra` `aerion`. `aurora` solo GM |
| clan_id | text | catálogo `nexum-terra/data/clans.json`; debe pertenecer a `kingdom_id`. Skills del clan vacías en primeras etapas; el tick no las aplica |
| honor | int | default 0. Misiones (cuando existan). GDD §11.6 |
| kills | int | default 0. Asesinatos persistidos (rooms PvP en A; no lobby) |
| kingdom_rank_id | text | default `novice`. Catálogo `nexum-terra/data/kingdom-ranks.json`. GDD §11.7 |
| level | int | default 1 |
| appearance | jsonb | |
| last_lobby_x, last_lobby_y | real | opcional persistencia de posición de hub |
| created_at | timestamptz | |

#### `character_stats`

| Columna | Tipo | Notas |
| --- | --- | --- |
| character_id | uuid PK/FK | |
| mana_max, stamina_max, vitality_max | numéricos | caps **base**; el combate usa snapshot + mods de equipo (maná/stamina). Ver `docs/GDD.md` §6. No hay `atk`/`def` |

#### `item_defs` (puede empezar como JSON en repo y promocionar a tabla)

Catálogo versionado. En Fase 1 puede vivir en `game/data` + tabla espejo cuando Inventory REST exista.

#### `item_instances`

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | |
| owner_character_id | uuid FK | |
| def_id | text | |
| qty | int | |
| bound | bool | |
| payload | jsonb | calidad, `item_level` 1–5, atributos crafteados, durabilidad futura; orbes compuestos: `{ "elementId", "compound": bool }`. Forma de rolls: GDD §8.10–8.13 |

#### `equipment`

| Columna | Tipo | Notas |
| --- | --- | --- |
| character_id | uuid | |
| slot | text | ids en `docs/GDD.md` §8: `head`, `armor`, armas, `orb_1`–`orb_3`, `ring_1`, `ring_2`, `necklace` |
| item_instance_id | uuid | unique |

#### `friendships`

| Columna | Tipo | Notas |
| --- | --- | --- |
| user_a | uuid | canonical menor |
| user_b | uuid | canonical mayor |
| status | text | pending / accepted / blocked |
| requested_by | uuid | |
| unique (user_a, user_b) | | |

#### `character_wallets` (economía de mundo; no Fase 1–4)

| Columna | Tipo | Notas |
| --- | --- | --- |
| character_id | uuid | FK |
| currency_id | text | canónico: **`hesedias` solamente** |
| amount | numeric | nunca negativo; fail closed |
| unique (character_id, currency_id) | | |

NC **no** vive aquí. Premium es de **cuenta**, no de personaje (sobrevive a borrar/cambiar character).

#### `user_wallets` (Nexum Coin; gancho desde A, uso en tienda/producción)

| Columna | Tipo | Notas |
| --- | --- | --- |
| user_id | uuid | FK a `users` |
| currency_id | text | canónico: **`nexum_coin`** |
| amount | numeric | nunca negativo; fail closed |
| unique (user_id, currency_id) | | |

#### `commerce_orders` (pagos; no PSP en Fases 1–4)

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | |
| user_id | uuid | |
| sku_id | text | catálogo versionado |
| status | text | `pending` / `paid` / `fulfilled` / `failed` / `refunded` |
| provider | text | psp (`stripe`, etc.) o `manual` / `patron` / `crowdfunding` |
| provider_ref | text unique null | id del PSP; **idempotencia** del webhook |
| amount_fiat | numeric | |
| fiat_code | text | p. ej. `USD` |
| created_at | timestamptz | |

El webhook **no** acredita dos veces el mismo `provider_ref`. Fail closed si la firma no valida.

#### `entitlements` (beta, suscripción, grants ops)

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | |
| user_id | uuid | |
| entitlement_id | text | de catálogo / ops (p. ej. `beta.patron.unique`) |
| source | text | `patron` / `crowdfunding` / `subscription` / `manual` |
| payload | jsonb | personalización; no lógica en el tick |
| expires_at | timestamptz null | suscripciones; null = permanente |
| unique (user_id, entitlement_id) donde aplique | | |

#### `market_listings` (mercado offline; no Fase 1–4)

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | |
| seller_character_id | uuid | |
| item_instance_id | uuid unique | el ítem **no** está en inventario del vendedor mientras la listing esté abierta |
| currency_id | text | **solo** `hesedias` (nunca `nexum_coin`) |
| price | numeric | |
| created_at | timestamptz | |

Al crear la listing: transferir posesión a escrow (p. ej. `owner_character_id` null + listing activa, o owner sintético de mercado). Al comprar: cobro atómico wallet comprador → vendedor, instancia → comprador, listing cerrada. Al cancelar: instancia vuelve al vendedor. Todo en Workers/Neon; Godot no participa.

Equipar exige (cuando existan) flags/skills de **portar nivel** en el personaje (GDD §8.11). Eso es Character/Progression + chequeo de Inventory, no el tick.

#### `match_records`

| Columna | Tipo | Notas |
| --- | --- | --- |
| id | uuid PK | = match_uuid |
| kind | text | pvp_duel, pvp_arena, pve_dungeon |
| map_id | text | |
| started_at, ended_at | timestamptz | |
| winner_side | text null | |
| ruleset_id | text | |

#### `match_participants`

| Columna | Tipo | Notas |
| --- | --- | --- |
| match_id | uuid | |
| character_id | uuid | |
| user_id | uuid | |
| side | text | |
| result | text | win/loss/draw/leave |
| stats | jsonb | daño, kills; **nunca** práctica de lobby |

#### Índices mínimos

- `users(email)`, `users(display_name)`
- `friendships(user_a)`, `friendships(user_b)`
- `match_records(started_at desc)`
- `match_participants(character_id, match_id)`
- `market_listings(created_at desc)` cuando exista la tabla
- `character_wallets(character_id)` cuando exista la tabla
- `user_wallets(user_id)` cuando exista la tabla
- `commerce_orders(user_id)`, `commerce_orders(provider_ref)` unique cuando exista
- `entitlements(user_id)` cuando exista la tabla

### 9.2 MongoDB — chat y efímeros

#### `chat_global`

```json
{
  "_id": "ObjectId",
  "channel": "global",
  "senderId": "uuid",
  "senderName": "string",
  "body": "string",
  "createdAt": "ISODate"
}
```

TTL opcional (ej. 30 días) + índice `{ createdAt: -1 }`.

#### `chat_private`

```json
{
  "threadKey": "minUser:maxUser",
  "senderId": "uuid",
  "recipientId": "uuid",
  "body": "string",
  "delivered": false,
  "createdAt": "ISODate"
}
```

Mensajes offline: `delivered: false` + índice `{ recipientId, delivered }`.

#### `lobby_events`

Eventos temporales de debug/auditoría ligera (join, leave, dummy_kill). TTL corto (24–72 h). **No** es analytics de producción a largo plazo.

#### Colecciones futuras

- `chat_party`, `chat_zone` (MMO)
- `moderation_flags`

### 9.3 Qué no va a la DB

- Posiciones de tick, cooldowns vivos, hitboxes → solo memoria del proceso Godot.
- Cola de matchmaking Fase 1–3 → memoria del proceso realtime en EC2.
- Lista de live matches → memoria del orquestador en EC2, proyectada a REST del realtime.

---

## 10. API REST

Prefijo `/api`. Auth Bearer salvo register/login.

**Dónde se sirve:** 10.1–10.3, **10.8 (comercio)** y persistencia de `match_records` → **Workers**. 10.4–10.6 (colas, live, join tokens, heartbeat Godot) → **HTTP del proceso realtime en EC2** (mismo proceso que el WS). El cliente tiene `SERVICES_URL` (Workers) y `REALTIME_URL` (EC2). No son microservicios: mismo repo, dos `app`.

### 10.1 Auth (`identity`)

| Método | Ruta | Auth | Descripción |
| --- | --- | --- | --- |
| POST | `/api/auth/register` | no | Crea user + character vacío de tutorial |
| POST | `/api/auth/login` | no | Access + refresh |
| POST | `/api/auth/refresh` | refresh | Rota tokens |
| POST | `/api/auth/logout` | sí | Revoca refresh |
| GET | `/api/me` | sí | Perfil + character resumen |

### 10.2 Character & inventory (Fase 1 parcial, se completa en base)

| Método | Ruta | Descripción |
| --- | --- | --- |
| GET | `/api/characters/me` | Character + `kingdom_id` + `clan_id` + caps de vitals + appearance |
| PATCH | `/api/characters/me/appearance` | Cosméticos |
| GET | `/api/inventory` | Items |
| POST | `/api/inventory/equip` | Equipa slot (rechaza si `in_session_combat`; más adelante: requisito de portar nivel) |
| POST | `/api/inventory/craft/orb` | Fusión / descraft de orbes (§8.7 GDD). No Fase 1–4 |
| POST | `/api/inventory/craft/attributes` | Rolls de atributo, strip, catalizadores (§8.12–8.14). No Fase 1–4 |
| POST | `/api/inventory/craft/upgrade` | Subida de nivel de ítem + runas (§8.11). No Fase 1–4 |
| GET | `/api/market` | Listings activas |
| POST | `/api/market/list` | Escrow: el ítem sale del inventario |
| POST | `/api/market/buy` | Compra offline |
| POST | `/api/market/cancel` | Devuelve el ítem al vendedor |

Durante una batalla PvP, equipar está **bloqueado** (flag de presencia). En lobby, permitido. Todo crafting y el mercado son Hono/REST, no ENet. Uso de orbes de sellado/teleport en mundo es simulación (cuando exista ruleset de mundo); la **posesión** sigue siendo Inventory.

### 10.3 Social

| Método | Ruta | Descripción |
| --- | --- | --- |
| GET | `/api/friends` | Lista |
| POST | `/api/friends/request` | Solicitud |
| POST | `/api/friends/accept` | Aceptar |
| DELETE | `/api/friends/:userId` | Eliminar o bloquear según query |

### 10.4 Matchmaking

| Método | Ruta | Descripción |
| --- | --- | --- |
| POST | `/api/match/quick` | Encola (body: `{ mode: "1v1" \| "5v5" }`). Retorna `{ status: "queued", ticketId }` |
| DELETE | `/api/match/quick` | Sale de cola |
| POST | `/api/match/private/create` | Crea sala privada. Retorna `{ roomId, code }` |
| POST | `/api/match/private/join` | Body `{ code }` |
| POST | `/api/match/invite` | Body `{ userId, roomId? }`. Dispara WS `match:invite_received` |
| POST | `/api/match/ready` | Ready-check de sala/cola |
| POST | `/api/match/cancel` | Cancela sala o ready |

### 10.5 Directory / espectador

| Método | Ruta | Descripción |
| --- | --- | --- |
| GET | `/api/matches/live` | Partidas en curso (id, kind, map, players, spectatorCount, endpoint no sensible) |
| GET | `/api/matches/active` | Alias de live (el brief original); **unificar a `/live`** y dejar redirect |
| POST | `/api/sessions/lobby/join-token` | Token para el lobby |
| POST | `/api/matches/:id/spectate-token` | Token spectator |

El cliente **nunca** inventa IP/puerto. Siempre los recibe de API o de `match:ready`.

### 10.6 Interno (Godot → realtime en EC2; realtime → Workers)

En **EC2** (Godot y orquestador), protegido por `INTERNAL_SERVICE_KEY`:

| Método | Ruta | Descripción |
| --- | --- | --- |
| POST | `/internal/sessions/:id/heartbeat` | Salud + playerCount |
| POST | `/internal/join-tokens/verify` | Opcional si no se usa HMAC local |

En **Workers**, el proceso realtime reporta el resultado para Neon:

| Método | Ruta | Descripción |
| --- | --- | --- |
| POST | `/internal/sessions/:id/ended` | Escribe `match_records` vía Hyperdrive |

### 10.7 Errores estándar

`400` validación, `401` token, `403` prohibido, `404`, `409` conflicto (ya en cola, ya amigo, pedido duplicado), `429` rate limit, `503` sin capacidad de battle nodes.

### 10.8 Commerce (Workers; tienda no Fase 1–4)

Auth Bearer salvo el webhook del PSP (firma del proveedor, no JWT del jugador).

| Método | Ruta | Auth | Descripción |
| --- | --- | --- | --- |
| GET | `/api/store/catalog` | sí | SKUs visibles (NC packs, ítems, suscripción). Vacío hasta producción/beta |
| POST | `/api/store/checkout` | sí | Crea `commerce_orders` pending y sesión en el PSP. **No** acredita NC aquí |
| GET | `/api/store/wallets` | sí | Saldo NC (cuenta) + Hesedias del character activo |
| POST | `/api/store/redeem` | sí | Gasta NC en un SKU no-fiat (ítem, Hesedias, puntos). Atómico |
| POST | `/api/webhooks/payments` | firma PSP | Idempotente: `paid` → cumplir pedido. Fail closed si firma inválida |
| POST | `/internal/commerce/grant` | service key | Grant ops / Patreon / crowdfunding → entitlement o NC. Nunca el cliente |

El cliente Godot **consulta** catálogo y saldos por REST. El tick **no** lee pedidos.

---

## 11. WebSocket (chat y señalización)

URL: `wss://<REALTIME_HOST>/ws` (**EC2**, no Workers). Primer mensaje o query: access token emitido por Workers.

### 11.1 Eventos

| type | Dirección | Payload | Notas |
| --- | --- | --- | --- |
| `chat:send` | C→S | `{ channel, body, toUserId? }` | channel: `global` \| `whisper` \| `system` (system solo servidor) |
| `chat:receive` | S→C | `{ senderId, senderName, channel, body, createdAt }` | |
| `chat:offline_delivered` | S→C | `{ messages: [...] }` | Al conectar |
| `match:invite_received` | S→C | `{ fromUserId, fromName, roomId, mode }` | Pop-up Godot |
| `match:invite_result` | S→C | `{ accepted, roomId }` | Al invitador |
| `match:queued` | S→C | `{ ticketId, mode }` | |
| `match:ready` | S→C | `{ matchId, ip, port, joinToken, mapId, kind, role }` | **Transferencia al battle node** |
| `match:cancelled` | S→C | `{ reason }` | |
| `presence:update` | S→C | `{ userId, state }` | `online` \| `lobby` \| `queued` \| `in_match` |
| `system:notice` | S→C | `{ body, severity }` | Mantenimiento, kick |

### 11.2 Rooms WS (Hono)

- `global`
- `user:{userId}` (unicast: invites, whispers, match:ready)
- `matchRoom:{roomId}` (chat de sala privada / party futura)

El tráfico de chat **no** entra al ENet del lobby. El nametag sobre el personaje puede pedir el último mensaje vía cliente (UI local), no vía servidor de juego.

### 11.3 Rate limits

- Chat global: N mensajes / 5 s
- Invites: N / minuto
- Conexión WS única por user (nueva conexión mata la anterior)

---

## 12. Red de juego (Godot)

### 12.1 Roles de conexión

```
Handshake ENet
  join_token
  character_snapshot_version
  role: player | spectator
```

Servidor:

1. Verifica token (HMAC con `JOIN_SECRET` de la sesión en EC2).
2. Si `player` y la sesión está llena / ya empezó en modo lock → reject.
3. Si `spectator` → suscribir a stream de snapshots, **ignorar** `InputCommand`.
4. Spawnea o asocia Entity.

### 12.2 Paquetes de simulación (v1)

Cliente → Servidor:

- `InputCommand` `{ tick, dt, moveX, moveY, buttons, aimX, aimY, skillSlot? }`
- `Ping`

Servidor → Cliente:

- `Welcome` `{ entityId, tick, rulesetId, mapId }`
- `Snapshot` `{ tick, entities: [{ id, x, y, vx, vy, hp, flags, anim }] }` (delta si es posible)
- `Event` `{ kind: spawn|despawn|hit|die|cc|skill_fx, ... }`
- `Reconcile` `{ ackTick, x, y }` al dueño
- `MatchState` `{ phase: waiting|countdown|active|ended, scores }`
- `Pong`

Espectador recibe Snapshot + Event + MatchState. No Reconcile.

### 12.3 Predicción

- Solo la entidad poseída.
- Servidor reconcilia si `|pos_client - pos_server| > epsilon`.
- Habilidades: el cliente reproduce animación al pulsar; el hit real llega por `Event.hit`. Si el servidor rechaza (cooldown, CC, out of range), el cliente cancela VFX (rollback de skill, no de todo el mundo).

### 12.4 Anti-trampas básico (autoridad)

- Ignorar velocidades enviadas por el cliente; el servidor integra velocidad desde input + stats.
- Clamp de magnitud de input a 1.
- Rechazar skills si cooldown/CC/recurso no cuadra.
- Kick por desync extremo o flood de paquetes.

---

## 13. Núcleo de simulación y combate

Este es el corazón que permite lobby → PvP → mazmorras → MMO **sin reescribir**.

### 13.1 Entity

```
EntityId: uint32
Entity:
  id
  kind: player | dummy | monster | boss | npc | projectile | summon
  team / faction
  transform (x, y, facing)
  movement (speed, controller: input | ai | none)
  combat (mana, stamina, vitality, pct_mods por tag, modifiers, guard)
  skills (loadout)
  status_effects[] (CC, DoT, buffs; p. ej. stun de rotura de guardia)
  flags (invulnerable, spectating, practice)
```

`guard`: barra, si está alta, si está rota (no se puede levantar hasta recargar). Números: `docs/GDD.md` §9.6 y `nexum-terra/data/melee.json`. Choque arma-vs-arma: empuje + onda de choque en resolución de hits, no skill de hotbar.

`PlayerID` / `user_id` es un **componente opcional** `IdentityLink`, no la clave del loop.

### 13.2 Tick

```
for each tick:
  1. recoger intents (inputs de players, decisiones de AI)
  2. aplicar movimiento (colisión mapa)
  3. resolver skills (cast, proyectiles)
  4. hit detection (shapes); choque melee vs melee si aplica (`docs/GDD.md` §9.6)
  5. aplicar daño (canales maná/stamina/vitalidad y overflow; `docs/GDD.md` §6), guardia y CC
  6. KO (vitalidad 0) / respawn según ruleset
  7. emitir eventos y snapshot
```

El mismo loop corre en lobby y en batalla. El ruleset apaga scoring, AI o persistencia.

### 13.3 Rulesets

| Ruleset | Scoring | AI | Persistencia | Friendly fire | Respawn |
| --- | --- | --- | --- | --- | --- |
| `practice_lobby` | no | dummies idle | no | no | inmediato |
| `pvp_duel` | sí | no | match_records | no | según modo |
| `pvp_arena` | sí | no | match_records | no | round o none |
| `pve_dungeon` | objetivos | sí | match + drops | configurable | checkpoints |
| `open_world` | **reservado Etapa C** | — | — | — | — |

`open_world` no se implementa ni se registra en data hasta el ADR de §18.1. Cuando exista: sin scoring de arena; AI sí; persistencia de posición + drops; PvP según color de zona (GDD §15); respawn cementerio/ciudad.

Añadir un modo nuevo = **nuevo ruleset + mapa + cola**, no un fork del motor. Un mapa de **región** no es un modo nuevo de rooms: es Etapa C.

### 13.4 Habilidades e items

Definidos en `game/data` (JSON/tres):

- `skill_id`, cooldown, costos, hitbox, `base` estático, `damage_channel`, tags de %, CC aplicado
- `item_def_id`, mods de caps (maná/stamina) y `%` por tag de daño; rolls de instancia (calidad, nivel, atributos) **ya resueltos** en el snapshot
- orbes: `element_id`; restos de campo y reacciones: tablas en `nexum-terra/data/elements.json` (GDD §8.9), no lógica en el nodo del jugador
- brillo legendario: solo cliente (shader); el servidor no simula el glow

El servidor carga los mismos archivos que el cliente (o un subset). El cliente no “inventa” números de daño.

### 13.5 Zona de práctica

Dummies = `kind: dummy`, `controller: none`, vitals altos o infinitos según data. Hits válidos para feedback (números, animación). **Cero** escritura a `match_records` y a progression.

### 13.6 IA (Fase 5, gancho ahora)

Puerto `AiController.decide(world_view) -> Intent`. El tick no distingue si el Intent vino de ENet o de AI. En lobby, dummies no registran AI o usan `IdleAi`.

### 13.7 Espectador

No es una entidad simulada (o es entidad `flags.spectating` sin colliders). Cámara libre o follow. Sin `IdentityLink` de combate.

---

## 14. Orquestación de sesiones

### 14.1 Lobby

- Proceso de larga vida **en EC2**.
- Puerto ENet fijo conocido por el realtime (`LOBBY_HOST` = host del EC2, `LOBBY_PORT`).
- Si un día hay 200 jugadores: **Lobby shards** (`lobby_0`, `lobby_1`) con el mismo mapa. El matchmaker social no cambia; Presence lleva `session_id`.

### 14.2 Battle nodes

Cuando Matchmaking emite `MatchFormed`:

1. Generar `match_uuid`.
2. Reservar puerto libre en el host (allocator).
3. Spawn:  
   `godot --headless --path nexum-terra --dedicated --session-kind pvp_duel --map maps/pvp/arena_01 --port 7778 --match-uuid ... --join-secret ...`
4. Health: TCP/ENet accept o heartbeat HTTP interno.
5. Timeout de boot (ej. 15 s) → fail match, devolver jugadores a cola.
6. WS `match:ready` a cada participante.
7. Al `ended` o timeout → realtime persiste en Neon vía Workers `/internal/sessions/:id/ended` → SIGTERM → liberar puerto.

Adaptador `SessionSpawner`:

- Fase 3: `child_process` / Docker **en el EC2** que ya corre el realtime y el lobby.
- Más nodos EC2 de batalla solo cuando una máquina no baste (sigue siendo el mismo módulo Orchestration, no un microservicio nuevo).

El dominio de Orchestration solo habla `spawn(spec) -> endpoint`.

### 14.3 Capacidad

Variable `MAX_BATTLE_NODES`. Si no hay puerto/CPU, REST `503` y WS `system:notice`.

### 14.4 Transferencia cliente

El cliente trata `match:ready` como **cambio de sesión de simulación**, no como “cerrar el juego”:

- WS Hono en EC2 permanece
- ENet lobby disconnect
- Load mapa
- ENet battle connect
- Al acabar: inverso

---

## 15. Cliente Godot

### 15.1 Responsabilidades

- Login/register UI
- Chat flotante (WS)
- Nametags
- Movimiento 8 direcciones, **multientrada** (teclado, ratón, gamepad, táctil virtual)
- Predicción
- HUD de cola, invites, lista de live matches
- Cámara top-down
- No autoridad de hit

### 15.2 Input

Capa `adapters/input` normaliza a `Intent { move: Vector2, aim: Vector2, skills: bitfield }`. El core no sabe si vino de un stick o de un joystick táctil.

### 15.3 Escenas

- `Boot` → auth
- `Lobby` → práctica + UI social
- `Battle` → combate / espectador (misma escena, flag)
- `World` → **Etapa C solamente.** Misma tubería de sesión, un mapa por región. No se crea la escena “para ir adelantando”.

### 15.4 Top-down 2D

Y-sort, colliders en mapa, aim hacia el cursor o stick derecho. El servidor usa la misma geometría de colisión exportada (TileMap/Physics layers compartidas).

---

## 16. Cómo añadir features sin colisionar

Checklist obligatorio antes de mergear una feature:

1. **¿A qué bounded context pertenece?** Si a ninguno, se crea módulo nuevo con puertos. No se añade “un endpoint suelto”.
2. **¿Es simulación o metajuego?** Simulación → `game/core`. Metajuego (amigos, cola, chat) → `backend/modules`. Nunca chat en ENet ni hits en REST.
3. **¿Hay datos nuevos?** Catálogo (`data/`) vs estado de jugador (Postgres) vs efímero (Mongo/memoria). Elegir uno.
4. **¿El combate necesita conocerlo en tick?** Si sí: componente de Entity + ruleset. Si no: snapshot al inicio de sesión.
5. **¿Rompe contratos?** Actualizar `contracts/` en el mismo cambio.
6. **¿El lobby se comporta distinto que PvP?** Eso es ruleset, no `if is_lobby` dentro de `apply_damage` salvo flags explícitos (`practice`).
7. **Pruebas del dominio** sin Godot y sin Hono (funciones puras / casos de uso con repos fake).
8. **¿Está `docs/modules/<modulo>.md` al día?** Si el módulo no existía, se crea desde `_TEMPLATE.md`. Si cambió negocio, técnica o gameplay, se actualiza el mismo día. Sin esto no se mergea.

Patrones de extensión previstos:

| Quiero añadir… | Dónde |
| --- | --- |
| Una skill | `game/data/skills/*.json` + VFX en cliente |
| Un mapa PvP | `game/maps/pvp` + registro en matchmaking modes |
| Un modo 2v2 | Queue nueva + ruleset (puede reutilizar arena) |
| Guilds | Nuevo contexto Social/Guilds, canal chat, Postgres |
| Crafting (orbes, atributos, nivel de ítem) | Inventory domain (REST) |
| Mercado de jugadores / Hesedias | Inventory & Economy (`market_listings`, `character_wallets`) — **Etapa B**, no C |
| Nexum Coin / IAP / Patreon / suscripción | Commerce (`user_wallets`, `commerce_orders`, `entitlements`) — ganchos en A; PSP en producción |
| World boss / zonas / conquista / regiones | **Etapa C.** No se engancha en Fases 1–5 |
| Mobile stick | Solo `adapters/input` |
| Redis matchmaking | Nuevo adapter de `MatchmakingStore` |

---

## 17. Fases de construcción

Cada fase cierra un corte jugable. No se adelanta PvE a Fases 1–4. **No se adelanta open world a ninguna fase de las Etapas A o B.** Sí se dejan ganchos de núcleo (`EntityID`, `SessionKind` listado, `Ruleset`, `party_id` opcional, **monedas duales y contexto Commerce**), no sistemas de mundo ni un PSP en el lobby.

### Fase 0 — Andamiaje (corta, antes de Fase 1)

- Monorepo, `docker-compose` (Postgres Neon-compatible local o Neon dev + Mongo), `contracts/` vacío versionado, proyecto Godot 4.7.2, export headless verificado en CI local.
- **Hecho cuando:** `godot --headless --quit` y `hono` healthcheck responden.

### Fase 1 — Cimientos del backend y bases de datos

Objetivo: servicios externos al motor gráfico.

- Migraciones Postgres: `users`, `refresh_tokens`, `characters`, `character_stats`, `friendships`, `match_records`, `match_participants` (tablas de match vacías de uso, pero creadas).
- Mongo: colecciones `chat_global`, `chat_private`, `lobby_events` + índices + TTL.
- Hono **Workers** + Neon/Hyperdrive: `/api/auth/register`, `/api/auth/login`, `/api/auth/refresh`, `/api/me`.
- Hono **realtime en EC2** + Mongo: WS `chat:send` / `chat:receive` global y whisper, mensajes offline.
- Módulos hexagonales Identity + Chat (+ Character lectura).
- Tests de dominio de auth y de chat (repos in-memory).

**Fuera de alcance:** Godot, colas, spawn de procesos.

**Hecho cuando:** se puede registrar, loguear, obtener perfil y chatear con dos clientes WS (p.ej. script o cliente de prueba) sin Godot.

### Fase 2 — Cliente y servidor del lobby (Godot)

Objetivo: hub interactivo.

- Movimiento 8 direcciones, multientrada.
- `core/` Entity + Movement + Ruleset `practice_lobby`.
- Dedicated lobby headless: recibe inputs, valida, broadcast snapshots.
- Cliente: predicción, interpolación de otros, cámara top-down.
- UI: login REST, chat flotante WS, nametag.
- Join token de lobby.
- Dummies: daño autoritativo **sin** persistir stats.
- Presencia `lobby`.

**Hecho cuando:** dos clientes se ven moverse, chatean, y ambos ven el dummy recibir el mismo daño (servidor manda el hit).

### Fase 3 — Matchmaking e instanciación

Objetivo: puente lobby → combate.

- Colas in-memory `1v1` y `5v5` (5v5 puede mockearse con bots de relleno **solo si** está flaggeado; preferible exigir N reales o alinear el modo).
- Salas privadas + invite WS.
- Orchestrator: `SessionSpawner` local, allocator de puertos, heartbeat, teardown.
- Flujo `match:ready` → cliente cambia de mapa y de puerto.
- API `POST /api/match/*`.
- Al volver: reconexión al lobby.

**Hecho cuando:** dos jugadores en lobby pulsan Quick Match 1v1, se levanta un headless, ambos cargan el mapa PvP (aunque el combate aún sea movimiento + placeholder).

### Fase 4 — Core de batalla y espectador

Objetivo: combate cerrado autoritativo.

- Skills, cooldowns, hitboxes, CC, muerte, win condition.
- Reporte del battle node al realtime EC2; persistencia Neon vía `POST` interno a Workers `/internal/sessions/:id/ended`.
- `GET /api/matches/live`.
- Spectator: join token `role=spectator`, snapshots, inputs ignorados.
- HUD de combate en cliente.

**Hecho cuando:** un 1v1 tiene ganador persistido, y un tercer cliente observa sin poder mover al luchador.

### Fase 5 — Etapa B: mazmorras y metajuego (después de rooms jugados)

Objetivo: reutilizar el combate de rooms; **no** abrir el overworld.

Esta fase es un **paquete de producto**, no un único sprint. Arranca cuando el juego de solo rooms (Fase 4) se ha podido **jugar de verdad**. Incluye, en el orden que el equipo cierre al entrar en B (pueden ser 5.1, 5.2, …):

- AI controller sobre Entity (monstruos/jefes).
- Party: reutilizar salas privadas (2–5) → `SessionKind.pve_dungeon`. Verdes son **1** jugador (`docs/GDD.md` §16.2).
- Mapas de **mazmorra** (instancias), objetivos, drops vía evento a Inventory. Entrada **desde el lobby**, no desde un overworld.
- Tipos verde / azul / roja y recompensas: GDD §16. **Fuera de Fase 5:** spawn periódico en zonas, carrera al portal, entrada que desaparece del mapa.
- Chat de party.
- Metajuego que el GDD ya describe y que **no** es mundo: build (calidad/nivel/crafting/orbes cuando toque), inventario persistente, mercado de **Hesedias**, skills de clan, profesiones, puntos/rebirth según el recorte que se elija al abrir B.
- Comercio: cumplir SKUs y wallets duales cuando el recorte de B lo pida; **PSP de tarjetas** cuando haya producción (puede ser posterior al primer meta jugable). Beta/Patreon puede grant-ear antes, vía ops.

**Fuera de Fase 5:** cualquier mapa de región, zona de peligro, loot de cadáver de overworld, estatua de conquista, chat por zona de mundo, persistencia `world_x/world_y`, AOI. **Fuera de Fases 1–4:** checkout de tarjeta, catálogo IAP cobrable, tienda en el HUD de combate.

**Hecho cuando (mínimo de mazmorra):** un party de 2 entra a una mazmorra, mata un dummy-AI con el mismo sistema de daño que el PvP, y recibe un item al terminar. El “hecho” de mercado/clanes se define al partir 5.x; no se usa como excusa para abrir Etapa C.

### Fuera de las Etapas A y B (congelado)

Ver sección 18. Las fases 6–8 **no son backlog activo**. El juego es publicable al cerrar la Fase 4. La Etapa B es el siguiente producto. El open world **espera una decisión explícita** tras meses de B.

---

## 18. Camino a open world MMORPG (Etapa C, congelada)

El open world **no** es un lobby más grande con 10.000 CharacterBody2D replicados a todos. Es el mismo núcleo con **regiones separadas**, **interés espacial (AOI)** y **persistencia de mundo**. El diseño de peligro, loot y conquista está en `docs/GDD.md` §15. Esta sección solo dice **cómo se enchufa** y **cuándo está prohibido tocarlo**.

### 18.1 Candado

| Regla | Detalle |
| --- | --- |
| Producto que se entrega primero | Etapa A (Fases 1–4): lobby + rooms PvP + espectador |
| Siguiente producto | Etapa B (Fase 5.x): mazmorras **y** metajuego (build, inventario, mercado, clanes, …) |
| Open world | **No** es “si hay margen”. **No** es un epic en el mismo tablero que rooms |
| Desbloqueo | Una **decisión explícita**: ADR nuevo en §22 (o enmienda de ADR-010) que diga “se abre Etapa C”. Hasta entonces, PRs de mundo se rechazan |
| Tiempo | Se espera **meses** de Etapa B jugable, no días |
| Qué sí se puede ahora | Escribir/actualizar GDD §15–§16 y este §18. Reservar nombres (`world_shard`) |
| Qué no | Mapas de región, TileMaps de overworld, `OpenWorld` ruleset, AOI, `world_x/world_y`, zonas en data, cadáveres de mundo, estatuas, chat `zone`, UI de conquista, “prototipo de campo detrás del lobby” |

Economía persistente (Hesedias, crafting, mercado) es **Etapa B**, no C. Nexum Coin y el procesador de pagos son **Commerce** (§4.14, §29): modelo desde A, cobro automático en producción. En C se reutiliza; no se espera al overworld para tener inventario ni tienda.

### 18.2 Regiones (optimización y gameplay)

El overworld **no** es un único mapa continuo cargado entero.

- El continente se parte en **varias regiones**. Cada región es (al menos) un `world_shard` con su mapa, su cap de jugadores y su proceso Godot.
- **Optimización:** un shard no simula ni replica la otra región. AOI recorta aún más *dentro* de la región.
- **Gameplay:** viajar entre regiones es un cambio de sesión (mismo handshake que lobby → arena). Drops, precio de territorio, densidad de recursos y facciones pueden diferir por región. Las reglas de **color de zona** (GDD §15) aplican *dentro* de cada región; Aurora es la única celeste.
- Transferencia entre regiones = Fase 8 de esta etapa, no un portal improvisado en B.

### 18.3 Norte técnico (solo válido tras el ADR de desbloqueo)

#### Fase 6 — Shard de una región mínima

- `SessionKind.world_shard` **implementado** (hoy solo el nombre)
- Un mapa de **una** región, cap de jugadores por shard
- Persistir posición de mundo en character (`region_id`, `world_x`, `world_y`)
- AOI: cada cliente solo recibe snapshots de entidades cercanas
- Ruleset `open_world`: peligro de zona, loot de cadáver según color (GDD §15), sin scoring de arena
- Instancias de mazmorra/PvP siguen existiendo: el jugador **sale del shard** hacia un battle node (igual que sale del lobby). Portales de mazmorra **spawnean en zonas** según tipo y ciclo (GDD §16.3); no se simula el interior en el shard.

#### Fase 7 — Contenido de mundo (no el metajuego de B)

- NPCs de overworld, quests (nuevo contexto `Quest`), spawns de mobs de mundo
- Canales de chat por zona/región
- Conquista de territorios (estatuas de poder, GDD §15.3)
- World bosses
- Reutilizar economía de Etapa B (impuestos de zona → tesorería de reino)

#### Fase 8 — Escala

- N regiones, N shards, transferencias entre regiones
- Extraer Chat, Matchmaking u Orchestration a procesos **solo si una réplica ya no basta** (no antes)
- Redis obligatorio para presencia y colas
- Allocator de battle nodes en cluster
- Interest management más agresivo, possibly binary protocol

**Invariante:** el cliente siempre hace `join_token → connect endpoint → simulate`. Da igual que el endpoint sea lobby, arena, mazmorra o shard `region_aurora_1`.

---

## 19. Seguridad

- Passwords: argon2id. Nunca logs de tokens en claro.
- JWT corto; refresh rotativo.
- Join tokens de un uso / TTL, atados a session y role.
- Internal API aislada (red privada / key).
- Rate limit REST y WS.
- Validación de input en dominio, no solo en el router.
- El cliente no es de confianza: ni posición, ni daño, ni inventario, **ni “ya pagué”**.
- Secretos en env, nunca en `contracts/` ni en el repo Godot exportado al jugador (el `JOIN_SECRET` vive en el servidor dedicado, no en el cliente; el cliente solo lleva su join token). Secretos del **PSP** igual: solo Workers.
- Webhooks de pago: verificar firma; idempotencia por `provider_ref`; no cumplir pedidos `pending` desde el cliente.
- Sanitizar chat (longitud, control chars). Moderación básica (mute) como puerto futuro.
- Spectator no recibe join tokens de player.
- **No cash-out:** Hesedias y NC no se convierten a dinero real. El mercado de jugadores no lista NC.

---

## 20. Observabilidad y operaciones

- Health: `GET /api/health` en Workers (Neon/Hyperdrive) y en el realtime EC2 (Mongo + Godot).
- Logs estructurados JSON en API (`request_id`).
- Heartbeat de cada sesión Godot (playerCount, tick_age).
- Métricas mínimas: colas depth, nodes vivos, ws connections, match start/fail.
- `lobby_events` en Mongo es telemetría corta, no APM.
- Apagado de battle nodes huérfanos (watchdog: sin heartbeat N segundos → kill + cancel match).

Entornos: `local` | `dev` | `prod`. Workers → Neon/Hyperdrive. EC2 → Mongo, path de Godot, `MAX_BATTLE_NODES`.

---

## 21. Convenciones

### Backend

- TypeScript strict.
- Un caso de uso = una función/clase por archivo, testeable.
- Nada de SQL en rutas Hono.
- Errores de dominio tipados; el adaptador HTTP los mapea a status.

### Godot

- `core/` sin `extends Node`, sin `Engine`, sin `MultiplayerAPI`.
- Nombres de paquetes idénticos a `contracts/game`.
- GDScript (salvo que se decida C# globalmente; no mezclar sin ADR).
- Escenas no contienen reglas de daño.

### Git / trabajo

- Features por dominio: `feat(identity): ...`, `feat(simulation): ...`.
- `PLAN.md` se actualiza en el mismo PR si cambia un contrato o un SessionKind.
- `docs/modules/*.md` se actualiza en el mismo PR que el código del módulo (negocio, técnica o gameplay si aplica). GDD si cambia balance/clases/items **o el diseño de monedas/tienda**.
- No commits de `.env`.

### Idioma

- Código (símbolos, rutas, contratos): **inglés**.
- Este plan y docs de diseño: **español**.
- Texto de UI del juego: se decidirá (tabla de localización desde el primer string visible).

---

## 22. Decisiones técnicas (ADR)

### ADR-001 — Modular monolith hexagonal, no microservicios

Aceptado. Extraer después por costura de puertos.

### ADR-002 — Postgres para verdad relacional, Mongo para chat/efímeros

Aceptado. Chat no merece joins relacionales; inventario y amigos sí.

### ADR-003 — Simulación en Godot headless, no en Node

Aceptado. Hitboxes, TileMap y tick 2D ya están en el motor. Node orquesta, no simula.

### ADR-004 — EntityID y Ruleset desde Fase 2, no como refactor de Fase 5

Aceptado. Fase 5 es contenido + AI, no rediseño de IDs.

### ADR-005 — Chat y señalización fuera de ENet

Aceptado. Permite chat durante carga de mapa y escala independiente.

### ADR-006 — Matchmaking in-memory en el realtime de EC2

Aceptado. Redis solo cuando una réplica de ese proceso no baste. Nunca colas en Workers.

### ADR-007 — Un proyecto Godot, dos entrypoints (client / dedicated)

Aceptado.

### ADR-008 — JSON en red de juego hasta que el profiler lo impida

Aceptado. Cambiar a binario es adapter de protocolo, no cambio de dominio.

### ADR-009 — Mazmorras después del núcleo PvP

Aceptado por producto. Ganchos de Party/PVE existen en el modelo, no en el código de AI.

### ADR-010 — Open world = Etapa C explícita; regiones + AOI, no un lobby infinito

Aceptado. El overworld es `world_shard` por **región** + AOI, no un lobby sin cap. **Congelado** hasta un ADR posterior que abra la Etapa C. Ese ADR solo se escribe tras Etapa A jugada y **meses** de Etapa B (mazmorras + meta). Hasta entonces no hay código ni prototipo de mundo. Diseño de zonas: `docs/GDD.md` §15.

### ADR-011 — Workers (servicios + Neon/Hyperdrive) y EC2 (WS + Mongo + Godot)

Aceptado. Todo servicio REST con datos relacionales corre en Cloudflare Workers contra Neon vía Hyperdrive. El proceso de WebSockets (Mongo), el matchmaker, el orquestador y el Godot headless corren en EC2. Dos entrypoints del mismo monolith; no son microservicios. Ver sección 26.

### ADR-012 — Diseño de juego y marca fuera de PLAN.md

Aceptado. Builds, items y stats van en `docs/GDD.md`. Marca en `docs/brand/BRAND.md`. PLAN.md solo describe cómo esos datos entran al sistema.

### ADR-013 — Un documento vivo por módulo/feature

Aceptado. Onboarding y mantenimiento se apoyan en `docs/modules/`. Toda modificación de negocio, técnica o gameplay actualiza ese documento en el mismo cambio que el código.

### ADR-014 — Comercio en Workers; dos monedas; sin cash-out

Aceptado. Dinero real, Nexum Coin y entitlements son el contexto **Commerce** (Hono/Workers + Neon). Hesedias y el mercado de jugadores siguen en Inventory. El procesador concreto (Stripe u otro) es un **adaptador**; el dominio no lo importa. Godot no cobra. No hay conversión a fiat. El modelo dual se declara **antes** de la primera tabla de wallets.

---


## 23. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación |
| --- | --- | --- |
| Spawn lento de Godot | Matches fallidos | Warm pool futuro; timeout + requeue |
| Una réplica del realtime EC2 y cola in-memory | Colas rotas | Redis **después** de que una réplica no baste |
| Cliente y servidor desalinean data de skills | Hits “fantasma” | Mismos archivos `game/data`, versión en handshake |
| Lobby lleno | Mala UX | Shards de lobby con el mismo SessionKind |
| Lógica de daño en el nodo del jugador | Trampas y forks | core/ puro + review |
| Adelantar mazmorras | Base de rooms inestable | Fase 4 jugable antes de AI / Fase 5 |
| Colar open world (zonas, regiones, conquista) en A o B | Scope infinito, combate de rooms sin probar | Candado §18.1; rechazar PRs de mundo sin ADR de Etapa C |
| Puertos abiertos al mundo | Abuse | Join token + firewall + lista de puertos del allocator |
| Neon cold start | Login lento | Pooling (PgBouncer) / compute Neon acorde |
| Fraude / doble crédito de NC | Economía rota, cargos chargeback | Webhook firmado, `provider_ref` unique, fail closed, grants solo ops |
| Meter NC en el mercado de jugadores | RMT y lavado | `market_listings.currency_id` solo `hesedias` |

---

## 24. Definición de hecho (Definition of Done)

Una fase está cerrada solo si:

1. Cumple el “Hecho cuando” de su sección.
2. Los contratos usados están en `contracts/`.
3. El dominio nuevo tiene tests sin I/O real.
4. No se ha introducido `PlayerID` en el loop de daño.
5. Chat sigue fuera de ENet.
6. `PLAN.md` refleja cualquier desvío (ADR nuevo).
7. Flujo manual documentado en README de esa fase (comandos para levantar API, DB, Godot).
8. Cada módulo tocado tiene `docs/modules/<modulo>.md` actualizado (negocio + técnica + gameplay si aplica).

---

## 25. Mapa de documentos (no hay PLAN2)

No se crea `PLAN2.md`. Un segundo plan director se desincroniza y el agente no sabe cuál manda.

| Archivo | Qué decide | Qué no decide |
| --- | --- | --- |
| `PLAN.md` | Arquitectura, dominios, red, fases, deploy, ADRs, **modelo de cobro (Commerce)** | Números de balance de combate, lore, paleta, nombres de skills |
| `docs/GDD.md` | Fantasía, reinos, builds, stats, items, skills, progresión, controles, **mazmorras (§16)**, **zonas de overworld (§15, congelado)**, **monedas y tienda de diseño (§17)** | Dónde se despliega ni cómo se nombra un puerto |
| `docs/brand/BRAND.md` | Color, tipo, voz visual, Theme de Godot | Lógica de combate |
| `contracts/` | Forma de los payloads | Significado de diseño (“por qué el dash de Aerion es Kawarimi”) |
| `nexum-terra/data/` | Catálogo ejecutable (JSON/tres) | Debe **reflejar** el GDD, no contradecirlo |
| `docs/modules/*.md` | Cómo está hecho *este* módulo hoy: flujo, archivos, reglas, gameplay | Constitución global (eso es PLAN) ni paleta (BRAND) |

**Cómo añadir builds/items/stats:** se escribe (o se amplía) `docs/GDD.md`. Si eso implica tablas o componentes nuevos, se actualiza `PLAN.md` en el mismo cambio (p. ej. `character_stats` o un componente `Build`). Luego se implementa el catálogo en `nexum-terra/data/` y el persistido en Postgres. El “cómo corre en código” de inventario/personaje va en `docs/modules/inventory.md` y `docs/modules/character.md`.

Orden: GDD (reglas de diseño) → PLAN (si hay hueco de arquitectura) → código + `docs/modules/` en el mismo cambio.

---

## 26. Despliegue: Cloudflare, AWS y el cluster de juego

Hay **dos planos de producción**. Mezclarlos (p. ej. ENet por Workers, o Neon desde el tick de Godot) es la forma más cara de no poder jugar.

### 26.1 Qué va en cada plano

| Plano | Dónde | Carga |
| --- | --- | --- |
| **Servicios** | **Cloudflare Workers** + **Neon + Hyperdrive** | REST: auth, perfiles, inventario, amigos, historial de batallas, `/internal/sessions/:id/ended` |
| **Tiempo real + simulación** | **AWS EC2** | Proceso Hono WS → MongoDB (chat, offline, lobby events); matchmaker in-memory; orquestador; Godot headless (lobby y battle nodes, ENet/UDP) |

El cliente usa `SERVICES_URL` (Workers) y `REALTIME_URL` (EC2). Cloudflare **no** proxifica UDP de Godot.

**Regla:** el orquestador vive **en el mismo EC2** (o flota EC2) que los binarios Godot. Workers nunca arrancan un headless ni abren un puerto ENet.

### 26.2 Topología (esta, no un menú de opciones)

1. **Cloudflare Workers:** único sitio de servicios REST y de Postgres (Neon vía Hyperdrive). R2/Pages opcionales para descargas.
2. **Un EC2 (o el mínimo necesario):** proceso realtime (WS + Mongo + colas + spawn) **y** Godot headless. Localmente, Docker Compose simula ese EC2 + Workers wrangler.
3. Más instancias EC2 de batalla o una réplica del realtime **solo** cuando una réplica ya no aguante. Eso todavía no son microservicios de dominio.

No hay etapa intermedia “todo en un VPS con el monolito HTTP+WS juntos” como destino: el split Workers/EC2 es el diseño. En `local`, los dos entrypoints se levantan juntos.

### 26.3 Cloudflare: útil y límites

**Usar:** Hono REST, Neon/Hyperdrive, DNS, TLS, WAF, rate limit de `/api` de servicios, R2/Pages.

**No usar:** Godot headless, proceso WS+Mongo, matchmaker in-memory, `child_process` de battle nodes, ENet/UDP.

### 26.4 Microservicios: cuándo y dónde

Hasta que **una réplica** del Worker o del realtime EC2 no baste, **no hay microservicios**. Dos entrypoints por plataforma no cuentan como extraer Chat o Matchmaking.

Cuando una réplica no funcione: Redis en el realtime, más EC2 de Godot, o partir un dominio. Sigue sin ir Godot a Workers ni Mongo de chat a Hyperdrive.

---

## 27. Branding y UI

Fuente de verdad visual: **`docs/brand/BRAND.md`**.

El agente (y las personas) leen ese archivo **antes** de inventar colores, Theme de Godot, CSS de landing o iconos. Tokens en código (`theme.tres`, variables) se generan **desde** BRAND.md, no al revés.

Regla de Cursor: `.cursor/rules/brand.mdc`.

---

## 28. Documentación por módulo y feature

El PLAN describe el sistema entero. **No sustituye** la explicación de cada pieza. Un desarrollador nuevo debe poder abrir `docs/modules/<nombre>.md` y entender qué hace el módulo, por qué, cómo está cableado y qué gameplay implica, sin reconstruir eso desde el código.

### 28.1 Qué es un “módulo” aquí

Coincide con un bounded context o una feature entregable, no con un archivo suelto.

Ejemplos (un markdown cada uno): `identity`, `character`, `inventory`, `commerce`, `social`, `chat`, `matchmaking`, `orchestration`, `match-directory`, `progression`, `simulation` (core Entity/tick/combate), `lobby`, `spectator`, `client-input-ui`, `party` (Etapa B), `dungeons` (Etapa B; spawn de mapa en C), `world` (Etapa C; no se rellena como implementación hasta el ADR).

Si nace un feature que no entra en ninguno, se crea **módulo nuevo** (código + `docs/modules/<id>.md`) en el mismo cambio. No se documenta “un poco” dentro de otro módulo ajeno.

### 28.2 Qué debe contener cada documento

Plantilla obligatoria: `docs/modules/_TEMPLATE.md`. Mínimo:

1. **Propósito** — qué problema resuelve, para quién.
2. **Lógica de negocio** — reglas, invariantes, casos borde, qué está prohibido.
3. **Lógica técnica** — flujo (secuencia), puertos/adaptadores, archivos clave, contratos REST/WS/paquetes, persistencia, fallos y retries.
4. **Gameplay** — si el jugador lo percibe: feedback, timings, restricciones de sesión, qué no persiste (ej. daño a dummies).
5. **Cómo extenderlo** — el cambio típico siguiente sin romper vecinos.
6. **Historial** — fecha + resumen de cambios de comportamiento (no cada typo).

Idioma: español (como el resto de docs de diseño). Símbolos de código en inglés, iguales al repo.

### 28.3 Cuándo se escribe y se actualiza

| Evento | Documentación |
| --- | --- |
| Se crea el módulo | Se copia `_TEMPLATE.md` → `docs/modules/<id>.md` y se rellena con lo implementado (no un wish list vacío) |
| Cambia una regla de negocio | Misma PR/commit: sección Negocio |
| Cambia un flujo, puerto, tabla, paquete | Misma PR: sección Técnica + contratos si aplica |
| Cambia sensación de juego, skill, UI de esa feature | Misma PR: sección Gameplay y, si es balance/clase/item, también `docs/GDD.md` |
| Se depreca o se extrae a servicio | El doc lo dice al inicio (estado: deprecated / extraído) y apunta al sucesor |

**Prohibido:** dejar “luego lo documentamos”, README de 3 líneas que solo dice “auth module”, o copiar el PLAN entero dentro de cada módulo.

### 28.4 Relación con PLAN y GDD

- **PLAN:** por qué el sistema se parte así y las reglas entre módulos.
- **GDD:** qué es un guerrero, qué stats existen, qué hace una skill en diseño.
- **Module doc:** cómo *está implementado hoy* (archivos, orden de llamadas, qué valida el servidor).

Si PLAN y el module doc chocan, se corrige el que esté obsoleto en el mismo cambio. La implementación no queda como única fuente: el module doc debe poder seguirse para onboarding.

### 28.5 Índice

`docs/modules/README.md` lista todos los módulos, su estado (`planned` / `active` / `deprecated`) y el enlace al markdown. Cada módulo nuevo se añade a ese índice.

---

## 29. Modelo de negocio y comercio

El cobro **no espera al open world**. Si las wallets y los pedidos se inventan al ir a producción, Identity e Inventory se parchean. Por eso el contexto Commerce y las dos monedas están **declarados desde Etapa A**; la tienda cobrable y el PSP no.

Producto (qué se vende, tasas, pay-to-win): `docs/GDD.md` §17. Sistema (tablas, REST, fail-closed): §4.14, §9, §10.8, ADR-014.

### 29.1 Dos monedas (nunca una)

| Moneda | `currency_id` | Cómo entra | Dónde se guarda | Dónde se gasta |
| --- | --- | --- | --- | --- |
| **Nexum Coin (NC)** | `nexum_coin` | CASH vía PSP, o grant ops (beta/Patreon) | `user_wallets` (cuenta) | Catálogo de tienda (`redeem` / fulfill de pack) |
| **Hesedia / Hesedias** | `hesedias` | Juego (drops, mercado, craft). Un SKU de tienda **puede** entregar Hesedias | `character_wallets` (personaje) | Mercado de jugadores, craft, sinks de mundo |

No se mezclan en la misma fila. No hay tipo de cambio libre NC ↔ Hesedias: si NC compra Hesedias, es un **SKU**, no un FX. **Prohibido** convertir a dinero real (cash-out). El mercado de jugadores **solo** `hesedias`.

Tasa de diseño de packs (no es FX en vivo; comisiones del PSP aparte): **1 USD → 15 NC**. Referencia: **100 USD → 1 500 NC**. Los SKU concretos (10 USD, etc.) se listan en catálogo al abrir tienda; no se hardcodean en Godot.

### 29.2 Before production (beta / primer impulso)

En betas se pueden ofrecer **beneficios únicos y personalizados** a testers vía **Patreon u otra plataforma de crowdfunding**. Eso es el primer impulso económico.

Implementación: `entitlements` + `commerce_orders.provider` `patron` | `crowdfunding` | `manual`. Un operador (o un job que lea la plataforma) llama `/internal/commerce/grant`. **No** hace falta checkout de tarjeta para esto. El contenido del beneficio se diseña por temporada de beta (GDD §17); no se mete un `if is_patron` en el tick.

### 29.3 In production (cobro automático)

Idea de negocio: pago con **cualquier tarjeta crédito/débito** vía procesador. Se recibe el pago y **automáticamente** se entrega lo comprado (pack de NC, SKU, suscripción).

Flujo canónico:

1. Cliente autenticado `POST /api/store/checkout` → pedido `pending` + sesión en el PSP.
2. El jugador paga en el PSP (hosted checkout / elemento de tarjeta). Godot no ve PAN.
3. Webhook `POST /api/webhooks/payments` con firma válida → `paid` → cumplimiento (acreditar NC o entregar SKU).
4. Reintentos del PSP: el mismo `provider_ref` no vuelve a cumplir.

Proveedor (Stripe u otro): **adaptador**. El dominio conoce Order + Payment + Fulfillment. Elegir PSP es un ADR de adaptador cuando se abra producción, no un `if` en Inventory.

### 29.4 Canales de ingreso

| Canal | Rol | Notas |
| --- | --- | --- |
| **Microtransacciones** | **Principal** | Elementos virtuales o mejoras de experiencia: cosméticos, armas, monedas, ventajas competitivas. Cada SKU lleva `tag`: `cosmetic` \| `currency` \| `power`. Ver GDD §17 (PvP puede equalizar `power`) |
| **Suscripción** | Alternativo | Membresía mensual/anual; `entitlements` con `expires_at`. Beneficios de catálogo, no código suelto |
| **Infoproductos** | Alternativo | Guías, tutoriales, trucos, estrategias. Fuera del tick; misma cuenta/checkout si se vende junto al juego, o store web. No van por ENet |

### 29.5 Qué no es Commerce

- Mercado jugador-jugador (Hesedias): Inventory.
- Drops de sesión: evento → Inventory.
- Simulación: no lee `commerce_orders`.

---

## Apéndice A — Glosario

| Término | Significado |
| --- | --- |
| Session | Proceso de simulación Godot con kind, mapa y ruleset |
| Battle node | Session efímera PvP o PvE |
| Lobby | Session persistente de práctica y social |
| Entity | Objeto simulado con EntityID |
| Ruleset | Plugin de reglas de la sesión |
| Join token | Credencial de un solo uso para ENet |
| Ticket | Solicitud de cola de matchmaking |
| PrivateMatchRoom | Sala de espera metajuego (no es el proceso Godot) |
| Party | Grupo persistente corto para PvE (Fase 5) |
| AOI | Área de interés: qué entidades se replican a cada cliente |
| Snapshot | Estado de entidades en un tick para replicar |
| Nexum Coin (NC) | Moneda premium de cuenta, se compra con CASH |
| Hesedias | Moneda de mundo por personaje; mercado de jugadores |

## Apéndice B — Modos de matchmaking iniciales

| mode | jugadores | SessionKind | mapa por defecto |
| --- | --- | --- | --- |
| `quick_1v1` | 2 | `pvp_duel` | `maps/pvp/arena_01` |
| `quick_5v5` | 10 | `pvp_arena` | `maps/pvp/arena_02` |
| `private` | 2–10 según host | elegido por host | elegido por host |
| `dungeon` (Fase 5) | 2–5 party | `pve_dungeon` | por dungeon_id |

## Apéndice C — Orden de implementación sugerido dentro de cada fase

No sustituye las fases; evita empezar por la UI.

1. Dominio + tests
2. Adaptador de persistencia
3. Adaptador HTTP/WS o spawn
4. Cliente mínimo que consuma el contrato
5. Pulido de UI
6. `docs/modules/<id>.md` (negocio, técnica, gameplay) + índice en `docs/modules/README.md`

---

*Fin del plan director. Cualquier feature, tabla, paquete de red o contexto nuevo debe poder señalar a una sección de este archivo.*
