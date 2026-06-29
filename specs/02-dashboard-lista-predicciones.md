# 02-dashboard-lista-predicciones

- **Estado:** Aprobado
- **Fecha:** 2026-06-29
- **Dependencias:** 01-dashboard-resultados-ranking (requiere la estructura de tabs del dashboard)
- **Objetivo:** Agregar un sub-tab "Predicciones" al dashboard que muestra, agrupado por partido o por usuario, las predicciones de todos los participantes para los partidos ya iniciados.

---

## Scope

### Incluido
- Tercer sub-tab "Predicciones" dentro del dashboard, al lado de "Resultados" y "Ranking"
- Toggle "Agrupar por: Partido / Usuario" dentro del sub-tab
- Vista agrupada por partido: lista de todos los partidos ya iniciados, cada uno expandido
  mostrando todos los usuarios con su predicción (o "—" si no predijo)
- Vista agrupada por usuario: lista de todos los usuarios, cada uno expandido mostrando
  sus predicciones para los partidos ya iniciados
- Visible para cualquier usuario logueado
- Los partidos aún no iniciados no aparecen en esta vista (sus predicciones permanecen ocultas)

### No incluido
- Predicciones de partidos que aún no han comenzado (ocultas hasta que el partido inicie)
- Información de resultados ni cálculo de puntos (eso corresponde al modal "Ver detalle" del spec 01)
- Filtros por fase, grupo u otros criterios
- Paginación o búsqueda
- Usuarios no logueados (requiere sesión activa para ver el dashboard)

---

## Modelo de datos

No se introducen tablas ni columnas nuevas.

### Consultas nuevas (JS, on-demand al activar el sub-tab)

**`cargarTodasLasPredicciones()`**
Consulta `predicciones` JOIN `usuarios` para todos los partidos ya iniciados
(`partidoAbiertoParaPrediccion(partido) === false`).

Estructura del resultado en memoria:
- `prediccionesGlobales` — array de objetos:
  `{ partido_id, usuario_nombre, goles_equipo_1, goles_equipo_2 }`

Se carga una sola vez al abrir el sub-tab y se invalida si el usuario
navega fuera y vuelve (recarga fresca cada vez).

---

## Plan de implementación

1. **Sub-tab "Predicciones":** Agregar el tercer botón "Predicciones" a los sub-tabs
   del dashboard, junto a "Resultados" y "Ranking". Click muestra la sección correspondiente
   y oculta las otras dos.

2. **Toggle de agrupación:** Dentro de la sección, agregar dos botones o un selector
   "Partido" / "Usuario". "Partido" activo por defecto.

3. **Carga de datos:** Al activar el sub-tab, llamar `cargarTodasLasPredicciones()`:
   consultar `predicciones` joined con `usuarios`, filtrar solo los partidos cuya
   `fecha`+`hora` ya pasó (usando la lógica existente de `partidoAbiertoParaPrediccion`),
   almacenar en `prediccionesGlobales`.

4. **Vista por partido:** Iterar `listaPartidos` filtrando los ya iniciados. Por cada
   partido renderizar un bloque con nombre de equipos, fecha y una fila por cada usuario
   registrado mostrando su predicción o "—" si no predijo ese partido.

5. **Vista por usuario:** Iterar todos los usuarios (obtenidos de `prediccionesGlobales`
   deduplicando por nombre). Por cada usuario renderizar un bloque con su nombre y una
   fila por cada partido ya iniciado mostrando su predicción o "—".

6. **CSS:** Estilos para el toggle de agrupación y los bloques de predicciones,
   coherentes con el diseño oscuro existente.

---

## Criterios de aceptación

- [ ] El dashboard muestra tres sub-tabs: "Resultados", "Ranking" y "Predicciones".
- [ ] Click en "Predicciones" muestra la nueva sección y oculta las otras dos.
- [ ] La sección contiene un toggle con dos opciones: "Partido" y "Usuario". "Partido" activo por defecto.
- [ ] Vista por partido: solo aparecen partidos ya iniciados (fecha+hora pasada).
- [ ] Vista por partido: cada bloque de partido muestra todos los usuarios registrados,
      con su predicción o "—" si no predijo ese partido.
- [ ] Vista por usuario: cada bloque de usuario muestra todos los partidos ya iniciados,
      con su predicción o "—" si no predijo ese partido.
- [ ] Partidos aún no iniciados no aparecen en ninguna de las dos vistas.
- [ ] Los datos se cargan frescos cada vez que se activa el sub-tab.
- [ ] No se muestran resultados ni puntos en esta vista.
- [ ] Las vistas "Resultados" y "Ranking" no sufren regresiones.
- [ ] Un usuario sin ninguna predicción aparece en la vista por usuario con "—" en todos los partidos.

---

## Decisiones tomadas y descartadas

- **Predicciones ocultas hasta que el partido inicie:** Mostrarlas antes quitaría el
  incentivo de predecir de forma independiente. Se reutiliza la lógica existente de
  `partidoAbiertoParaPrediccion`.

- **Toggle en lugar de sub-tabs anidados:** Agregar un cuarto nivel de navegación
  (tab → sub-tab → sub-sub-tab) haría la UI compleja. Un toggle dentro del sub-tab
  es suficiente para alternar entre agrupaciones.

- **Carga on-demand al activar el sub-tab:** Consistente con la decisión del spec 01
  para el modal "Ver detalle". Evita penalizar el tiempo de carga inicial.

- **Sin resultados ni puntos en esta vista:** Ya cubierto por el modal "Ver detalle"
  del spec 01. Mezclar ambas responsabilidades en una sola vista la haría más compleja
  sin beneficio adicional.

- **Descartado:** Paginación o filtros por fase — el volumen de datos (pocos usuarios,
  64 partidos máximo) no lo justifica.

- **Descartado:** Ocultar la vista a usuarios no logueados con mensaje específico —
  el dashboard completo ya requiere sesión, no se necesita manejo adicional aquí.
