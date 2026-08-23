# Guía 4 — Servicios: las reglas del negocio

> **Qué construirás hoy:** la capa donde viven las decisiones: hash de contraseñas, tokens de sesión, validación de tráilers, la regla de recalificar, y la abstracción de almacenamiento (el DIP del ramo, en un problema real).
> **Al terminar tendrás:** toda la lógica del sistema, funcionando y verificada por script, sin que exista todavía ni una ruta web.
> **Necesitas:** la guía 3 terminada y verificada.

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Servicio** | Una clase/módulo de la capa de negocio: implementa un proceso del DFD con sus reglas (ADR-001) |
| **Hash** | Transformación de un solo sentido de la contraseña; con buena "sal", no se revierte ni se repite (RNF-01) |
| **Token JWT** | Un "carné" firmado que el servidor emite al iniciar sesión y que porta id y rol sin necesidad de recordar nada (ADR-003) |
| **Interfaz (ABC)** | Un contrato de métodos que una clase promete cumplir; el resto del código depende del contrato, no de una implementación (DIP) |
| **Upsert** | "Actualizar si existe, insertar si no": exactamente lo que exige recalificar (RN-01 + RF-09) |

---

## Paso 0 — Dependencias y configuración

```powershell
pip install bcrypt PyJWT
pip freeze > requirements.txt
```

🧠 **El desarrollador piensa:** *uso `bcrypt` directamente y NO passlib, que es el envoltorio que muestran muchos tutoriales viejos: su última versión rompe con bcrypt moderno y ese error le pasaría factura a cada alumno en su casa. Elegir dependencias también es arquitectura. Y ya que estamos, agrego a `config.py` las dos variables que hoy necesito: la vida del token y la carpeta de carátulas — todo lo configurable sigue viviendo en un solo lugar.*

Actualiza **`app/config.py`** completo:

```python
"""Configuración central de Cartelera.

Toda la configuración vive en variables de entorno con valores por defecto
pensados para desarrollo local. En producción (Render) se configuran en el
dashboard y este archivo no cambia. Ver docs/04_arquitectura (RNF-03).
"""
import os
from pathlib import Path

# Carpeta raíz del proyecto (este archivo es app/config.py, la raíz está un nivel arriba)
BASE_DIR = Path(__file__).resolve().parent.parent

# Firma de los tokens JWT (ADR-003)
SECRET_KEY = os.getenv("SECRET_KEY", "dev-secreto-cambiar-en-produccion")

# URL de la base de datos (ADR-002)
DATABASE_URL = os.getenv(
    "DATABASE_URL", f"sqlite:///{(BASE_DIR / 'cartelera.db').as_posix()}"
)

# Carpeta de carátulas subidas (ADR-004)
UPLOAD_DIR = os.getenv("UPLOAD_DIR", str(BASE_DIR / "uploads"))

# Vida útil del token, en días (ADR-003)
DIAS_TOKEN = 7
```

---

## Paso 1 — `almacenamiento.py`: el DIP hecho carne

🧠 **El desarrollador piensa (la decisión de la semana):** *las carátulas son archivos, y los archivos hoy viven en el disco local… pero en producción gratis el disco es efímero (ADR-004): el día que se vuelva a desplegar, las imágenes vuelan. Podría ignorarlo; prefiero convertirlo en una **puerta abierta**: el resto del código dependerá de una **interfaz** con tres métodos, y la elección de la implementación concreta vivirá en **una sola línea** al final del archivo. Mañana, agregar Cloudinary es escribir una clase nueva y cambiar esa línea — cero cirugía en el resto del sistema. Es exactamente el ejemplo `08_dip_inyeccion.py` del ramo, pero pagando consecuencias reales.*

Crea la carpeta `app/servicios/` y el archivo **`app/servicios/almacenamiento.py`**:

```python
"""Almacenamiento de carátulas (ADR-004) — el ejemplo de DIP del proyecto.

El resto del código depende de la INTERFAZ Almacenamiento, nunca de la clase
concreta. Por eso cambiar "disco local" por "Cloudinary/S3" mañana significa
escribir UNA clase nueva y cambiar UNA línea en este archivo. Ninguna otra
capa se entera.
"""
import uuid
from abc import ABC, abstractmethod
from pathlib import Path

from app.config import UPLOAD_DIR


class Almacenamiento(ABC):
    """Contrato: lo que debe saber hacer cualquier almacén de carátulas."""

    @abstractmethod
    def guardar(self, nombre_original: str, contenido: bytes) -> str:
        """Guarda el archivo y devuelve la ruta/nombre con que se registra en la BD."""

    @abstractmethod
    def eliminar(self, ruta: str) -> None:
        """Elimina un archivo guardado previamente."""

    @abstractmethod
    def url_publica(self, ruta: str | None) -> str:
        """URL con la que el navegador puede pedir el archivo."""


class AlmacenamientoLocal(Almacenamiento):
    """Implementación de desarrollo: archivos en la carpeta uploads/.

    ADVERTENCIA (ADR-004): en Render free el disco es EFÍMERO. Los archivos
    subidos desaparecen cuando el servicio se reinicia o se vuelve a
    desplegar. En desarrollo local esto nunca molesta; en producción es la
    lección en vivo del ramo.
    """

    def __init__(self, carpeta: str = UPLOAD_DIR):
        self.carpeta = Path(carpeta)
        self.carpeta.mkdir(parents=True, exist_ok=True)

    def guardar(self, nombre_original: str, contenido: bytes) -> str:
        # Nombre nuevo aleatorio: evita choques y caracteres problemáticos
        # (una carátula llamada "shrek 2 FINAL(2).jpg" nunca debe romper nada)
        extension = Path(nombre_original).suffix.lower()
        nombre = f"{uuid.uuid4().hex}{extension}"
        (self.carpeta / nombre).write_bytes(contenido)
        return nombre

    def eliminar(self, ruta: str) -> None:
        archivo = self.carpeta / ruta
        if archivo.exists():
            archivo.unlink()

    def url_publica(self, ruta: str | None) -> str:
        if not ruta:
            return "/static/img/placeholder.svg"  # sin carátula (RN-05)
        return f"/uploads/{ruta}"


# ---------------------------------------------------------------------------
# PUNTO ÚNICO DE ELECCIÓN. El día que exista AlmacenamientoCloudinary, esta es
# la única línea del proyecto que cambia (junto con el archivo de la clase nueva).
# ---------------------------------------------------------------------------
almacenamiento: Almacenamiento = AlmacenamientoLocal()
```

## Paso 2 — `autenticacion.py`: contraseñas y tokens

```python
"""Servicio de autenticación: hash, tokens JWT y registro (ADR-003).

La lógica de "quién es" y "qué puede hacer" vive aquí, no en las rutas.
"""
from datetime import datetime, timedelta, timezone

import bcrypt
import jwt
from sqlalchemy.orm import Session

from app.config import DIAS_TOKEN, SECRET_KEY
from app.modelos import Usuario
from app.repositorios import RepositorioUsuarios

ALGORITMO = "HS256"


# ---------- Contraseñas ----------

def hashear(contrasena: str) -> str:
    """Convierte una contraseña en su hash bcrypt.

    bcrypt incorpora la "sal" automáticamente: la misma contraseña produce un
    hash distinto cada vez (por eso no se puede "des-hashear", solo reintentar
    con checkpw).
    """
    return bcrypt.hashpw(contrasena.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")


def verificar(contrasena: str, hash_almacenado: str) -> bool:
    return bcrypt.checkpw(contrasena.encode("utf-8"), hash_almacenado.encode("utf-8"))


# ---------- Tokens JWT ----------

def crear_token(usuario: Usuario) -> str:
    """Emite el carné: porta el id (sub) y el rol, firmados, con expiración."""
    datos = {
        "sub": str(usuario.id),
        "rol": usuario.rol,
        "exp": datetime.now(timezone.utc) + timedelta(days=DIAS_TOKEN),
    }
    return jwt.encode(datos, SECRET_KEY, algorithm=ALGORITMO)


def leer_token(token: str) -> dict | None:
    """Devuelve los datos del token si la firma y la expiración son válidas."""
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITMO])
    except jwt.PyJWTError:
        return None  # token falsificado, malformado o vencido


# ---------- Casos de uso ----------

def registrar(sesion: Session, nombre: str, email: str, contrasena: str) -> Usuario:
    """Crea una cuenta (RF-01). Lanza ValueError con mensaje para el usuario."""
    nombre = nombre.strip()
    email = email.strip().lower()

    # Validaciones duplicadas a propósito: la frontera (esquemas, guía 5)
    # también valida, pero el servicio no confía en nadie (defensa en profundidad)
    if len(nombre) < 2:
        raise ValueError("El nombre debe tener al menos 2 caracteres.")
    if len(contrasena) < 8:
        raise ValueError("La contraseña debe tener al menos 8 caracteres.")

    repo = RepositorioUsuarios(sesion)
    if repo.por_email(email) is not None:
        raise ValueError("Ya existe una cuenta con ese email.")

    usuario = Usuario(nombre=nombre, email=email, password_hash=hashear(contrasena))
    return repo.crear(usuario)


def entrar(sesion: Session, email: str, contrasena: str) -> str:
    """Verifica credenciales y devuelve el token de sesión (RF-02).

    Mensaje genérico a propósito: no le decimos al atacante SI el email existe.
    """
    usuario = RepositorioUsuarios(sesion).por_email(email)
    if usuario is None or not verificar(contrasena, usuario.password_hash):
        raise ValueError("Email o contraseña incorrectos.")
    return crear_token(usuario)
```

🧠 **El desarrollador piensa:** *fíjate en el tamaño de `entrar`: si el usuario no existe O la contraseña no calza, el mismo error, en el mismo instante. Un mensaje distinto ("ese email no está registrado") convierte el login en un oráculo gratis para inventariar cuentas ajenas.*

## Paso 3 — `peliculas.py`: tráilers y reglas de publicación

```python
"""Servicio de películas: reglas del catálogo (RN-03, RN-05) y armado de vistas."""
import os
from urllib.parse import parse_qs, urlparse

from fastapi import HTTPException, UploadFile
from sqlalchemy.orm import Session

from app.modelos import Pelicula
from app.repositorios import RepositorioPeliculas
from app.servicios.almacenamiento import almacenamiento

# RN-05: formatos y tamaño máximo de la carátula
EXTENSIONES_PERMITIDAS = {".jpg", ".jpeg", ".png", ".webp"}
TAMANO_MAXIMO = 5 * 1024 * 1024  # 5 MB


def normalizar_trailer(url: str) -> str:
    """Valida que el link sea de YouTube y lo convierte a formato embed (RN-03).

    Acepta los tres formatos que la gente pega:
      - https://www.youtube.com/watch?v=ID
      - https://youtu.be/ID
      - https://www.youtube.com/shorts/ID
    Lanza ValueError con mensaje si el link no sirve.
    """
    partes = urlparse(url.strip())
    dominio = partes.netloc.lower().removeprefix("www.")

    if dominio not in ("youtube.com", "m.youtube.com", "music.youtube.com", "youtu.be"):
        raise ValueError("El tráiler debe ser un link de YouTube.")

    video_id = ""
    if dominio == "youtu.be":
        video_id = partes.path.lstrip("/")
    else:
        video_id = parse_qs(partes.query).get("v", [""])[0]
        if not video_id and "/shorts/" in partes.path:
            video_id = partes.path.split("/shorts/")[1].split("/")[0]

    video_id = video_id.strip("/")
    if not video_id:
        raise ValueError("No se pudo encontrar el video en el link de YouTube.")

    return f"https://www.youtube.com/embed/{video_id}"


def crear_pelicula(
    sesion: Session,
    titulo: str,
    anio: int,
    sinopsis: str,
    url_trailer: str,
    caratula: UploadFile | None,
) -> Pelicula:
    """Crea la película validando las reglas (RF-04, RN-03, RN-05)."""
    titulo = titulo.strip()
    if not titulo:
        raise HTTPException(400, "El título es obligatorio.")

    try:
        trailer_embed = normalizar_trailer(url_trailer)
    except ValueError as error:
        raise HTTPException(400, str(error))

    ruta = None
    if caratula is not None and caratula.filename:
        extension = os.path.splitext(caratula.filename)[1].lower()
        if extension not in EXTENSIONES_PERMITIDAS:
            raise HTTPException(
                400, f"Carátula no permitida ({extension}). Usa JPG, PNG o WebP."
            )
        contenido = caratula.file.read()
        if len(contenido) > TAMANO_MAXIMO:
            raise HTTPException(400, "La carátula supera el máximo de 5 MB.")
        ruta = almacenamiento.guardar(caratula.filename, contenido)

    pelicula = Pelicula(
        titulo=titulo,
        anio=anio,
        sinopsis=sinopsis.strip(),
        url_trailer=trailer_embed,
        ruta_caratula=ruta,
    )
    return RepositorioPeliculas(sesion).crear(pelicula)


def eliminar_pelicula(sesion: Session, pelicula: Pelicula) -> None:
    """Elimina la película y su carátula del disco (RF-07)."""
    if pelicula.ruta_caratula:
        almacenamiento.eliminar(pelicula.ruta_caratula)
    RepositorioPeliculas(sesion).eliminar(pelicula)


def a_resumen(pelicula: Pelicula, promedio: float, total: int) -> dict:
    """Cómo una película se muestra en listas (lo usarán la web y la API)."""
    return {
        "id": pelicula.id,
        "titulo": pelicula.titulo,
        "anio": pelicula.anio,
        "promedio": round(promedio, 1),
        "total_calificaciones": total,
        "imagen": almacenamiento.url_publica(pelicula.ruta_caratula),
    }


def a_detalle(pelicula: Pelicula, promedio: float, total: int) -> dict:
    """La ficha completa (agrega sinopsis y tráiler al resumen)."""
    datos = a_resumen(pelicula, promedio, total)
    datos["sinopsis"] = pelicula.sinopsis
    datos["url_trailer"] = pelicula.url_trailer
    return datos
```

🧠 **Una honestidad necesaria:** *los puristas dirán que un servicio no debería usar `HTTPException` ni `UploadFile` (cosas del mundo web). Razón tienen… en un sistema grande. Aquí lo hacemos **pragmático**: el mensaje y su código de error viajan juntos y las capas de arriba no traducen nada. Es la misma decisión registrada en el documento de arquitectura §5: SOLID donde paga, no por ritual. (Pregunta de clase: ¿qué tendríamos que hacer para desacoplarlo del todo?)*

## Paso 4 — `calificaciones.py`: la regla RN-01, lado amable

```python
"""Servicio de calificaciones: la regla RN-01 vive aquí (ADR-005).

Tanto el widget HTML como el endpoint JSON llegarán a esta MISMA función:
una sola lógica, dos caras (ADR-001).
"""
from datetime import datetime, timezone

from fastapi import HTTPException
from sqlalchemy.orm import Session

from app.modelos import Calificacion, Usuario
from app.repositorios import RepositorioCalificaciones, RepositorioPeliculas


def calificar(sesion: Session, usuario: Usuario, pelicula_id: int, estrellas: int) -> Calificacion:
    """Guarda o reemplaza la calificación del usuario (RF-08, RF-09, RN-01).

    "Recalificar" no crea una segunda fila: actualiza la existente (upsert).
    Si dos peticiones simultáneas lograran colarse, la restricción UNIQUE de
    la base de datos frena a la segunda (lo viste en la guía 2).
    """
    if RepositorioPeliculas(sesion).por_id(pelicula_id) is None:
        raise HTTPException(404, "Película no encontrada.")

    repo = RepositorioCalificaciones(sesion)
    existente = repo.por_usuario_y_pelicula(usuario.id, pelicula_id)

    if existente is not None:
        existente.estrellas = estrellas
        existente.actualizado_en = datetime.now(timezone.utc)
        sesion.commit()
        return existente

    return repo.guardar(
        Calificacion(usuario_id=usuario.id, pelicula_id=pelicula_id, estrellas=estrellas)
    )


def mi_calificacion(sesion: Session, usuario: Usuario | None, pelicula_id: int) -> int:
    """Estrellas que dio este usuario a esta película (0 si no ha calificado)."""
    if usuario is None:
        return 0
    fila = RepositorioCalificaciones(sesion).por_usuario_y_pelicula(usuario.id, pelicula_id)
    return fila.estrellas if fila else 0
```

Y **`app/servicios/__init__.py`**:

```python
from app.servicios import autenticacion, calificaciones, peliculas

__all__ = ["autenticacion", "calificaciones", "peliculas"]
```

---

## ✅ Verificación de la guía 4

Crea **`verificar_servicios.py`** en la raíz (desechable):

```python
"""Verificación de la guía 4: hash, token, tráiler, almacenamiento y upsert."""
from app.database import SesionLocal, crear_tablas
from app.modelos import Pelicula
from app.repositorios import RepositorioPeliculas
from app.servicios import autenticacion, calificaciones
from app.servicios.peliculas import normalizar_trailer
from app.servicios.almacenamiento import almacenamiento

crear_tablas()

# 1) Contraseña: hash y verificación
hash1 = autenticacion.hashear("clave-segura-123")
hash2 = autenticacion.hashear("clave-segura-123")
print("1) Mismas contraseñas, hashes distintos (sal):", hash1 != hash2)
print("   Verificación correcta:", autenticacion.verificar("clave-segura-123", hash1))
print("   Verificación incorrecta:", autenticacion.verificar("otra-clave", hash1))

with SesionLocal() as sesion:
    # 2) Registro + login (con el hash de verdad esta vez)
    ana = autenticacion.registrar(sesion, "Ana", "ana@test.cl", "clave-segura-123")
    token = autenticacion.entrar(sesion, "ana@test.cl", "clave-segura-123")
    datos = autenticacion.leer_token(token)
    print("2) Token legible:", datos["sub"] == str(ana.id), "| rol:", datos["rol"])
    print("   Token falsificado rechazado:", autenticacion.leer_token(token + "x") is None)
    try:
        autenticacion.registrar(sesion, "Ana2", "ana@test.cl", "otra-clave-123")
        print("   ¡El duplicado se aceptó! MAL ❌")
    except ValueError as e:
        print("   Email duplicado rechazado:", e)

    # 3) Tráiler: los tres formatos y el rechazo
    print("3) watch?v= :", normalizar_trailer("https://www.youtube.com/watch?v=abc123"))
    print("   youtu.be :", normalizar_trailer("https://youtu.be/abc123"))
    print("   shorts=  :", normalizar_trailer("https://www.youtube.com/shorts/abc123"))
    try:
        normalizar_trailer("https://vimeo.com/123")
    except ValueError as e:
        print("   Vimeo rechazado:", e)

    # 4) Almacenamiento: guardar, URL y eliminar (DIP: nadie sabe que es disco local)
    ruta = almacenamiento.guardar("mi caratula.jpg", b"bytes-falsos-de-imagen")
    print("4) Guardado como:", ruta, "| URL:", almacenamiento.url_publica(ruta))
    almacenamiento.eliminar(ruta)
    print("   Sin carátula →", almacenamiento.url_publica(None))

    # 5) La regla RN-01 por el lado amable: votar y RECALIFICAR sin errores
    peli = RepositorioPeliculas(sesion).crear(
        Pelicula(titulo="Coco", anio=2017, sinopsis="…",
                 url_trailer="https://www.youtube.com/embed/x")
    )
    calificaciones.calificar(sesion, ana, peli.id, 5)
    print("5) Promedio tras voto 5★:", RepositorioPeliculas(sesion).promedio(peli.id))
    calificaciones.calificar(sesion, ana, peli.id, 2)   # recalifica: reemplaza
    promedio, total = RepositorioPeliculas(sesion).promedio(peli.id)
    print("   Tras recalificar a 2★ → promedio:", promedio, "| total votos:", total, "(debe seguir siendo 1)")
```

Con el servidor **detenido**:

```powershell
python verificar_servicios.py
```

Debes ver los tres formatos de tráiler convertidos a `embed`, Vimeo rechazado, el hash distinto en cada llamada pero verificable, el token legible y el falsificado rechazado, el duplicado de email rebotado, el archivo guardado con nombre aleatorio y URL pública correcta, y — lo más importante — **el total de votos sigue en 1 después de recalificar**. Después limpia:

```powershell
del verificar_servicios.py
del cartelera.db
```

---

## 📝 Punto de control

1. ¿Por qué `hash1 != hash2` si las contraseñas son iguales? ¿Y por qué eso es una virtud?
2. ¿Qué línea exacta del proyecto habría que tocar para pasar de disco local a Cloudinary? ¿Cuántos archivos se modifican?
3. En `calificar`, ¿qué pasaría si quitáramos la búsqueda de voto existente y confiáramos solo en la restricción UNIQUE? (Pista: ¿qué vería la socia en su pantalla?)

## Lo que acabas de aprender

- Hash con sal y tokens JWT firmados (ADR-003)
- Dependencia de interfaces, no de implementaciones (DIP, ADR-004)
- Upsert: la misma operación "calificar" sirve para votar y para cambiar el voto
- Verificar reglas de negocio con scripts, antes de cualquier interfaz

**Siguiente:** `guia-05-api-json.md` — abrimos la primera ventana hacia afuera: la API que debe cumplir el contrato OpenAPI de la fase 4, al pie de la letra.
