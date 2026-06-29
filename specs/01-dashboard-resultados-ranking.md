# 01-dashboard-resultados-ranking

- **Estado:** Aprobado
- **Fecha:** 2026-06-29
- **Dependencias:** ninguna (primer spec del proyecto)
- **Objetivo:** Agregar un dashboard con vista de resultados de partidos y ranking de usuarios, accesible mediante tabs en el header de la página existente.

---

## Scope

### Incluido
- Dos tabs en el header: "Predicciones" (vista actual) y "Dashboard"
- Vista "Resultados" dentro del Dashboard: tarjetas de partidos mostrando marcador final con etiqueta "FINAL", igual estilo visual que las tarjetas actuales
- Botón "Ver detalle" en cada tarjeta con resultado: abre modal con predicciones de todos los usuarios para ese partido
- Botón "Cargar resultado" en cada tarjeta (solo visible para el usuario "adminmlc"): permite ingresar `resultado_equipo_1` y `resultado_equipo_2`
- Vista "Ranking" dentro del Dashboard: tabla con posición, nombre y puntos totales de todos los usuarios registrados
- Cálculo de puntos: acierto exacto = 3 pts, acierto de ganador o empate = 1 pt, fallo = 0
- Nuevas columnas `resultado_equipo_1` y `resultado_equipo_2` (integer, nullable) en la tabla `partidos` de Supabase

### No incluido
- Autenticación real de administrador (solo validación por nombre de usuario)
- Historial de cambios de resultados
- Notificaciones a usuarios cuando se carga un resultado
- Resultados en tiempo real o integración con API de fútbol
- Sub-rankings por fase o grupo
- Partidos sin resultado aún no aparecen en la vista de resultados del Dashboard

---

## Modelo de datos

### Tabla `partidos` (modificación)
Agregar dos columnas nuevas via Supabase dashboard:

| Columna              | Tipo    | Default | Descripción                                      |
|----------------------|---------|---------|--------------------------------------------------|
| `resultado_equipo_1` | integer | null    | Goles del equipo local. NULL = sin resultado     |
| `resultado_equipo_2` | integer | null    | Goles del equipo visitante. NULL = sin resultado |

Un partido tiene resultado cuando ambas columnas son non-null.

### Cálculo de puntos (lógica JS, sin tabla nueva)
Función `calcularPuntos(prediccion, resultado)`:
- `resultado_equipo_1 === null` → partido sin resultado, no suma puntos
- predicción exacta (`goles_1 === res_1 && goles_2 === res_2`) → 3 pts
- mismo ganador o ambos empate (`signo(goles_1 - goles_2) === signo(res_1 - res_2)`) → 1 pt
- cualquier otro caso → 0 pts

### Estado global nuevo
- `prediccionesTodosUsuarios` — map de `partido_id → [{ usuario_nombre, goles_1, goles_2 }]` cargado on-demand al abrir el modal "Ver detalle"

---

## Plan de implementación

1. **Supabase:** Agregar columnas `resultado_equipo_1` y `resultado_equipo_2` (integer, nullable) a la tabla `partidos` via el dashboard de Supabase.

2. **Tabs en el header:** Agregar dos botones "Predicciones" y "Dashboard" al header existente. Click en cada tab muestra/oculta la sección correspondiente. La vista de predicciones es la activa por defecto.

3. **Estructura del Dashboard:** Dentro de la sección dashboard, agregar dos sub-tabs: "Resultados" y "Ranking". "Resultados" activo por defecto.

4. **Vista Resultados:** Renderizar tarjetas de todos los partidos que tengan resultado (ambas columnas non-null). Cada tarjeta muestra banderas, nombres, marcador con etiqueta "FINAL", fecha y botón "Ver detalle".

5. **Botón "Cargar resultado" (solo adminmlc):** En cada tarjeta de la vista Resultados (y también en partidos sin resultado), mostrar el botón solo si `usuarioActual.nombre === "adminmlc"`. Click abre un mini-formulario inline o modal con dos inputs numéricos y botón guardar. Upsert a Supabase y recarga la tarjeta.

6. **Modal "Ver detalle":** Al hacer click, consultar `predicciones` JOIN `usuarios` filtrado por `partido_id`. Mostrar tabla con nombre de usuario, predicción y puntos obtenidos (calculados con `calcularPuntos`).

7. **Vista Ranking:** Consultar todas las predicciones con resultado disponible, calcular puntos por usuario usando `calcularPuntos`, ordenar descendente, renderizar tabla con posición, nombre y puntos totales.

8. **CSS:** Estilos para tabs, sub-tabs, etiqueta "FINAL" en tarjetas, tabla de ranking y modal "Ver detalle", coherentes con el diseño oscuro existente.

---

## Criterios de aceptación

- [ ] El header muestra dos tabs: "Predicciones" y "Dashboard". Click alterna entre vistas.
- [ ] La vista de predicciones existente no sufre regresiones al navegar entre tabs.
- [ ] El dashboard muestra dos sub-tabs: "Resultados" y "Ranking".
- [ ] La vista Resultados muestra únicamente partidos con ambas columnas de resultado non-null.
- [ ] Cada tarjeta de resultado muestra banderas, nombres de equipos, marcador y etiqueta "FINAL".
- [ ] El botón "Cargar resultado" es visible únicamente cuando `usuarioActual.nombre === "adminmlc"`.
- [ ] El admin puede ingresar y guardar resultados; la tarjeta se actualiza sin recargar la página.
- [ ] El botón "Ver detalle" abre un modal con la lista de predicciones de todos los usuarios para ese partido, incluyendo puntos calculados por predicción.
- [ ] Un empate predicho correctamente (cualquier empate cuando el resultado es empate) suma 1 pt.
- [ ] Un marcador exacto suma 3 pts. Un resultado incorrecto suma 0 pts.
- [ ] La vista Ranking muestra todos los usuarios ordenados por puntos totales descendente, con posición, nombre y puntos.
- [ ] Partidos sin resultado no aparecen en Resultados ni cuentan en el Ranking.
- [ ] Un usuario sin predicciones aparece en el Ranking con 0 puntos.

---

## Decisiones tomadas y descartadas

- **Dashboard en la misma página (no página separada):** Mantiene el proyecto sin rutas ni servidor. Una segunda página HTML requeriría pasar estado entre páginas.

- **Admin por nombre hardcodeado ("adminmlc"), no por flag en BD:** Suficiente para un juego privado. Agregar un campo `es_admin` en Supabase requeriría lógica de roles innecesaria para este caso.

- **Resultado en tabla `partidos`, no en tabla separada:** Un partido tiene un único resultado final. No hay historial ni versiones, así que columnas directas son más simples.

- **Puntos calculados en JS, no en BD:** Evita funciones o vistas en Supabase. El volumen de datos (pocos usuarios, pocos partidos) no justifica cálculo server-side.

- **Empate predicho correctamente = 1 pt (no 3 pts):** Solo el marcador exacto vale 3. Acertar "empate" sin acertar el marcador exacto es parcialmente correcto.

- **"Ver detalle" carga predicciones on-demand:** No se pre-cargan todas las predicciones de todos los usuarios al iniciar la app para no penalizar el tiempo de carga inicial.

- **Descartado:** Sub-rankings por fase o grupo (fuera de scope, puede ser un spec futuro).
- **Descartado:** Notificaciones al cargar resultados (fuera de scope).

---

## Riesgos identificados

- **Cálculo de puntos inconsistente si faltan predicciones:** Si un usuario no predijo un partido con resultado, ese partido simplemente no suma puntos. Verificar que `calcularPuntos` maneje el caso `prediccion === undefined` → 0 pts.

- **Concurrencia al cargar resultados:** Si el admin guarda un resultado mientras otro usuario está viendo el modal "Ver detalle", los puntos mostrados pueden estar desactualizados. Mitigación: el modal recarga datos frescos de Supabase cada vez que se abre.

- **Nombre "adminmlc" en código fuente visible:** El JS está en el HTML sin ofuscación. Cualquier usuario puede ver el nombre del admin en el código. Riesgo aceptado dado el contexto de juego privado; la única acción protegida es cargar resultados.

- **Columnas nuevas en `partidos` rompen queries existentes:** Al agregar `resultado_equipo_1` y `resultado_equipo_2`, verificar que el `SELECT *` en `cargarPartidos()` las incluya automáticamente (lo hace por ser `*`) y que no rompan el renderizado de tarjetas existentes al ser `null`.
