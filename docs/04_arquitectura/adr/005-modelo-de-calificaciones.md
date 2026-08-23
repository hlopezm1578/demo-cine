# ADR-005 — Calificaciones: UNIQUE por socio+película, upsert y promedio calculado al leer

- **Estado:** Aceptada
- **Fecha:** 2026-08-23
- **Resuelve:** cómo garantizar RN-01 (un voto por socio) y mantener el promedio (RF-08, RF-09, CS3)

## Contexto

El corazón del producto es la estrella: enteros 1–5, promedio visible en catálogo y ficha, y la regla RN-01: un socio tiene a lo más una calificación por película (recalificar reemplaza). El diseño ya dejó la restricción UNIQUE en el modelo de datos (fase 3, §2.3.1); este ADR decide cómo se comporta el código sobre esa base.

## Opciones consideradas

| Aspecto | Opción A | Opción B (elegida) |
|---|---|---|
| **Unicidad** | Chequear en Python antes de insertar | Restricción UNIQUE **en la base** + upsert en el servicio (existe → se actualiza) |
| **Promedio** | Columna desnormalizada en PELICULA, recalculada al escribir | AVG calculado en la consulta de lectura |

## Decisión

**Unicidad en la base de datos.** El `if` en Python tiene una ventana: dos peticiones simultáneas del mismo socio pasan ambas el chequeo e insertan dos filas. La restricción UNIQUE cierra esa puerta aunque el código tenga un bug. El servicio implementa el "recalificar reemplaza": busca el voto existente y lo actualiza, de modo que la interfaz es un simple widget de estrellas.

**Promedio en lectura** con promedio y conteo agrupados por película, calculados por la base. Con decenas o cientos de filas la consulta es instantánea y la fuente de la verdad es una sola: los votos.

## Consecuencias

**Positivas**
- RN-01 se cumple por diseño de datos, no por disciplina del programador.
- El promedio nunca se desincroniza: no existe como dato guardado (diseño §2.3.6).
- "Recalificar" es natural en la interfaz: clic sobre otra estrella.

**Negativas**
- Con millones de votos, el AVG en cada carga se encarece → ahí aparece la desnormalización (columna o tabla resumen actualizada en la misma transacción del voto). Es una evolución, no un error.
- El upsert "leer-entonces-escribir" conserva una micro-ventana de carrera; en bases serias se resuelve con INSERT … ON CONFLICT UPDATE.

## Para conversar en clase

1. Dos pestañas del mismo socio votan a la vez: dibujen la línea de tiempo con el `if` en Python y con la restricción UNIQUE. ¿Qué pasa en cada caso?
2. ¿Qué habría que cambiar para guardar el historial de calificaciones?
3. YouTube muestra los "me gusta" instantáneos con millones de usuarios: ¿columna desnormalizada o AVG? ¿Cómo la mantendrían consistente?
