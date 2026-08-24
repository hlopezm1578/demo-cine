# Guía 5 — La API JSON: cumpliendo el contrato

> **Qué construirás hoy:** las 7 operaciones del contrato `contrato_api.yaml` (fase 4): esquemas Pydantic, autenticación por dependencia y el router de la API.
> **Al terminar tendrás:** la API completa funcionando en `/docs`, probada operación por operación **contra el contrato** — API-first de verdad.
> **Necesitas:** la guía 4 terminada y verificada.

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Esquema (Pydantic)** | La forma que deben tener los datos que entran y salen; Pydantic valida solo (RN-02, por ejemplo) |
| **Dependencia (FastAPI)** | Una función que FastAPI resuelve ANTES de tu ruta e inyecta el resultado — inyección de dependencias de verdad |
| **Esquema de seguridad (OpenAPI)** | La declaración formal de CÓMO se autentica cada petición. Sin ella, `/docs` no muestra el botón **Authorize** — por más que el código lea tokens |
| **response_model** | El contrato de salida: lo que no está declarado, no se serializa (el hash de contraseñas jamás sale) |
| **Código de estado** | El idioma de resultados del HTTP: 201 creado, 401 sin sesión, 409 conflicto… (ver portada del contrato) |

---

## Paso 0 — Dependencia y configuración

```powershell
pip install email-validator
pip freeze > requirements.txt
```

🧠 **El desarrollador piensa:** *`email-validator` es lo que permite `EmailStr` en los esquemas: sin él, Pydantic no sabe validar formatos de email. Y agrego a `config.py` el nombre de la cookie de sesión: la API leerá el token del encabezado Bearer **o** de esa cookie (ADR-003) — así la misma dependencia servirá a la API y a la web de la guía 6.*

Agrega al final de **`app/config.py`**:

```python
# Nombre de la cookie de sesión (ADR-003)
NOMBRE_COOKIE = "sesion"
```

---

## Paso 1 — Los esquemas: la frontera de validación

🧠 **El desarrollador piensa:** *estos esquemas son la traducción EXACTA de los componentes del contrato OpenAPI (`components/schemas`). No estoy diseñando: estoy **cumpliendo**. Fíjate en `UsuarioOut`: no incluye `password_hash`. Eso no es un descuido — es la regla "lo que no se declara, no se serializa" trabajando a favor de la seguridad. Y `model_config = ConfigDict(from_attributes=True)` le permite a Pydantic construir el esquema directo desde el objeto ORM.*

Crea la carpeta `app/esquemas/` con estos archivos.

**`app/esquemas/usuario.py`**:

```python
"""Esquemas Pydantic de usuarios (contrato: schemas UsuarioCrear, SolicitudSesion, UsuarioOut)."""
from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UsuarioCrear(BaseModel):
    nombre: str = Field(min_length=2, max_length=80)
    email: EmailStr
    contrasena: str = Field(min_length=8, max_length=100)


class SolicitudSesion(BaseModel):
    email: EmailStr
    contrasena: str


class UsuarioOut(BaseModel):
    """Un usuario visible en respuestas. El hash jamás aparece: no está declarado."""
    model_config = ConfigDict(from_attributes=True)

    id: int
    nombre: str
    email: EmailStr
    rol: str  # 'socio' | 'admin' — enum del contrato
```

**`app/esquemas/pelicula.py`**:

```python
"""Esquemas Pydantic de películas (contrato: schemas PeliculaResumen, PeliculaDetalle)."""
from pydantic import BaseModel


class PeliculaResumen(BaseModel):
    id: int
    titulo: str
    anio: int
    promedio: float
    total_calificaciones: int
    imagen: str  # URL pública de la carátula o placeholder (RN-05)


class PeliculaDetalle(PeliculaResumen):
    sinopsis: str
    url_trailer: str
```

**`app/esquemas/calificacion.py`**:

```python
"""Esquemas Pydantic de calificaciones (contrato: schemas CalificacionEntrada/Respuesta)."""
from pydantic import BaseModel, Field


class CalificacionEntrada(BaseModel):
    pelicula_id: int
    estrellas: int = Field(ge=1, le=5)  # RN-02: entero 1–5; fuera de rango → 422


class CalificacionRespuesta(BaseModel):
    promedio: float
    total_calificaciones: int
    mi_calificacion: int
```

**`app/esquemas/__init__.py`**:

```python
from app.esquemas.calificacion import CalificacionEntrada, CalificacionRespuesta
from app.esquemas.pelicula import PeliculaDetalle, PeliculaResumen
from app.esquemas.usuario import SolicitudSesion, UsuarioCrear, UsuarioOut

__all__ = [
    "CalificacionEntrada",
    "CalificacionRespuesta",
    "PeliculaDetalle",
    "PeliculaResumen",
    "SolicitudSesion",
    "UsuarioCrear",
    "UsuarioOut",
]
```

## Paso 2 — `dependencias.py`: ¿quién llama? ¿puede?

🧠 **El desarrollador piensa:** *la pregunta "¿quién es el que llama?" se resuelve en un solo lugar (ADR-003). Y aquí hay una trampa que descubrí a los golpes: si leo el encabezado `Authorization` a mano con `request.headers.get(...)`, la autenticación **funciona**… pero `/docs` no muestra el botón **Authorize**. ¿Por qué? Porque ese botón aparece únicamente cuando el OpenAPI generado declara un **esquema de seguridad**, y FastAPI solo lo declara si usamos sus utilidades de seguridad. La solución hace doble trabajo: `HTTPBearer` extrae el encabezado por nosotros Y registra el esquema (el candadito aparece). Con `auto_error=False` devuelve `None` cuando el encabezado no viene, en vez de rechazar la petición — necesario porque nuestro usuario de la web llega por cookie. Sobre esa base, dos reglas más: "exigir sesión" y "exigir coordinadora". Y FastAPI cachea el resultado por petición: aunque tres dependencias la usen, la base se consulta una vez.*

Crea **`app/dependencias.py`**:

```python
"""Dependencias de FastAPI: identificar y autorizar al usuario (ADR-003)."""
from fastapi import Depends, HTTPException, Request
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from sqlalchemy.orm import Session

from app.config import NOMBRE_COOKIE
from app.database import obtener_sesion
from app.modelos import Usuario
from app.repositorios import RepositorioUsuarios
from app.servicios import autenticacion

# Declarar el esquema Bearer hace DOS trabajos: FastAPI extrae el encabezado
# Authorization por nosotros Y publica el esquema de seguridad en el OpenAPI
# generado — sin esto, /docs no muestra el botón Authorize.
# auto_error=False: sin encabezado devuelve None (nuestro usuario web llega
# por cookie), en vez de rechazar con un 403 automático.
seguridad_bearer = HTTPBearer(auto_error=False)


def obtener_usuario_actual(
    request: Request,
    credenciales: HTTPAuthorizationCredentials | None = Depends(seguridad_bearer),
    sesion: Session = Depends(obtener_sesion),
) -> Usuario | None:
    """Identifica al usuario a partir del token, o None si es anónimo.

    Acepta el token de dos formas (ADR-003):
      - cabecera Authorization: Bearer <token> (pruebas desde /docs)
      - cookie 'sesion' (navegador web — la setea el login de la guía 6)
    """
    token = credenciales.credentials if credenciales is not None else None
    if not token:
        token = request.cookies.get(NOMBRE_COOKIE)

    if not token:
        return None

    datos = autenticacion.leer_token(token)
    if datos is None:
        return None

    return RepositorioUsuarios(sesion).por_id(int(datos["sub"]))


def usuario_obligatorio(
    usuario: Usuario | None = Depends(obtener_usuario_actual),
) -> Usuario:
    """Para rutas que exigen sesión (RF-03)."""
    if usuario is None:
        raise HTTPException(status_code=401, detail="Debes iniciar sesión.")
    return usuario


def requiere_admin(
    usuario: Usuario = Depends(usuario_obligatorio),
) -> Usuario:
    """Para rutas solo de la coordinadora (RN-04)."""
    if usuario.rol != "admin":
        raise HTTPException(status_code=403, detail="Esta acción es solo para administradores.")
    return usuario
```

## Paso 3 — `rutas/api.py`: el contrato, operación por operación

🧠 **El desarrollador piensa:** *abro `contrato_api.yaml` al lado y lo traduzco línea a línea. Cada código de respuesta del contrato tiene su `HTTPException` correspondiente: 409 para el email duplicado, 401 para credenciales malas, 404 para película inexistente. Las rutas no piensan: llaman servicios y traducen resultados. Y notarás que las mismas funciones de la guía 4 (`a_resumen`, `calificar`) alimentan todo — eso es ADR-001 pagándose solo.*

Crea la carpeta `app/rutas/` y el archivo **`app/rutas/api.py`**:

```python
"""API JSON, documentada automáticamente en /docs (RF-10, contrato_api.yaml).

Punto pedagógico clave (ADR-001): estas rutas llaman a los MISMOS servicios
que usarán las páginas HTML de la guía 6. Dos caras, una sola lógica.
"""
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app.database import obtener_sesion
from app.dependencias import usuario_obligatorio
from app.esquemas import (
    CalificacionEntrada,
    CalificacionRespuesta,
    PeliculaDetalle,
    PeliculaResumen,
    SolicitudSesion,
    UsuarioCrear,
    UsuarioOut,
)
from app.modelos import Usuario
from app.repositorios import RepositorioPeliculas
from app.servicios import autenticacion
from app.servicios import calificaciones as servicio_calificaciones
from app.servicios import peliculas as servicio_peliculas

router = APIRouter(prefix="/api", tags=["API JSON"])


@router.get("/peliculas", response_model=list[PeliculaResumen])
def listar_peliculas(sesion: Session = Depends(obtener_sesion)):
    repo = RepositorioPeliculas(sesion)
    promedios = repo.promedios()  # una sola consulta para todo el catálogo
    return [
        servicio_peliculas.a_resumen(p, *promedios.get(p.id, (0.0, 0)))
        for p in repo.listar()
    ]


@router.get("/peliculas/{pelicula_id}", response_model=PeliculaDetalle)
def ver_pelicula(pelicula_id: int, sesion: Session = Depends(obtener_sesion)):
    pelicula = RepositorioPeliculas(sesion).por_id(pelicula_id)
    if pelicula is None:
        raise HTTPException(status_code=404, detail="Película no encontrada")
    promedio, total = RepositorioPeliculas(sesion).promedio(pelicula_id)
    return servicio_peliculas.a_detalle(pelicula, promedio, total)


@router.post("/usuarios", response_model=UsuarioOut, status_code=201)
def crear_usuario(datos: UsuarioCrear, sesion: Session = Depends(obtener_sesion)):
    try:
        return autenticacion.registrar(sesion, datos.nombre, datos.email, datos.contrasena)
    except ValueError as error:
        raise HTTPException(status_code=409, detail=str(error))


@router.post("/sesion")
def iniciar_sesion(datos: SolicitudSesion, sesion: Session = Depends(obtener_sesion)):
    """Devuelve el token JWT para usar en las demás rutas (cabecera Bearer)."""
    try:
        token = autenticacion.entrar(sesion, datos.email, datos.contrasena)
    except ValueError as error:
        raise HTTPException(status_code=401, detail=str(error))
    return {"token": token}


@router.get("/yo", response_model=UsuarioOut)
def quien_soy(usuario: Usuario = Depends(usuario_obligatorio)):
    """Prueba rápida del token (botón Authorize en /docs)."""
    return usuario


@router.post("/calificaciones", response_model=CalificacionRespuesta)
def calificar(
    datos: CalificacionEntrada,
    usuario: Usuario = Depends(usuario_obligatorio),
    sesion: Session = Depends(obtener_sesion),
):
    fila = servicio_calificaciones.calificar(
        sesion, usuario, datos.pelicula_id, datos.estrellas
    )
    promedio, total = RepositorioPeliculas(sesion).promedio(datos.pelicula_id)
    return CalificacionRespuesta(
        promedio=round(promedio, 1),
        total_calificaciones=total,
        mi_calificacion=fila.estrellas,
    )
```

**`app/rutas/__init__.py`**:

```python
from app.rutas import api

__all__ = ["api"]
```

## Paso 4 — Conectar la API a la aplicación

Edita **`app/main.py`** hasta que quede así:

```python
"""Punto de composición: arma la aplicación y conecta las piezas (ADR-001)."""
from fastapi import FastAPI

from app.database import crear_tablas
from app.rutas import api

app = FastAPI(
    title="Cartelera",
    description=(
        "Catálogo de películas del CineClub Barrio con calificación por "
        "estrellas. La misma lógica alimenta la web y esta API."
    ),
    version="1.0.0",
)

crear_tablas()

app.include_router(api.router)  # la API JSON (/docs)


@app.get("/salud")
def salud() -> dict:
    """Chequeo mínimo para saber que el servicio está vivo."""
    return {"estado": "ok"}
```

---

## ✅ Verificación de la guía 5 (el corazón de API-first)

**Prepara una película de prueba** — crea **`crear_pelicula_demo.py`** en la raíz (desechable):

```python
from app.database import SesionLocal, crear_tablas
from app.modelos import Pelicula
from app.repositorios import RepositorioPeliculas

crear_tablas()
with SesionLocal() as sesion:
    RepositorioPeliculas(sesion).crear(
        Pelicula(titulo="Coco", anio=2017,
                 sinopsis="Miguel viaja a la Tierra de los Muertos.",
                 url_trailer="https://www.youtube.com/embed/Ga6RYejo6Hk")
    )
    print("Película demo creada (id=1)")
```

```powershell
python crear_pelicula_demo.py
python run.py
```

Ahora abre **http://127.0.0.1:8000/docs** y recorre el contrato, en orden:

1. **`POST /api/usuarios`** con `{"nombre":"Fernanda","email":"fer@test.cl","contrasena":"clave-segura-123"}` → **201**. Prueba de nuevo con el mismo email → **409** (duplicado, como el contrato).
2. **`POST /api/sesion`** con las credenciales → **200** y un `token` en la respuesta. Cópialo.
3. Botón **Authorize** (candado, arriba a la derecha): pega el token. Ahora estás autenticada/o.
4. **`GET /api/yo`** → tu usuario con `"rol": "socio"`.
5. **`POST /api/calificaciones`** con `{"pelicula_id": 1, "estrellas": 5}` → promedio 5.0, total 1. Repite con `"estrellas": 2` → promedio 2.0 y **total sigue en 1** (RN-01, ahora visible por la API).
6. Prueba `"estrellas": 7` → **422** (RN-02). Cierra Authorize y prueba calificar → **401** (RF-03).
7. **`GET /api/peliculas`** y **`GET /api/peliculas/{pelicula_id}`** → el catálogo con promedios e `imagen` placeholder.

**Y la verificación que cierra la fase (ADR-008):** con el contrato abierto en Swagger Editor (o impreso), compara **operación por operación** contra el `/docs` generado: rutas, parámetros, códigos de respuesta, esquemas. Anota cualquier diferencia — aquí no debería haber ninguna, pero este ritual es el que detecta desvíos cuando el proyecto crece.

Detén el servidor (`Ctrl+C`) y limpia:

```powershell
del crear_pelicula_demo.py
del cartelera.db
```

---

## 📝 Punto de control

1. ¿Por qué el registro duplicado responde **409** y no 400? ¿Y por qué "estrellas: 7" es 422 y no 400?
2. `UsuarioOut` no declara `password_hash`. ¿Qué pasaría si lo declarara por accidente?
3. La dependencia `obtener_usuario_actual` se usa en varias rutas por petición. ¿Cuántas consultas a la base hace FastAPI por petición gracias al caché de dependencias?

## Lo que acabas de aprender

- Esquemas Pydantic como frontera de validación (RN-02 en una línea)
- Dependencias de FastAPI: inyección real para "¿quién?" y "¿puede?"
- Traducir un contrato OpenAPI a código, y verificar el cumplimiento
- El flujo completo de autenticación por Bearer en `/docs`

**Siguiente:** `guia-06-web.md` — lo que ve la gente: las 4 pantallas con plantillas Jinja2 y el widget de estrellas.
