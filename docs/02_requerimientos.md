# Fase 2 — Documento de Requerimientos: Cartelera

> **Módulos:** ISI601 Taller de Diseño de Sistemas · ISI602 Arquitectura de Software
> **Fase del ciclo de vida:** 2. Análisis de requerimientos
> **Insumo obligatorio:** `01_necesidad_del_cliente.md` (cliente: CineClub Barrio) — cada requerimiento de este documento **nace de una petición (P), condición (C) o criterio de éxito (CS)** de ese documento. La tabla de trazabilidad (§13) demuestra el hilo.
> **Producto:** Cartelera — catálogo web de películas con carátula, tráiler y calificación por estrellas
> **Fecha:** 2026-08-23

---

## 1. Contexto y objetivo

**Cartelera** es el sitio web del CineClub Barrio: un catálogo de películas donde la coordinadora publica cada título con su carátula y tráiler, los vecinos navegan libremente, y los socios registrados califican con estrellas (1 a 5) las películas que vieron. El promedio de estrellas orienta la elección de la próxima temporada.

**Objetivo del sistema:** permitir la **publicación, exploración y calificación de películas** para que el cineclub decida con datos y comunique con orden.

---

## 2. Alcance

**Dentro del alcance** (lo que pide el cliente, §4 doc 01):
- Registro e inicio de sesión de socios (C3).
- Gestión del catálogo por la coordinadora: crear y eliminar películas con carátula y tráiler (P1, P4).
- Catálogo público navegable sin cuenta (P2, C5).
- Ficha de película con tráiler embebido y promedio de estrellas (P1, P3).
- Calificación con estrellas 1–5, una por socio por película (P3).
- API JSON documentada para integraciones futuras (soporte técnico de P2/P3).

**Fuera del alcance** (lo que el cliente NO pide, §6 doc 01):
- Pagos, venta de entradas, membrecías.
- Reseñas escritas o comentarios.
- Recomendaciones automáticas.
- Red social: perfiles públicos, avatares, mensajería.

---

## 3. Actores y roles

| Rol | Descripción | Origen | Permisos clave |
|---|---|---|---|
| **Visitante (anónimo)** | Cualquier vecino que llega al sitio | P2, C5 | Navegar catálogo y fichas con tráiler y promedio |
| **Socio (usuario registrado)** | Miembro del cineclub con cuenta | P3, C3 | Todo lo del visitante + calificar y recalificar películas |
| **Coordinadora (administradora)** | Macarena, única administradora | P4, C2 | Todo lo anterior + crear y eliminar películas |

---

## 4. Requerimientos funcionales (RF)

### Gestión de cuentas (soportan P3, C3)
- **RF-01:** El sistema debe permitir registrar un socio con nombre, email y contraseña (mínimo 8 caracteres, email único).
- **RF-02:** El sistema debe autenticar socios por email y contraseña, y permitir cerrar sesión.
- **RF-03:** El sistema debe exigir sesión iniciada para calificar; los visitantes solo navegan.

### Catálogo de películas (soportan P1, P2, P4, C2)
- **RF-04:** La administradora debe poder crear una película con título, año, sinopsis, carátula (imagen) y link del tráiler de YouTube.
- **RF-05:** El sistema debe mostrar el catálogo público con carátula, título, año y promedio de estrellas.
- **RF-06:** El sistema debe mostrar una ficha por película con sinopsis, tráiler reproducible y promedio de calificaciones.
- **RF-07:** La administradora debe poder eliminar una película del catálogo (junto con sus calificaciones).

### Calificaciones (soportan P3, CS2, CS3)
- **RF-08:** El socio debe poder calificar una película con un entero de 1 a 5 estrellas.
- **RF-09:** El sistema debe permitir recalificar: la nueva calificación **reemplaza** la anterior (el promedio considera un voto por socio).
- **RF-10:** El sistema debe exponer una API JSON documentada automáticamente (pantalla interactiva en `/docs`) con las operaciones del sistema.

---

## 5. Requerimientos no funcionales (RNF)

| Código | Categoría | Descripción | Origen |
|---|---|---|---|
| RNF-01 | Seguridad | Las contraseñas se almacenan con hash (bcrypt), nunca en texto plano | Práctica profesional |
| RNF-02 | Seguridad | Autorización por rol: solo la administradora gestiona el catálogo | P4 |
| RNF-03 | Seguridad | Los secretos del sistema (claves, direcciones de base de datos) viven en variables de entorno, no en el código | Práctica profesional |
| RNF-04 | Mantenibilidad | Arquitectura en capas con decisiones documentadas (ADRs), respetando SOLID en su versión pragmática | Objetivo pedagógico |
| RNF-05 | Portabilidad | La misma base de código corre en desarrollo (SQLite) y producción (Postgres) cambiando una variable de entorno | C1, CS4 |
| RNF-06 | Usabilidad | Interfaz responsiva, usable desde el celular, navegable sin JavaScript salvo el widget de estrellas | C4 |
| RNF-07 | Operación | Publicable en plataformas gratuitas, sin servidor propio ni tarjeta de crédito; funcionamiento desatendido | P5, C1, CS4 |

---

## 6. Reglas de negocio (RN)

- **RN-01:** Un socio tiene **a lo más una calificación por película**; recalificar reemplaza su voto anterior. *(P3, CS3: el promedio refleja una opinión por socio.)*
- **RN-02:** Las estrellas son un **entero entre 1 y 5**; cualquier otro valor se rechaza. *(P3)*
- **RN-03:** El tráiler debe ser un **link de YouTube**; el sistema lo convierte al formato reproducible embebido. *(P1)*
- **RN-04:** Solo la **administradora** puede crear y eliminar películas. *(P4)*
- **RN-05:** La carátula es opcional (se muestra una imagen genérica si falta); si se sube, debe ser JPG, PNG o WebP de máximo 5 MB. *(P1, CS1: subir una película debe ser simple y a prueba de errores.)*

---

## 7. Historias de usuario (con criterios de aceptación Gherkin)

### HU-01 — Explorar el catálogo
*Como* visitante, *quiero* ver el catálogo de películas, *para* decidir qué ver sin necesidad de una cuenta.
- **Dado** que estoy en la página principal,
  **Cuando** cargo el sitio desde mi celular,
  **Entonces** veo todas las películas con su carátula, año y promedio de estrellas.

### HU-02 — Ver la ficha de una película
*Como* visitante, *quiero* ver la ficha de una película, *para* conocer su sinopsis y ver su tráiler antes de la función.
- **Dado** que seleccioné una película del catálogo,
  **Cuando** se abre su ficha,
  **Entonces** veo carátula, sinopsis, tráiler reproducible y el promedio de calificaciones.

### HU-03 — Registrarme como socio
*Como* visitante, *quiero* crear mi cuenta con nombre, email y contraseña, *para* poder calificar.
- **Dado** que completo el formulario con datos válidos,
  **Cuando** lo envío,
  **Entonces** mi cuenta queda creada y puedo iniciar sesión.
- **Dado** que uso un email ya registrado,
  **Cuando** envío el formulario,
  **Entonces** el sistema me avisa sin crear una cuenta duplicada.

### HU-04 — Calificar una película
*Como* socio, *quiero* calificar con estrellas la película que vi, *para* que mi opinión cuente.
- **Dado** que inicié sesión y estoy en la ficha,
  **Cuando** hago clic en una estrella (1–5),
  **Entonces** mi calificación queda guardada y el promedio se actualiza.
- **Dado** que ya califiqué esa película,
  **Cuando** elijo otra cantidad de estrellas,
  **Entonces** mi voto anterior se reemplaza (RN-01) y el promedio refleja un voto mío, no dos.

### HU-05 — Saber que necesito cuenta para calificar
*Como* visitante, *quiero* que me inviten a iniciar sesión, *para* no perder tiempo intentando calificar.
- **Dado** que no he iniciado sesión,
  **Cuando** abro una ficha,
  **Entonces** veo una invitación a iniciar sesión en lugar del widget de estrellas.

### HU-06 — Publicar una película (administradora)
*Como* coordinadora, *quiero* publicar una película con carátula y tráiler en menos de 5 minutos, *para* mantener el catálogo al día sin ayuda técnica.
- **Dado** que inicié sesión como administradora y estoy en el panel,
  **Cuando** completo el formulario y subo la carátula,
  **Entonces** la película aparece de inmediato en el catálogo. *(CS1)*

### HU-07 — Eliminar una película (administradora)
*Como* coordinadora, *quiero* retirar una película que ya se exhibió, *para* mantener el catálogo al día.
- **Dado** una película existente,
  **Cuando** la elimino desde el panel confirmando la acción,
  **Entonces** desaparece del catálogo junto con sus calificaciones.

---

## 8. Modelo de datos preliminar (insumo para la fase de diseño)

| Entidad | Atributos clave | Relaciones |
|---|---|---|
| **Usuario** | id, nombre, email (único), password_hash, rol ('socio' / 'admin'), creado_en | 1:N con Calificación |
| **Pelicula** | id, titulo, anio, sinopsis, url_trailer (embebida), ruta_caratula, creado_en | 1:N con Calificación |
| **Calificacion** | id, usuario_id, pelicula_id, estrellas (1–5), actualizado_en | N:1 con Usuario y Película. **UNIQUE(usuario_id, pelicula_id)** = RN-01 |

> *La restricción UNIQUE es la decisión de datos más importante: la base de
> datos misma impide RN-01 aunque el código falle. Se desarrolla en la fase de
> diseño (03).*

---

## 9. Procesos principales (insumo para DFD en diseño)

1. **Calificar una película:** socio → inicia sesión → abre ficha → elige estrellas → el sistema guarda/reemplaza su voto → recalcula el promedio.
2. **Publicar una película:** coordinadora → inicia sesión → panel → completa formulario + carátula → el sistema valida (RN-03, RN-05) → guarda → película visible.
3. **Explorar el catálogo:** visitante → catálogo → ficha → tráiler / promedio.

---

## 10. Entradas del sistema (formularios y validaciones)

| Entrada | Actor | Validaciones clave |
|---|---|---|
| Registro de socio | Visitante | Nombre ≥ 2, email con formato válido y único, contraseña ≥ 8 |
| Inicio de sesión | Todos | Credenciales correctas (mensaje genérico si falla) |
| Nueva película | Administradora | Título obligatorio, año numérico, tráiler solo YouTube (RN-03), carátula JPG/PNG/WebP ≤ 5 MB (RN-05) |
| Calificación | Socio | Estrellas entero 1–5 (RN-02), película existente, un voto por socio (RN-01) |

## 11. Salidas del sistema

| Salida | Audiencia | Medio |
|---|---|---|
| Catálogo con promedios | Visitante / socio | Pantalla web (responsiva, C4) |
| Ficha con tráiler y promedio | Visitante / socio | Pantalla web |
| Ranking de promedios de la temporada | Coordinadora | Pantalla web (insumo de su decisión, CS3) |
| API JSON documentada | Integraciones futuras | `/docs` |

## 12. Pantallas principales (insumo para prototipado)

1. **Catálogo (home):** grilla de tarjetas con carátula, título, año, estrellas y promedio.
2. **Ficha de película:** carátula grande, sinopsis, tráiler embebido, widget de estrellas (o invitación a iniciar sesión).
3. **Registro e inicio de sesión:** formularios con mensajes de error.
4. **Panel de administración:** tabla de películas + formulario de carga + eliminar.

---

## 13. Trazabilidad: necesidad → requerimiento

El hilo completo del ciclo de vida. **Ninguna petición queda sin requerimiento y ningún requerimiento nace de la nada.**

| Necesidad (doc 01) | Se convierte en | Se verifica con |
|---|---|---|
| P1 Publicar con carátula y tráiler | RF-04, RF-06, RN-03, RN-05 | HU-02, HU-06 |
| P2 Catálogo público | RF-05, RF-06, C5 | HU-01, HU-02 |
| P3 Socios califican con estrellas | RF-01 a RF-03, RF-08, RF-09, RN-01, RN-02 | HU-03, HU-04, HU-05 |
| P4 Solo la coordinadora administra | RF-07, RN-04, RNF-02 | HU-06, HU-07 |
| P5 Todo gratis, sin servidor propio | RNF-05, RNF-07 | Fase de despliegue (07) |
| C1 Presupuesto $0 | RNF-07 | Fase de despliegue |
| C3 Registro simple | RF-01, RF-02 | HU-03 |
| C4 Funciona en el celular | RNF-06 | HU-01 |
| CS1 Publicar en < 5 min, sin ayuda | RF-04, RN-05, §12 pantalla 4 | HU-06 |
| CS2 Mitad de socios califica | RF-03, RF-08, RF-09, HU-05 (fricción mínima) | Uso real |
| CS3 Promedios para decidir | RF-09, RN-01, §11 ranking | Uso real |
| CS4 Desatendido | RNF-05, RNF-07 | Fase de despliegue |

---

## 14. Aprobación de la fase

| Rol | Nombre | Decisión | Fecha |
|---|---|---|---|
| Analista | ______________ | ☐ Aprobado ☐ Con observaciones | 2026-08-__ |
| Cliente (validación de requerimientos) | Macarena — CineClub Barrio | ☐ Aprobado ☐ Con observaciones | 2026-08-__ |

> **Nota para la clase:** este documento es el contrato de QUÉ hace el sistema,
> jamás de CÓMO se construye. El CÓMO arranca en la fase 3 (`03_diseno.md`):
> modelo de datos detallado, diagramas de procesos (DFD) y prototipo de
> pantallas; sigue en la fase 4 con las decisiones de arquitectura (ADRs) y el
> cumplimiento de SOLID (RNF-04). Misma estructura que el proyecto guía
> PixelStore: los alumnos pueden contrastar ambos documentos.
