# Nexum Terra — Diseño de juego (GDD)

**Este archivo manda sobre fantasía, builds, stats, items, skills y sensación de combate.** La arquitectura (sesiones, EntityID, Postgres vs catálogo) está en `PLAN.md`. Cómo está implementado cada sistema en código: `docs/modules/`. Si un número de balance cambia, se edita aquí, el module doc si cambia el flujo, y luego `nexum-terra/data/`. Si cambia *cómo se persiste o se simula*, se edita `PLAN.md`.

Fuentes actuales: GDD Notion, Gameplay, Daño y Build, Clanes, cinco reinos, Elementos y los cuatro básicos, Items y Crafting, Pasivas, Sistema de Puntos, Sistema de Rebirth, Obtención de Skills, Combates Melee, Clasificación, Rangos de Reinos, Sistema de Zonas, Dungeons, Misiones, Divisiones, Renegados, Robo, KO y Death, Regeneración, Logout bajo ataque, Médico, Espadachín, Sensor; skills de clan: Omnivisus, Gadgetrix, Entomante, Nachtsoldaten, Pulmonarius, Sangrafilos, Umbromante, Geisteswaffen (2026-09-13). Pendiente de volcar: skills de los 8 clanes sin PDF.

Estado: **normativo en lo escrito; incompleto donde se marca abierto.** Caps, melee A, kit, cámara 640×360 y mapas de rooms: §0.1. Etapa B/C y el resto de §14 siguen abiertos.

Inspiración de ritmo (no de lore ni de calendario): [StormEdge](https://store.steampowered.com/app/2321350/StormEdge/). Acción frenética. El overworld partido en regiones es **Etapa C** (`PLAN.md` §18), no el juego que se construye ahora.

---

## 0. Alcance por etapas

Tres eras, alineadas con `PLAN.md` §1.1. **No se mezclan.** El modelo de personaje *admite* piezas futuras; el catálogo y el tick no las simulan hasta su era.

| Era | Qué es | Cuándo |
| --- | --- | --- |
| **A — Rooms** | Lobby, práctica, PvP instanciado (Fases 1–4) | Ahora |
| **B — Mazmorras + meta** | Dungeons en instancia + build, inventario, mercado, clanes (skills), etc. | Tras **jugar** A |
| **C — Open world** | Regiones, zonas de color, conquista, loot de cadáver en overworld | Tras **meses** de B y **decisión explícita** |

| Pieza | Etapa A (rooms) | Etapa B (mazmorras + meta) | Etapa C (open world) |
| --- | --- | --- | --- |
| Identidad: nombre, apariencia, `kingdom_id`, `clan_id` | sí (1 reino jugable + 1 clan de ese reino) | skills/pasivas/activas del clan | — (ya en B) |
| Chat de distancia con gentilicio / nombre | sí (reglas §4) | amigos + olvido: ya en §4 | chat por zona/región |
| Melee, guardia, dash, hotbar, swap de arma | sí (guardia, rotura y choque §9.6); **sin** instancias de arma/pecho | combos mágicos / elemento; arma y pecho | — |
| Clasificación D–SSS (honor + asesinatos) | asesinatos de rooms PvP; honor aún 0 sin misiones | honor por PvE/misiones; umbrales | asesinatos de overworld |
| Rangos de reino (Novicio→Élite) | todos **Novicio** (display) | desafío Soldado y torneo Élite como **rooms de evento**; Comandante por umbrales | sede Aurora, Órdenes/territorios |
| Tres vitals + daño estático + % de equipo | sí (§6); en A el `%` de equipo es **0** (no hay gear) | pasivas de clan/elemento/profesión | — |
| Equipo (armadura + armas) | **no** (melee y skills salen de data; slots de §8 en B) | orbes, anillos, collar, calidad/nivel, crafting; 1 arma + 1 pecho por reino | — |
| Skills genéticas de clan (activas/pasivas) | no; catálogo vacío | sí | — |
| Elementos (orbes, crafting, ciclo, skills básicas) | reglas en §8.6–8.9 y §10.1; no se simulan aún | sí | — |
| Crafting de atributos, subida de nivel de ítem, mercado | no | sí (§8.10–8.18) | se **reutiliza**; impuestos de zona → reino |
| Dual currency NC / Hesedias + Commerce | **modelo** (ids, wallets; sin PSP) | mercado Hesedias; SKUs; grants beta | se **reutiliza** |
| Profesiones | no | sí | — |
| Rebirths y pasivas especiales de rebirth | no | sí | — |
| Árbol de puntos / grind de activas | no | sí (PVE de mazmorra) | grind también en overworld cuando exista |
| Mazmorras instanciadas | no | sí: **rooms desde el lobby** (tipos y recompensas §16) | mismas instancias; el **portal** spawnea en zonas del overworld (§16.1, §16.3). Se **sale** del shard al battle node |
| Misiones | contadores de rooms (ej. gana 3 1v1) | + PvE de mazmorra / honor de reino si se cataloga | escoltas, robo de mercancía, mundo |
| Sistema de zonas, regiones, conquista, robo en cadáveres de mundo | **no** | **no** | sí (§15). **Prohibido** prototipar en A/B |

Todo es reemplazable en una build **excepto el clan en el que naces**, hasta un rebirth (§11.3). En Etapa A ese ancla ya se elige; aún no otorga skills.

### 0.1 Recorte cerrado Etapa A (rooms)

Decisiones de producto (2026-09-13). Si el código contradice esto, se corrige el código o se actualiza esta tabla.

| Tema | Regla |
| --- | --- |
| `pvp_duel` win | Primera **Death** gana. Sin respawn en la arena. Disconnect = forfeit (`PLAN.md` §12.7 rooms). |
| KO → Death | Vitalidad 0 = KO. Levantarse en A = **mantener una tecla** el tiempo de canal del GDD (§12.5: 8 s en el 1.er KO / Novicio). **Sin** minijuego de puntería (eso es diseño posterior). Si se suelta la tecla o vencen 30 s → Death. |
| Dummy de lobby | Vitals **infinitos** (no KO). Solo feedback de hit. |
| Equipo | **Ningún** `item_instance` en A. Pecho/arma/placeholder por reino = Etapa B. PvP nace equalizado. |
| Regen de quieto (hasta 50 %) | **Solo open world (Etapa C).** Nunca en arena ni en mazmorra. |
| Lobby | Sin regen de quieto. **Sí** tiles de reposo (sillas/camas): stamina y vitalidad según §12.6 Reposo. Hold Z recarga maná desde reserva. |
| Mazmorra (B) | Igual que arena: hold Z + pociones; sin quieto. Sillas solo si el mapa las pone. |
| Pociones | Inventario/catálogo = **Etapa B**. En A no hay poción que beber. Cuando existan: en arena/mazmorra se beben **en cualquier momento**, pero **sin moverse** mientras. El rival no tiene que estar KO. Cap de usos por match: abierto. |
| Salto | **No** en A (ni en 2D hasta que un mapa lo pida). Espacio = golpe ligero. |
| Kit A | **Las mismas 4** para todos (§10.0). No se gated por reino. |
| Colas A | `quick_1v1` y `quick_5v5` (10 jugadores reales, sin bots). |
| `pvp_arena` win | Al mejor de **3 rondas**. Cada ronda es **team wipe** (5 Deaths del equipo rival). Respawn entre rondas; no durante la ronda. |
| Skills A | Kit compartido §10.0: `iaido-dash`, `reverse-dash`, `kawarimi`, `magic-armor`. |
| Sprint | Andar **1,5 tiles/s** (simulado en px). Correr = **×1,5** (+50 %). Alternativa de playtest: ×2. Sin gasto de stamina en A. |
| Combo melee | Ligero **100**, pesado **250**, combo 🔳🔳🔳 = **100+100+150 (350)**. Cancel post-hit a dash/skill. Carga 5 s → **+25 %** melee solo al **100 %**. |
| Guardia A | Brazos **40 %**. Pool **500** (por daño, no por tiempo). Hit a **carga 100 %** rompe: **0** daño, stun 400 ms, separación **1** tile. Clash arma-arma: **2** tiles. |
| Maná | Reserva = **3×** cap (6000). Hold Z: **5 % del cap / s** desde la reserva. Equipo/pasivas de carga = B+. |
| Sillas | **30 s** a llenar vit + stam + maná + reserva. |
| 5v5 lectura | Tinte sutil **azul** aliado / **rojo** rival (silueta). Sin nametag. |
| Misiones A | `win-3-duels`, `play-10-matches`, `win-5-arenas`. Recompensas luego. |
| Presentación | Native **640×360**. Tiles **32**. Viewport **20×11** tiles. Arena 1v1 **25×15**. 5v5 **100×100**. Sprite: ancho cuerpo **32 px**; lienzo 48×48 o 32×62. Arte = mockup/greybox. Icono de reino = cuadrado verde. Fuente **Pixeloid Sans**. Mando/móvil: después. |
| Create | Nombre + **1 reino** + **1 clan** de ese reino. Sprite genérico por reino (tinte). Sin editor de cara. |
| Nametag | **Lobby:** nombre + icono de reino. **Luchador en battle:** sin nametag ni barras flotantes. **Espectador:** overlay con **nombres** (broadcast); los peleadores no lo ven. Chat §4. |
| HUD combate | Solo **tus** 3 vitals. En 5v5: tinte azul/rojo de silueta. Sin barras flotantes. |
| Mapa lobby | Un rectángulo greybox: social (sillas) en un lado, dummy(s) en el otro. |
| Mapas PvP | `arena_01`: **25×15** tiles. `arena_02`: **100×100**, con hitos para que los equipos se encuentren. |

---

## 1. Pitch y fantasía

Nexum Terra es un **RPG online de acción**. La **fantasía** es un mundo abierto dividido en regiones (culturas y épocas en el mismo suelo). El **producto que se construye** es, en este orden: rooms de combate → mazmorras con metajuego → y solo después, si se decide por escrito, ese overworld. Farmer, guerrero o millonario **todos pelean**; en A/B eso ocurre en instancias, no en el campo.

La magia es “la ciencia” de este mundo: **todos son magos**, aunque el primer recorte de combate sea melee + movilidad + hotbar.

Arquetipos de jugador (economía / fantasía, no clases de combate): **Farmer**, **Guerrero**, **Millonario**.

---

## 2. Lore

Nexum lo habitaba la gente de **Aurora**. Un suceso de origen desconocido abrió portales y trajo **fragmentos de diversos mundos** y su gente. Las lenguas se distorsionaron en el tránsito: todos hablan el idioma Nexum y olvidan el original.

No tardaron los **cuatro reinos** en guerrear por el poder y querer conquistar Nexum. Al acercarse al centro del planeta, Aurora oyó el estruendo. El rey de Aurora luchó días contra los cuatro reyes. La historia pública dice que los cuatro **perdieron y murieron**.

La verdad (aún no pública para los pueblos): los cuatro reyes **dejaron de luchar y ascendieron**.

Notas de lore a conservar:

- Dios invocó cuatro reinos al Nexum para convivir con Aurora y completar lo faltante; la intención era que se llevaran bien. No se ha logrado.
- El suceso tiene nombre en Aurora: **La Inmersión**. El cielo se tornó de diversos colores; los sabios detectaron grandes grietas dimensionales. Se alude a **varias grietas**, no solo cuatro mundos, para poder traer más adelante.
- Recuerdo de otro cielo: “en nuestro planeta el cielo era morado y los atardeceres azules.”
- Cada uno de los cuatro mundos tiene **Prime / Titan / Arconte** distintos (dioses puestos por Dios). Detalle por reino: página de Notion pendiente.
- **Terrara:** menor capacidad cognitiva; recibieron poderes mágicos para cubrir esa falta.
- **Spectra:** más comunicación con lo espiritual; tienen ventajas en ese eje.
- **Fontaine:** científicos / magos de la tecnología (detalle de reino en §3).

Se descubren más secretos jugando (lugares, NPCs, lectura). No se spoilea el resto en UI de onboarding.

---

## 3. Reinos y clanes

Catálogo: `nexum-terra/data/kingdoms.json` y `nexum-terra/data/clans.json`. Añadir un reino o clan = fila de data + arte, no un fork de combate.

### 3.1 Selección (primeras etapas)

Al crear personaje el jugador elige **un** reino jugable y **un** clan de ese reino.

| Invariante | Regla |
| --- | --- |
| Reinos jugables | `fontaine`, `terrara`, `spectra`, `aerion` |
| `aurora` | Solo GMs; no aparece en el create de jugador |
| `clan_id` | Obligatorio; su `kingdom_id` debe coincidir con el reino elegido |
| Skills | Kit A **igual para todos** (§10.0). Clan `skills: []`. El tick no lee reino ni clan para daño. |
| Permanencia | Fuera de rebirth, el clan de nacimiento no se cambia (es lo único no reemplazable de la build). El rebirth **corta toda afiliación** al clan anterior: naces de nuevo en otro reino/clan; **ninguna habilidad** de esa vida se guarda (§11.3) |

La fantasía de cada clan (ojos, insectos, cristales, etc.) se escribe aquí para no perderla. **No se implementa** hasta que existan entradas de skill en data.

Idea de mundo (Etapa C, no A ni B): **tiendas de ropa distintas por reino**. Vestirte de ninja sin ser de Aerion implica ir a comprar allá.

Movilidad y defensas de A: kit compartido de Nexum Terra (§10.0). El reino/clan es identidad (y, en B, skills de clan). No se reparte 1 skill por reino.

### 3.2 Aurora (`aurora`)

Ciudad antigua del **shogunato**. Murallas de índice Kamakura. Colores **negro y blanco**. Época: pasado. Hostilidad: **neutral**. Arquitectura de samurái: madera alzada contra inundaciones, paredes que rodean el área frente a enemigos del exterior.

El ambiente de **ciudad** en overworld es **zona celeste** (§15.2): no se ataca y no se usan habilidades. Eso sustituye la idea previa de “puedes pegar pero un NPC te parte”. Fuera de Etapa C, Aurora es identidad/lore (GMs en create). El dash samurái del kit A (`iaido-dash`) es **de todos**, no un racial de Aurora.

Gentilicio de chat: no aplica a jugadores (no nacen aquí).

### 3.3 Fontaine (`fontaine`)

Ambiente de **futuro**; referencia de sensación: capital del oeste tipo ciudad de Bulma. Gris, celeste, cristal. Territorio alrededor invadido por aliens (campos extra: TBD). Época: futuro. Hostilidad: **activa**. Arquetipos de fantasía: **científicos** (magos de la tecnología). Edificios altos que rematan en punta circular, colores llamativos, césped junto a carreteras con vehículos voladores.

Nota de diseño (Omnivisus): ojo inspirado en byakugan en un clan científico; cuatro clanes que estudian artes distintas. Skills: §12.9.

Gentilicio de chat: **Fontainer**.

### 3.4 Terrara (`terrara`)

Reino de la **fuerza**. Gran bosque, desierto y alto valle. Paleta medieval: hierro, rojo, naranja. Época: pasado. Hostilidad: **activa**. Arquetipos: soldados, caballeros y magos. Arquitectura mezclada: **gótica** en edificios importantes, **románica** en hogares, **bizantina** en centros de la monarquía.

Gentilicio: **Terrano**.

### 3.5 Spectra (`spectra`)

Reino oscuro: pantano venenoso, cementerio, bosque oscuro. Colores **verde y púrpura**. Época: no registrada. Hostilidad: **activa**. Arquetipos: ninjas / magos / elfos oscuros. Edificios góticos oscuros.

Gentilicio: **Spectro**.

### 3.6 Aerion (`aerion`)

Ciudad futurista en el **cielo**, nubes y cielo azul. Verde esmeralda, blanco y dorado. Época: futuro. Hostilidad: **activa**. Arquetipos: guerreros del cielo. Planta en estrella con círculos en puntas y centros; flota por maquinaria de los habitantes. Rascacielos de cristal junto a edificios de aspecto contemporáneo.

Gentilicio: **Aerion**.

### 3.7 Clanes (16, 4 por reino jugable)

Los clanes son de los reinos en los que puedes nacer. Aurora no tiene clan jugable en este catálogo. La fuente escribe a veces “Fountaine”; el `kingdom_id` canónico es `fontaine`.

**Fontaine**

| `clan_id` | Nombre | Fantasía |
| --- | --- | --- |
| `omnivisus` | Omnivisus | Ojos científicos: estado físico de una persona y visión 360° a larga distancia (una región entera). Combate: puntos vulnerables / sistema nervioso en melee. Skills §12.9. |
| `sagitta` | Sagitta (flecha, latín) | Clan antiguo. Magia concentrada en “arcos” (hoy varían). Largo alcance, tiros precisos. Magia interior canalizada con **canalizadores cibernéticos**. Skills: sin PDF. |
| `gadgetrix` | Gadgetrix | Gadgets de protección y ataque, para sí o para aliados. Skills §12.9. Crafting de orbes +25 % (§8.7). |
| `ciberlance` | Ciberlance | Científicos de mejoras biónicas de alto nivel. El melee más fuerte de Fontaine frente a la precisión de Omnivisus. Skills: sin PDF. |

**Spectra**

| `clan_id` | Nombre | Fantasía |
| --- | --- | --- |
| `herbora` | Herbora | Control de plantas de origen mítico o demoníaco. Skills: sin PDF. |
| `entomante` | Entomante | *ento-* insectos + *-mante* dominio. Bichos que habitan en su cuerpo. (Canal `mana_steal` cuando existan skills.) Skills §12.9. |
| `pyrofauces` | Pyrofauces | Conexión de maná al inframundo; llamas del infierno y otros elementos de ese eje. Skills: sin PDF. |
| `nachtsoldaten` | Nachtsoldaten | “Soldados de la noche”. Naturaleza + artes oscuras + disciplina de soldado. Skills §12.9. |

**Terrara**

| `clan_id` | Nombre | Fantasía |
| --- | --- | --- |
| `stellamante` | Stellamante | Maná en ataques luminosos tan rápidos que los llaman estrellas. Skills: sin PDF. |
| `pulmonarius` | Pulmonarius | Fuerza sobrehumana; distancia y melee. Skills §12.9. |
| `vitamancers` | Vitamancers | Manipulan energía vital: absorber cadáveres; proyectiles/orbes lentos que drenan vitalidad; materia extraña (minas, alfombra que roba vital a quien la toca); sacrificio propio para curar a otro (coste: un KO al caster, el aliado queda a 50% vit y stamina). Skills: sin PDF. |
| `sangrafilos` | Sangrafilos | Maná en metal: filos, cobertura, combate de **sangrado** (efecto secundario). Skills §12.9. |

**Aerion**

| `clan_id` | Nombre | Fantasía |
| --- | --- | --- |
| `umbromante` | Umbromante | Umbra: dominio de la sombra. Referencia de sensación: Nara. Skills §12.9. |
| `geisteswaffen` | Geisteswaffen | Clan antiguo. Poder en herramientas: espada moldeable de tecnología. Medio alcance; hojas de chakra tipo samurái. Skills §12.9. |
| `crystallomante` | Crystallomante | Convierte materia elemental en cristal; proyectiles, muros, dragón de cristal, encierro. Necesita cristales cerca: invoca **generadores** que sueltan cristales periódicamente. Skills: sin PDF. |
| `sonomantes` | Sonomantes | Sonido para deshabilitar: daño directo o ilusiones. Potenciado por tecnología. Ilusiones sonoras; ondas que crecen en círculo. Skills: sin PDF. |

Listas detalladas de skills: §12.9 (8 clanes volcados; 8 sin PDF). Un **5.º clan por reino** se desbloquea por rebirth de cuenta (§11.3); aún no hay ids en data. **No** se llenan `skills` en `clans.json` ni el tick lee el clan hasta Etapa B.

---

## 4. Conversación e identidades

En el **chat de distancia**, si no te conocen, el mensaje no lleva tu nombre de personaje, sino el gentilicio del reino:

- Fontainer: Hola, viejo.
- Terrano: Sos rarito, científico.
- Aerion: No más raro que vivir en la tierra.
- Spectro: 💀 *(nota de diseño: Spectra vive en un pantano venenoso)*

Las personas se **presentan** escribiendo su nombre en ese chat. Hay que recordar al otro por aspecto y habilidades.

Al **desconectarse**, el personaje **olvida** los nombres de quienes no son amigos y se presentaron. **Los amigos siempre leen tu nombre** en el chat de distancia.

Contrato de sistema (cuando se implemente): la resolución nombre vs gentilicio es del servicio de Chat + Social (amigos), no del tick ENet. Ver `PLAN.md`.

**Nametag sobre el cuerpo (no es chat):**

- **Lobby:** nombre + icono de reino. Sin clan, sin barra de vitals en el tag.
- **Combate (rooms de batalla):** el **luchador** no ve nametag ni barras flotantes. El **espectador** ve nombres en overlay de UI (no pegados al sprite).

---

## 5. Capas de un personaje

```
Account (user)
  └── Character                 # 1 por cuenta en Fase 1; N más adelante
        ├── Identity            # name, kingdom_id, clan_id, appearance
        ├── Progression         # level, xp (PVE); puntos de habilidad / rebirth; PvP puede usar rating aparte
        ├── Vitals              # caps y actuales: mana, stamina, vitality
        ├── Loadout             # skillbar + melee; no hay “build template”
        ├── Equipment           # instancias en slots (% y caps)
        └── CombatSnapshot      # caps + % por tag de daño → tick
```

En una sesión Godot entra **solo el snapshot** (caps, vitals actuales al spawn, % por tag, skill ids). El tick no lee Postgres. No hay `atk` / `pow` / `def` que escale el daño.

**Composición objetivo** (cuando estén todas las etapas):

- Clan de nacimiento (skills genéticas cuando el catálogo deje de estar vacío)
- 3 slots de orbe elemental (`orb_1`, `orb_2`, `orb_3`); solo el 3º acepta compuestos (§8.6)
- Profesión
- 2 anillos + collar
- Equipo: Head, Armor, Gloves, Pants, Boots
- Armas: mano izquierda, mano derecha, arma secundaria (Gameplay: 2 armas en izquierda + 1 en derecha; se rota la izquierda)
- Un estilo de melee (solo uno)
- Árbol de pasivas (puntos de habilidad) y pasivas especiales (puntos de rebirth) cuando existan (§11)

Snapshot de primeras etapas: **reino + clan (sin skills de clan) + kit A de 4 técnicas + tres vitals + melee**. Sin `%` de equipo.

---

## 6. Vitals, daño y “build”

No hay clases ni un árbol que te preasigne el rol. Puedes ir a magia elemental, a cuerpo a cuerpo, o a mental / ilusiones. Los clanes **no** definen un build: nada está atado a un template. El daño de las habilidades es **estático** y sube con el **nivel de pasivas** (clan, elemento, profesión, etc.; referencia de sensación: pasivas de Albion) y con **equipo que da %** a un tag (fuego, melee, …).

Especializar sigue importando: quien pone *todo* el equipo a fuego pega más fuego que quien parte el equipo entre fuego y melee. Puedes **cambiar el loadout cuando quieras** (fuera de combate / fuera de match congelado).

No se usa la fórmula antigua `atk`/`pow` × escala + mitigación `K`. Eso queda derogado.

### 6.1 Vitals (salud del personaje)

Tres pools. IDs de data en inglés.

| id | Nombre | Qué es |
| --- | --- | --- |
| `mana` | Maná | Pool de combate. **Reserva** (`mana_reserve`) = **3×** el cap (base **6000**). Hold Z recarga el pool desde la reserva al **5 % del cap / s** (100/s con cap 2000). Equipo y pasivas podrán subir esa tasa (B+). |
| `stamina` | Stamina | Salud física y mental. Permite **correr** y **resistir golpes**. |
| `vitality` | Vitalidad / vida | Salud vital. A **0 → KO**. |

Caps **base** de un jugador (sin equipo; el equipo modifica maná y stamina):

| Vital | Cap base |
| --- | --- |
| Maná | 2000 |
| Reserva de maná | 6000 (3× maná) |
| Stamina | 2000 |
| Vitalidad | 1500 |

Locomoción (px en runtime; data en tiles): andar **1,5 tiles/s**; sprint **×1,5**. Ver `nexum-terra/data/movement.json`.

### 6.2 A qué pool pega cada tipo de daño

| Canal (`damage_channel`) | Ejemplos | Pool que golpea |
| --- | --- | --- |
| `stamina` | Melee, mental / ilusiones | Stamina |
| `mana_steal` | Entomante, algunos estilos de tai | Maná (robo / consumo del objetivo) |
| `vitality` | Elementos (salvo excepción en data, p. ej. `whirlwind`) | Vitalidad |

`mana_steal` y daño elemental son contenido de etapas siguientes; el **canal y el overflow** se implementan en el ruleset desde el núcleo para no refactorizar el tick.

### 6.3 Jerarquía hasta el KO

Una sola jerarquía de overflow:

```
Maná → Stamina → Vitalidad
```

- Daño a **stamina**: no toca maná. Overflow: stamina → vitalidad.
- **Robo de maná** con maná a 0: el resto pasa a stamina; si stamina está a 0, a vitalidad.
- Vitalidad a **0**: el usuario cae **KO**. Conteos, minijuego de levantarse, death, cadáver y hospital: §12.5. Regen de pools y de conteo de KO: §12.6.

Presupuesto para tumbar a alguien con caps base (sin equipo):

| Camino | Puntos necesarios | Cuentas |
| --- | --- | --- |
| Daño stamina | 3500 | 2000 stamina + 1500 vitalidad |
| Daño vitalidad | 1500 | solo vitalidad |
| Robo de maná / chakra | 5500 | 2000 + 2000 + 1500 |

### 6.4 Fórmula de daño (estático + %)

```
raw    = skill.base + skill.tag_flat      # p.ej. 800 + 50 de ese elemento
final  = raw * (1 + sum(pct_mods del tag))
```

Los `pct_mods` salen de pasivas y de equipo (anillos, collar y rolls de crafting cuando existan). Se aplican al **tag** de la skill (`melee`, `fire`, `mental`, …), no a un stat `atk` global. El % **general** de anillo/collar (§8.13) es otra palanca; no sustituye al tag.

Ejemplo de la fuente: skill elemental más fuerte ≈ 800 base + 50 de ese elemento = 850; con +30% daño elemental → **1105**. Con eso, una super técnica puede dejar KO a alguien que aún tenía ~90% de vitalidad.

Ritmo: las peleas deben aguantar **varias técnicas pequeñas**. Las súper técnicas son más difíciles de acertar y tienen un **coste negativo** para quien las usa (p. ej. no puede moverse mientras las ejecuta). Esos riesgos van en data de la skill (`self_applies`, `cast_ms`, flags), no en el nodo del jugador.

Anillo de Poder (cuando exista): +20% a **todos** los daños causados; +15% maná consumido por el portador.

### 6.5 Dónde vive cada número

| Dato | Catálogo (`nexum-terra/data`) | Postgres | Memoria de tick |
| --- | --- | --- | --- |
| Caps base (maná 2000, reserva 6000, stam 2000, vit 1500), melee, movimiento | sí (`vitals.json`, `melee.json`, `movement.json`) | no | no |
| Level, xp, `kingdom_id`, `clan_id` | no | `characters` | no |
| `base` y `damage_channel` / tags de cada skill | sí | no | no |
| Mods `%` o add de `item_def` (caps de maná/stamina, % por tag) | sí | no | no |
| Instancia (calidad, nivel, rolls, owner) | no | `item_instances.payload` | no (el tick ve mods resueltos en el snapshot) |
| Maná / stamina / vitalidad actuales, CC, cooldowns | no | no | Entity.combat |

---

## 7. Loadout (primeras etapas vs objetivo)

No hay un **build predefinido**. El “build” en la práctica es: qué armas y skills llevas y a qué **tags** apunta tu equipo (y, más adelante, tus pasivas). El daño no sale de un stack de `atk`.

### 7.1 Primeras etapas

| Pieza | Qué es |
| --- | --- |
| Skillbar | En A: **4** hotslots iguales para todos (§10.0). Rueda no cambia de barra hasta B. |
| Melee kit | Un estilo; golpes ligero/pesado, guardia, carga, choque de armas (§9.6). Daño canal `stamina`, `base` estático. En A **no** hay `% melee` de equipo (0). |
| Movilidad | `iaido-dash` y `reverse-dash` en el kit (C / V). Kawarimi en hotslot 3. |
| Loadout | El mismo kit A + estilo melee. Se congela al entrar a 1v1. Sin snapshot de items en A. |

Persistencia sugerida: jsonb `characters.loadout` `{ skillBar: [], meleeStyleId: "" }` o tabla equivalente. En cola/ready-check se **congela**; no se cambia a mitad del 1v1. Fuera de match, el jugador puede cambiar cuando quiera.

### 7.2 Objetivo (no implementar ahora)

| Pieza | Qué es |
| --- | --- |
| Clan de nacimiento (16, §3.7) | Lo único no reemplazable **hasta un rebirth**. Skills: cuando `clans.json` deje de tener `skills: []` |
| Elementos | 3 orbes; slot 3 = compuesto; ciclo y reacciones §8.6. Skills §10.1 |
| Equipo avanzado | Calidad, nivel I–V, collar, crafting de atributos, mercado, orbes de uso §8.10–8.18 |
| Profesión | Pasivas que modifican el personaje u otras activas. Pensadas: Médico, Espadachín, Sensor. Originalmente iban a ser activas; no hay controles libres, así que son pasivas. |
| Árbol de puntos | Pasivas **generales** se entrenan (puntos opcionales para adelantar). **Elementales:** activas se compran con puntos y suben con el uso; pasivas se mejoran con puntos. Catálogo generales: §11.2. Flujo: §11.5 |
| Activas | Familia/profesión: camino de uso (§11.5). Elementales: se **obtienen** con puntos de habilidad y **suben de nivel** usándolas. No AFK |
| Rebirth | Cada rebirth es más difícil hasta el #10; después la dificultad se estabiliza. Al renacer, en selección de personaje aparece un **alma flotante**, no un slot vacío. Flujo, requisitos y qué se conserva: §11.3. Puntos de rebirth alimentan **pasivas especiales**, no el árbol de §11.2 |

Los 16 clanes ya están en catálogo. El diseño de skills de 8 clanes está en §12.9. Data (`clans.json` `skills: []`) y tick **no** las leen hasta la primera skill de Etapa B.

---

## 8. Items y equipo

### 8.1 Definición vs instancia

- **`item_def`:** plantilla inmutable versionada (rarity, slot, mods, tags).
- **`item_instance`:** objeto poseído (`id`, `def_id`, `qty` si stackable, `payload` para rolls).

El combate solo ve mods y skills habilitadas. No ve el UUID del item en el tick.

### 8.2 Slots de equipo (objetivo de diseño)

Armadura: `head`, `armor` (pecho), `gloves`, `pants`, `boots`.

Anillos: `ring_1`, `ring_2`. Collar: `necklace`. Anillos con reglas especiales (§8.5) y rolls genéricos (§8.13) comparten slot.

Armas (inventario):

| Slot | Rol |
| --- | --- |
| `weapon_right` | Mano derecha |
| `weapon_left` | Mano izquierda (arma “activa” de esa mano) |
| `weapon_left_alt` | Segunda arma de la mano izquierda; se **rota** con la activa |
| `weapon_secondary` | Cambia de lugar con la mano izquierda, o con **ambas** si el arma ocupaba dos manos |

Gameplay aclara: puedes llevar **dos armas en la izquierda y una en la derecha** y rotar las de la izquierda. El scroll cambia **primaria ↔ secundaria**. Armas a dos manos (p. ej. arco) siguen permitiendo secundaria (espada o pistola). Ejemplos válidos: Staff + escudo, Staff + grimorio, espada + escudo + pistola, arco + pistola.

Nexum mezcla culturas y tiempos: fuego, hierro, inteligentes y arcaicas.

Orbes elementales (§8.6): `orb_1`, `orb_2`, `orb_3`.

**Primeras etapas:** implementar al menos `weapon_right`, `weapon_left` y un swap a secundaria; `weapon_left_alt`, anillos, collar y orbes pueden esperar. Equipar es Inventory (REST), bloqueado si `presence == in_match`. En etapas siguientes, equipar un nivel de pieza exige la **habilidad de portar** ese nivel (§8.11).

Un item declara `slot` + `tags`. El reino o el clan puede restringir `allowed_tags` cuando exista esa regla; hasta entonces, kit abierto para prototipar.

### 8.3 Stackables vs únicos

- Consumibles / materiales: `stackable: true`, `qty`.
- Equipo: `stackable: false`, una instancia por ítem.

### 8.4 Mods

Lista de `{ stat_or_tag, op: add|pct, value }`.

En primeras etapas los mods relevantes son:

- `add` / `pct` sobre caps `mana` y `stamina` (el equipo **modifica maná y stamina**; la vitalidad base 1500 no se cita como modificable por gear)
- `pct` sobre un **tag de daño** (`melee`, más adelante `fire`, `mental`, …)

Nada de scripts en el item en Fases 1–4. Nada de `atk`/`def`. Efectos raros = `effect_id` del catálogo de combat.

Crafting de **atributos y nivel de equipo:** §8.10–8.15. Crafting de **orbes compuestos:** §8.7. Orbes de uso / sellado: §8.17.

### 8.5 Anillos (etapas siguientes; no primeras)

Los anillos dan características **distintas** a los mods de armadura. Tradeables. Algunos raros.

**Anillos de Pureza** — permiten beneficios de elemento **puro** aunque tengas otros orbes (distinto del camino de pureza de §8.8, que **bloquea** slots a uno solo). Cinco tipos citados (cuatro elementos + el quinto no está en el export):

- Anillo de Pureza Ígnea (Fire)
- Anillo de Pureza Terrestre (Earth)
- Anillo de Pureza Acuática (Water)
- Anillo de Pureza Aérea (Wind)

**Anillo de Poder** — +20% a todos los daños causados; +15% maná consumido.

Un anillo más está cortado en la fuente (“Anillo de”).

Los anillos y el collar **crafteados** (maná, stamina, daño/reducción general) son §8.13. Un Anillo de Poder y un anillo crafteado de daño general no se fusionan en un solo objeto; si ambos existen, sus mods se suman en el snapshot (caps de §8.13 aplican al roll crafteado, no al anillo nombrado).

### 8.6 Orbes, ciclo y uso

Cuatro elementos básicos: `fire`, `wind`, `earth`, `water`. Catálogo `nexum-terra/data/elements.json`.

| Slot | Acepta | Cómo se abre |
| --- | --- | --- |
| `orb_1` | Básico | Al existir el sistema de elementos |
| `orb_2` | Básico | Entrenamiento (umbral TBD) |
| `orb_3` | Básico **o compuesto** (fusión de 2 orbes) | Entrenamiento (umbral TBD) |

Regla dura: **para usar skills de un elemento hay que tener un orbe de ese elemento equipado**. El staff al ≥50% de carga (Gameplay) dispara el elemento del modo/orbe activo; sin orbe, no hay disparo elemental.

Ciclo de **ventaja** (importan los niveles de poder). Solo estas parejas reciben modificación; el resto son “cruzados”:

```
wind  > earth
earth > water
water > fire
fire  > wind
```

Cruzados (p. ej. agua vs viento, tierra vs fuego): **ningún efecto extra**. Los ataques se destruyen ambos / no aplican el modificador de ventaja. No se inventan otras ventajas.

Daño elemental al cuerpo: canal `vitality`, salvo que la skill declare otro (p. ej. `whirlwind` → stamina).

Items y Crafting lista además un **Orbe de Rayo** (§8.17). El ciclo de ventaja y el catálogo de skills §10.1 siguen siendo **cuatro básicos** hasta que una página de elementos cierre `lightning` / `rayo`. No se inventan skills ni ventaja de rayo aquí.

### 8.7 Crafting de orbes

Los orbes se fusionan para crear un **elemento compuesto**. Nombres de compuestos: no están en este export (no inventar). Hay mención previa de compuestos de 3; este sistema solo describe fusión de **2**.

| Operación | Éxito por defecto | Fallo |
| --- | --- | --- |
| Fusionar 2 orbes → compuesto | 60% | **Pierde todos los ítems** del craft |
| Descraftear compuesto → orbes individuales | 50% | (no detallado; no inventar pérdida) |

Modificadores de fusión (cuando existan esas piezas):

- Clan **Gadgetrix**: +25% (60 → 85). Pasiva `crafting` §12.9. No se aplica mientras las features de clan estén apagadas.
- Cada **runa de crafteo**: +5%. Cap de runas: no cerrado.

Crafting es Inventory (REST / servicio Hono), no el tick. El resultado es un `item_instance` con `def_id` de compuesto.

### 8.8 Pureza

Camino de usuario **puro**: dominar por completo 1 elemento + misiones que **bloquean** los slots elementales a uno solo. Entonces aplican los **beneficios puros** de ese elemento.

- Desventaja: no puedes aprender (ni equipar) compuesto; solo un slot.
- Ventaja: mejora cada técnica de ese elemento y sus efectos.

Eso es distinto del **Anillo de Pureza** (§8.5), que da el beneficio puro *sin* dejar los otros orbes.

Beneficios puros volcados:

| Elemento | Beneficios puros |
| --- | --- |
| `fire` | El fuego se ve **azul**. Técnicas de fuego +20% daño. CD de técnicas de fuego −15% |
| `wind` | Técnicas de viento +50% probabilidad de **Cortar**. CD −15%. Consumo de maná de viento −20% |
| `earth` | **Pendiente** |
| `water` | **Pendiente** |

### 8.9 Restos en el suelo y reacciones

Cada elemental puede dejar un **resto de campo** (tile). La fuente marca las interacciones “divertidas” de terreno como contenido *para después*; la tabla siguiente ya está escrita y es normativa cuando existan restos.

| Resto | Efecto al pisar / al paso |
| --- | --- |
| Fuego | Llamas; **quemadura** |
| Agua | Agua; **ralentizamiento** |
| Tierra | Tierra; **ralentizamiento** |
| Viento | Corriente; **empuje** en la dirección de la corriente |

Reacciones cuando dos restos (o un resto y un ataque del otro elemento) coinciden:

| A + B | Resultado |
| --- | --- |
| Fuego + Viento | Las llamas **se expanden**; desaparece la corriente de aire |
| Fuego + Tierra | Permanecen **ambos** |
| Fuego + Agua | **Vapor** en el área; sube la temperatura; **daño de calor** |
| Tierra + Agua | El agua pasa a **lodo**; ralentización al **100%** de probabilidad |
| Viento + Tierra | Permanecen **ambos** |
| Viento + Agua | El agua **se congela**; desaparece la corriente de aire |

Números de quemadura, slow, empuje, daño de calor y duración: no están en el export.

Implementación: restos = entidades o celdas de mapa con `element_id`; reacciones en data, no `if fire and water` en el player.

### 8.10 Calidad, nivel y requisitos

Fuente: Items y Crafting. **No Fases 1–4.** El tick solo ve mods resueltos; calidad y nivel viven en `item_instance.payload` (`PLAN.md` §9.1).

Cada pieza de equipo tiene:

| Campo | Valores | `id` de data |
| --- | --- | --- |
| Calidad | Normal, Bueno, Destacado, Raro, Legendario | `normal`, `good`, `notable`, `rare`, `legendary` |
| Nivel | I–V (1–5) | `item_level` 1–5 |
| Beneficios | Suben alguna pasiva y/o rolls de atributo (§8.12) | mods en payload |
| Requisitos | Stat, pasiva y/o nivel **y siempre** la habilidad de portar equipo de ese nivel | chequeo al equipar (Inventory) |

La misma `item_def` puede dropear de bosses en **cualquier** calidad y nivel.

Armadura **Legendaria** brilla (shader de cliente; no es un mod de combate). Probar look antes de comprometer arte.

Calidad = cuántos atributos crafteados **puede** llevar la instancia (no los otorga al drop por sí sola):

| Calidad | Atributos crafteados máx. |
| --- | --- |
| Normal | 0 |
| Bueno | 1 |
| Destacado | 2 |
| Raro | 3 |
| Legendario | 4 |

Subir de **nivel** del ítem aumenta el **%** (o magnitud) de los atributos que ya tiene. No desbloquea un slot extra de atributo; eso es calidad.

### 8.11 Subir de nivel un ítem

Ejemplo de la fuente: Armor III → Armor IV.

1. Consumir **X runas de nivel** (cantidad X no cerrada).
2. Haber **combatido** PvE o PvP **con esa pieza en el nivel actual** para desbloquear la habilidad de **portar** el nivel siguiente.
3. Equipar el nivel nuevo exige esa habilidad. Sin ella, el ítem subido no se puede poner.

La fuente quiere **familias de armadura** con habilidades desbloqueables distintas (no un único árbol global). Ids de esas skills: al volcar el set concreto, no aquí.

El grind de “portar IV” es progresión de personaje (flag/skill), no un roll del ítem.

### 8.12 Crafting de atributos

Operación distinta de fusionar orbes (§8.7). El jugador mete un ítem en el banco de craft y pide **añadir / rerollear atributos**.

- El resultado es un buff **aleatorio** (p. ej. % daño o resistencia en una pasiva/tag: ilusiones, fuego, …).
- Si el roll no gusta: venderlo en el mercado (§8.18) o **quitar ese atributo** y volver a craftear.
- Un mismo atributo **no se repite** en la misma instancia. Un ítem puede llevar los **cuatro** atributos de **un** elemento (daño, resistencia, CD, coste de maná de ese elemento).
- Los % por atributo deben ser **bajos**: hay 5 piezas de armadura; un 20% por pieza sería +100% en un tag.

Solo se puede añadir **un orbe elemental** por craft de atributos. Ese orbe **sesga** el roll hacia atributos de su elemento. Los orbes son caros a propósito (§8.17).

Catalizadores de éxito (se pueden combinar con el sesgo; no inventar tablas de peso):

| Catalizador | Efecto |
| --- | --- |
| Arma / espada en el craft | Sesga a daño/resistencia **de armas**. La calidad del arma sube el % de **éxito**: Destacado 50%, Raro 75%, Legendario 100% |
| Piedra fina, gema dorada, etc. | Suben el % de éxito (magnitud por piedra: no cerrada) |

Crafting = Inventory (REST / Hono), no el tick. El snapshot de combate recibe los mods ya resueltos.

### 8.13 Pools de atributos por slot

**Helmet, Armor, Pants, Gloves, Boots**

| Familia de atributo | Notas |
| --- | --- |
| Daño en X elemento / pasiva | tag de % |
| Resistencia a X elemento / pasiva | tag de % |
| Reducción de cooldowns de X elemento | |
| Reducción de coste de maná de X elemento | |
| Daño o resistencia a **armas** (espadas, etc.) | sesgo con arma en el craft |

**Anillo izquierdo, anillo derecho, collar** (`ring_1`, `ring_2`, `necklace`)

Características distintas a la armadura. Caps **por ítem** (no por personaje; 3 piezas pueden apilar hasta 3× el cap):

| Atributo | Cap por ítem | Craft |
| --- | --- | --- |
| Maná máximo (cantidad) | 750 | Piezas de monstruos **mágicos**. Cada pieza suma poco % de éxito; varias o ítems de alta calidad para acercarse a 100% |
| Stamina máxima | 500 | Piezas de monstruos **sin magia**. Misma lógica de éxito |
| Reducción de daño **general** | 10% | Ítem de defensa / escudo + gema dorada |
| Aumento de daño **general** | 10% | Ítem de ataque / arma + piedra fina |

La magnitud de maná/stamina **varía con el nivel del ítem**.

Fantasía de build: tres joyas a daño general + armadura ofensiva = mucha presión, pocos hits para tumbarte. Se **contrarresta** con joyas a reducción de daño. No es un bug; es el tradeoff.

### 8.14 Riesgo si el éxito no llega a 100%

Si el craft es **solo el ítem** + “añadir atributos” (sin catalizadores que suban el éxito):

| % de éxito | Riesgo |
| --- | --- |
| Éxito por debajo de 50% | Puedes **perder el ítem** |
| Éxito por debajo de 100% | Puedes **perder todos los atributos** del ítem |

No se detalla si esos fallos son excluyentes o acumulativos; al implementar, un solo resultado por intento (éxito / strip attrs / destroy). No inventar % de cada fallo aquí.

### 8.15 Secretos de crafting

Habrá secretos de mundo (Etapa C): p. ej. **estatuas mágicas** que incrementan el % de éxito. Objetos específicos en el craft sesgan un atributo concreto (escudo + gema dorada → reducción de daño; arma + piedra fina → aumento de daño). Lista cerrada de secretos: con contenido de zona, no en A/B. Las estatuas de **conquista** de §15.3 son otro sistema; no mezclarlas con estas.

### 8.16 Sets — Armaduras de Fontaine

**Set Mazinger Z:** cinco piezas (los slots de armadura). Calidad prevista **Raro**. Beneficios de set y requisitos: **se escriben luego**; no inventar números ni pasivas. `kingdom_id` canónico `fontaine`.

### 8.17 Orbes de uso, sellado y economía

Distintos de la **fusión a compuesto** (§8.7). Estos orbes se **craftean con grind** y se venden; el mercado de jugadores es el sink/source principal.

**Orbes de elemento** (equipables / habilitan skills; precio de referencia de mercado **15 000 Hesedias**). Relación con grind de mazmorra: §16.5. Una verde enterada paga **1 500** Hes de cofres (15 verdes = **22 500**, no 15 000); al implementar economía, **reconciliar** este precio con esas tablas. La fuente de azules habla de “chance” de comprar un orbe con **10 azules enteras** (hay contienda; no es un precio fijo).

| Orbe | Efecto |
| --- | --- |
| Fuego | Permite habilidades de Fuego |
| Viento | Permite habilidades de Viento |
| Rayo | Permite habilidades de Rayo (ver tensión con §8.6) |
| Tierra | Permite habilidades de Tierra |
| Agua | Permite habilidades de Agua |

**Orbes de sellado y utilidad** (consumibles de uso; no son el slot `orb_n` de ciclo):

| Orbe | Efecto |
| --- | --- |
| De los 5 Elementos | Sella al objetivo: no puede ejecutar **ningún** skill de elemento. Sellado con toque |
| Bloqueo de regeneración | El oponente no regenera tras el sello. Sellado con toque |
| Destrucción de maná | El sellado consume **el doble** de maná y pierde una cantidad pequeña de maná a intervalos. Sellado con toque |
| De Luz | Barrera **de fuego** grande alrededor del caster; nadie entra ni sale. 60 s. Encerrar a una víctima |
| De Teletransportación | Marca un punto (región + X,Y). Consumir el orbe + tecla `*` abre opciones de teleport a marcas. Si ya hay marca: esperar **N s** (no cerrado) **emanando magia** para llegar. Cada marca se puede borrar desde esas opciones |

**Sellado con toque:** los orbes “con toque” cargan sellos **en la mano** (preparados de antemano). Los sellos en mano **caducan a 15 s**. Un golpe **melee** en esa ventana aplica los sellos al golpeado.

Tecla `*` y N de canal de teleport: conflicto posible con el mapa de input §9; unificar al implementar, no inventar bind final.

### 8.18 Mercado de jugadores

El mercado es **importante**: poner en venta y que otro compre **sin estar online**. No sustituye el trade directo; lo complementa.

Al listar, el ítem **sale del inventario** (escrow). Evita duplicar o vender algo que sigues usando.

Moneda citada: **Hesedias** (unidad Hesedia). Drops de dungeon / craft de orbes alimentan este loop (farmer / millonario). Persistencia: `PLAN.md` Inventory & Economy. **Nexum Coin no se lista aquí** (§17, Commerce).

---

## 9. Gameplay y controles

Plataformas objetivo: **PC (teclado y mouse), mando y móvil**. Mando y móvil se estructuran **después**; A prioriza mecánicas en teclado.

Convención en tablas: 🔳 / 🔼 / ⭕ / ❎ = face buttons del mando; L2/R2 = gatillos. En teclado, 🔳 ≈ golpe izquierdo / Space según contexto de arma; ver filas.

### 9.1 Controles (teclado | mando)

| Acción | Teclado | Mando (propuesta) |
| --- | --- | --- |
| Movimiento | WASD | Joystick izquierdo |
| Cámara | Mouse | Joystick derecho |
| Modo maná (tap) o recargar maná desde la **reserva** (hold) | Z | L2 |
| Dash adelante (`iaido-dash`) | C **o** 1 | L2 + ❎ |
| Reverse dash | V **o** 2 | (provisional) |
| Kawarimi | R **o** 3 | Doble R2 (provisional) |
| Armadura mágica (toggle) | Q **o** 4 | (provisional) |
| Barra de skills anterior / siguiente | Scroll up / down | ← / → en stick derecho |
| Guardia | X | R2 |
| Golpe ligero (2H) o arma mano izquierda | Espacio | 🔳 |
| Golpe pesado (2H) o arma mano derecha | Click derecho | 🔼 |
| Hotslot 1 | 1 | L2 + 🔳 |
| Hotslot 2 | 2 | L2 + 🔼 |
| Hotslot 3 | 3 | L2 + ⭕ |
| Hotslot 4 | 4 | ← |
| Hotslot 5 | 5 | ↓ |
| Hotslot 6 | 6 | → |
| Saltar | — (no en A) | — |
| Correr (hold) | Shift | ❎ hold |
| Primaria ↔ secundaria | — (no en A; B) | — |
| Levantarse de KO (hold, A) | F (provisional) | ❎ tap (provisional; ❎ hold sigue siendo sprint) |

**Etapa A cerrado:** Espacio = ligero. Rueda = barras (una sola en A). Swap no. **C/1** dash, **V/2** reverse dash, **R/3** Kawarimi, **Q/4** armadura mágica. Guardia = X. Bind F de KO provisional.

### 9.2 Carga de golpes

Ligero y pesado tienen **barra de carga**. Llena en **5 s**. Solo al **100 %**: **+25 %** al daño melee de **ese** golpe (se consume al soltar). Por debajo de 100 % no hay bonus.

Un melee a carga 100 % **rompe guardia** (§9.6) y hace **0** daño.

### 9.3 Combos de arma (melee)

En esta sección 🔳 = click izquierdo, 🔼 = click derecho.

| Secuencia | Resultado | Daño stamina (A) |
| --- | --- | --- |
| 🔳 | Ligero | **100** |
| 🔼 | Pesado | **250** |
| 🔳 + 🔳 + 🔳 | Combo básico | **100 + 100 + 150** = **350** (el tercero lleva **+50**) |
| 🔳 + 🔼 + 🔳 | Combo secundario | B+ (números luego) |
| 🔳 + 🔼 + 🔼 | Habilidad especial de arma | B+ |
| 🔳 + 🔳 + 🔼 + 🔼 | Habilidad especial de arma | B+ |

Aplica a espada, hacha, lanza, daga. **Etapa A:** se implementan ligero, pesado y combo básico. Ligero **cancelable a dash y a skill después del hit**, no en el startup. Data: `nexum-terra/data/melee.json`.

**Arco:** no usa esos combos. Skills propias (triple flecha, lluvia de flechas, etc.).

**Staff:** no tiene combos de arma. Permite ejecutar **combos mágicos sin pulsar modo maná**. Si cargas el staff a **≥ 50%** y sueltas, dispara magia del orbe/elemento activo (§8.6). Sin orbe, no hay disparo elemental.

### 9.4 Combos mágicos (etapas siguientes)

Mezclados con el arma equipada. El staff no hace nada en el combo salvo la regla del 50% de carga.

| Secuencia | Resultado |
| --- | --- |
| 🔳 + 🔳 + L2 | Ofensiva #1 |
| 🔳 + 🔼 + L2 | Ofensiva #2 |
| 🔳 + 🔼 + 🔳 + L2 | Ofensiva #3 |

### 9.5 Target

Se abandonó el click-sobre-enemigo como requisito (móvil y mando).

**Etapa A:** no hay lock. Hitbox / arco frente al facing. Cámara top-down **640×360** (20×11 tiles visibles). En 5v5, tinte de bando (§0.1), no nametag.

Gameplay Notion: [Gameplay](https://app.notion.com/p/Gameplay-2891e055ee8d813598c4d30709684bd1?pvs=21).

### 9.6 Guardia, rotura y choque de armas

Etapa **A** (tick de rooms). Números en `nexum-terra/data/melee.json`. La guardia es **estado** (input), no skill de hotbar. El tick no usa `if arma == espada` en el nodo del jugador: hit melee vs guardia vs hit melee es ruleset + data.

#### Barra de guardia

Baja **por hit**, no por tiempo. Pool **500** (referencia: 5 ligeros de 100). El hit resta su `base` de stamina (sin el 40 %; el 40 % es lo que llega al pool de stamina del defensor mientras la guardia aguanta).

| Defensa | Reducción de daño (stamina del cuerpo) |
| --- | --- |
| Escudo | **60–80%**, `item_def` (B; no hay escudo en A) |
| Brazos (A) | **40%** |

Si el pool llega a 0, la guardia **se rompe**. Hasta recargar la barra, no se levanta.

**Excepción:** un melee soltado a **carga 100 %** rompe la guardia **aunque el pool no esté vacío**. Ese golpe hace **0** daño. Stun **400 ms** + separación **1 tile** (misma forma que el choque, otra distancia).

#### Rotura de guardia (pool vacío o carga 100 %)

- Stun **400 ms**.
- Separación **1 tile** (no los 2 del clash arma-arma).
- Carga 100 %: daño **0**. Rotura por pool: el golpe que vacía aplica aún el 40 % al cuerpo (salvo que data diga lo contrario).

#### Choque de armas

Cuando dos golpes melee **conectan arma contra arma** (no contra hitbox de cuerpo):

- Las armas **rebotan**.
- Ambos se **separan 2 tiles**.
- Onda de choque (VFX + empuje).

Arco y staff no chocan así. Ventana de clash: al implementar.

---

## 10. Skills (catálogo)

Catálogo `nexum-terra/data/skills/*.json` (ids y nombres de data en inglés).

Tipos: **pasivas** y **activas**. Solo las activas van a hotslot.

Campos mínimos de una activa:

- `id`, `tags` de daño (`melee` | `mental` | elemento | …) y de gameplay (`cc` | `movement`)
- `damage_channel`: `stamina` | `vitality` | `mana_steal`
- `base` (daño estático), `tag_flat` opcional (p. ej. +50 de ese elemento)
- `cooldown_ms`, `resource_cost` (qué pool gasta el caster; maná o stamina)
- `range`, `hitbox` (shape + tamaño)
- `applies[]` / `self_applies[]` status effect ids (p. ej. root en súper técnica)
- `cast_ms` / `channel`

No hay `scale_atk` / `scale_pow`. El cliente usa el mismo `id` para animación. El servidor aplica números. “Dash invulnerable 200 ms” es un `status_effect` `invuln` en data, no `if skill == dash` en el player.

**Etapa A — kit base de Nexum Terra (las mismas 4 para todos).** Data: `nexum-terra/data/skills/kit-a.json`. Sabor del universo (samurái, ninja del cielo, magia como ciencia), **no** 1 skill por reino. El tick lee `skill_id`. Guardia sigue siendo estado, no hotslot. Sin i-frames en dashes (0 ms). Números v1 (la reducción de armadura y su CD post-off están marcados para playtest).

| Slot | `id` | Fantasía | Qué hace | Coste | CD |
| --- | --- | --- | --- | --- | --- |
| 1 | `iaido-dash` | Corte-paso samurái | Dash **adelante** 2,5 tiles. Sin daño. | 40 stam | 0,8 s |
| 2 | `reverse-dash` | Mismo arte, retirada | Dash **atrás** 2,5 tiles. Sin daño. CD **independiente** del dash. | 40 stam | 0,8 s |
| 3 | `kawarimi` | Reemplazamiento | Cuerpo falso 400 ms, invis 250 ms, blink 3 tiles. | 90 stam | 6 s |
| 4 | `magic-armor` | Armadura de maná | Toggle. **−30 %** daño **melee** recibido, **30 s**. Se apaga al vencer el tiempo **o** al pulsar de nuevo. **El CD (20 s v1) empieza al apagarse**, no al encender. Coste de maná **al activar**. Se acumula con guardia de brazos. Distinto de `rock-armor` elemental (§10.1). | 150 maná | 20 s tras off |

**No van a hotbar:** melee de arma, guardia.

**Luego:** genéticas de clan, profesión, pasivas generales y elementales. Elementales: §10.1. Obtención: §11.5.

### 10.1 Elementales — cuatro básicos

Requisito: orbe del elemento equipado (§8.6). Ids como en las páginas de Notion. Números de daño/CD no vienen en el export salvo el canal de `whirlwind`.

Cómo se aprenden y ramifican: §11.5. Activas elementales suben de nivel con el uso; pasivas elementales con puntos de habilidad. El tick no aplica este árbol en Fases 1–4.

#### Fuego (`fire`)

| id | Tipo | Qué hace |
| --- | --- | --- |
| `fire-incineration` | pasiva | Sube la probabilidad de quemar con tu fuego |
| `fire-combustion` | pasiva | Más daño de skills de fuego si tu salud está **< 50%** (vitalidad) |
| `fire-ashes` | pasiva | Reduce el daño **que te hace tu propio fuego** |
| `fire-ball` | activa | Bola en la dirección apuntada. Al subir de nivel: **3 bolas** (las extra a los lados de la original) |
| `fire-whip` | activa | Látigo de corto alcance; media circunferencia al frente. **No** puede cambiar de dirección ni moverse mientras dura |
| `sea-of-fire` | activa | Desde la boca, mar de fuego **cuadrado** al frente. **No** puede moverse mientras lo crea |

#### Viento (`wind`)

| id | Tipo | Qué hace |
| --- | --- | --- |
| `wind-evasion` | pasiva | Sube probabilidad de esquivar |
| `wind-hurricane` | pasiva | Sube probabilidad de **sangrado** en tus ataques |
| `wind-speed` | pasiva | Más velocidad de movimiento si hay viento en el entorno |
| `wind-cutter` | activa | Onda de **1 tile** en la dirección apuntada. Al subir de nivel: tamaño hasta **3 tiles** |
| `wind-slaves` | activa | 2 eslabones de viento en zigzag; se encuentran y se separan |
| `whirlwind` | activa | Remolino: atrae a  **3 tiles**, daño al colisionar. Canal **`stamina`**. Sangrado. Si atrae una skill de fuego, puede **quemar** |

#### Tierra (`earth`)

| id | Tipo | Qué hace |
| --- | --- | --- |
| `earth-strength` | pasiva | Más resistencia al **aturdimiento** |
| `earth-shaking` | pasiva | Más probabilidad de aturdir con tu tierra |
| `earth-roots` | pasiva | Si vitalidad **< 25%**: armadura de roca pesada **sin** gastar maná. Quita la armadura de roca anterior y esa queda inutilizable mientras esta está activa |
| `earth-spikes` | activa | Picos en cada tile en línea recta; daño al impacto. Alcance base **4 tiles**. Puede sangrar. Upgrade: +1 tile por nivel, **3** niveles, tope **+2 tiles** (fuente: ambos números; no redondear) |
| `earthquake` | activa | Daño a gente cercana. **Destruye criaturas de agua** si están sobre tierra; **no** si están sobre agua. Shake de cámara a los afectados |
| `rock-armor` | activa | Menos daño de espada y melee; evita **empuje**. Consume maná mientras está activa |

#### Agua (`water`)

| id | Tipo | Qué hace |
| --- | --- | --- |
| `water-flow` | pasiva | Más velocidad de movimiento al moverte **por el agua** |
| `water-impulse` | pasiva | Menos CD de skills de agua si estás **sobre agua** |
| `water-density` | pasiva | Más probabilidad de ralentizar con tu agua |
| `water-gun` | activa | Bala de agua rápida. La fuente describe además una **pasiva** (mismo id en el export) que dispara en **3 direcciones** (frente + 2 diagonales = V). Hasta tener id propio, es upgrade/pasiva de `water-gun` |
| `water-creatures` | activa | Criaturas que pegan a distancia y melee **mientras hay objetivo**. Sin objetivo **no se mueven** |
| `water-barrier` | activa | Cuadrado de agua **8×8** alrededor del usuario; encierra lo que esté dentro. Se sale con **flicker** (dash / reemplazo; unificar id al implementar) |

---

## 11. Progresión vs PvP

- **Lobby / práctica:** no XP, no drops, no rating.
- **PvP (Fases 3–4):** `match_records` + rating futuro. Caps de vitals pueden **equalizarse** al baseline (2000 / 2000 / 1500) o respetar gear de maná/stamina. El % por tag de equipo es la palanca de especialización; si el 1v1 debe ser fair, se capean o se ignoran mods de ítem en el ruleset `pvp_duel`. Decisión aún abierta; el modelo de tres pools no cambia.
- **PvE / mundo:** gear importa; drops al terminar la sesión. Mejora de activas por uso (cuando exista).

Clasificación D–SSS: §11.6. Rangos de reino: §11.7. Misiones, divisiones, renegados, KO/Death: §12.

### 11.1 Puntos de habilidad y de rebirth

Dos monedas distintas. No se mezclan. No existe una moneda “puntos elementales”: es el mismo recurso, y se llama **puntos de habilidad**.

**Puntos de habilidad:** compran **elemento**, **activas elementales** (obtenerlas) y **mejoran pasivas elementales**. En pasivas **generales** son opcionales: esas se entrenan; los puntos solo adelantan. Las activas elementales, una vez obtenidas, **no** suben de nivel con más puntos: suben **usándolas**.

| Regla | Valor |
| --- | --- |
| Al crear personaje | **50** puntos |
| Primer elemento | **0** puntos |
| Aprender un elemento nuevo | **10** |
| Aprender una profesión nueva | **10** |
| Una skill elemental | **1–15** |
| Un elemento + todas sus skills (incl. pasivas) | ~**10 + 30–80** (media de la fuente) |

Obtención: subir de **rango** (cantidades fijas por umbral de §11.6 / §11.7; el número de puntos por rango **aún no está en la fuente**), eventos, misiones (probabilidad de 1–2 puntos). También se **compran**: con monedas de juego (Hesedias u otras sinks, precios abiertos) y, de forma **importante**, con **dinero real** vía **Nexum Coin** / SKU de tienda (§17). El canal IAP es requisito de diseño, no un extra opcional. SKU y precio de puntos: abiertos.

**Puntos de rebirth:** 1 por cada rebirth. Solo mejoran **pasivas especiales** (§11.4).

### 11.2 Árbol de pasivas generales

Ramas I→V. Nivel I desbloquea II, etc., salvo requisitos extra anotados. Ids como en la fuente. Se **entrenan** (uso / kills); los puntos de habilidad pueden adelantar, no son el requisito. Esto **no** es el mismo sistema que las profesiones Médico / Espadachín / Sensor (§12.8); no fusionar. Las skills **elementales** no viven aquí: se compran con puntos (§11.1, §11.5).

Este árbol y las pasivas especiales de rebirth se seguirán afinando al implementar **Etapa B** (PVE de mazmorra / meta). No simular en Fases 1–4. No esperar al overworld.

#### General

| Rama | id | Efecto |
| --- | --- | --- |
| I | `explorer` | +5% velocidad de movimiento |
| I | `elementary-novice` | Controlar **1** elemento |
| II | `arcane` | +10% daño mágico |
| II | `elementary-apprentice` | Controlar **2** elementos a la vez |
| III | `fast-regeneration` | Más velocidad de regeneración de stamina (cuando aplica §12.6) |
| III | `elementary-master` | Controlar **3** elementos a la vez |
| IV | `extreme-survival` | Sobrevivir en condiciones extremas. Pequeña probabilidad de **no caer KO** al llegar vitalidad a 0 |
| V | `elementary-genius` | Controlar **4** elementos a la vez. Requiere `elementary-master` **y** `extreme-survival` |

#### Experiencia (portar equipo)

Encaja con el requisito de “habilidad de portar equipo de ese nivel” (§8). Se entrena matando enemigos con el equipo del nivel anterior (fuente: Aventurero).

| Rama | id | Efecto |
| --- | --- | --- |
| I | `expertise-novice` | Equipar armas/armaduras nivel 1 |
| II | `expertise-adventurer` | Nivel 2 |
| III | `expertise-hero` | Nivel 3 |
| IV | `expertise-champion` | Nivel 4 |
| V | `expertise-titan` | Nivel 5 |

#### Crafting

| Rama | id | Efecto |
| --- | --- | --- |
| I | `artisan` | Más probabilidad de loot de mayor calidad |
| II | `collector` | Más recursos al recolectar |
| II | `forger` | Crear armas/armaduras de mayor calidad |
| III | `alchemist` | Crear pociones con efectos beneficiosos |
| III | `charmer` | Encantar armas/armaduras |
| IV | `repairman` | Reparar equipo dañado (evita pagar al NPC) |
| IV | `extractor` | Recursos extra de enemigos derrotados |
| V | `inventor` | Crear objetos mágicos con efectos |
| V | `transmuter` | Transmutar objetos de un tipo a otro |

#### Guerrero

| Rama | id | Efecto |
| --- | --- | --- |
| I | `warrior-apprentice` | −5% CD de habilidades **básicas** |
| I | `warrior-survivor` | −10% daño recibido de enemigos de **nivel inferior** |
| II | `experienced-warrior` | +10% daño de habilidades. Requiere `warrior-apprentice` |
| II | `stealth-master` | +10% probabilidad de esquivar. Requiere `warrior-survivor` |
| III | `strategist` | −10% CD de habilidades **especiales** |
| III | `treasure-hunter` | Más probabilidad de objetos raros en cofres |
| IV | `conqueror` | +10% daño y velocidad de movimiento |
| V | `legend` | +15% salud máxima (vitalidad cap) |

### 11.3 Rebirth

Largo plazo. No hay código ni persistencia todavía.

**Requisitos** (fuente): rango **Élite** + **1 millón de Hesedias**. (Hesedias = moneda de mundo; no está en el modelo de primeras etapas.)

**Qué se conserva (cosas, no skills):** equipamiento, inventario y baúl. Capa de rebirth: recuento de rebirths, puntos de rebirth y pasivas especiales ya compradas (§11.4). Quintos clanes de **cuenta**.

**Afiliación:** se pierde **todo** vínculo con el clan (y reino) anterior. Naces de nuevo, literalmente: eliges clan y reino otra vez.

**Habilidades:** **ninguna** de la vida anterior se guarda. Ni genéticas de clan, ni elementales, ni de profesión, ni el árbol de pasivas/activas entrenadas o compradas con puntos de habilidad. Las vuelves a aprender en la vida nueva (§11.5).

**Qué más se reinicia:** clasificación (asesinatos y honor, §11.6), rango de reino (§11.7) y el personaje.

**Recompensas:** según el número de rebirth (p. ej. runas para subir nivel de ítems). **~3%** de nacer con un anillo legendario (referencia: anillo de Pureza). **+1 punto de rebirth.**

**Dificultad:** cada rebirth es más difícil hasta el **#10**; después se estabiliza (§7.2). UI: en selección de personaje, un **alma flotante**, no un slot vacío.

**Quintos clanes:** cada **5** rebirths puedes desbloquear **uno** de los 5.ºs clanes de los reinos (el jugador elige cuál). El desbloqueo es de **cuenta**. A **20** rebirths están los **4** (uno por reino jugable). Si un personaje de la cuenta tiene 5+ rebirths, otro personaje nuevo puede nacer en el 5.º clan ya desbloqueado. Esos clanes **no** están en `clans.json` (hoy hay 4 por reino).

**Abierto a propósito:** lista completa de recompensas por número de rebirth.

### 11.4 Pasivas especiales (puntos de rebirth)

No son el árbol de §11.2 ni skills de la vida anterior. Sobreviven al rebirth. Cada punto de rebirth se gasta aquí.

Patrón (ejemplo de diseño, no catálogo cerrado):

| Ejemplo | Cap | Por punto | Al máximo |
| --- | --- | --- | --- |
| Reducción de cooldown | 0/10 | +1% | 10% |

La UI de la fuente es del estilo: `Reducción de cooldown (10%) (0/10) (Aumenta 1% por cada punto de rebirth)`.

El resto del catálogo está en planificación. No inventar más filas. Se irá cerrando al desarrollar rebirth en **Etapa B**, no en Fases 1–4 ni en C.

### 11.5 Obtención de skills

Largo plazo. El árbol enseña las pasivas de cada sección (clan/familia, profesión, elemento, …) y las ramas que salen de las activas.

**Camino.** Aprender una habilidad exige avanzar en esa familia: si eres espadachín, matando NPCs con espada o con skills de espada; lo mismo por elemento (tras haberlo comprado), etc. No AFK.

**Pasivas generales** (§11.2). Se entrenan **sin** gastar puntos: uso / kills. Más rápido según el **nivel del NPC**. Primera skill + matar un **boss** → se obtiene al momento, con chance de la siguiente. Los **puntos de habilidad** son opcionales y **adelantan** ese progreso (si falta la mitad, se puede cubrir gastando un coste de skip). Ese coste es un número de **balance**, no un N de diseño: en la fuente era un placeholder; se fija al implementar, no hay valor canónico ahora.

**Elementales.** Se pagan con **puntos de habilidad**: 10 el elemento (0 el primero); **1–15** cada skill elemental al **obtenerla** (§11.1). Al comprar el elemento: **orbe** + **una activa ofensiva**.

| Tipo | Obtener | Mejorar / nivel |
| --- | --- | --- |
| Activas | Puntos de habilidad | **Uso** (no AFK). Los puntos no suben el nivel de una activa ya comprada |
| Pasivas | Puntos de habilidad | **Puntos de habilidad** |

**Ramas de activas** (ejemplo fuego): la bola de fuego es la primera; desde ella se abren **3**; desde cada una de esas 3, **2** (6 nuevas); desde cada una de esas 6, **1**. Cada nodo se **compra** con puntos cuando el árbol lo permite; el **nivel** de esa activa sube usándola. El catálogo de §10.1 lista ids; el grafo de desbloqueo se escribe en data cuando se implemente, no alinear a ojo con el recuento actual.

**Tras un rebirth:** este progreso de skills parte de cero (§11.3). El equipo que conservaste no otorga las habilidades de la vida anterior.

### 11.6 Clasificación (Nexumer)

Fuente: *Sistema de Clasificación*. Catálogo: `nexum-terra/data/classification-ranks.json`.

Un Nexumer tiene un **rango de clasificación** de **D** a **SSS**. Más adelante se pueden añadir **SSSS** y **SSSSS**; no hay umbrales para esos hasta que se escriban aquí.

Subir de rango exige **los dos** umbrales a la vez (AND): **Honor** y **asesinatos**. Honor es el contador ligado a **misiones completadas** (y otras fuentes cuando existan; no inventar tasas). Asesinatos son kills persistidos del personaje.

| Rango | id | Honor mín. | Asesinatos mín. |
| --- | --- | --- | --- |
| D | `d` | 0 | 0 |
| C | `c` | 30 | 10 |
| B | `b` | 100 | 30 |
| A | `a` | 150 | 50 |
| S | `s` | 300 | 100 |
| SS | `ss` | 800 | 300 |
| SSS | `sss` | 1500 | 500 |

El rango **se deriva** de los contadores; no se elige a mano. El lobby de práctica **no** suma honor ni asesinatos.

**Por era:** en A se persisten asesinatos de rooms PvP cuando el KO cuenta como kill (§12.5). Honor puede subir en A con **misiones de rooms** (ej. gana 3 1v1); las misiones de reino / escolta son B o C (§12.1). Sin misiones de honor, el HUD puede quedar en D. Un rebirth pone honor, asesinatos y rango de clasificación a D / 0 / 0 (§11.3).

Puntos de habilidad “al subir de rango”: cantidades fijas **aún no listadas** en la fuente; no inventar la tabla de puntos.

Esto **no** es el rango de reino (§11.7) ni las Divisiones (§12.2). Los asesinatos y el honor que se ganan/pierden al matar siguen §12.5.

### 11.7 Rangos de reino

Fuente: *Rangos de Reinos*. Catálogo: `nexum-terra/data/kingdom-ranks.json`. Distinto de la clasificación §11.6. Hay **6** rangos previstos; cuatro están nombrados. Los especiales (Sabio, Héroe y otros) se idean después; no hay ids ni requisitos.

| Orden | id | Nombre | Cómo se obtiene |
| --- | --- | --- | --- |
| 1 | `novice` | Novicio | Al crear personaje (y tras rebirth). Estado inicial. |
| 2 | `soldier` | Soldado | Superar el **desafío mundial** de supervivencia (abajo). Requisitos de entrada: abiertos. |
| 3 | `commander` | Comandante | **Automático** al tener **150 Honor** y clasificación **B** (§11.6). |
| 4 | `elite` | Élite | Top **3** del torneo mundial (abajo). |
| 5–6 | — | Sabio, Héroe, … | Pendiente. |

**Etapa A:** todos los personajes son Novicio (display). No hay desafío ni torneo.

**Soldado — desafío.** Los novicios que cumplan requisitos (aún no listados) entran a una misión estratégica de supervivencia: arena cerrada, combate **entre jugadores y contra NPCs**. Fantasía de evento mundial; implementación como **room de evento** (B) o evento de overworld (C). **No** es un mapa de región ni AOI. No prototipar el overworld para esto.

**Comandante.** Umbrales 150 Honor + rango B. Puede **crear una Orden**: poseer territorios del reino y gestionar la economía de ese territorio. Órdenes y territorios = **Etapa C** (`PLAN.md` §18, GDD §15). El título de Comandante puede existir en B; crear Orden no.

**Élite — torneo.** Organizado en **Aurora** (fantasía). Se eligen combatientes de cada reino; **3** se convierten en Élites **ese día** (puestos 1–3). Mínimo **8 participantes** (número par). Bracket: **4** combates iniciales → **2** semifinales → **1** final. Las primeras 4 **no** van en paralelo: **misma arena**, en serie, para que se puedan ver todos. En B esto es un **room de evento** (ruleset de bracket); la sede Aurora en overworld es C.

Recompensas por puesto: **abiertas**. Intención: que valga la pena el 1.º (candidato: **1 punto de rebirth**). Un premio fuerte + requisito Élite para rebirth (§11.3) empuja a **rebirthear para volver a entrar** al torneo y al desafío de Soldado.

Rebirth exige rango **Élite** (§11.3); al renacer el rango de reino vuelve a Novicio.

---

## 12. Misiones, mundo social y profesiones

Fuentes volcadas: Misiones, Divisiones, Renegados, Robo, KO y Death, Regeneración, Logout bajo ataque, Médico, Espadachín, Sensor, skills de 8 clanes (§12.9). Ocho clanes aún sin PDF.

Números con ceros: el export de Notion pierde glifos `0`/`D`/`R` en CID; aquí se reconstruyen con el resto del GDD (p. ej. 150 Honor = suelo de Élite/Comandante en §11.6–11.7). Si un valor no encaja al implementar, se corrige aquí, no en el tick.

| Pieza | A (rooms) | B (mazmorras + meta) | C (open world) |
| --- | --- | --- | --- |
| KO / death de combate | sí (conteos, minijuego; loot de cadáver **no**) | sí; respawn de mazmorra §16.4 | sí + cadáver 45 s + hospital |
| Misiones de contador (gana N 1v1, etc.) | sí, catálogo pequeño | más PvE de instancia | + mundo |
| Misiones de reino (honor tabla D–S) | no hace falta el mapa | honor de mazmorra / meta si se cataloga | escolta / robo de mercancía |
| Divisiones | no | no (exige Élite) | sí (overworld + uniformes) |
| Renegados / desertar reino | no (clan de nacimiento fijo) | ítem de redención se puede vender en meta | efecto de mundo |
| Robo de cadáver | no | no | sí; % por zona §15.2 |
| Logout “Bajo ataque” | rooms: disconnect = forfeit del match, no cadáver de mundo | igual | sí (30 s, cadáver) |
| Profesiones | no | sí (10 puntos de habilidad, §11.1) | se reutilizan |
| Regeneración | quieto al 50 %; hold Z desde reserva; sillas del lobby si hay tile | pociones + reposo de instancia | sillas/camas de mundo |

### 12.1 Misiones

Clasificación de **dificultad** D / C / B / A / S. Más rango → más recompensa. Las de **reino** suelen dar **honor y dinero** (Hesedias). Honor por rango de misión:

| Rango de misión | Honor |
| --- | --- |
| D | 1 |
| C | 1 |
| B | 3 |
| A | 5 |
| S | 10 |

El catálogo se **añade con el juego**. No hay lista cerrada en la fuente; el brainstorm admite cosas como “matar 5 miembros de un reino”.

**Etapa A / B (rooms e instancias).** Catálogo A (`nexum-terra/data/missions.json`):

| id | Texto |
| --- | --- |
| `win-3-duels` | Gana 3 matchs 1v1 |
| `play-10-matches` | Participa en 10 matchs 1v1 o 5v5 |
| `win-5-arenas` | Gana 5 matchs 5v5 |

Recompensas: se añaden después. El `progress` lee `match_records`. Más misiones se agregan **a medida**.

**Etapa C — escoltar y robar mercancía** (misión **compartida**: varias personas la toman y cooperan).

Cargar mercancía **ralentiza** el caminar (NPC y jugadores que la cargan).

**Escoltar.** Un NPC lleva carga de un punto a otro. La misión **empieza** al hablarle. Todos los que la tomaron pueden escoltar.

**Robar.** En cuanto alguien **empieza** a escoltar, aparece en los **otros reinos** la misión de robar esa mercancía. Los ladrones matan al NPC, toman la carga y la llevan al **mercado negro**.

**Recuperar.** Los escoltas pueden quitar la carga a los ladrones y seguir la ruta del NPC: entonces **un escolta** camina lento con la carga (ya no el NPC). Eso **priva** de muchos movimientos y habilidades; el resto debe protegerlo.

No prototipar rutas de escolta, mercado negro ni NPCs de mundo en A/B.

### 12.2 Divisiones

Cada reino tiene **2** organizaciones. Cualquier mago se une **a partir de rango Élite** (§11.7, torneo). Dan **habilidades especiales** y **uniforme**; refuerzan la fantasía del reino.

| Reino | División | Uniforme / notas | Requisito extra en fuente |
| --- | --- | --- | --- |
| Terrara | Caballería Bélica | Full armadura hierro/acero | Alta Fuerza o Poder |
| Terrara | Magos Blancos | Túnicas blancas; skills médicas para sanar | Alto Control |
| Spectra | Escuadrón de Torturas | Vestimenta oscura, aspecto sanguinario | — |
| Spectra | Infiltración | Nombre solo; kit abierto | — |
| Aerion | Brigada de Reconocimiento | Vestimenta ligera, camuflaje | — |
| Aerion | Maestros de las Barreras | Barreras impenetrables; cubren el cuerpo propio y el de aliados | — |
| Fontaine | Equipo de Investigación | Vestimenta científica; buffs de **crafting** y afines | — |
| Fontaine | (falta) o Infiltración Tecnológica | Segunda división **no cerrada** en la fuente | — |

**Unirse.** Se puede cambiar de división cuando se quiera (del mismo reino). Coste: **50 Honor**. Hay que ser Élite **y** tener esos 50 **por encima** del suelo **150** Honor: se entra con **200** y, al pagar, se vuelve a **150**.

**Mantener.** **2 Honor por día** (14 por semana ≈ 3 misiones B de §12.1 para cubrirlo).

**Perder.** Si el Honor cae **por debajo de 150**, se pierde la membresía **y** el rango Élite según esta fuente. Eso choca con Élite-por-torneo de §11.7: al implementar, el suelo 150 corta la **división**; si también despoja el título de torneo, se decide aquí, no en silencio.

Fuerza / Poder / Control de la tabla son requisitos de **fantasía** de la fuente; el modelo de daño actual no tiene esos stats (§6). No inventar umbrales hasta alinear con vitals / profesión.

### 12.3 Renegados (desertores)

- **¿Cambiar de reino?** No. El reino nace del **clan** que elegiste (§3.1). (El rebirth **sí** corta afiliación y elige otro clan/reino, §11.3; no es “cambiar de reino” en vida.)
- **Abandonar el reino** → **desertor** (renegado).
- **Volver a un reino:** ítem de redención, **10 000 Hesedias** o **100 NC**. El rey **no** puede meter a un desertor gratis; paga el jugador.
- **¿El rey expulsa?** No (abuso).

Etapa A: nadie deserta. El ítem puede existir en catálogo Commerce/Inventory en B; el estado “sin reino” en overworld es C.

### 12.4 Robo (cadáveres, Etapa C)

Al **morir** (no al KO) queda cadáver un tiempo (§12.5: **45 s**). Se saquean ítems **encima** del cuerpo; la **cantidad** la fija el **color de zona** (§15.2). Por eso bancos y baúles importan.

No es auto-loot: hay que **acercarse**. Abrir el menú de robo tarda **unos segundos** (número exacto abierto), **se interrumpe con cualquier golpe**, y **nadie “Bajo ataque”** (§12.7) puede abrirlo.

Sirve para que un compañero lootee, para que varios peleen el cadáver, o para que **no dé tiempo** y se protejan los bienes del aliado.

Rooms A/B: no hay cadáver de mundo ni menú de robo.

### 12.5 KO y Death

Vitalidad a **0** → **KO**, no muerte inmediata.

**Levantarse** es una **elección** (no auto-stand). Si no te esfuerzas (soltar la tecla o dejar pasar el tiempo) → **mueres**.

**Etapa A (rooms):** mantener **una tecla** el tiempo de canal de la tabla de abajo (Novicio = 8 s). Sin minijuego. La tecla no puede ser la de guardia (X); bind concreto al cerrar input §9.1.

**Más adelante:** fantasía de minijuego (puntería, orbes/espíritus); más **conteo de KO** → más difícil; el **último** KO del cupo debe ser **casi imposible**. El hold de A queda como fallback.

**Cupo de KOs según rango de reino** (§11.7):

| Rango | KOs antes de death |
| --- | --- |
| Novicio, Soldado | 1 |
| Comandante | 2 |
| Élite | 3 |

Cada KO abre una ventana de **30 s** para morir si no te levantas. El **%** es cuánto de esos 30 s hay que canalizar para levantarse:

| Nº de KO | Canal para levantarse | Tiempo de canal |
| --- | --- | --- |
| 1 | 26,66 % de 30 s | 8 s |
| 2 | 50 % | 15 s |
| 3 | 80 % | 26 s |
| 4+ | 150 % cada vez | un médico puede aportar ese %; **se puede interrumpir** |

Al vencer los 30 s en el suelo, **mueres** sea cual sea el número de KO.

**Penalización de movimiento:** a partir de cierto conteo corres más lento — **1** KO acumulado si eres Comandante, **2** si Élite. En el **último** KO del cupo **no puedes correr**.

**Al levantarte:** vitalidad y stamina al **10 %**. Maná y reserva de maná **no** se tocan. Para bajar el conteo de KO hace falta vitalidad al **100 %** (§12.6).

**Death / cadáver (overworld):** el cuerpo dura **45 s**. Tras fallar el KO: mueres → **15 s** → spawn en **hospital** → **1 min 30 s** para poder levantarte y salir.

**Honor y asesinatos al morir** (clasificación §11.6, no rango de reino):

Cuando **tú mueres:**

- Pierdes honor (y “otras cosas” no listadas) si el asesino es de clasificación **igual o inferior**.
- Igual rango de clasificación: **−2** Honor.
- Asesino inferior: **−5** Honor **por cada rango** de diferencia (2 rangos abajo → **−10**).
- Excepción **Special** (S/SS/SSS…): si un Special mata a un SSS, el SSS pierde solo **5**. Si **tú** eres Special, pierdes **3** Honor (da igual el número de S) **más 1** por cada S que te falte respecto al asesino. Detalle fino de “Special” vs umbrales S/SS/SSS: al implementar, no inventar un rango nuevo.

Cuando **tú asesinas:**

- Si la víctima es de clasificación **igual o superior**, ganas **1 asesinato**.
- Ganas el **mismo** Honor que pierde la víctima.

En **rooms** A/B: mismos conteos de KO y kill/honor de clasificación; **sin** cadáver, hospital ni % de zona. Práctica: sin persistir (§11).

Mazmorra: wipe de party y reentrada §16.4; el KO de combate es este apartado.

### 12.6 Regeneración

Fuente: *Sistema de Regeneración*. Pociones: segundos de beber y cap por match **abiertos**. Sillas A: **30 s** a full (§ abajo). Hold Z: **5 % cap / s**.

**No hay regen automática genérica** (andar, pelear, o un stat de personaje que suba pools al 100 %). Lo que pase de las reglas de abajo es **equipamiento** (`item_def` / orbes), no un tick aparte.

Jerarquía (la misma que el overflow de daño, §6.3, al revés del “pozo”):

```
Maná → Stamina → Vitalidad
```

#### Quieto (techo 50 %) — solo Etapa C (overworld)

Hay que estar **quieto** (sin movimiento). Cada pool solo sube **hasta el 50 %** de su cap. **No** aplica en lobby, `pvp_duel`, `pvp_arena` ni `pve_dungeon`.

Orden **estricta**, un peldaño a la vez:

1. Regenera **maná** hasta 50 %.
2. Si el maná está **≥ 50 %**, regenera **stamina** hasta 50 %.
3. Si la stamina está **≥ 50 %**, regenera **vitalidad** hasta 50 %.

Si el maná ya iba al 80 %, se salta al paso 2. Moverse corta esta regen. **No** baja conteos de KO.

#### Reposo (sillas / camas)

Tiles de reposo. En **Etapa A** el lobby **tiene** sillas/camas. En mazmorra/arena, solo si el mapa las coloca. En C, también en mundo.

Ahí, en **30 s**, se llenan **vitalidad, stamina, maná y reserva de maná** (tasas = cap / 30). Hold Z sigue existiendo fuera de la silla.

El **conteo de KOs** puede bajar hasta **0** en reposo, pero **exige vitalidad al 100 %**. Sin eso, el conteo no se mueve.

Cualquier **curación de vitalidad** que la lleve al 100 % (médico, poción de vida, etc.) puede **bajar el conteo de KO a la vez** que rellena stamina y maná (si la cura o el contexto lo permite; no inventar que una cura de 10 HP vacíe tres KOs).

#### Pociones

Se compran pociones de **maná** y de **vida** (Hesedias; catálogo Etapa B). **No te puedes mover** unos segundos mientras las bebes. Son **más rápidas** que las sillas.

**Arena y mazmorra:** se **pueden** beber en cualquier momento del match (el rival **no** tiene que estar KO). **No te puedes mover** mientras. Cap de usos por match: abierto.

**Quieto (techo 50 %):** **solo open world (Etapa C).**

**En arena / mazmorra, recuperación permitida:** (1) recargar maná desde la **reserva** (hold Z); (2) pociones (desde B). Nada más, salvo sillas si el mapa las tiene.

Etapa A: hold Z sí. Pociones **no** (sin inventario). Lobby: sillas sí, quieto no.

### 12.7 Logout “Bajo ataque”

Al **recibir un ataque** de alguien: estado **Bajo ataque**.

- No permite **pasar a otras zonas** del mapa (C).
- Caduca **30 s** después del último golpe recibido.
- Si te **desconectas** bajo ataque: dejas **cadáver**, se cuenta la **muerte**, recompensa al **último** que te pegó.
- Si te desconectas **en KO**: igual (cadáver + muerte + reward al último atacante).

Etapa A/B: no hay zonas que cruzar. Disconnect en duelo/arena = **forfeit** del match (ruleset); no spawnea cadáver de §12.4.

### 12.8 Profesiones (Etapa B, antes del open world)

Distinto del árbol de pasivas generales (§11.2). Aprender una profesión: **10** puntos de habilidad (§11.1). Números de cura/daño/CD: abiertos (data al implementar). Canal de daño de skills médicas: **vitalidad** salvo que data diga otra cosa. Sangrado / veneno / taunt = status effects del tick.

#### Médico (`medic`)

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `healing-mastery` | Más vida recuperada con skills médicas |
| `protection-aura` | Aura permanente: daño de **tu reino** al **75 %**; de tu **party** al **0 %** (100 % reducción) |
| `persistent-poison` | Más duración del secundario Veneno en víctimas |
| `poison-resist` | **−85 %** daño de venenos ajenos; **inmune** a los propios |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `heal` | Cura instantánea al jugador en la misma dirección. Si deja vitalidad al **100 %**, puede bajar conteo de KO (§12.6) |
| `heal-aura` | Cura aliados en área. Alt: cura a **cualquiera** dentro del aura |
| `potion` | Quita **un** estado negativo (el de **más daño** primero) |
| `resurrection` | Revive a un aliado **antes** de que el cadáver desaparezca. **15 s**, interrumpible. Coste: **50 %** del maná y **+1** conteo de KO al médico. El revivido: maná que tenía **+75 %**, stamina y vitalidad al **75 %**, y **3** conteos de KO si 3 es el máximo de su rango |
| `poison-bomb` | Bomba **3×3**; Veneno + daño directo si pega a un jugador |
| `toxic-cloud` | Nube **6×6**; DoT/s (DoT) y chance de secundario (DoT fuera de la nube) |

#### Espadachín (`swordsman`)

Mejora el uso de **espadas** frente al resto del arma melee.

**Pasivas:** `sword-mastery` (más velocidad de ataques con espada); `tenacity` (**−50 %** daño de sangrado); `iron-will` (**−25 %** daño de efectos secundarios).

**Activas:** `simple-cut` (línea frontal, sangrado); `double-cut` (dos cortes en X, cono, daño moderado, sangrado); `charge` (dash-corte, stun + sangrado); `shadow-step` (invulnerable, gran velocidad, breve; Flicker **controlable** y un poco más largo); `slash` (giro en área, sangrado).

#### Sensor (`sensor`)

**Pasivas:** `mana-control` (menos maná para ocultar presencia); `mana-knowledge` (ver vitals del objetivo en **3 etapas**: maná → stamina → vitalidad). En discusión: gastar un **slot elemental** para ver secundarios y KOs como Omnivisus — **no cerrado**. `night-vision` (mejor visión en oscuridad).

**Activas:** `hide-magic` (nadie ve info sensorial ni la **identidad**); `detect-presence` (flechas en órbita: dirección, **color de maná**, distancia); `false-presences` (**5** clones, taunt fuerte, `group_name` Taunt); `distortion` (invisible **mientras no te mueves**).

### 12.9 Skills de clan (Etapa B; catálogo vacío en A)

Genéticas del **clan de nacimiento**. Cómo se aprenden: §11.5. El rebirth las borra todas (§11.3). En Etapa A `clans.json` sigue con `skills: []` y el tick **no** lee el clan para daño (§3.1). Números de daño/CD/consumo: **abiertos** salvo lo escrito aquí. Ids propuestos para data; no hay `if` en el nodo del jugador.

**Volcados (8):** Omnivisus, Gadgetrix, Entomante, Nachtsoldaten, Pulmonarius, Sangrafilos, Umbromante, Geisteswaffen. **Sin PDF (8):** Sagitta, Ciberlance, Herbora, Pyrofauces, Stellamante, Vitamancers, Crystallomante, Sonomantes. Quintos clanes: §11.3.

Glifos `0`/`×`/`(`/`)` reconstruidos del CID de Notion como en el resto de §12 (p. ej. áreas **10×10**). Si al implementar un valor no encaja, se corrige aquí.

**Reglas compartidas**

- Canal de daño: el de cada skill en data (`stamina` melee, `mana` el resto, salvo Entomante `mana_steal` / Sangrafilos sangrado).
- Visión de “una región entera” (Omnivisus): fantasía de **Etapa C**. En B = el mapa de la **instancia**.
- No prototipar overworld para range, sombras de mundo ni “noche” de servidor: en rooms/mazmorra, “noche” es flag del ruleset o se deja apagada hasta C.

#### Omnivisus (`omnivisus`, Fontaine)

Todas las habilidades **exigen el ojo activo**. Fantasía: ojos científicos; melee a puntos vulnerables / sistema mágico interno.

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `omnicontrol` | Menos consumo de maná de skills Omnivisus |
| `precision` | Sube la chance de secundarios Omnivisus de **0 a 100 %** |

**Maestría del ojo** (`omnivisus-mastery`): 3 desbloqueos al entrenar.

| Orden | Efecto |
| --- | --- |
| Inicial | Círculo bajo las personas; según “fuerza” muestra **cantidad de maná** |
| 2 | El círculo cambia de color / aura elemental (**5** elementos o compuestos) para leer el tipo de skill que viene |
| 3 | Card flotante con target: vida, stamina, maná y secundarios |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `range-extension` | Ve más mapa que el resto |
| `repulsion` | Campo en un diámetro: empuja jugadores u objetos dentro del círculo |
| `fist-of-uncontrol` | Golpe: la víctima **gasta más maná** al usar skills |
| `fist-of-vertigo` | Golpe: invierte movimiento (**A↔D**, etc.) |
| `twin-eagles` | Un águila por mano; cargas de puño las lanzan; puños normales que tocan **roban maná** |

#### Gadgetrix (`gadgetrix`, Fontaine)

Gadgets de protección y ataque (propio o aliados). Fuente: “habilidades de **1** familia”. **3** bots invocables (los de abajo). Crafting de orbes: **+25 %** de éxito (§8.7); apagado mientras el clan no esté en data.

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `crafting` | +probabilidad de éxito en **cualquier** crafting, hasta **25 %** |
| `attack-ai-companion` | Inicial. Ataca físicamente al objetivo y a quien golpee al dueño. Ofensivo, controlable a distancia |
| `evasion-ai-companion` | Bot volador pequeño: evade, entrega auto-target, mantiene target fijo. Si profesión **Médico** (§12.8): **sana** aliados cercanos |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `laser-barrier` | 4 bots a las esquinas de un cuadrado **10×10**. Al unir láseres **nadie sale** |
| `chain-sacrifice` | Muchos bots en un círculo grande; caminan hacia un círculo chico. Colisión → explosión en cadena; círculo chico → **gran** explosión |

#### Entomante (`entomante`, Spectra)

Insectos en el cuerpo. **Dos tipos:** los que **comen** maná y los que **almacenan** maná. Debilidades de los que comen: Fuego, Agua, Viento, Rayo y Armadura de Tierra. El clan debe **ver** con facilidad al invadido (indicador). Canal `mana_steal` cuando exista.

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `insect-mastery` | Distinguen tu maná del ajeno |
| `mana-store` | Más maná almacenado y entregado al usuario. Tope: **75 %** del maná de la víctima |
| `strong-bite` | Más fuerza para comer armadura de tierra |
| `elemental-resist` | Más chance de no morir ante fuego, agua, viento y rayo |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `invade-body` | Puño: deja un insecto. Detecta el cuerpo real, lo encuentra a larga distancia o ataca. Al **KO** el insecto vuelve. **Imposible** saber si te invaden |
| `steal-mana` | El invasor roba maná; el usuario lo recibe **cuando el insecto vuelve** |

Los siguientes envían come-maná en su mayoría; algunos llevan almacenadores → chance de **invadir cuerpo**.

| id (propuesto) | Efecto |
| --- | --- |
| `swarm` | Línea recta, **6** tiles de altura. Al tocar, insectos se quedan **comiendo** maná (no lo roban) |
| `insect-barrier` | **10×10**; se puede salir, pero tocar la barrera deja come-maná |
| `insect-shield` | Cubre el cuerpo, menos daño. Melee recibido → insectos al golpeador y comen su maná (pequeña chance de invadir) |
| `insect-wave` | Círculo que crece desde el usuario |
| `insect-storm` | Libera todos + nube **8×8** centrada en él. **Gran** consumo de maná |

#### Nachtsoldaten (`nachtsoldaten`, Spectra)

Al nacer reciben la espada del clan y se **bloquea el slot de mano izquierda**. “Natchsoldaten” en la fuente = este clan.

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `nachtsoldaten-sword-mastery` | Menos maná al usar la espada; **evoluciona** la espada |
| `dark-sword` | 1.ª etapa: misma velocidad que Luna, **menos** daño |
| `moon-sword` | Más daño que Oscura |
| `night-sword` | Más daño **y** velocidad que Luna |
| `night-domain` | Todos los ataques de esta espada (melee o técnicas) pasan a **base de poder mágico** |
| `lunar-blessing` | Hijos de la Luna: menos daño cuando “el tiempo favorece” = **más resistentes de noche** |
| `cut-mastery` | El Corte nocturno no te daña con sus explosiones/ataques |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `moon-blade` | Invoca espada (alma + noche). Larga, una mano, algo pesada, cuerda en el mango. Evoca técnicas y evoluciona. Se desbloquea **Giro celestial** |
| `throw-sword` | Arroja al frente a alta velocidad; reaparece en la mano al destruirse |
| `celestial-spin` | Domo giratorio de daño alrededor. Con **Espada de la noche** puede **cancelar** técnicas que le arrojen |

Con **Espada de la noche**:

| id (propuesto) | Efecto |
| --- | --- |
| `night-cut` | Canal continua; el **próximo básico** suelta un haz según carga. Empieza al ejecutar la skill; **puedes moverte**; se libera con slash o con `tensa`. La espada se ilumina por cargas |
| | **≤ 30 %:** haz de **1** tile, muy rápido |
| | **≤ 60 %:** haz de **3** tiles, velocidad normal |
| | **100 %:** haz de **3** tiles, alta velocidad, daño **explosivo** al impactar |
| `tensa` | Requiere Corte nocturno en carga. Melee: suelta el % acumulado en corte explosivo + onda. Golpea al nivel del % |

#### Pulmonarius (`pulmonarius`, Terrara)

Fuerza sobrehumana, distancia y melee. **Cambio de sol cada 2 s.** Cambiar consume **maná y stamina** (anti-abuso).

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `sun-power` | Desbloquea soles según maestría |
| `inner-sun` | Más fuerza solar que el sol central reparte al cuerpo **de noche** |
| `solar-resist` | Hasta **−50 %** consumo de maná al usar poder del sol |
| `stellar-speed` | Más chance de **esquivar automático** ataques físicos y armas |

Maestría de soles (vía `sun-power`):

| Sol | Umbral |
| --- | --- |
| Norte | Inicial |
| Sur | **25 %** |
| Este | **50 %** |
| Oeste | **75 %** |
| Centro | **100 %** |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `opening` | Entra en respiración y enfoca el Sol (beneficios del sol activo) |

**Apertura — beneficios por sol**

| Sol | Foco | Trade-off |
| --- | --- | --- |
| Norte | Parte superior / postura | Menos daño físico recibido; **menos fuerza** |
| Sur | Piernas | Más velocidad; **menos resistencias** |
| Este | Brazo derecho | Más fuerza, menos stamina; ágil y abrasadora |
| Oeste | Brazo izquierdo | Más fuerza, menos stamina; más precisa y directa |
| Centro | Corazón (bombea más sangre) | Beneficios **menores** que los otros; **menor** consumo de stamina y maná |

`stellar-meteors` y `fleeting-trail` son técnicas físicas: **varían con el sol** y el daño sigue las capacidades físicas del usuario.

**Meteoros estelares** (`stellar-meteors`)

| Sol | Forma |
| --- | --- |
| Norte | Corto alcance: varios golpes consecutivos al de enfrente |
| Sur | Línea: el usuario avanza golpeando en recta a alta velocidad |
| Este | Ráfaga de puños / proyectiles de aire comprimido al frente |
| Oeste | Un cañón de aire en línea; puede desplazarse hacia donde apunta |
| Centro | Ráfagas rápidas y cierra con el cañón del brazo; fricción tipo llamas |

**Estela fugaz** (`fleeting-trail`)

| Sol | Forma |
| --- | --- |
| Norte | Dos patadas en combo corto alrededor del oponente (sensación: Konoha senpuu) |
| Sur | Se abalanza en línea como flecha ardiente |
| Este | Ráfaga de patada / proyectiles de energía |
| Oeste | Versión concentrada: aire hacia el objetivo |
| Centro | Patada envuelta en maná: bola de fuego en la dirección apuntada |

| id (propuesto) | Efecto |
| --- | --- |
| `stellar-descent` | Aparece detrás, sube a la víctima y cae en rotación hasta estrellarle la cabeza en el suelo |

#### Sangrafilos (`sangrafilos`, Terrara)

Metal / huesos de acero; combate de **sangrado**. La **armadura ósea** es superior a los **5** elementos (rompe tierra, aparta agua, etc.).

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `metamorphosis` | Convierte huesos en acero. Se entrena consumiendo el **elixir secreto** de los Sangrafilos, que sale en **cofres de mazmorra** (Etapa B, §16) |
| `bone-resist` | Más resistencia a golpes cuerpo a cuerpo y de espada |
| `bone-control` | Menos tiempo para dar forma a los huesos de acero |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `bone-armor` | Cubre la piel; reduce daño elemental, de espada y melee |
| `steel-fingers` | **10** dedos al frente como balas en **V**; el centro de la V es **1** tile frente al usuario |
| `steel-lance` | Barra puntiaguda al frente a alta velocidad |
| `steel-whip` | Látigo (cuchillas + punta de lanza); gira y daña lo cercano |
| `thoracic-strip` | Costillas como cuchillas en **8** direcciones en órbita |

#### Umbromante (`umbromante`, Aerion)

Dominio de la sombra. Referencia de sensación: Nara. Notas de fuente “(Aragami)” = idea de viaje entre sombras, no un 5.º clan.

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `shadow-mastery` | Mejor control de sombras |
| `shadow-range` | Más distancia que recorre la sombra |
| `concentration` | Menos maná al controlar la sombra |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `shadow-extension` | Círculo **5×5**; atrapa los pies de quien cae dentro. Se puede usar manipulación de sombra **sin moverse** mientras está activa |
| `shadow-pierce` | Sombra sólida atraviesa a la víctima **atrapada**; **sangrado** |
| `shadow-needles` | Agujas en las **8** direcciones; sangrado al impacto |
| `shadow-travel` | Viajar de una sombra a otra |
| `create-shadow` | Crea una sombra temporal a la que puedes viajar |

#### Geisteswaffen (`geisteswaffen`, Aerion)

Clan antiguo: poder en herramientas; espada moldeable de tecnología; **medio alcance**.

**Pasivas**

| id (propuesto) | Efecto |
| --- | --- |
| `infinite-reserve` | Regenera **reservas de maná** automáticamente; **bajo ataque** (§12.7) baja un **gran %** de esa regen |
| `mana-control` | Hasta **−10 %** consumo de maná de **todas** las skills |

**Activas**

| id (propuesto) | Efecto |
| --- | --- |
| `sharp-tsunami` | Cuchillas delante que se extienden a alto rango |
| `quick-draw` | Espada de maná a alta velocidad; daña a todos los cercanos (**1** o **1,5** tiles) |
| `sword-barrier` | **4** espadas persiguiéndose en un cuadrado **10×10**. Tocar = herida grave y esa espada se clava y desaparece. Con las 4 juntas la velocidad impide salir sin tocarlas |
| `minefield` | Minas alrededor: explotan al contacto o a los pocos segundos |

---

## 13. Audio (nota)

Bandas sonoras citadas: **Base**; **Base Boss — Tenebroso — Acción**. Catálogo de música fuera de este GDD hasta haber lista.

---

## 14. Trabajo pendiente de diseño (aquí, no en PLAN)

- [x] Volcar skills de clan (§12.9) — 8/16; faltan Sagitta, Ciberlance, Herbora, Pyrofauces, Stellamante, Vitamancers, Crystallomante, Sonomantes
- [x] Sillas: 30 s a full (vit/stam/maná/reserva). Hold Z 5 % cap/s. Cap pociones por match: abierto (B).
- [x] Misiones A: win 3 1v1; participa 10; gana 5 5v5. Más a medida. Recompensas luego.
- [ ] Segunda división de Fontaine; kit de Infiltración Spectra
- [x] KO en rooms A: hold de tecla (tiempos de §12.5). Minijuego = más adelante. Bind de tecla al cerrar input.
- [x] Guardia: pool 500 por daño; carga 100 % rompe a 0 dmg, stun + 1 tile. Clash 2 tiles.
- [ ] % exacto por `item_def` de escudo (banda 60–80; B). A = brazos 40% solamente.
- [ ] Puntos de habilidad por umbral de clasificación / rango de reino
- [ ] Requisitos de entrada al desafío de Soldado
- [ ] Recompensas 1.º / 2.º / 3.º del torneo Élite (candidato: 1 punto de rebirth al 1.º)
- [ ] Rangos especiales de reino (Sabio, Héroe, …) y umbrales SSSS / SSSSS
- [ ] Catálogo de pasivas especiales de rebirth (además del ejemplo de CD)
- [ ] Quintos clanes: ids, fantasía y skills; no están en `clans.json`
- [ ] Tienda de puntos de habilidad: precios en Hesedias y/o NC; SKU IAP concreto (canal obligatorio, §17)
- [ ] Catálogo v1 de SKU de microtransacciones (cosmetic / currency / power) y packs de NC
- [ ] Beneficios concretos de Patreon/crowdfunding por temporada de beta
- [ ] Beneficios de suscripción (si se abre el canal alternativo)
- [x] Kit A compartido: dash, reverse dash, Kawarimi, armadura mágica (§10.0). No es 1 por reino. Sprint A ilimitado. Combo cancel post-hit.
- [x] Layout de mando y móvil: **después** (prioridad = mecánicas).
- [x] Target A: facing + hitbox; sin lock / tab-target
- [x] PvP A: sin gear → caps iguales y `%` de equipo = 0. Equalizar explícito innecesario hasta B.
- [x] Pixeloid Sans; icono de reino = cuadrado verde; arte mockup/greybox; 640×360; tiles 32; 1v1 25×15; 5v5 100×100.
- [ ] Autocoste de súper técnicas (root / no moverse al castearla)
- [x] Equipo en A: **ninguno**. 1 arma + 1 pecho por reino = Etapa B.
- [ ] Beneficios puros de `earth` y `water`
- [ ] Nombres y recetas de compuestos (fusión de 2); ¿existen compuestos de 3?
- [ ] Números: quemadura, slow, empuje, vapor/calor, freeze; CD y `base` de skills §10.1
- [ ] Umbral de entrenamiento para `orb_2` / `orb_3`
- [ ] Cap de runas de crafteo de **compuestos**; qué se pierde si falla el descraft
- [ ] Cantidad X de runas para subir nivel de ítem (I→V)
- [ ] % exactos por roll de atributo (deben ser bajos; 5 piezas apilan)
- [ ] % que aportan piedra fina, gema dorada y piezas de monstruo al éxito
- [ ] Tabla de fallo único: destroy vs strip si el éxito no llega a 50% o a 100%
- [ ] Reconciliar Orbe de Rayo / “5 elementos” con el ciclo de cuatro básicos
- [ ] Bind de teleport (`*`) vs controles §9; N segundos de canal
- [ ] Set Mazinger Z: 5 piezas, beneficios y requisitos
- [ ] Skills de “portar” por familia de armadura / nivel
- [ ] Id propio para el disparo en V de `water-gun` si no es la misma skill
- [ ] Quinto anillo de pureza y el “Anillo de…” cortado
- [ ] Prime/Titan/Arconte por reino
- [ ] Campos extra del territorio de Fontaine (además del área alien)
- [ ] Primera skill de clan: llenar `skills` en `clans.json` + GDD, no un `if` en el player (Etapa B)
- [ ] Etapa C: lista cerrada de regiones (ids, adyacencia, qué shard); no inventar el mapa en A/B
- [ ] Etapa C: elegir el split de tesorería al conquistar todos los territorios (§15.4)
- [ ] Segundos de canal del menú de robo (§12.4 ya enlaza % de §15.2)
- [ ] Reconciliar precio de orbe elemental (15 000 Hes) con tablas de cofres §16.5
- [ ] Ruleset de mazmorra: wipe vs conteos de KO §12.5 y reentrada §16.4
- [ ] Layouts de verde y roja (asentamientos / jefes) más allá de lo cerrado en azul §16.5
- [ ] Catálogo de drops de jefe (además de Hesedias de cofre)
- [ ] Eventos que disparan mazmorras globales §16.6

---

## 15. Mundo abierto, regiones y zonas (Etapa C, congelado)

**No se implementa** en Etapa A ni B. Arquitectura y candado: `PLAN.md` §18. Este apartado guarda el diseño para cuando el equipo escriba el ADR de desbloqueo, **meses** después de mazmorras + meta jugables.

El sistema de zonas se vincula al **robo de cadáveres** (§12.4). Los % de Hesedias e ítems de esta sección son la regla de zona; quién lootea, canal interrumpible y cadáver de 45 s están en §12.4–12.5. KO no instancia cadáver.

### 15.1 Regiones (no un mapa único)

El overworld se parte en **varias regiones** separadas:

- **Optimización:** cada región es su propio shard de simulación. No se carga ni se replica el continente entero. AOI recorta entidades *dentro* de la región (`PLAN.md` §18.2).
- **Gameplay:** viajar entre regiones es cambio de sesión. Recursos, precio de territorio y densidad de peligro pueden diferir. Las reglas de color de zona aplican **dentro** de cada región.

Lista de regiones (ids, adyacencia, arte): **pendiente**. No se dibuja ni se mete TileMap “de prueba” en el cliente de rooms.

Mazmorras y arenas **no** son regiones: siguen siendo battle nodes instanciados. En C se entra igual que hoy se sale del lobby.

### 15.2 Colores de zona

Todo el overworld es **PvP 100%** excepto la ciudad de **Aurora**. Aurora debe ser **gigante**: honor al reino más poderoso y lugar tranquilo.

| Color | Dónde | Combate | Hesedias en mano al morir | Ítems al morir | Territorios | Recursos / drops |
| --- | --- | --- | --- | --- | --- | --- |
| **Celeste** | Solo Aurora | No se puede atacar. No se puede usar **ninguna** habilidad | N/A (no hay combate) | N/A | N/A | Ciudad; no es farmeo de zona |
| **Verde** | Overworld | Puedes morir | Pierdes **50%** | No pierdes ítems del inventario | Más baratos que amarilla | Más comunes. Muy poca probabilidad de loot de zona roja |
| **Amarilla** | Overworld | Puedes morir | Pierdes **75%** | Puedes perder **la mitad** de los ítems **no equipados** | Más baratos que roja | Más importantes |
| **Roja** | Overworld | Puedes morir | Pierdes **100%** | Puedes perder **todos** los ítems, **equipados y no** | Los más caros | Máxima calidad |

“Hesedias en mano” ≠ Hesedias en banco/almacén. El detalle de dónde se guarda moneda es Etapa B (economía); en C esta tabla consume ese modelo.

KO de combate en rooms (A/B) **no** aplica esta tabla. Rooms no tienen color de zona.

### 15.3 Conquista de territorios

Se conquistan territorios **dispersando el maná** de la **estatua de poder** del territorio (estilo capture zone de MOBA).

| Regla | Valor |
| --- | --- |
| Tiempo para conquistar | **3 minutos** |
| Recaptura (misma facción u otra) | Tras **12 horas** |
| Al **50%** de captura | Se dispersa el maná del territorio; pasa a **neutral** (nadie lo posee) |
| Cap de jugadores otorgando maná | **10** por estatua |
| Facción enemiga en rango | La captura **se detiene**. Si la otra facción no está y los enemigos sí, **baja** el porcentaje |
| A **0%** + **30 s** | La facción que capturaba **pierde la oportunidad** de capturar |
| KO | Mientras estás KO **no** capturas (evitar capturar durante ~25 s de KO) |

### 15.4 Beneficios de conquista

- **% de suerte** de farmeo en esa zona (número exacto: al implementar C).
- **Todo el dinero** generado ahí (NPC, impuestos, tiendas, etc.) va al **reino**.
- Los reinos tienen **gestión automática**. A partir de cierto rango de reino (§11.7, Comandante+) se ve el estado (si falta dinero, si no cubre mantenimiento: radar, mejoras, etc.).

**Si un reino conquista todos los territorios:** los demás pierden **50%** de su tesorería, entregada al conquistador. Cómo se reparte ese botín (elegir **una** al abrir C; no las dos):

1. El top **#10** de jugadores que más contribuyeron al reino recibe **una décima parte** de ese dinero.
2. El **50%** del botín se reparte entre las **órdenes** que conquistaron territorios, **dividido por la cantidad de zonas**. Las órdenes con más territorios reciben más.

### 15.5 Qué no es esto

- El lobby de Etapa A no es zona celeste ni proto-Aurora jugable como overworld.
- El **interior** de una mazmorra no es un tile de zona: cofres y jefes son de instancia (§16). En Etapa C, al morir **dentro**, las penalizaciones de Hesedias/ítems siguen el **color de la zona donde spawneó el portal** (§16.4), no un color pintado en el mapa de la mazmorra.
- No se copian estas reglas al tick de `pvp_duel` / `pvp_arena`.

---

## 16. Mazmorras

Fuente: página Notion *Dungeons*. El **contenido de instancia** (tipos, jefes, cofres, PvP interior) es **Etapa B**. El **spawn aleatorio en el overworld** es **Etapa C** (`PLAN.md` §18). No se prototipa portal de mundo en A/B.

Arquitectura: misma tubería que un room PvP (`SessionKind.pve_dungeon`, battle node). No es una región ni un `world_shard`.

### 16.1 Cómo se entra (por era)

| Era | Cómo llega el jugador | Qué no hay |
| --- | --- | --- |
| **A — Rooms** | Nada. No hay mazmorra. | — |
| **B — Mazmorras + meta** | **Room desde el lobby**, como el PvP: cola / party / sala → orquestar instancia. Los tres tipos (verde / azul / roja) y las globales de evento existen como **modos de room**, no como portales en un mapa. | Ciclo de spawn en zonas, carrera “el que llega primero”, entrada que desaparece del overworld, bloqueo por *Bajo Ataque* de mundo |
| **C — Open world** | Las mismas instancias. El portal **aparece en el mapa** (zonas según tipo). Quien llega primero puede farmear. Entrar = salir del shard al battle node (igual que salir del lobby). | No se reescribe el interior; se añade el spawn |

### 16.2 Tipos

Los tipos varían, en C, según **la zona donde aparezca el portal**. En B el tipo se **elige o rota en el lobby**; las restricciones de zona no aplican porque no hay overworld.

#### Verdes

- **C (spawn):** zonas **verdes y amarillas**.
- **Jugadores:** solo **1**. La entrada **desaparece** cuando un jugador entra (en C: del mapa; en B: la instancia es solo).
- Un jugador **no puede entrar si está Bajo Ataque** (flag de mundo; Etapa C. En B no hay combate de overworld).

#### Azules

- **C (spawn):** cualquier zona **PvP** (verde / amarilla / roja; no celeste).
- Aceptan **varias personas**. Los cofres de camino quedan **vacíos si alguien ya los tomó**.
- Cofres de **jefe** dan recompensa a **todos los que participaron** del jefe. En el camino hay recompensas menores matando a los pequeños.
- **3 jefes**, cada uno más fuerte que el anterior. Diseño: **no es posible matarlos solo**.
- **PvP permitido** dentro: parties vs parties por quedarse la dungeon.

#### Rojas

- **C (spawn):** zonas **amarillas y rojas**. Dificultad y recompensas **más altas** que las azules.
- PvP permitido dentro. Las **muertes** (Hesedias / ítems) siguen la zona de aparición del portal (§16.4).
- Niveles más difíciles y mejor loot.

### 16.3 Ciclos de spawn (solo Etapa C)

En el overworld las mazmorras aparecen así (totales = 24 h a ese ritmo, sin eventos extra):

| Tipo | Cada | Por día |
| --- | --- | --- |
| Verde | 15 minutos | 96 |
| Azul | 1 hora | 24 |
| Roja | 2 horas | 12 |

En Etapa B **no** hay este reloj de mapa. La oferta de rooms del lobby la cierra el equipo al implementar (colas, rotación, límite de instancias); no se copia este calendario al matchmaking “para simular el MMO”.

### 16.4 Qué pasa si te matan

Varía según la **zona de aparición del portal** (colores y %: §15.2). El combate usa KO/Death §12.5; el saqueo de cadáver solo aplica si esa muerte es de **overworld** (C). En la instancia, loot de cofre/jefe es §16.5, no el menú de robo.

Comportamiento de **reentrada** (pensado para C, con portal en el mundo):

- Reapareces **fuera** y puedes volver a la entrada para reunirte con tu party.
- Si **matan a todos** los miembros de la party **dentro**, **no pueden volver a entrar**. Evita hostigar a quienes ya ganaron el combate reentrando en bucle.

En Etapa B (room desde lobby, sin portal): la reentrada al battle node la define el ruleset al implementar (mismo espíritu: wipe de party cierra el run; no inventar un “portal invisible” en el lobby). Conteos de KO dentro del run: §12.5.

### 16.5 Recompensas y recorrido

Hesedias de cofre son las de la fuente. **Drops** de jefe: catálogo abierto (no inventar ítems).

| Tipo | Cofres de camino | Cofre(s) de jefe | Recorrido cerrado en fuente |
| --- | --- | --- | --- |
| Verde | 3 × **250** Hes | 1 × **750** Hes + drops | — |
| Azul | 8 × **350** Hes | 3 × **1 000** Hes + drops | 2 asentamientos → jefe → 3 asentamientos → jefe → 3 asentamientos → jefe |
| Roja | 12 × **500** Hes | 3 × **1 500** Hes + drops | Al matar al **último** jefe se **bloquea la salida 2 minutos** (ventana de PvP para robar ítems matando) |

Totales de cofres si se lootea todo (sin drops): verde **1 500**; azul **5 800**; roja **10 500**.

Fantasía de grind (azules): un jugador tiene **chance** de juntar para un orbe elemental con **10 azules enteras**, pero **no siempre**: otras personas entran a limpiar. No tratar esas 10 runs como precio de tienda.

### 16.6 Mazmorras globales

Dungeons de **eventos importantes**: mucho más grandes; varias dificultades en una (fantasía: “5 dungeons distintas en 1”). Al final, bosses de los **reyes** con **almas corruptas**.

- Aparecen **cerca de los 4 reinos**.
- El boss final es **uno de los antiguos reyes** según el reino.

**C:** el evento spawnea cerca de esos reinos en el mapa. **B:** puede existir como **room de evento** desde el lobby (mismo interior); la colocación geográfica espera el overworld. No implementar mapa de reinos para “colocar” el evento en A/B.

### 16.7 Qué no se inventa aquí

- Layout tile a tile de verdes, rojas y globales (salvo el orden de asentamientos/jefes de azul).
- Lista de drops no-Hes.
- IA, skills de monstruo, HP de jefes.
- UI de cola del lobby (marca: `docs/brand/BRAND.md`).
- Código de spawn, AOI o zonas: `PLAN.md` §18.

---

## 17. Economía de jugador y modelo de negocio

Arquitectura de cobro, tablas y REST: `PLAN.md` §4.14, §10.8, §29, ADR-014. Aquí va **qué siente el jugador** y las tasas de diseño. No se implementa PSP en Etapa A; el modelo de dos monedas **sí** se respeta desde el primer wallet.

### 17.1 Dinero del juego

Dos monedas. No se mezclan. Plural canónico de la de mundo: **Hesedias** (unidad: Hesedia).

| Moneda | Qué es | Cómo se obtiene | Dónde vive |
| --- | --- | --- | --- |
| **Nexum Coin (NC)** | Premium | CASH (packs) o grant de beta/ops | Cuenta (`user_wallets`) |
| **Hesedias** | Moneda normal del juego | Drops, mercado, craft, sinks; un SKU **puede** entregar Hesedias | Personaje (`character_wallets`) |

Tasa de packs de NC (diseño, no FX en vivo; el PSP se queda comisión): **1 USD = 15 NC**. Referencia: **100 USD = 1 500 NC**. Packs concretos (montos de tarjeta) se listan en catálogo al abrir tienda.

Reglas:

- El mercado de jugadores (§8.18) **solo** Hesedias.
- No hay cambio libre NC ↔ Hesedias. NC → Hesedias solo como **SKU**.
- **No cash-out** a dinero real.
- Puntos de habilidad (§11.1) son un tercer recurso de build, no una moneda de mercado; se pueden vender como SKU (NC o Hesedias).

### 17.2 Before production (beta)

En versiones beta se pueden ofrecer **beneficios únicos y personalizados** a testers en **Patreon u otra plataforma de crowdfunding**. Es el primer impulso económico del proyecto.

Los beneficios son **entitlements de cuenta** (cosmético, título, item bound, NC de cortesía, etc.). Se diseñan por temporada de beta **aquí** cuando existan; no un `if` en el nodo del jugador. Cumplimiento: grant ops (`PLAN.md` §29.2), no checkout de tarjeta.

### 17.3 In production

Pagos automatizados con **tarjeta crédito/débito**: el jugador paga y recibe lo comprado, como cualquier servicio online. El cliente no confirma “ya pagué”.

Canales:

| Canal | Prioridad | Qué vende |
| --- | --- | --- |
| **Microtransacciones** | **Principal** | Elementos virtuales o mejoras de experiencia: objetos **decorativos**, **armas**, **monedas**, **ventajas competitivas** |
| **Suscripción** | Alternativo | Membresía mensual/anual con beneficios de catálogo |
| **Infoproductos** | Alternativo | Guías, tutoriales, trucos, estrategias, consejos. Fuera del tick (web o mismo checkout de cuenta) |

Cada SKU de microtransacción lleva un tag de catálogo:

| `tag` | Ejemplos | Combate |
| --- | --- | --- |
| `cosmetic` | skins, títulos, VFX de presentación | No entra al snapshot de daño |
| `currency` | packs NC, SKU de Hesedias, puntos de habilidad | Economía / build |
| `power` | armas, gear, ventajas competitivas | **Sí** puede entrar al loadout. En `pvp_duel` / `pvp_arena` sigue abierta la palanca de **equalizar** gear (§11). El tag obliga a no olvidar esa decisión; no la cierra |

Armas y power IAP son **producto aceptado**, no un accidente. Si un modo debe ser fair, lo filtra el **ruleset**, no se borra el canal de venta.

### 17.4 Qué no se inventa aquí

- Proveedor de pagos (adaptador en PLAN).
- Lista cerrada de SKUs ni precios de puntos.
- Lista de recompensas Patreon de una temporada concreta.
- Impuestos de zona / tesorería de reino (eso es Etapa C, §15.4) sobre Hesedias de mundo, no sobre NC.

