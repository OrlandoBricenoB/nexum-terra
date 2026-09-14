# Nexum Terra — Marca y UI

**Única fuente de verdad visual.** Antes de Theme Godot, iconos, landing o colores “de gusto”, se lee y se actualiza este archivo.

Estado: **paleta de marca cerrada**. No inventar una paleta paralela en código. El 20 % de acento es el oro; el 80 % es el lienzo oscuro o blanco más grises cálidos.

---

## Identidad

- **Nombre:** Nexum Terra
- **Tono:** terrestre, nexo, misterio contenido — no cartoon neón genérico ni sci-fi clínico.
- **Vista:** top-down 2D. UI densa de MMO ligero (chat, nametag, barras), no mobile-casual vacío.

---

## Color (regla)

**Pareto 80 / 20.** El oro de marca ocupa **como máximo ~20 %** de la superficie (CTA, focus, iconos activos, XP, highlights). El **80 %** es lienzo: `bg.deep` en UI de juego / menús oscuros, o `bg.light` / blanco en superficies claras (landing, docs). El resto del cromo (bordes, texto secundario, hover de panel, chat) es **gris cálido o blanco**, no un segundo acento de marca.

No hay teal ni paleta “secundaria de marca”. Selección y links usan el oro o un gris, no un tercer hue.

---

## Ejes de marca

| Rol | Hex | Uso |
| --- | --- | --- |
| `brand.primary` | `#dcc552` | Oro de marca (= `brand.400`). CTA, focus, XP, whispers |
| `bg.deep` | `#160e04` | Lienzo oscuro: menús, HUD, paneles |
| `bg.light` | `#ffffff` | Lienzo claro: landing, docs |

---

## Escala de oro (`brand.*`)

El primario de marca es el **400**. Usar 50–300 para tintes suaves sobre oscuro; 500–950 para hover, pressed y texto oro sobre claro.

| Token | Hex |
| --- | --- |
| `brand.50` | `#fcfbee` |
| `brand.100` | `#f5f2d0` |
| `brand.200` | `#ebe49c` |
| `brand.300` | `#e1d268` |
| `brand.400` | `#dcc552` |
| `brand.500` | `#d1a72f` |
| `brand.600` | `#b88527` |
| `brand.700` | `#996424` |
| `brand.800` | `#7d4f23` |
| `brand.900` | `#68421f` |
| `brand.950` | `#3b220d` |

---

## Neutrales (grises cálidos sobre `#160e04`)

| Token | Hex | Uso |
| --- | --- | --- |
| `bg.deep` | `#160e04` | Fondo raíz |
| `bg.panel` | `#24180a` | Cards, chat, ventanas |
| `bg.panel_alt` | `#332414` | Hover / header |
| `line` | `#4a3828` | Bordes |
| `neutral.500` | `#7a6e62` | Iconos inactivos, divisores suaves |
| `text.muted` | `#9a9084` | Secundario, timestamps |
| `text.primary` | `#ffffff` | Títulos, chat propio, cuerpo sobre oscuro |
| `text.on_light` | `#160e04` | Cuerpo sobre blanco |
| `bg.light` | `#ffffff` | Superficies claras |
| `bg.light_muted` | `#f7f6f4` | Fondos claros alternos |

---

## Tokens de UI (mapeo)

| Token | Hex | Uso |
| --- | --- | --- |
| `accent.primary` | `#dcc552` | Alias de `brand.400` |
| `state.danger` | `#C45C4A` | HP, error, invite urgente (semántico, no marca) |
| `state.ok` | `#6FAE6A` | Ready, connected (semántico) |
| `state.warn` | `#D4A056` | Cola, aviso (semántico; no sustituye al oro de marca) |
| `state.mana` | `#9a9084` | Maná: gris, no un segundo acento |
| `team.ally` | `#4A7CB8` | Tinte 5v5 aliado (sutil; solo combate) |
| `team.enemy` | `#C45C4A` | Tinte 5v5 rival (sutil; mismo rojo que danger) |

Los `state.*` y `team.*` son **legibilidad de juego**, no paleta de marca. Se usan en barras, tintes y toasts; no en CTAs ni cromo de menú.

**Chat:** fondo `bg.panel` a ~90 % opacidad; burbuja sistema usa `text.muted`; whispers `brand.400` suave (`brand.200` / 20 % overlay).

**Barras:** vitalidad `state.danger`, stamina `state.warn`, maná `state.mana`, XP (cuando exista) `brand.400`. Tres vitals; ver `docs/GDD.md` §6.

---

## Tipografía

- UI y números: **Pixeloid Sans** (única familia de UI en A). Tamaños enteros (8 / 12 / 16 px) a resolución nativa.
- No mezclar una segunda sans “de diseño” hasta que se decida.

## Presentación (cliente A)

- Resolución nativa: **640×360**. Tiles **32×32**. Viewport ≈ **20×11** tiles.
- Personaje: ancho de cuerpo **~32 px**. Lienzo candidato **48×48** o más alto **32×62** (elegir en greybox).
- **Todo el arte A es mockup:** primitivas, bocetos, greybox. Icono de reino: **cuadrado verde** (el mismo para todos hasta hay iconos).
- 5v5: tinte de silueta `team.ally` / `team.enemy` (azul / rojo, sutil).

---

## Espaciado y forma

- Radio de panel: 6–8 px (poco). Nada de pill buttons en todo el HUD.
- Padding de ventana: 12–16 px.
- Chat flotante: esquina (recomendado inferior izquierda), no centro.
- Nametag **lobby:** nombre `text.primary` + icono de reino. **Combate:** sin nametag ni barra HP sobre la cabeza (las tres vitals van al HUD).

---

## Godot

- Un `Theme` principal: `nexum-terra/ui/theme/nexum_theme.tres` (cuando exista) mapeado a estos tokens.
- Prohibido hardcodear `Color(0.2, 0.6, 1)` o el teal viejo `#3D8B7A` en scripts de UI. Usar colores del Theme o constantes en `ui/tokens.gd` generadas desde esta tabla. No existe `accent.secondary`.
- Icono de app actual (`icon.svg`) se reemplaza cuando haya logo; hasta entonces no se usan otros logos sueltos.

---

## Voz de copy (UI)

- Español del jugador: claro, corto, de tú.
- Sistema: “En cola 1v1…” no “Matchmaking ticket queued”.
- Errores: qué pasó + qué hacer (“No se pudo entrar a la sala. Reintenta.”).

---

## Cómo actualizar

1. Cambiar hex/tokens aquí.
2. Reflejar en Theme / `tokens.gd` / CSS de landing en el mismo cambio.
3. No dejar paletas viejas “por si acaso” (`#C4A35A`, `#3D8B7A`, `#0E1412`, etc.).
