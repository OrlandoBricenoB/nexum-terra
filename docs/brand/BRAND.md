# Nexum Terra — Marca y UI

**Única fuente de verdad visual.** Antes de Theme Godot, iconos, landing o colores “de gusto”, se lee y se actualiza este archivo.

Estado: **tokens temporales**. Sustituir hex cuando el branding esté cerrado. No inventar una paleta paralela en código.

---

## Identidad

- **Nombre:** Nexum Terra
- **Tono:** terrestre, nexo, misterio contenido — no cartoon neón genérico ni sci-fi clínico.
- **Vista:** top-down 2D. UI densa de MMO ligero (chat, nametag, barras), no mobile-casual vacío.

---

## Color (tokens)

| Token | Hex temporal | Uso |
| --- | --- | --- |
| `bg.deep` | `#0E1412` | Fondos de menú, paneles |
| `bg.panel` | `#16201C` | Cards, chat, ventanas |
| `bg.panel_alt` | `#1C2A24` | Hover / header |
| `line` | `#2F453C` | Bordes |
| `text.primary` | `#E8F0EA` | Títulos, chat propio |
| `text.muted` | `#8FA399` | Secundario, timestamps |
| `accent.primary` | `#C4A35A` | CTA, focus, oro terroso |
| `accent.secondary` | `#3D8B7A` | Selección, links, mana/ok |
| `state.danger` | `#C45C4A` | HP, error, invite urgente |
| `state.ok` | `#6FAE6A` | Ready, connected |
| `state.warn` | `#D4A056` | Cola, aviso |

**Chat:** fondo `bg.panel` a ~90% opacidad; burbuja sistema usa `accent.secondary`; whispers `accent.primary` suave.

**Barras:** vitalidad `state.danger`, stamina `state.warn`, maná `accent.secondary`, XP (cuando exista) `accent.primary`. Tres vitals; ver `docs/GDD.md` §6.

---

## Tipografía

- UI: sans geométrica legible a 13–16 px (candidato: *Source Sans 3* / *IBM Plex Sans* — decidir uno y no mezclar).
- Números de daño: tabular / semibold, outline oscuro para top-down.
- No más de **dos** familias.

---

## Espaciado y forma

- Radio de panel: 6–8 px (poco). Nada de pill buttons en todo el HUD.
- Padding de ventana: 12–16 px.
- Chat flotante: esquina (recomendado inferior izquierda), no centro.
- Nametag: nombre `text.primary` + barra HP estrecha debajo.

---

## Godot

- Un `Theme` principal: `nexum-terra/ui/theme/nexum_theme.tres` (cuando exista) mapeado a estos tokens.
- Prohibido hardcodear `Color(0.2, 0.6, 1)` en scripts de UI. Usar colores del Theme o constantes en `ui/tokens.gd` generadas desde esta tabla.
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
3. No dejar paletas viejas “por si acaso”.
