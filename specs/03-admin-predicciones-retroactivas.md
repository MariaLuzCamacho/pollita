# 03-admin-predicciones-retroactivas

- **Estado:** Aprobado
- **Fecha:** 2026-06-29
- **Dependencias:** ninguna (comparte lógica de modal y upsert de predicciones con el código base)
- **Objetivo:** Permitir al usuario adminmlc ingresar predicciones faltantes en nombre
  de otros usuarios para partidos ya cerrados, desde la vista principal de Predicciones.

---

## Scope

### Incluido
- Botón "Agregar predicción" visible solo para adminmlc en las tarjetas de partidos
  ya cerrados (fecha+hora pasada) en la vista principal de Predicciones
- Modal con dropdown de usuarios que aún no tienen predicción para ese partido,
  dos campos numéricos de marcador y botón guardar
- Upsert a `predicciones` solo si el usuario seleccionado no tiene predicción previa
  para ese partido (no sobreescribe)
- El dropdown se filtra dinámicamente: solo muestra usuarios sin predicción
- Tras guardar, el modal permite ingresar otra predicción faltante para el mismo
  partido (dropdown actualizado, sin el usuario recién guardado)

### No incluido
- Sobreescritura de predicciones existentes (el admin solo completa las faltantes)
- Ingresar predicciones para partidos aún no cerrados
- Registro de auditoría ("ingresada por admin")
- Notificaciones al usuario cuando se carga su predicción
- Acceso a esta funcionalidad desde el Dashboard

---

## Modelo de datos

No se introducen tablas ni columnas nuevas.

### Consultas nuevas (JS, on-demand al abrir el modal admin)

**`cargarUsuariosSinPrediccion(partido_id)`**
Consulta todos los usuarios de `usuarios` y las predicciones existentes en
`predicciones` filtradas por `partido_id`. Retorna la lista de usuarios cuyo
`id` no aparece en las predicciones de ese partido.

Estructura del resultado en memoria (local al modal, no global):
- `usuariosSinPrediccion` — array de `{ id, nombre }` ordenado por nombre

### Escritura

Reutiliza el upsert existente sobre `predicciones` con
`onConflict: "usuario_id,partido_id"`. Como solo se llama cuando el usuario
no tiene predicción previa, en la práctica siempre es un INSERT.

---

## Plan de implementación

1. **Botón en tarjetas cerradas (solo adminmlc):** En `renderizarTarjetaPartido()`,
   cuando el partido está cerrado (`partidoAbiertoParaPrediccion === false`) y
   `usuarioActual.nombre === "adminmlc"`, agregar un botón "Agregar predicción"
   debajo del mensaje de cierre existente.

2. **Modal admin:** Crear un modal separado del modal de predicción normal (o
   reutilizar su estructura HTML con contenido distinto). Contiene:
   - Nombre del partido en el encabezado
   - Dropdown `<select>` con los usuarios sin predicción (cargado dinámicamente)
   - Mensaje "Todos los usuarios ya tienen predicción" si el dropdown está vacío
   - Dos inputs numéricos (goles equipo 1 y equipo 2)
   - Botón "Guardar"

3. **Carga de usuarios sin predicción:** Al abrir el modal, llamar
   `cargarUsuariosSinPrediccion(partido_id)`: consultar `usuarios` y
   `predicciones` filtrado por `partido_id`, calcular la diferencia y poblar
   el dropdown. Si el resultado es vacío, mostrar mensaje y deshabilitar el form.

4. **Guardar predicción:** Al hacer click en "Guardar", upsert a `predicciones`
   con `usuario_id` del usuario seleccionado, `partido_id` del partido, y los
   goles ingresados. Tras el upsert exitoso, remover ese usuario del dropdown
   y limpiar los inputs para permitir ingresar la siguiente predicción faltante.

5. **Actualización de la tarjeta:** Tras guardar, actualizar `prediccionesUsuario`
   en memoria si el usuario guardado es `usuarioActual`, y re-renderizar la
   tarjeta del partido afectado.

6. **CSS:** Estilos para el botón "Agregar predicción" y el estado vacío del
   dropdown, coherentes con el diseño oscuro existente.

---

## Criterios de aceptación

- [ ] El botón "Agregar predicción" aparece en tarjetas de partidos cerrados
      únicamente cuando el usuario logueado es adminmlc.
- [ ] El botón NO aparece para usuarios distintos de adminmlc.
- [ ] El botón NO aparece en tarjetas de partidos aún abiertos.
- [ ] Al hacer click, se abre el modal con el nombre del partido en el encabezado.
- [ ] El dropdown contiene únicamente usuarios que no tienen predicción para ese partido.
- [ ] Si todos los usuarios ya tienen predicción, el dropdown está vacío y el
      formulario muestra "Todos los usuarios ya tienen predicción".
- [ ] El admin puede seleccionar un usuario, ingresar dos valores numéricos y guardar.
- [ ] Tras guardar, el usuario recién guardado desaparece del dropdown sin cerrar el modal.
- [ ] Los inputs se limpian tras guardar para facilitar la siguiente entrada.
- [ ] El upsert no sobreescribe predicciones existentes (la consulta previa garantiza
      que solo usuarios sin predicción aparecen en el dropdown).
- [ ] Las tarjetas de predicciones de usuarios normales no sufren regresiones.
- [ ] La vista principal de Predicciones no sufre regresiones para usuarios no admin.

---

## Decisiones tomadas y descartadas

- **Modal separado del modal de predicción normal:** El flujo admin (seleccionar usuario +
  ingresar marcador) es distinto al flujo normal (solo ingresar marcador para uno mismo).
  Mezclarlos en un solo modal con lógica condicional complicaría el código sin beneficio.

- **Filtro en cliente, no en BD:** `cargarUsuariosSinPrediccion` trae todos los usuarios
  y todas las predicciones del partido, y calcula la diferencia en JS. El volumen es bajo
  (pocos usuarios, un partido) y evita una query SQL con LEFT JOIN en Supabase.

- **No sobreescribir predicciones existentes:** Protege la integridad del juego.
  Un usuario que predijo a tiempo no debería ver su predicción alterada por el admin.

- **Dropdown en lugar de lista con inputs inline:** Más simple de implementar y evita
  que el admin envíe accidentalmente un formulario vacío para un usuario. El volumen
  de usuarios faltantes por partido es pequeño.

- **Sin auditoría ("ingresada por admin"):** Añadiría una columna nueva en `predicciones`
  y lógica de visualización. Para un juego privado, la confianza implícita en el admin
  es suficiente.

- **Descartado:** Acceso desde el Dashboard — el usuario eligió la vista principal de
  Predicciones para mantener el flujo de entrada en un solo lugar.

- **Descartado:** Notificaciones al usuario — fuera de scope, puede ser un spec futuro.

---

## Riesgos identificados

- **Concurrencia al abrir el modal:** Si un usuario envía su predicción mientras el admin
  tiene el modal abierto, ese usuario seguirá visible en el dropdown hasta que el admin
  lo intente guardar. Mitigación: el upsert con `onConflict` no sobreescribe, así que
  el peor caso es un upsert que actualiza con los mismos datos que ya existían. Riesgo
  aceptado dado el contexto de uso privado y baja concurrencia.

- **Inputs sin validación de rango:** Los campos numéricos podrían recibir valores
  negativos o decimales. Agregar `min="0"` y `step="1"` en los `<input type="number">`
  es suficiente mitigación del lado cliente.

- **adminmlc aparece en el dropdown de usuarios faltantes:** Si adminmlc no predijo
  ese partido, su propio nombre aparecerá en la lista. Es comportamiento correcto
  (puede ingresar su propia predicción retroactiva), pero puede sorprender. No requiere
  mitigación especial.
