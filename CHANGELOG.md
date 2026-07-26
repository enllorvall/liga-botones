# Changelog — Liga de Botones

Registro de cambios y mejoras de la aplicación web `index.html`.

---

## [v0.14] — 2026-07-26

### Migración de Supabase a Firebase Realtime Database

Motivo: Supabase pausaba el proyecto tras 7 días de inactividad (ya ocurrido dos veces, con solo 1 reactivación disponible en el plan gratuito). Firebase Spark (gratuito) no pausa proyectos por inactividad y tiene realtime nativo.

- Nuevo proyecto Firebase `liga-botones`, Realtime Database en `europe-west1` (Bélgica), reglas en modo de prueba (expiran 2026-08-25 — **pendiente fijar reglas permanentes antes de esa fecha**).
- SDK: `firebase-app-compat.js` + `firebase-database-compat.js` vía CDN (sin build/NPM, mismo patrón que `supabase-js`).
- `db` sigue exponiendo la misma interfaz (`from(tabla).select().order()`, `.insert()`, `.update().eq()`, `.delete().eq()`) mediante un shim (`_fbTable`) que traduce a llamadas de Realtime Database, para no tocar los ~30 puntos de uso existentes en el resto del fichero.
- Canal realtime de Supabase (`postgres_changes`) sustituido por listeners `onValue` de Firebase sobre `jugadores`/`plantilla`/`ligas`/`torneos`/`partidos`, ignorando el evento inicial para replicar el comportamiento anterior.
- Datos migrados de Supabase (1 liga, 8 equipos, 14 jornadas, 84 partidos, 101 jugadores de plantilla) al nuevo Realtime Database, verificado tras migración.
- Eliminado `supabaseKeepalive()` y su `setInterval` (ya no hace falta ping de actividad).
- Pendiente: retirar `keepalive.html` y `.github/workflows/supabase-keepalive.yml` (obsoletos), y decidir qué hacer con el proyecto Supabase antiguo.

## [v0.13] — 2026-07-26

### Responsive, modales rápidos, estadísticas y PDF

**Correcciones responsive móvil / tablet:**
- `@media (max-width:480px)`: `.grid2` pasa a `1fr` (soluciona overflow de porteros y todos los grids de 2 columnas).
- `.banda-cell` reducido a 18×18 px con `gap:2px` en móvil; la fila de 8 clavados ya no desborda.
- `.card` con `padding:12px` en móvil para ganar espacio horizontal.
- `input, select, textarea`: `max-width:100%!important; min-width:0!important` en móvil.
- `.inp`: `min-width:0` global para evitar que `<input type="date">` desborde en grids.
- `.grid2 > *`: `min-width:0` global (fix definitivo del overflow del campo fecha en iOS/Chrome).
- `input[type="date"]`: `-webkit-appearance:none` + pseudo-elementos `::-webkit-date-and-time-value` y `::-webkit-datetime-edit` para forzar ancho correcto.
- `#app`: `overflow-x:hidden` para evitar scroll horizontal global.
- Grid Jornada/Fecha: reemplazado `grid2` por `grid-template-columns:minmax(0,90px) minmax(0,1fr)` con `overflow:hidden` en celda Fecha.

**Modales rápidos de gol y fauts (acta en curso):**
- Botones `+ Gol` y `+ Faut` abren modales independientes en lugar de formularios inline.
- El minuto se rellena automáticamente con el minuto actual del cronómetro (`calcSegDisplay()`), editable.
- Selector de jugador con optgroups por equipo; si se elige del desplegable, el equipo se auto-detecta y no se muestra el selector manual.
- Si se escribe un nombre libre (`✏ Escribir nombre...`), aparece el selector de equipo.
- Funciones globales: `abrirModalGol()`, `abrirModalFauts()`, `_detectarEqJugador()`, `_inyectarModal()`.

**Estadísticas — "Ver más":**
- Los rankings (goleadores, porteros, faltas, expulsados) muestran los 5 primeros (`STATS_LIMIT = 5`).
- Si hay más, aparece `<details>` nativo con `Ver N más ▾` sin JS adicional.
- Expulsados: unificado con `rl()`, eliminado el `.slice(0,8)` hardcoded.

**Exportación a PDF:**
- Botón `📄 PDF` en la cabecera de cada competición en la pantalla Estadísticas → `imprimirStats()`.
- Genera PDF con todos los rankings, clavados por equipo, tiros a gol y goles por franja de minutos.
- Botón `📄 Actas PDF` en la cabecera de cada liga → `imprimirActasLiga(ligaId)`.
- Genera PDF con portada de liga + todas las actas completadas ordenadas por jornada, con salto de página entre cada una.
- `buildActaPDFHtml(p)` extraída como función reutilizable; `imprimirActa(p)` la usa internamente.

---

## [v0.12] — 2026-05-16

### Mejoras de acta, marcador y modal de edición

**Aviso de próxima expulsión por clavados:**
- Al llegar al clavado 7, 11, 15 o 19, suena un buzzer (MP3 integrado en base64) y aparece el aviso "⚠ Próxima expulsión" en la sección de clavados del acta.
- En la pantalla Marcador aparece el badge "PRÓXIMA EXP." en carmín bajo el nombre del equipo afectado.
- Los cambios de fueras llaman a `sincronizarEnCurso()` para que el espectador también reciba el dato actualizado.

**Pantalla Marcador — mejoras:**
- Botón "Actualizar" eliminado (la actualización es automática cada 7 segundos).
- Cronómetro más grande: de 56 px a 72 px.
- Sin acta abierta muestra "Sin partido en juego" en lugar del último partido disputado, tanto en el dispositivo del árbitro como en el espectador.
- El botón Marcador del header ahora funciona desde cualquier pantalla (listener movido a `attachEvents` global).
- El espectador solo ve partido activo si `cron_estado.estado` es un estado en juego (`1run`, `1pause`, `fin1`, `2run`, `2pause`); los partidos con `idle` o `fin` ya no aparecen.

**Modal de edición de acta — correcciones:**
- Typo "Cancelarr" → "Cancelar".
- Nueva sección Córners: lista de córners existentes con badge Gol/Sin gol, botón para eliminar y formulario para añadir (equipo + ¿acabó en gol?).
- Expulsiones: el minuto ya no es obligatorio.

**Plantilla de jugadores:**
- Los jugadores se muestran ordenados por dorsal ascendente; los que no tienen dorsal aparecen al final ordenados alfabéticamente.
- En el acta, los campos de portero muestran un datalist con los jugadores de posición "portero" de cada equipo. Si no hay ninguno registrado, sigue siendo texto libre.

---

## [v0.11] — 2026-05-16

### Rediseño completo del cronómetro

**Nueva máquina de estados explícita:**
- Campo `CRON.estado`: `'idle' | '1run' | '1pause' | 'fin1' | '2run' | '2pause' | 'fin'`.
- Sustituye el anterior sistema de flags booleanos (`run`, `fin`, `alerta`, `parte`, `descuento`).

**Flujo del partido:**
- **Inicio partido** → cronómetro arranca desde 00:00, suena `sonidoiniciopartido64.mp3`.
- **Final 1ª parte** → cronómetro se detiene, suena `sonido final partido.mp3`.
- **Inicio 2ª parte** → cronómetro arranca desde 15:00 (configurable) y sigue acumulando. Debajo aparece «2ª parte · Minuto X» actualizado en tiempo real.
- **Final del partido** → cronómetro se detiene, suena `sonido final partido.mp3`.
- **Pausar / Reanudar** → disponible en cualquier estado en marcha.
- **Reiniciar** → pide confirmación, vuelve a `idle`.

**Botón bocina eliminado:** redundante con el nuevo flujo; los sonidos se lanzan automáticamente.

**Eliminado código muerto:** funciones `calcSegParte`, `calcSegTotal`, `calcLimite`, `acumularSegParte`, `cronDescuento`, `cronIniciar`, `cronSig2`, `cronFin` y variables `cronPartidoIniciado`, `cronAlertado`.

**Marcador en directo actualizado:** tanto el árbitro (CRON local) como el espectador (vía `cron_estado` de Supabase) muestran el tiempo correcto con el nuevo formato de payload.

---

## [v0.10] — 2026-05-14

### Plantilla de jugadores + editor de kit

**Nueva tabla Supabase `plantilla`:**
- Campos: `id`, `equipo_id`, `nombre`, `posicion`, `dorsal`, `created_at`.
- Permisos explícitos para `anon`, `authenticated` y `service_role`.
- Columnas nuevas en `jugadores`: `color_primario`, `color_secundario`, `patron`.

**Modal de equipo:**
- Al clicar cualquier tarjeta en la pantalla Equipos se abre un modal de equipo con dos secciones: editor de kit y plantilla.

**Editor de kit en vivo:**
- Dos `color-picker` (color primario / secundario) y 6 patrones: `solid`, `halves`, `vstripes`, `hoops`, `sash`, `quarters`.
- Preview SVG del botón que se actualiza en tiempo real al cambiar color o patrón.
- Botón "Guardar kit" persiste los valores en Supabase (`jugadores.color_primario`, `color_secundario`, `patron`).
- `kitEq()` actualizado: prioridad → BD → mapa estático `TEAM_KIT` → fallback gris.
- Nueva función `buttonSVGFromKit(kit, size)` para preview a partir de un objeto kit directo.

**Plantilla de jugadores:**
- Lista de jugadores del equipo con dorsal (en ámbar), nombre y posición.
- Formulario para añadir jugador (nombre obligatorio, posición y dorsal opcionales).
- Eliminar jugador con confirmación. Solo disponible en modo árbitro.

**Autocompletado en el acta:**
- Los campos de goleador, jugador (faltas/expulsiones) y sale/entra (cambios) muestran `<datalist>` con sugerencias de la plantilla de ambos equipos.
- Aplica tanto al acta en curso (WIP) como al modal de edición de acta.
- Texto libre como fallback si no hay plantilla registrada.

**Realtime:** suscripción añadida para la tabla `plantilla`.

---

## [v0.9] — 2026-05-13

### Rediseño visual completo — Sistema editorial «Mesa de juego»

Se aplica el sistema visual diseñado en `entrega-claude-code/app-original/index.html` directamente sobre la app funcional, sin tocar ninguna lógica de Supabase, WIP, PIN ni vanilla JS.

**Identidad visual:**
- Fondo de fieltro verde profundo (`--felt-deep: #0c1f14`) en toda la app.
- Tipografía rediseñada: **Anton** para titulares y nombres de equipo, **DM Sans** para texto corrido, **JetBrains Mono** para cifras, kickers y etiquetas.
- Paleta de tokens CSS (`--chalk`, `--amber`, `--carmin`, `--win`, `--loss`, `--draw`, etc.).
- Textura de fieltro en el `body` (puntitos + tramado diagonal) vía `::before` / `::after`.
- Cero emojis en la interfaz; reemplazados por texto tipográfico o SVG.
- Sin `border-radius` mayor de 4px — estética plana y editorial.

**Componentes nuevos:**
- `TEAM_KIT`: mapa de colores y patrones por equipo (18 equipos precargados con fallback genérico).
- `buttonSVG(nombre, size)`: función que genera el SVG del botón cosido (círculo bicolor, agujeros, hilo), con ID único por instancia para evitar conflictos de clipPath.
- Botón cosido integrado en: clasificación (24px), cards de partido (24px), modal de partido (36px), acta header (40px), marcador LED (48px), lista de equipos (32px).

**Pantalla por pantalla:**
- **Masthead**: wordmark `LIGA DE BOTONES` con "DE" en carmín cursiva, tamaño fluido `clamp(36px, 8vw, 72px)`.
- **Home**: cards de liga y torneo con nueva tipografía y hover en verde/ámbar.
- **Liga / Torneo**: tabla de clasificación rediseñada con botón cosido, número de posición, líder en ámbar, colista en carmín.
- **Calendario**: resultado en LED ámbar con glow, botones cosidos junto a cada nombre.
- **Modal de partido**: marcador LED en fondo negro, botones cosidos con porteros.
- **Acta**: header con botones cosidos a 40px, cronómetro sobre fondo negro con scanlines, contadores rediseñados, notas con borde carmín.
- **Marcador LED**: panel con scanlines, marcador score-num en ámbar con glow (`text-shadow`), botones cosidos a 48px.
- **Equipos**: lista con botón cosido a 32px + nombre en Anton uppercase.
- **App container**: `max-width` ampliado a 900px para aprovechar tablets/desktop.

---

## [v0.8] — 2026-05-12

### Estadísticas por competición
- Las estadísticas ahora son **independientes por liga**. Cada liga tiene sus propias estadísticas, separadas de las demás competiciones.
- **Pantalla de selección**: el botón 📊 Estadísticas muestra un picker con todas las ligas (incluyendo las terminadas), torneos y partidos amistosos. Se puede consultar el histórico de cualquier liga en cualquier momento.
- **Desde la liga**: nuevo botón "📊 Estadísticas" en la cabecera de cada liga que abre directamente las estadísticas filtradas de esa liga.
- **Desde el acta**: el botón 📊 del acta va directamente a las estadísticas de la competición de ese partido (liga, torneo o amistosos).
- **Breadcrumb**: desde cualquier vista de estadísticas filtradas, se puede volver al picker pulsando "Estadísticas" en el breadcrumb.
- Los amistosos y torneos siguen teniendo estadísticas propias separadas.

---

## [v0.7] — 2026-05-12

### Tiros a gol
- **Nuevo contador en el acta**: sección "🎯 Tiros a gol" con contador independiente para local y visitante, igual que el de paradas.
- **Guardado**: se persiste en Supabase como `tiros_local` / `tiros_visitante` al finalizar el acta y también durante la sincronización en curso.
- **PDF**: aparece en el acta impresa si algún equipo tiene tiros registrados.
- **Estadísticas**: nueva sección "Tiros a gol por equipo" en la pantalla de estadísticas con total, media por partido y porcentaje de eficacia (goles/tiros) para cada equipo.

---

## [v0.6] — 2026-05-12

### Mejoras responsive (móvil)
- **Clasificación**: reducido el padding de celdas y el `min-width` de la tabla en pantallas ≤480px para que quepan todas las columnas sin scroll horizontal.
- **Clavados (fueras de banda)**: en móvil los dos equipos se apilan verticalmente (antes en dos columnas), de modo que las 8 celdas de la primera fila ya no se salen de pantalla.
- **Marcador (pantalla pública)**: reducido el tamaño del marcador numérico (64px→44px) y el espaciado del grid en móvil para evitar que los nombres largos desplacen el resultado.

### Navegación
- **Clic en el título "Liga de Botones"**: navega al menú inicio desde cualquier pantalla.

---

## [v0.5] — 2026-05-11

### Marcador en tiempo real (correcciones)
- **Partido correcto en espectador**: el dispositivo espectador ahora selecciona el partido con el `cron_estado.tsSync` más reciente (el que el árbitro está sincronizando activamente), en lugar del partido más antiguo con goles o el de mayor `created_at`.
- **Sin parpadeo 00:00**: el ticker del cronómetro ya no se destruye y recrea en cada actualización. Ahora es una variable global (`marcadorCronTicker`) que sigue corriendo de forma continua.
- **Eventos en tiempo real**: los cambios de Supabase Realtime y el botón Actualizar ya no hacen un `render()` completo (que reconstruía todo el DOM y perdía el estado del ticker). En su lugar, solo actualizan los elementos de marcador, goles y resultado mediante `actualizarMarcadorDatos()`.
- **Botón Actualizar funcional**: antes hacía un `render()` completo que reiniciaba el ticker; ahora llama a `_recargarPartidosYActualizar()` correctamente.
- **Auto-refresco de datos**: intervalo cambiado de 5s a 7s; ya no reconstruye el DOM completo.

---

## [v0.4] — 2026-05-09

### Cronómetro rediseñado
- **Sistema basado en timestamps**: reemplazado el contador por incremento (`seg++`) por un sistema con `tsInicio` + tiempo acumulado (`segParte1`/`segParte2`). Más preciso, resistente a cambios de pestaña y bloqueos de pantalla.
- **Duración configurable**: campo numérico (5–45 min, por defecto 15) antes de iniciar el cronómetro.
- **Botón "2.ª Parte"**: el árbitro decide manualmente cuándo empieza la segunda parte; el tiempo total se muestra de forma acumulada (p. ej. minuto 19 si estamos en el min 4 de la 2.ª parte).
- **Tiempo de descuento (+1 min)**: botón disponible al final de cada parte para añadir minutos extra.
- **Persistencia en localStorage**: el estado del cronómetro se guarda en `lb_cron`; navegar fuera del acta y volver no resetea el tiempo.
- **Botón Reset**: visible desde que el partido se inicia (no solo al final).

### Marcador en tiempo real
- **Sincronización cross-device**: el campo `cron_estado` (JSONB) se guarda en Supabase con un `tsSync` timestamp. El dispositivo espectador calcula el tiempo en vivo como `segBase + (Date.now() - tsSync) / 1000` sin necesidad de recargar.
- **`sincronizarCron()`**: nueva función que actualiza `cron_estado` en Supabase en cada cambio de estado del cronómetro.
- **Supabase Realtime**: ya existía la suscripción; ahora también actualiza el marcador cuando cambia `cron_estado`.
- **Auto-refresco**: reducido de 15s a 5s.

> **Requisito**: ejecutar `ALTER TABLE partidos ADD COLUMN IF NOT EXISTS cron_estado JSONB;` en Supabase.

---

## [v0.3] — 2026-05-06

### Sección Córners
- Nueva sección en el acta para registrar córners: equipo que los lanza y si acabaron en gol.
- Incluidos en la sincronización con Supabase y en la exportación PDF.

### Exportación PDF
- Corregida la ausencia de **paradas de portero** en el PDF.
- Corregido error `undefined'` en el minuto de las expulsiones (campo eliminado previamente).
- Los córners aparecen en el PDF.

### Expulsiones
- Eliminado el campo **Minuto** de la sección de expulsiones (no es necesario en fútbol de botones).
- Reordenadas las opciones del desplegable **Motivo**: "Clavados (fueras de banda)" aparece primero.
- Renombrado "Doble falta" → **"Doble fauts"**.

### Sonido de inicio de partido
- Corregido el audio `snd-inicio`: el base64 original estaba corrompido y contenía caracteres no ASCII.
- Sustituido por referencia a archivo externo: `<audio src="sonidoiniciopartido64.mp3">`.
- Separados los bloques `try-catch` de `currentTime` y `play()` para mayor robustez.

---

## [v0.2] — 2026-05-03

### Preservación de datos del acta (WIP)
- Los datos introducidos en un acta en curso ya no se pierden al navegar a Marcador, Estadísticas u otras secciones.
- Implementado objeto global `WIP` + persistencia en `localStorage` (`lb_wip`).
- `salvarWipActual()` guarda el estado del DOM antes de navegar.
- Al volver al acta (desde Liga → clic en partido), se comprueba si el partido es el mismo (`WIP.partidoId === p.id`) antes de resetear; si coincide, se restauran los datos.

### Banner "Volver al acta"
- Aparece en las pantallas **Marcador**, **Estadísticas**, **Equipos** e **Inicio** cuando hay un acta en curso.
- Implementado con `onclick="volverAlActa()"` (función global) para evitar problemas de timing con el DOM dinámico.

---

## [v0.1] — Versión inicial

- Gestión de ligas y torneos (todos contra todos).
- Calendario automático por jornadas.
- Acta de partido: porteros, paradas, clavados/fueras de banda, goles, faltas, expulsiones, cambios, notas.
- Clasificación en tiempo real.
- Marcador público (pantalla separada).
- Exportación PDF del acta.
- Modo árbitro con contraseña.
- Conexión con Supabase (PostgreSQL + Realtime).
- Sonidos: inicio de partido, gol y pitido final.
