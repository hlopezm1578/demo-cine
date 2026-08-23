# ADR-008 — API-first: el contrato OpenAPI se diseña y aprueba antes del código

- **Estado:** Aceptada
- **Fecha:** 2026-08-23
- **Resuelve:** cómo se define la interfaz de la API JSON (RF-10) y su relación con el desarrollo

## Contexto

El sistema expone una API JSON (RF-10) además de la web. Con FastAPI existe una tentación concreta: escribir el código primero y dejar que el framework **genere** la documentación automáticamente (`/docs`). Eso es **code-first**: el contrato llega después y hereda todas las decisiones del código, buenas y malas. El ramo exige aplicar **API-first**: la interfaz es un acuerdo explícito que se diseña, comenta y aprueba **antes** de implementarse, igual que el resto del ciclo de vida (necesidad → requerimientos → diseño → **contrato** → código).

## Opciones consideradas

| Opción | A favor | En contra |
|---|---|---|
| **A. Code-first** (FastAPI genera `/docs` desde el código) | Cero pasos extra; la doc nunca se desactualiza del código | El contrato llega tarde: no se puede discutir ni aprobar antes; renombrar una ruta o cambiar un código de respuesta es "gratis" y nadie se entera |
| **B. API-first manual** (contrato OpenAPI versionado primero; el código lo implementa) | La interfaz se discute en lenguaje de contrato; sirve como mock mientras no hay backend; evaluación objetiva del desarrollo | Doble mantenimiento (contrato + código) sin herramienta que los sincronice |
| **C. API-first con generación** (del contrato se genera el esqueleto de código) | Drift imposible por construcción | Tooling pesado para el alcance del ramo; esconde justo lo que se quiere enseñar |

## Decisión

**Opción B.** El contrato vive en [`contrato_api.yaml`](../contrato_api.yaml) (OpenAPI 3.0), es **la fuente de la verdad de la interfaz** y forma parte de la fase 4:

1. **Se aprueba antes de codificar** — mismo estándar de firmas que el resto de las fases.
2. **La fase 5 lo implementa sin desviarse**: nombres de rutas, parámetros, códigos de respuesta y esquemas del código deben coincidir con el contrato.
3. **Detección de desvío (drift) como actividad de clase:** al terminar el desarrollo, se abre el `/docs` generado por FastAPI **al lado** del contrato y se comparan. Cualquier diferencia es un hallazgo: o el código corrige, o el contrato se versiona y se aprueba de nuevo — nunca cambia en silencio.
4. Mientras no exista backend, el contrato permite **mockear** la API (importarlo en Swagger Editor o Postman) y adelantar trabajo de consumo.

## Consecuencias

**Positivas**
- La interfaz se critiquea en su momento: "¿por qué 409 y no 400 para el email duplicado?" se discute sobre el contrato, no sobre el código ya escrito.
- Consumidores (un futuro frontend separado, una app móvil, otro equipo del curso) pueden trabajar contra el contrato sin esperar al backend.
- La evaluación de la fase 5 es objetiva: ¿el endpoint cumple el contrato, sí o no?

**Negativas**
- Sin generación automática, el contrato y el código pueden diverdir; se mitiga con la comparación en clase (y como evolución: pruebas de contrato automáticas).
- Más un documento que mantener.

## Para conversar en clase

1. El código y el contrato discrepan: ¿quién manda y por qué? ¿Qué harían en un equipo real con clientes ya integrados?
2. Importen `contrato_api.yaml` en Swagger Editor **antes** de que exista código: ¿qué se puede hacer con una API que solo existe en papel?
3. ¿Qué código de respuesta corresponde para "email duplicado" (409) vs "estrellas fuera de rango" (422) vs "no iniciaste sesión" (401)? ¿Quién más en la industria usa estos códigos igual?
