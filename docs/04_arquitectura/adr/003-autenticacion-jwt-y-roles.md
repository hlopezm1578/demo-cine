# ADR-003 — Autenticación con JWT en cookie HttpOnly y rol en la tabla de usuarios

- **Estado:** Aceptada
- **Fecha:** 2026-08-23
- **Resuelve:** cómo el sistema sabe quién llama y qué puede hacer (socio / coordinadora)

## Contexto

Tres audiencias: visitante (nada), socio (calificar) y coordinadora (gestionar catálogo). Necesitamos identificar al llamador tanto en las páginas HTML como en la API JSON. Restricción dura: el plan gratis de Render **duerme el servicio** y lo despierta en otra máquina; no hay memoria entre peticiones. El cliente exige registro simple: nombre, email, contraseña, nada más (C3).

## Opciones consideradas

| Opción | A favor | En contra |
|---|---|---|
| **A. Sesiones en base de datos + cookie con id** | Se puede cerrar una sesión desde el servidor | Una consulta extra por petición; tabla que crece y hay que limpiar |
| **B. Token JWT firmado en cookie HttpOnly** | Sin estado: el servidor no recuerda nada; el token porta id y rol firmados | Revocar un token antes de su expiración es difícil |
| **C. OAuth con Google** | Sin contraseñas propias | Cuenta externa, aprobaciones, complejidad muy superior al demo |

## Decisión

**Opción B**, con estos detalles:

- **Firma:** HS256 con clave secreta leída de variables de entorno, nunca en el código (RNF-03).
- **En la web:** cookie `sesion` con `HttpOnly` (JavaScript no puede leerla: mitiga robo por XSS) y `SameSite=Lax`. Duración: 7 días.
- **En la API:** además acepta cabecera `Authorization: Bearer`, para probar todo desde `/docs` con el botón Authorize.
- **Roles:** columna `rol` en usuarios (`'socio'` / `'admin'`). Con dos roles, una columna basta.
- **Contraseñas:** hash **bcrypt** directo. No se usa passlib porque su última versión rompe con bcrypt ≥ 4.1: un error que golpearía a cada alumno en su casa.

## Consecuencias

**Positivas**
- Funciona igual con el servicio dormido y despierto: no hay estado en memoria.
- Un solo mecanismo (token) sirve para la web y para la API.
- Login = emitir token; logout = borrar la cookie. Concepto simple de explicar.

**Negativas**
- "Cerrar sesión" solo borra la cookie del navegador: el token sigue válido si alguien lo copió.
- Si la clave secreta se filtra, cualquiera fabrica tokens de admin. En producción se rota.
- Sin lista negra no hay "cerrar todas las sesiones".

## Para conversar en clase

1. ¿Qué puede hacer un atacante que robe una cookie HttpOnly? ¿Y una que no lo es?
2. ¿Por qué el mensaje de login fallado es genérico y no dice qué dato estuvo mal?
3. ¿Cómo agregarían un tercer rol ("moderador") sin romper lo existente?
