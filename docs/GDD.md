# Nexum Terra — Diseño de juego (GDD)

**Este archivo manda sobre fantasía, builds, stats, items, skills y sensación de combate.** La arquitectura (sesiones, EntityID, Postgres vs catálogo) está en `PLAN.md`. Cómo está implementado cada sistema en código: `docs/modules/`. Si un número de balance cambia, se edita aquí, el module doc si cambia el flujo, y luego `nexum-terra/data/`. Si cambia *cómo se persiste o se simula*, se edita `PLAN.md`.

Fuentes actuales: GDD Notion, Gameplay, Daño y Build, Clanes, cinco reinos, Elementos y los cuatro básicos, Items y Crafting, Pasivas, Sistema de Puntos, Sistema de Rebirth, Obtención de Skills (2026-09-13). Otras páginas de Notion siguen sin volcar.

Estado: **normativo en lo escrito; incompleto en sistemas con página pendiente.** Caps base de maná / stamina / vitalidad y el modelo de daño estático + % están cerrados en §6. Calidad, nivel I–V, crafting de atributos, orbes de uso y mercado de jugadores están en §8.10–8.18 (no se simulan en Fases 1–4). Tasas de sprint, regeneración y % exactos por roll de atributo siguen abiertos.

Inspiración de ritmo (no de lore ni de sistemas): [StormEdge](https://store.steampowered.com/app/2321350/StormEdge/). Acción frenética, mundo abierto partido en regiones para rendimiento.

---

## 0. Alcance por etapas

Las **primeras etapas** (lobby, práctica, PvP instanciado: Fases 1–4 de `PLAN.md`) **no** entregan el build completo. El modelo de personaje *admite* esas piezas más adelante; el catálogo y el tick no las simulan todavía.

| Pieza | Primeras etapas | Etapas siguientes (contenido continuo) |
| --- | --- | --- |
| Identidad: nombre, apariencia, `kingdom_id`, `clan_id` | sí (1 reino jugable + 1 clan de ese reino) | skills/pasivas/activas del clan |
| Chat de distancia con gentilicio / nombre | sí (reglas §4) | amigos + olvido: ya en §4 |
| Melee, guardia, dash, hotbar, swap de arma | sí | combos mágicos / elemento |
| Tres vitals + daño estático + % de equipo | sí (§6) | pasivas de clan/elemento/profesión |
| Equipo (armadura + armas) | sí, slots de §8 | orbes, anillos, collar, calidad/nivel, crafting |
| Skills genéticas de clan (activas/pasivas) | no; catálogo vacío | sí |
| Elementos (orbes, crafting, ciclo, skills básicas) | reglas en §8.6–8.9 y §10.1; no se simulan aún | sí |
| Crafting de atributos, subida de nivel de ítem, mercado | no | sí (§8.10–8.18) |
| Profesiones | no | sí |
| Rebirths y pasivas especiales de rebirth | no | sí |
| Árbol de puntos de pasivas / grind de activas en PvE | no | sí (PVE/mundo) |
| Misiones, rangos, divisiones, dungeons, zonas, death/loot, etc. | KO a vitalidad 0 sí (§6) | el resto, al volcar cada doc |

Todo es reemplazable en una build **excepto el clan en el que naces**, hasta un rebirth (§11.3). En primeras etapas ese ancla ya se elige; aún no otorga skills.

---

## 1. Pitch y fantasía

Nexum Terra es un **RPG online de acción** con mundo abierto dividido en regiones. Mezcla culturas y épocas: hierro, armas de fuego, lo arcaico y lo inteligente en el mismo suelo. El jugador arma el personaje que quiera; farmer, guerrero o millonario **todos pelean** (defender territorio, sobrevivir, o lo que el mundo pida).

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
| Skills | `skills: []` en data. **Cero** activas y **cero** pasivas de clan o de reino. El tick no lee el clan para daño |
| Permanencia | Fuera de rebirth, el clan de nacimiento no se cambia (es lo único no reemplazable de la build). El rebirth **corta toda afiliación** al clan anterior: naces de nuevo en otro reino/clan; **ninguna habilidad** de esa vida se guarda (§11.3) |

La fantasía de cada clan (ojos, insectos, cristales, etc.) se escribe aquí para no perderla. **No se implementa** hasta que existan entradas de skill en data.

Idea de mundo (no primera etapa): **tiendas de ropa distintas por reino**. Vestirte de ninja sin ser de Aerion implica ir a comprar allá.

Movilidad citada en el GDD original (dash samurái / Kawarimi / Fontain): documentada para más adelante. En primeras etapas el dash de combate, si existe, es un skill genérico de kit, no un racial del reino.

### 3.2 Aurora (`aurora`)

Ciudad antigua del **shogunato**. Murallas de índice Kamakura. Colores **negro y blanco**. Época: pasado. Hostilidad: **neutral**. Arquitectura de samurái: madera alzada contra inundaciones, paredes que rodean el área frente a enemigos del exterior.

El ambiente debe **desincentivar peleas** sin quitar la posibilidad de hacer daño: penalizaciones graves (p. ej. un NPC muy fuerte que parta al agresor). Eso es regla de zona, no de create. Dash / samurái asociado en fuentes previas; no se otorga ahora.

Gentilicio de chat: no aplica a jugadores (no nacen aquí).

### 3.3 Fontaine (`fontaine`)

Ambiente de **futuro**; referencia de sensación: capital del oeste tipo ciudad de Bulma. Gris, celeste, cristal. Territorio alrededor invadido por aliens (campos extra: TBD). Época: futuro. Hostilidad: **activa**. Arquetipos de fantasía: **científicos** (magos de la tecnología). Edificios altos que rematan en punta circular, colores llamativos, césped junto a carreteras con vehículos voladores.

Nota de diseño (Omnivisus): ojo inspirado en byakugan en un clan científico; cuatro clanes que estudian artes distintas. Skills: más adelante.

Gentilicio de chat: **Fontainer**.

### 3.4 Terrara (`terrara`)

Reino de la **fuerza**. Gran bosque, desierto y alto valle. Paleta medieval: hierro, rojo, naranja. Época: pasado. Hostilidad: **activa**. Arquetipos: soldados, caballeros y magos. Arquitectura mezclada: **gótica** en edificios importantes, **románica** en hogares, **bizantina** en centros de la monarquía.

Gentilicio: **Terrano**.

### 3.5 Spectra (`spectra`)

Reino oscuro: pantano venenoso, cementerio, bosque oscuro. Colores **verde y púrpura**. Época: no registrada. Hostilidad: **activa**. Arquetipos: ninjas / magos / elfos oscuros. Edificios góticos oscuros.

Gentilicio: **Spectro**.

### 3.6 Aerion (`aerion`)

Ciudad futurista en el **cielo**, nubes y cielo azul. Verde esmeralda, blanco y dorado. Época: futuro. Hostilidad: **activa**. Arquetipos: guerreros del cielo. Planta en estrella con círculos en puntas y centros; flota por maquinaria de los habitantes. Rascacielos de cristal junto a edificios de aspecto contemporáneo.

Gentilicio: **Aerion**. Kawarimi / ninja: más adelante, no se otorga al elegir el reino.

### 3.7 Clanes (16, 4 por reino jugable)

Los clanes son de los reinos en los que puedes nacer. Aurora no tiene clan jugable en este catálogo. La fuente escribe a veces “Fountaine”; el `kingdom_id` canónico es `fontaine`.

**Fontaine**

| `clan_id` | Nombre | Fantasía (skills más adelante) |
| --- | --- | --- |
| `omnivisus` | Omnivisus | Ojos científicos: estado físico de una persona y visión 360° a larga distancia (una región entera). Combate: puntos vulnerables / sistema nervioso en melee. |
| `sagitta` | Sagitta (flecha, latín) | Clan antiguo. Magia concentrada en “arcos” (hoy varían). Largo alcance, tiros precisos. Magia interior canalizada con **canalizadores cibernéticos**. |
| `gadgetrix` | Gadgetrix | Gadgets de protección y ataque, para sí o para aliados. |
| `ciberlance` | Ciberlance | Científicos de mejoras biónicas de alto nivel. El melee más fuerte de Fontaine frente a la precisión de Omnivisus. |

**Spectra**

| `clan_id` | Nombre | Fantasía (skills más adelante) |
| --- | --- | --- |
| `herbora` | Herbora | Control de plantas de origen mítico o demoníaco. |
| `entomante` | Entomante | *ento-* insectos + *-mante* dominio. Bichos que habitan en su cuerpo. (Canal `mana_steal` cuando existan skills.) |
| `pyrofauces` | Pyrofauces | Conexión de maná al inframundo; llamas del infierno y otros elementos de ese eje. |
| `nachtsoldaten` | Nachtsoldaten | “Soldados de la noche”. Naturaleza + artes oscuras + disciplina de soldado. |

**Terrara**

| `clan_id` | Nombre | Fantasía (skills más adelante) |
| --- | --- | --- |
| `stellamante` | Stellamante | Maná en ataques luminosos tan rápidos que los llaman estrellas. |
| `pulmonarius` | Pulmonarius | Fuerza sobrehumana; distancia y melee. |
| `vitamancers` | Vitamancers | Manipulan energía vital: absorber cadáveres; proyectiles/orbes lentos que drenan vitalidad; materia extraña (minas, alfombra que roba vital a quien la toca); sacrificio propio para curar a otro (coste: un KO al caster, el aliado queda a 50% vit y stamina). |
| `sangrafilos` | Sangrafilos | Maná en metal: filos, cobertura, combate de **sangrado** (efecto secundario). |

**Aerion**

| `clan_id` | Nombre | Fantasía (skills más adelante) |
| --- | --- | --- |
| `umbromante` | Umbromante | Umbra: dominio de la sombra. Referencia de sensación: Nara. |
| `geisteswaffen` | Geisteswaffen | Clan antiguo. Poder en herramientas: espada moldeable de tecnología. Medio alcance; hojas de chakra tipo samurái. |
| `crystallomante` | Crystallomante | Convierte materia elemental en cristal; proyectiles, muros, dragón de cristal, encierro. Necesita cristales cerca: invoca **generadores** que sueltan cristales periódicamente. |
| `sonomantes` | Sonomantes | Sonido para deshabilitar: daño directo o ilusiones. Potenciado por tecnología. Ilusiones sonoras; ondas que crecen en círculo. |

Listas detalladas de skills por clan (páginas Notion) se vuelcan cuando se activen las skills, no antes. Un **5.º clan por reino** se desbloquea por rebirth de cuenta (§11.3); aún no hay ids en data.

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

Snapshot de primeras etapas: **reino + clan (identidad, sin skills) + tres vitals + % de equipo + melee + hotbar + dash**.

---

## 6. Vitals, daño y “build”

No hay clases ni un árbol que te preasigne el rol. Puedes ir a magia elemental, a cuerpo a cuerpo, o a mental / ilusiones. Los clanes **no** definen un build: nada está atado a un template. El daño de las habilidades es **estático** y sube con el **nivel de pasivas** (clan, elemento, profesión, etc.; referencia de sensación: pasivas de Albion) y con **equipo que da %** a un tag (fuego, melee, …).

Especializar sigue importando: quien pone *todo* el equipo a fuego pega más fuego que quien parte el equipo entre fuego y melee. Puedes **cambiar el loadout cuando quieras** (fuera de combate / fuera de match congelado).

No se usa la fórmula antigua `atk`/`pow` × escala + mitigación `K`. Eso queda derogado.

### 6.1 Vitals (salud del personaje)

Tres pools. IDs de data en inglés.

| id | Nombre | Qué es |
| --- | --- | --- |
| `mana` | Maná | Energía de poder. En la fuente de kill-budget también se llama **chakra** al robo. |
| `stamina` | Stamina | Salud física y mental. Permite **correr** y **resistir golpes**. |
| `vitality` | Vitalidad / vida | Salud vital. A **0 → KO**. |

Caps **base** de un jugador (sin equipo; el equipo modifica maná y stamina):

| Vital | Cap base |
| --- | --- |
| Maná | 2000 |
| Stamina | 2000 |
| Vitalidad | 1500 |

`speed` de movimiento sigue existiendo como dato de locomoción, no como stat de daño.

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
- Vitalidad a **0**: el usuario cae **KO**. Muerte / downed / loot: página KO y Death, aún no volcada.

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
| Caps base (2000 / 2000 / 1500) y curva si deja de ser plana | sí | no | no |
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
| Skillbar | 6 hotslots + barras de skills (rueda del mouse cambia de barra) |
| Melee kit | Un estilo; golpes ligero/pesado, guardia, carga. Daño canal `stamina`, número estático + % `melee` del equipo |
| Movilidad | Dash (y/o reemplazo según reino cuando esté en data) |
| Loadout | Un snapshot por personaje; se congela al entrar a 1v1 |

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

Los 16 clanes ya están en catálogo. Las páginas Notion de skills por clan (ver §12) se fusionan cuando se implemente la primera skill, no al elegir el clan.

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

- Clan **Gadgetrix**: +25% (60 → 85). No se aplica mientras las features de clan estén apagadas.
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

Habrá secretos de mundo: p. ej. **estatuas mágicas** que incrementan el % de éxito. Objetos específicos en el craft sesgan un atributo concreto (escudo + gema dorada → reducción de daño; arma + piedra fina → aumento de daño). Lista cerrada de secretos: cuando exista contenido de zona, no ahora.

### 8.16 Sets — Armaduras de Fontaine

**Set Mazinger Z:** cinco piezas (los slots de armadura). Calidad prevista **Raro**. Beneficios de set y requisitos: **se escriben luego**; no inventar números ni pasivas. `kingdom_id` canónico `fontaine`.

### 8.17 Orbes de uso, sellado y economía

Distintos de la **fusión a compuesto** (§8.7). Estos orbes se **craftean con grind** y se venden; el mercado de jugadores es el sink/source principal.

**Orbes de elemento** (equipables / habilitan skills; precio de referencia **15 000 Hesedias** = recompensa de **15 dungeons verdes** enteras):

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

Moneda citada: **Hesedias**. Drops de dungeon / craft de orbes alimentan este loop (farmer / millonario). Persistencia: `PLAN.md` Inventory & Economy.

---

## 9. Gameplay y controles

Plataformas objetivo: **PC (teclado y mouse), mando y móvil**. El esquema de mando no está cerrado.

Convención en tablas: 🔳 / 🔼 / ⭕ / ❎ = face buttons del mando; L2/R2 = gatillos. En teclado, 🔳 ≈ golpe izquierdo / Space según contexto de arma; ver filas.

### 9.1 Controles (teclado | mando)

| Acción | Teclado | Mando (propuesta) |
| --- | --- | --- |
| Movimiento | WASD | Joystick izquierdo |
| Cámara | Mouse | Joystick derecho |
| Modo maná (tap) o recargar maná (hold) | Z | L2 |
| Reemplazamiento (Kawarimi: cuerpo falso + invisibilidad breve para reposicionar) | R | Doble R2 *o* Doble L2 (la fuente duplica R; unificar al implementar) |
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
| Saltar (tap) | Espacio | ❎ |
| Correr (hold) | Shift | ❎ hold |
| Primaria ↔ secundaria | Rueda del mouse | R2 *(conflicto con Guardia: resolver en implementación)* |
| Dash (habilidad básica, corta distancia adelante) | C | L2 + ❎ |

Notas: Espacio está asignado a golpe ligero **y** a salto (tap). Rueda a cambio de barra **y** a swap de arma. R2 a guardia **y** swap. Eso hay que desambiguar (hold vs tap, o mover swap a otro botón) al cerrar input; no se inventa el layout final aquí.

### 9.2 Carga de golpes

Click izquierdo (mano izquierda) y click derecho (mano derecha) tienen **barras de carga**.

### 9.3 Combos de arma (melee)

En esta sección 🔳 = click izquierdo, 🔼 = click derecho.

| Secuencia | Resultado |
| --- | --- |
| 🔳 + 🔳 + 🔳 | Combo básico |
| 🔳 + 🔼 + 🔳 | Combo secundario |
| 🔳 + 🔼 + 🔼 | Habilidad especial de arma |
| 🔳 + 🔳 + 🔼 + 🔼 | Habilidad especial de arma |

Aplica a espada, hacha, lanza, daga. Detalle de frames / cancel: [Sistema de Combates Melee](https://app.notion.com/p/Sistema-de-Combates-Melee-2891e055ee8d81f4889bfe3041dc02e6?pvs=21) (pendiente).

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

Se abandonó el click-sobre-enemigo como requisito (móvil y mando). **Target y “drop target” están abiertos.** No implementar un tab-target de MMO clásico hasta cerrar esta decisión.

Gameplay Notion: [Gameplay](https://app.notion.com/p/Gameplay-2891e055ee8d813598c4d30709684bd1?pvs=21).

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

**Primeras etapas — kit mínimo a definir en data (placeholders hasta balance):**

- Dash (y variante de reino cuando exista)
- Reemplazamiento (Kawarimi)
- Guardia (puede ser estado, no skill de hotbar)
- 4–6 activas genéricas de prueba para PvP
- Melee como skills de arma (`weapon_skill` ids), no lógica en el nodo

**Luego:** genéticas de clan, profesión, pasivas generales (entrenables) y elementales (activas: puntos para obtener, uso para nivel; pasivas: puntos). Elementales de los 4 básicos: §10.1 (catálogo listo; el tick las ignora hasta el recorte). Cómo se aprenden: §11.5. Puntos y rebirth: §11.1–11.4.

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

Rangos, misiones, divisiones: no definidos aquí. Enlaces en §12.

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

Obtención: subir de **rango** (cantidades fijas), eventos, misiones (probabilidad de 1–2 puntos). También se **compran**: con varias monedas de juego (el medio exacto no está cerrado) y, de forma **importante**, con **dinero real**. No hay SKU, precio ni tienda definidos; el canal de compra con dinero real es requisito de diseño, no un extra opcional. Números exactos de rango: cuando se vuelque Rangos.

**Puntos de rebirth:** 1 por cada rebirth. Solo mejoran **pasivas especiales** (§11.4).

### 11.2 Árbol de pasivas generales

Ramas I→V. Nivel I desbloquea II, etc., salvo requisitos extra anotados. Ids como en la fuente. Se **entrenan** (uso / kills); los puntos de habilidad pueden adelantar, no son el requisito. Esto **no** es el mismo sistema que las profesiones Médico / Espadachín / Sensor (§12); no fusionar. Las skills **elementales** no viven aquí: se compran con puntos (§11.1, §11.5).

Este árbol y las pasivas especiales de rebirth se seguirán afinando al implementar mundo/PVE. No simular en Fases 1–4.

#### General

| Rama | id | Efecto |
| --- | --- | --- |
| I | `explorer` | +5% velocidad de movimiento |
| I | `elementary-novice` | Controlar **1** elemento |
| II | `arcane` | +10% daño mágico |
| II | `elementary-apprentice` | Controlar **2** elementos a la vez |
| III | `fast-regeneration` | Más velocidad de regeneración de stamina |
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

**Qué más se reinicia:** clasificación (asesinatos y honor de reino), rango y el personaje.

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

El resto del catálogo está en planificación. No inventar más filas. Se irá cerrando al desarrollar mundo/rebirth, no en Fases 1–4.

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

---

## 12. Sistemas con página pendiente (no rellenar de memoria)

Hay diseño en Notion que **no** está en los PDF volcados. Hasta el siguiente documento, no se inventan reglas.

| Sistema | Etapa probable | Notion |
| --- | --- | --- |
| Misiones | mundo / PvE | [Misiones](https://app.notion.com/p/Sistema-de-Misiones-2891e055ee8d819399dbd20a84ba97bb?pvs=21) |
| Clasificación | PvP | [Clasificación](https://app.notion.com/p/Sistema-de-Clasificaci-n-2891e055ee8d81099a35d445fcbce797?pvs=21) |
| Rangos de reinos | mundo | [Rangos de Reinos](https://app.notion.com/p/Rangos-de-Reinos-2891e055ee8d8107a6eadb40e938c9fc?pvs=21) |
| Divisiones | PvP / mundo | [Divisiones](https://app.notion.com/p/Sistema-de-Divisiones-2891e055ee8d8171adcddabcdc9ee393?pvs=21) |
| Renegados | mundo | [Renegados](https://app.notion.com/p/Sistema-de-Renegados-2891e055ee8d811b9ba3c68c9c51b4e8?pvs=21) |
| Robo | mundo | [Robo](https://app.notion.com/p/Sistema-de-Robo-2891e055ee8d81c4aa95d08690fc7bcb?pvs=21) |
| KO y Death | todas | [KO y Death](https://app.notion.com/p/Sistema-de-KO-y-Death-2891e055ee8d819a8269d89a6d3fabaf?pvs=21) |
| Regeneración | combate | [Regeneración](https://app.notion.com/p/Sistema-de-Regeneraci-n-2891e055ee8d8173a477eb396bca8b6f?pvs=21) |
| Logout bajo ataque | mundo | [Logout bajo ataque](https://app.notion.com/p/Sistema-de-Logout-Bajo-Ataque-2891e055ee8d816f9acbc7f44028196c?pvs=21) |
| Órdenes | mundo / party | [Órdenes](https://app.notion.com/p/Sistema-de-rdenes-2891e055ee8d8101b7a0f3d0f391aa65?pvs=21) |
| Zonas | mundo | [Zonas](https://app.notion.com/p/Sistema-de-Zonas-2891e055ee8d81588edce4a3132073d2?pvs=21) |
| Dungeons | Fase 5+ | [Dungeons](https://app.notion.com/p/Dungeons-2891e055ee8d81a28844e2edf7155084?pvs=21) |
| Pasivas especiales (resto del catálogo) | largo plazo | [Pasivas especiales](https://app.notion.com/p/Pasivas-Especiales-2891e055ee8d8187a6adfd3f1464d662?pvs=21) — patrón y ejemplo en §11.4 |
| Profesiones | largo plazo | [Médico](https://app.notion.com/p/M-dico-2891e055ee8d81469a7ec0a5834ac4ee?pvs=21), [Espadachín](https://app.notion.com/p/Espadach-n-2891e055ee8d81989991f2e73c183956?pvs=21), [Sensor](https://app.notion.com/p/Sensor-2891e055ee8d81ad9844f33f42e312b7?pvs=21) |
| Skills de clan | cuando se activen | [Omnivisus](https://app.notion.com/p/Omnivisus-Skills-2891e055ee8d81bca43ef96c5cc18267?pvs=21), [Gadgetrix](https://app.notion.com/p/Gadgetrix-Skills-2891e055ee8d81fd9507c492928316b8?pvs=21), [Entomante](https://app.notion.com/p/Entomante-Skills-2891e055ee8d8108bdb5d54ffc692d91?pvs=21), [Nachtsoldaten](https://app.notion.com/p/Nachtsoldaten-Skills-2891e055ee8d8156b831cbf162e250cd?pvs=21), [Pulmonarius](https://app.notion.com/p/Pulmonarius-Skills-2891e055ee8d8173a00df68feceaf190?pvs=21), [Sangrafilos](https://app.notion.com/p/Sangrafilos-Skills-2891e055ee8d8117af0bd8299cd1546b?pvs=21), [Umbromante](https://app.notion.com/p/Umbromante-Skills-2891e055ee8d818aab16ee84dfac6767?pvs=21), [Geisteswaffen](https://app.notion.com/p/Geisteswaffen-Skills-2891e055ee8d81ae9388e1e94d8502bf?pvs=21). El resto de clanes aún no tiene página volcada. |

---

## 13. Audio (nota)

Bandas sonoras citadas: **Base**; **Base Boss — Tenebroso — Acción**. Catálogo de música fuera de este GDD hasta haber lista.

---

## 14. Trabajo pendiente de diseño (aquí, no en PLAN)

- [ ] Volcar páginas Notion de §12 (reinos/clanes en §3; pasivas, puntos, rebirth y obtención de skills en §11)
- [ ] Catálogo de pasivas especiales de rebirth (además del ejemplo de CD)
- [ ] Quintos clanes: ids, fantasía y skills; no están en `clans.json`
- [ ] Tienda de puntos de habilidad: qué monedas de juego, precios; IAP con dinero real (canal obligatorio, SKU/precio abiertos)
- [ ] Cerrar dash racial de Fontaine, Terrara, Spectra (Aurora/Aerion documentados, no otorgados)
- [ ] Unificar botones duplicados (R / R2 / L2, Space, rueda, guardia vs swap)
- [ ] Layout de mando y móvil
- [ ] Target / drop target
- [ ] PvP: equalizar caps de vitals y/o ignorar % de gear en `pvp_duel`
- [ ] Tasa de gasto de stamina al correr y recarga de maná (hold Z); regeneración en combate (página pendiente)
- [ ] Lista v1 de tags de % (`melee` suficiente para primeras etapas)
- [ ] Kit v1 de skills de prueba (4–6) **sin** skills de clan/elemento/profesión; melee con `damage_channel: stamina` y `base` estático
- [ ] Autocoste de súper técnicas (root / no moverse al castearla)
- [ ] 1 arma y 1 pecho placeholder por reino
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
- [ ] Primera skill de clan: llenar `skills` en `clans.json` + GDD, no un `if` en el player
