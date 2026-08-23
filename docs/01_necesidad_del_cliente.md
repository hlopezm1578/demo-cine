# Fase 1 — La Necesidad del Cliente: CineClub Barrio

> **Módulo:** ISI601 Taller de Diseño de Sistemas · ISI602 Arquitectura de Software
> **Fase del ciclo de vida:** 1. Necesidad
> **Regla de esta fase:** el documento habla el idioma del CLIENTE. Cero tecnología, cero jerga de sistemas. La traducción a lenguaje técnico recién ocurre en la fase 2 (requerimientos).
> **Fecha:** 2026-08-23

---

## 1. El cliente

**CineClub Barrio** es un círculo de cine del barrio Yungay, Santiago. Funciona
desde 2019: cada mes alquilan una sala comunitaria, exhiben una película y
conversan después con once té de por medio. Tiene unos **40 socios** de todas
las edades y una coordinadora: **Macarena** (54, profesora de lenguaje), que
organiza todo desde su notebook y su teléfono.

## 2. Cómo funcionan hoy (situación actual)

- La lista de películas de la temporada vive en un **Excel** que Macarena
  actualiza a mano y manda por WhatsApp cada vez que cambia.
- Las **carátulas** están sueltas en la galería del teléfono: cuando un socio
  pregunta "¿cuál es la de este mes?", hay que mandarle la foto.
- Los **tráilers** no se comparten: cada socio busca el suyo en YouTube, si es
  que se acuerda.
- Después de cada función pasa una **encuesta en papel** ("ponle de 1 a 5
  estrellas") que completa menos de la mitad y que Macarena luego debe contar a
  mano.
- La selección de la próxima temporada la decide Macarena **por intuición**.

## 3. El problema (por qué duele)

| # | Dolor | Consecuencia |
|---|---|---|
| D1 | El catálogo llega desordenado y feo por WhatsApp | Los socios nuevos no se enteran bien y baja la asistencia |
| D2 | Nadie ve el tráiler antes de la función | La gente llega sin saber qué va a ver y deserta a mitad de temporada |
| D3 | Las estrellas en papel se pierden o nadie las cuenta | Las próximas películas se eligen a ciegas |
| D4 | Todo depende del teléfono y la memoria de Macarena | Si ella falla, el cineclub se detiene |

## 4. Lo que el cliente necesita (en sus propias palabras)

> *"Quiero una página donde estén las películas de la temporada con su carátula
> y su tráiler, que los socios le pongan estrellas después de verla, y que yo
> pueda agregar o sacar películas sin tener que pedirle ayuda a nadie."*
> — Macarena, coordinadora del CineClub Barrio

Formalizado como peticiones del cliente:

| # | Petición |
|---|---|
| **P1** | Publicar cada película con su **carátula** y el **link del tráiler** |
| **P2** | Un **catálogo** que cualquiera pueda mirar, con o sin cuenta |
| **P3** | Que los socios **califiquen con estrellas** (1 a 5) las películas |
| **P4** | Que **solo Macarena** administre el catálogo (agregar, quitar) |
| **P5** | Que todo funcione **gratis**: sin servidor propio, sin pagar hosting |

## 5. Objetivos del cliente (de negocio, no técnicos)

- **O1:** Aumentar la asistencia mensual de socios y visitas nuevas.
- **O2:** Elegir la próxima temporada con datos reales de gustos, no por intuición.
- **O3:** Proyectar una imagen ordenada y profesional del cineclub hacia el barrio.

## 6. Lo que el cliente NO pide (gestión de expectativas)

- Vender entradas ni cobrar membrecías (**no hay pagos**).
- Reseñas escritas ni comentarios (solo estrellas).
- Recomendaciones automáticas ("si te gustó X, mira Y").
- Red social entre socios, perfiles públicos, avatares.

Dejar esto por escrito evita el clásico "¿y si le agregamos…?" a mitad del proyecto.

## 7. Condiciones y restricciones del cliente

| # | Condición |
|---|---|
| C1 | **Presupuesto: $0.** Plataformas gratuitas, sin tarjeta de crédito. |
| C2 | **Personal:** Macarena sola como administradora, sin equipo técnico. |
| C3 | Los socios se registran con **nombre, email y contraseña** (nada más). |
| C4 | La página debe verse bien **en el celular** (la mayoría llega por WhatsApp). |
| C5 | El catálogo es **público**: cualquier vecino puede mirarlo sin registrarse. |

## 8. Criterios de éxito (cómo sabe el cliente que quedó resuelto)

| # | Criterio | Cómo se verifica |
|---|---|---|
| CS1 | Subir una película nueva toma **menos de 5 minutos** y no requiere ayuda técnica | Macarena lo hace sola, cronómetro en mano |
| CS2 | Al menos la **mitad de los socios califica** cada película exhibida | Conteo de calificaciones por película |
| CS3 | Macarena puede ver los **promedios** antes de elegir la próxima temporada | Revisa el catálogo antes de cada decisión |
| CS4 | La página funciona **sin que nadie cuide un servidor** | Pasan dos meses sin intervención técnica |

## 9. Aprobación de la fase

| Rol | Nombre | Decisión | Fecha |
|---|---|---|---|
| Cliente (coordinadora) | Macarena — CineClub Barrio | ☐ Aprobado ☐ Con observaciones | 2026-08-__ |
| Analista | ______________ | ☐ Aprobado ☐ Con observaciones | 2026-08-__ |

> *En el ciclo de vida formal, la necesidad se **aprueba y se firma** antes de
> analizar nada: es el primer hito del proyecto.*

---

> **Nota para la clase:** este documento es el insumo directo de la fase 2
> (`02_requerimientos.md`): cada petición (P) y criterio de éxito (CS) se
> traducirá allí a requerimientos funcionales y no funcionales numerados, y la
> tabla de trazabilidad permitirá recorrer el hilo completo — ningún
> requerimiento "aparece de la nada" y ninguna petición queda olvidada.
