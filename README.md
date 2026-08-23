# 🎬 Cartelera — un proyecto para enseñar desarrollo de software

**Este repositorio es un proyecto educativo.** No es un producto comercial: es el
recorrido **completo y documentado** del ciclo de vida del software, construido
sobre una aplicación pequeña pero real, pensado para que estudiantes de diseño
de sistemas y arquitectura de software vean el proceso entero — desde la
necesidad de un cliente hasta la publicación en la nube — con todas las
decisiones a la vista.

## La aplicación

**Cartelera** es el sitio web de un cliente ficticio, el *CineClub Barrio*: un
catálogo de películas donde la coordinadora publica cada título con su carátula
y tráiler de YouTube, los vecinos navegan libremente, y los socios registrados
califican con estrellas (1 a 5, un voto por persona y película).

Es pequeña a propósito: se lee completa en una sentada. Pero toca todo lo que
importa: roles y permisos, autenticación, subida de archivos, reglas de datos,
base de datos relacional, API con contrato OpenAPI y despliegue gratuito.

## El enfoque: una fase a la vez, nada aparece de la nada

Cada fase produce documentos que la siguiente usa como insumo, con
**trazabilidad completa** (cada requerimiento nace de una petición del cliente;
cada decisión de código nace de un documento). Ninguna fase se escribe sin
aprobar la anterior.

| # | Fase | Documento | Estado |
|---|---|---|---|
| 1 | Necesidad del cliente | [`docs/01_necesidad_del_cliente.md`](docs/01_necesidad_del_cliente.md) | ✅ |
| 2 | Requerimientos | [`docs/02_requerimientos.md`](docs/02_requerimientos.md) | ✅ |
| 3 | Diseño (datos, DFD, pantallas) | [`docs/03_diseno.md`](docs/03_diseno.md) | ✅ |
| 4 | Arquitectura + 8 ADRs + contrato OpenAPI | [`docs/04_arquitectura/`](docs/04_arquitectura) | ✅ |
| 5 | Desarrollo (8 guías paso a paso) | [`docs/05_desarrollo/`](docs/05_desarrollo) | ✅ |
| 6 | Pruebas | `docs/06_pruebas.md` | Pendiente |
| 7 | Despliegue | `docs/07_despliegue.md` | Pendiente |
| 8 | Mantenimiento | `docs/08_mantenimiento.md` | Pendiente |

El índice detallado del ciclo vive en [`docs/README.md`](docs/README.md).

## Qué hace especial a este material

- **ADRs con consecuencias honestas**: las 8 decisiones de arquitectura
  (`docs/04_arquitectura/adr/`) registran contexto, opciones descartadas,
  ventajas **y desventajas**, más preguntas para discutir en clase.
- **API-first de verdad**: el contrato OpenAPI
  ([`contrato_api.yaml`](docs/04_arquitectura/contrato_api.yaml)) se diseñó y
  aprobó **antes** del código; el desarrollo debe cumplirlo y la divergencia se
  detecta comparándolo con la documentación generada.
- **Guías de desarrollo "senior → junior"** (`docs/05_desarrollo/`): 8 guías
  que construyen la aplicación completa **de adentro hacia afuera** (datos →
  almacenes → reglas → API → web → panel), con el razonamiento de un
  desarrollador experimentado narrado paso a paso, bloques de código para
  copiar y verificaciones al final de cada paso.
- **SOLID pragmático**: los principios se aplican donde pagan (la abstracción
  de almacenamiento es el ejemplo estrella), y esa decisión —no el ritual— es
  material de discusión.

## Stack y cómo construir la aplicación

Python 3.11+ · FastAPI · SQLAlchemy 2 · Jinja2 · SQLite en desarrollo / Postgres
(Neon) en producción · JWT + bcrypt · Render como plataforma gratuita.

El código completo del proyecto vive **narrado en las 8 guías** de la fase 5:
siguiéndolas en orden —copiando los bloques y pasando cada ✅ verificación— la
aplicación queda construida y probada de punta a punta. Ese recorrido guiado
es la experiencia diseñada para el aula.

## Uso en clases

Pensado para módulos de taller de diseño de sistemas y arquitectura de
software: los documentos de las fases 1–3 sirven como modelos de entregables,
los ADRs como base de discusión, las guías como laboratorio guiado, y el
despliegue gratuito permite que cada estudiante termine con su propia
aplicación publicada con URL pública, sin costo.

> Repo mantenido con fines exclusivamente educativos. El cliente (CineClub
> Barrio, su coordinadora y sus socios) es ficticio; cualquier parecido con un
> cineclub real es pura nostalgia de barrio.
