# ADR-006 — Frontend: páginas de servidor con Jinja2 (y JavaScript solo donde suma)

- **Estado:** Aceptada
- **Fecha:** 2026-08-23
- **Resuelve:** cómo se sirven las 4 pantallas del diseño (§4) — RNF-06, C4

## Contexto

El diseño definió 4 pantallas con lineamiento explícito: celular primero (C4), un solo JavaScript en todo el sitio (el widget de estrellas). El foco del producto es el backend, pero la demo debe verse bien y funcionar. Se despliega gratis en un único servicio (ADR-007).

## Opciones consideradas

| Opción | A favor | En contra |
|---|---|---|
| **A. SPA (React/Vue) + API separada** | Experiencia de app moderna; equipos separados como en la industria | Dos proyectos, dos despliegues, CORS, Node/npm: el doble de pasos antes de ver algo |
| **B. Server-side rendering con Jinja2 desde el propio FastAPI** | Un solo proyecto y un solo despliegue; HTML generado junto a los datos; sin CORS | Interactividad limitada; el frontend "vive" dentro del backend |
| **C. Sitio estático + API** | Baratísimo de servir | Las páginas no pueden adaptarse al socio logueado sin construir la A de todos modos |

## Decisión

**Opción B.** FastAPI sirve plantillas Jinja2 (las 4 pantallas del diseño) y archivos estáticos (CSS, el widget de estrellas). El único JavaScript es el widget, que llama a la API JSON y recarga.

Detalle pedagógico clave (ADR-001 en acción): la página HTML y el endpoint JSON llaman **al mismo servicio** de calificaciones. Dos caras, una sola lógica — el valor de las capas visto en vivo.

## Consecuencias

**Positivas**
- Un `git push` → un servicio → todo el producto publicado. El camino del alumno hasta "mi app con URL pública" es el más corto posible (CS4).
- El HTML llega renderizado con los datos: funciona sin JavaScript salvo las estrellas.

**Negativas**
- Cada interacción nueva (buscador en vivo, paginación sin recarga) empuja a más JavaScript o a recargar páginas.
- No refleja la separación de equipos frontend/backend de la industria — hay que decirlo y mostrar cuándo conviene partir.

## Para conversar en clase

1. ¿Qué pantalla convertirían primero en SPA y por qué? (pista: la que más recargas genera).
2. ¿Qué es CORS y por qué esta arquitectura no lo necesita pero la opción A sí?
3. El widget de estrellas habla con la API JSON y no con un endpoint HTML: ¿qué gana y qué pierde ese diseño?
