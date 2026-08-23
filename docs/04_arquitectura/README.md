# Fase 4 — Documento de Arquitectura: Cartelera

> **Módulo:** ISI602 Arquitectura de Software (con soporte de ISI601)
> **Fase del ciclo de vida:** 4. Arquitectura
> **Insumo obligatorio:** `03_diseno.md` — la arquitectura **no cambia el diseño**: elige tecnología y define cómo se organiza el código para construir exactamente lo diseñado.
> **Decisiones detalladas:** cada elección de esta fase tiene su ADR (Architecture Decision Record) en `adr/`, formato: contexto → opciones → decisión → consecuencias → preguntas para la clase.
> **Fecha:** 2026-08-23

---

## 1. La arquitectura en una página

**Nombre:** arquitectura **cliente-servidor en tres capas**.

```
┌──────────────────┐         ┌───────────────────────────────┐         ┌─────────────────┐
│   CAPA 1         │  HTTP   │        CAPA 2                 │   SQL   │    CAPA 3       │
│   PRESENTACIÓN   │◄───────►│   LÓGICA DE NEGOCIO           │◄───────►│   DATOS         │
│                  │         │                               │         │                 │
│  Navegador       │         │  Servidor FastAPI             │         │  Base de datos  │
│  HTML + CSS + JS │         │  · Rutas (reciben peticiones) │         │  SQLite /       │
│  (lo que ve      │         │  · Servicios (las reglas)     │         │  Postgres       │
│   el usuario)    │         │  (lo que decide)              │         │  (lo que queda  │
└──────────────────┘         └───────────────────────────────┘         └─────────────────┘
```

**Las tres preguntas** (memorizables):

| Capa | Pregunta que responde | En Cartelera |
|---|---|---|
| **1. Presentación** | ¿Qué ve y toca el usuario? | Catálogo, ficha, login, panel |
| **2. Lógica de negocio** | ¿Qué decide el sistema? | "Un voto por socio", "solo la coordinadora publica" |
| **3. Datos** | ¿Qué se recuerda cuando nadie está mirando? | Usuarios, películas, calificaciones |

**Regla de oro:** *cada capa solo conversa con su capa vecina inmediata. La presentación nunca pregunta directo a la base de datos, y la base de datos nunca decide una regla de negocio.*

**Frase para el informe:**

> *"El sistema se construye con una arquitectura cliente-servidor en tres capas: presentación (interfaz de usuario), lógica de negocio (reglas del sistema) y datos (persistencia), donde cada capa se comunica únicamente con la capa inferior."*

> **Pregunta inevitable del jurado: "¿no es eso MVC?"** Casi. MVC es un patrón para organizar la interacción con el usuario (Modelo–Vista–Controlador); las capas son sobre la **dirección de las dependencias**. Mapeo: Vista = plantillas, Controlador = rutas, Modelo = servicios + repositorios + modelos. Las capas agregan la regla estricta que hace que la web y la API compartan lógica.

---

## 2. Del diseño a la arquitectura (traducción elemento a elemento)

| Elemento de diseño (fase 3) | Dónde vive en la arquitectura |
|---|---|
| Entidades USUARIO, PELICULA, CALIFICACION (§2) | **Modelos ORM** (capa de datos) |
| Almacenes D1, D2, D3 (§3.2) | **Repositorios** (acceso a datos, un almacén = un repositorio) |
| Procesos 1.0–6.0 (§3.3–3.5) | **Servicios** (la lógica de cada proceso) |
| Pantallas 1–4 (§4) | **Rutas + plantillas** (capa de presentación) |
| Almacén A1 de carátulas (§2.3.3) | **Servicio de almacenamiento** (ADR-004) |
| Validaciones RN-01…RN-05 | Esquemas Pydantic en la frontera + servicios + restricciones de la base |

Detalle clave para la clase: **el DFD del diseño se convierte literalmente en la estructura del código.** Proceso 2.0 "Registrar calificación" del DFD = un servicio `calificaciones` con su regla; el almacén D3 = un repositorio que solo habla con la tabla de votos.

---

## 3. Stack tecnológico (y por qué)

| Pieza | Elección | Por qué (decisión completa) |
|---|---|---|
| Lenguaje | **Python 3.11+** | Requerimiento del ramo y del mercado laboral del perfil |
| Framework web | **FastAPI** | API JSON documentada automáticamente en `/docs` (RF-10), validación declarativa con Pydantic y **inyección de dependencias nativa** — puente directo entre la teoría DIP y un framework real |
| Plantillas | **Jinja2** | SSR servido por el mismo FastAPI: un solo despliegue, sin CORS (ADR-006) |
| ORM | **SQLAlchemy 2** | Mismo código para SQLite y Postgres, restricciones declaradas una vez (ADR-002) |
| Base de datos | **SQLite** (desarrollo) / **Postgres en Neon** (producción) | Cero instalación local; gratis y persistente en la nube (ADR-002, ADR-007) |
| Autenticación | **PyJWT + bcrypt** | Tokens sin estado para un servicio que se duerme; contraseñas con hash (ADR-003) |
| Pruebas | **pytest** | Estándar de facto; el flujo completo se prueba sin navegador (fase 6) |
| Publicación | **Render + Neon** | Gratis, sin tarjeta, deploy automático desde GitHub (ADR-007) |

---

## 4. Organización del código (la regla de oro hecha carpetas)

```
app/
├── main.py            # composición: arma la aplicación (no tiene lógica)
├── config.py          # variables de entorno: única fuente de configuración
├── dependencias.py    # ¿quién llama? ¿puede? (sesión y roles)
├── modelos/           # CAPA 3 — tablas: usuario, pelicula, calificacion
├── repositorios/      # CAPA 3 — solo acceso a datos, sin reglas
├── servicios/         # CAPA 2 — las reglas del negocio (los procesos del DFD)
├── esquemas/          # frontera: validación de entrada y forma de salida
├── rutas/             # CAPA 1 — web (HTML), api (JSON), admin
├── plantillas/        # CAPA 1 — Jinja2 (las 4 pantallas del diseño)
└── static/            # CAPA 1 — CSS, widget de estrellas, imagen genérica
```

**Reglas de dependencia entre carpetas** (verificables en revisión de código):

1. `rutas/` importa de `servicios/` y `esquemas/`; **jamás** de `repositorios/` ni escribe SQL.
2. `servicios/` importa de `repositorios/` y `modelos/`; **jamás** decide cómo se responde al usuario.
3. `repositorios/` importa de `modelos/`; **jamás** contiene una regla de negocio.
4. Todos pueden leer `config.py`. Nadie más que `main.py` arma la aplicación.

---

## 5. SOLID — versión pragmática (decisión registrada)

Cumplimiento de RNF-04. Se abstrae **lo que es razonable esperar que cambie**; no se abstrae todo por ritual.

| Principio | Cómo se cumple |
|---|---|
| **S** — Responsabilidad única | La estructura de carpetas ES el principio: cada clase tiene un motivo para cambiar. Cambia una regla → `servicios/`; cambia una consulta → `repositorios/`; cambia una respuesta → `rutas/`. |
| **O** — Abierto/cerrado | Caso emblemático: agregar Cloudinary/S3 mañana = una clase nueva de almacenamiento, cero cambios en el resto (ADR-004). |
| **L** — Sustitución de Liskov | El contrato del almacenamiento se cumple en cualquier implementación: guardar siempre devuelve una ruta utilizable; eliminar nunca explota si el archivo no existe. En proyectos chicos, L vive en el contrato de la interfaz. |
| **I** — Segregación de interfaces | La interfaz de almacenamiento tiene solo los 3 métodos que el resto usa; las dependencias de autenticación son granulares: una para "usuario actual", otra para "exigir socio", otra para "exigir coordinadora". |
| **D** — Inversión de dependencias | Los servicios dependen de la **interfaz** de almacenamiento, no de la implementación; la elección concreta vive en una línea. Bonus: el mecanismo de dependencias de FastAPI es inyección de dependencias real — el puente con el ejemplo `08_dip_inyeccion.py` del ramo. |

**La decisión pragmática, explícita:** el almacenamiento SÍ se abstrae (cambia entre desarrollo y producción, ADR-004); los repositorios NO (su implementación no va a cambiar en la vida de este sistema; una interfaz por repo duplicaría archivos sin beneficio).

> **Pregunta para la clase:** ¿por qué abstraemos el almacenamiento sí y los repositorios no? ¿Qué tendría que cambiar en el sistema para que convenga abstraer también los repositorios?

---

## 6. Índice de ADRs

| ADR | Decisión | Resuelto por |
|---|---|---|
| [001](adr/001-arquitectura-en-capas.md) | Arquitectura en capas (rutas → servicios → repositorios → modelos) | Cómo organizar el código |
| [002](adr/002-persistencia-con-orm.md) | SQLAlchemy ORM: SQLite en dev, Postgres en prod | Cómo persistir (D1–D3) |
| [003](adr/003-autenticacion-jwt-y-roles.md) | JWT en cookie HttpOnly + rol en la tabla | Cómo identificar socio/coordinadora |
| [004](adr/004-almacenamiento-de-caratulas.md) | Interfaz de almacenamiento + implementación local (DIP) | Cómo guardar A1 |
| [005](adr/005-modelo-de-calificaciones.md) | UNIQUE + upsert + promedio en lectura | Cómo garantizar RN-01 |
| [006](adr/006-frontend-ssr-jinja2-vs-spa.md) | SSR con Jinja2 desde el propio servidor | Cómo servir las 4 pantallas |
| [007](adr/007-despliegue-render-y-neon.md) | Render (web) + Neon (Postgres), gratis | Cómo publicar (P5, C1, CS4) |
| [008](adr/008-api-first.md) | API-first: contrato OpenAPI antes del código | Cómo se define la interfaz (RF-10) |

---

## 7. API-first: el contrato antes del código

La interfaz de la API JSON (RF-10) se diseña **antes** de implementarse. El contrato es [`contrato_api.yaml`](contrato_api.yaml) (OpenAPI 3.0) y es la **fuente de la verdad de la interfaz**: rutas, parámetros, códigos de respuesta y esquemas. La fase 5 debe implementarlo sin desviarse, y la comparación del `/docs` generado contra el contrato es una actividad de clase (ADR-008).

Flujo de la decisión:

```
contrato_api.yaml (se aprueba)  →  el código lo implementa  →  /docs generado ≈ contrato (verificación de desvío)
```

---

## 8. Aprobación de la fase

| Rol | Nombre | Decisión | Fecha |
|---|---|---|---|
| Arquitecto | ______________ | ☐ Aprobado ☐ Con observaciones | 2026-08-__ |
| Revisión (pares) | ______________ | ☐ Aprobado ☐ Con observaciones | 2026-08-__ |

> **Nota para la clase:** con esta fase el CÓMO queda cerrado y **versionado en ADRs**: si dentro de un año alguien pregunta "¿por qué JWT y no sesiones?", la respuesta no está en la memoria de quien estuvo ese día — está en el ADR-003. La fase 5 (`05_desarrollo/`) ya puede escribir código sabiendo exactamente qué debe respetar.
