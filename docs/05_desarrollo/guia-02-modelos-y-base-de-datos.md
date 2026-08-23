# Guía 2 — Los datos: modelos y base de datos

> **Qué construirás hoy:** las tres tablas del sistema (usuarios, películas, calificaciones) y su primera regla de negocio blindada.
> **Al terminar tendrás:** un archivo `cartelera.db` con tus tablas creadas, y una demostración en vivo de que la base de datos **rechaza sola** un segundo voto de la misma persona (RN-01).
> **Necesitas:** la guía 1 terminada y verificada.

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Base de datos relacional** | Un archivador de tablas que se relacionan entre sí (la nuestra: SQLite, que es un solo archivo) |
| **ORM** | "*Object Relational Mapping*": un traductor entre tablas y objetos de Python. Escribimos clases; él escribe el SQL (ADR-002) |
| **Modelo** | Una clase de Python que representa una tabla |
| **Restricción** | Una regla que la base de datos hace cumplir solita, sin pedirle permiso al programa |
| **Migración** | El procedimiento para *modificar* tablas ya creadas. Nosotros no lo usaremos: creamos tablas nuevas y listo (ver ADR-002, consecuencias) |

---

## Paso 1 — Instalar el ORM

```powershell
pip install sqlalchemy
pip freeze > requirements.txt
```

🧠 **El desarrollador piensa:** *elegí SQLAlchemy por una razón que verás en la guía de despliegue: el mismo código conversa con SQLite (desarrollo) y con Postgres (producción) cambiando **una sola línea de configuración** — la URL que ya vive en `config.py`. Si escribiera SQL a mano, tendría dos versiones de cada consulta.*

---

## Paso 2 — `app/database.py`: la conexión

🧠 **El desarrollador piensa:** *este archivo es el único del proyecto que sabe hablar con la base de datos "en bruto". Define tres cosas: el **motor** (la conexión, elegida por la URL de `config.py`), la **sesión** (conversaciones cortas: se abre, se hace algo, se cierra — en la guía 3 verás que FastAPI abrirá una por cada petición), y la **Base** (la clase padre de todos los modelos). Todo lo demás del proyecto importará desde aquí, nunca creará conexiones propias.*

Crea **`app/database.py`**:

```python
"""Conexión a la base de datos y sesión de SQLAlchemy (ADR-002).

El motor se elige por la URL de config.py: SQLite en desarrollo, Postgres en
producción. El resto del código nunca sabe cuál está usando.
"""
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker

from app.config import DATABASE_URL

# SQLite necesita este parámetro porque FastAPI atiende peticiones desde varios
# hilos y, por defecto, la librería sqlite3 rechaza conexiones cruzadas.
es_sqlite = DATABASE_URL.startswith("sqlite")
motor = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False} if es_sqlite else {},
)

# expire_on_commit=False: los objetos siguen útiles después de guardarlos
# (podremos leer u1.id después de un commit sin recargar nada)
SesionLocal = sessionmaker(bind=motor, autoflush=False, expire_on_commit=False)


class Base(DeclarativeBase):
    """Clase padre de todos los modelos ORM."""


def crear_tablas() -> None:
    """Crea las tablas si no existen.

    Cómodo para un proyecto de aprendizaje; en un sistema real se usan
    migraciones versionadas (Alembic), porque create_all NO modifica tablas
    ya creadas. Ver ADR-002.
    """
    from app import modelos  # noqa: F401 — importar registra los modelos en Base

    Base.metadata.create_all(bind=motor)


def obtener_sesion():
    """Dependencia de FastAPI: una sesión por petición, cerrada al final."""
    sesion: Session = SesionLocal()
    try:
        yield sesion
    finally:
        sesion.close()
```

> ⚠️ **Detalle que ahorra una tarde de dolor:** la clase `Session` se importa desde `sqlalchemy.orm`, **no** desde `sqlalchemy`. Si te aparece `ImportError: cannot import name 'Session'`, revisa esa línea. (Pregunta de clase: ¿por qué creen que SQLAlchemy tiene esta trampa?)

---

## Paso 3 — Los modelos: traducir el diccionario de datos

🧠 **El desarrollador piensa:** *no estoy inventando nada: el diccionario de datos de la fase de diseño (§2.2) ya definió atributo por atributo con sus tipos y restricciones. Mi trabajo como desarrollador es **traducir fielmente**, no reinterpretar. Tres decisiones del diseño merecen atención especial: el hash en vez de la contraseña (RNF-01), la carátula como ruta opcional (RN-05), y la restricción UNIQUE que blinda RN-01 (§2.3.1).*

Crea la carpeta `app/modelos/` con estos cuatro archivos.

**`app/modelos/__init__.py`** — importa todo para que `crear_tablas()` los registre:

```python
"""Modelos ORM de Cartelera.

Importarlos todos aquí hace que database.crear_tablas() los registre en Base.
"""
from app.modelos.calificacion import Calificacion
from app.modelos.pelicula import Pelicula
from app.modelos.usuario import Usuario

__all__ = ["Usuario", "Pelicula", "Calificacion"]
```

**`app/modelos/usuario.py`**:

```python
"""Tabla de usuarios (diseño §2.2): socios y la coordinadora."""
from datetime import datetime, timezone

from sqlalchemy import DateTime, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.database import Base


class Usuario(Base):
    __tablename__ = "usuarios"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(80))
    # unique=True: la base impide cuentas duplicadas (RF-01)
    email: Mapped[str] = mapped_column(String(120), unique=True, index=True)
    # Nunca la contraseña en texto plano (RNF-01): aquí vive su hash bcrypt
    password_hash: Mapped[str] = mapped_column(String(200))
    # Dos roles alcanzan (ADR-003): 'socio' califica, 'admin' además gestiona
    rol: Mapped[str] = mapped_column(String(20), default="socio")
    creado_en: Mapped[datetime] = mapped_column(
        DateTime, default=lambda: datetime.now(timezone.utc)
    )

    # Si se elimina un usuario, sus calificaciones se van con él
    calificaciones = relationship(
        "Calificacion", back_populates="usuario", cascade="all, delete-orphan"
    )
```

**`app/modelos/pelicula.py`**:

```python
"""Tabla de películas (diseño §2.2)."""
from datetime import datetime, timezone

from sqlalchemy import DateTime, Integer, String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.database import Base


class Pelicula(Base):
    __tablename__ = "peliculas"

    id: Mapped[int] = mapped_column(primary_key=True)
    titulo: Mapped[str] = mapped_column(String(120), index=True)
    anio: Mapped[int] = mapped_column(Integer)
    sinopsis: Mapped[str] = mapped_column(Text, default="")
    # Ya normalizada a formato embed (RN-03); la guía 4 hará la conversión
    url_trailer: Mapped[str] = mapped_column(String(300), default="")
    # Nombre del archivo en uploads/ (ADR-004); vacía → imagen genérica (RN-05)
    ruta_caratula: Mapped[str | None] = mapped_column(String(300), nullable=True)
    creado_en: Mapped[datetime] = mapped_column(
        DateTime, default=lambda: datetime.now(timezone.utc)
    )

    # Si se elimina una película, sus calificaciones se van en cascada (HU-07)
    calificaciones = relationship(
        "Calificacion", back_populates="pelicula", cascade="all, delete-orphan"
    )
```

**`app/modelos/calificacion.py`** — la tabla donde vive la regla estrella del proyecto:

```python
"""Tabla de calificaciones (diseño §2.2, ADR-005)."""
from datetime import datetime, timezone

from sqlalchemy import DateTime, ForeignKey, Integer, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.database import Base


class Calificacion(Base):
    __tablename__ = "calificaciones"

    # RN-01: la BASE DE DATOS impide dos votos del mismo socio a la misma
    # película, aunque el código tenga un bug (diseño §2.3.1, ADR-005)
    __table_args__ = (
        UniqueConstraint("usuario_id", "pelicula_id", name="uq_usuario_pelicula"),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    usuario_id: Mapped[int] = mapped_column(ForeignKey("usuarios.id"))
    pelicula_id: Mapped[int] = mapped_column(ForeignKey("peliculas.id"))
    estrellas: Mapped[int] = mapped_column(Integer)  # 1–5, se valida en la frontera (RN-02)

    actualizado_en: Mapped[datetime] = mapped_column(
        DateTime, default=lambda: datetime.now(timezone.utc)
    )

    usuario = relationship("Usuario", back_populates="calificaciones")
    pelicula = relationship("Pelicula", back_populates="calificaciones")
```

🧠 **El desarrollador piensa (la decisión del día):** *podría confiar en un chequeo de Python: "si ya existe un voto de esta socia para esta película, no insertes otro". Pero ese `if` tiene una ventana: dos clics simultáneos pasan ambos el control y crean dos filas. La restricción UNIQUE cierra esa ventana **en el único lugar que no puede mentir: la base de datos**. En la guía 4 escribiré el código amable (el que busca y reemplaza el voto); aquí acabo de instalar el guardián que atrapa lo que ese código no alcance a ver.*

---

## Paso 4 — Conectar la base al arranque

🧠 **El desarrollador piensa:** *la composición de tablas va en `main.py`, que es exactamente su trabajo: decidir qué existe al partir. `crear_tablas()` solo crea lo que falta; si las tablas ya existen, no toca nada — por eso partirlo dos veces no rompe nada.*

Edita **`app/main.py`** hasta que quede así:

```python
"""Punto de composición: arma la aplicación y conecta las piezas (ADR-001)."""
from fastapi import FastAPI

from app.database import crear_tablas

app = FastAPI(
    title="Cartelera",
    description=(
        "Catálogo de películas del CineClub Barrio con calificación por "
        "estrellas. La misma lógica alimenta la web y esta API."
    ),
    version="1.0.0",
)

# Crea las tablas si no existen (ADR-002; en producción real: migraciones)
crear_tablas()


@app.get("/salud")
def salud() -> dict:
    """Chequeo mínimo para saber que el servicio está vivo."""
    return {"estado": "ok"}
```

✅ **Verificación intermedia:** corre `python run.py`, abre `/salud`, y detén con `Ctrl+C`. En la raíz del proyecto debe haber aparecido el archivo **`cartelera.db`** — tu base de datos, en un archivo. Ábrelo si quieres con la extensión "SQLite Viewer" de VS Code: tres tablas esperándote.

---

## Paso 5 — La demostración: RN-01 vivito y coleando

🧠 **El desarrollador piensa:** *antes de seguir construyendo, quiero **probar la regla más importante** sin esperar a tener la API. Escribo un script desechable que intenta lo que el diseño prohíbe: dos votos de la misma socia a la misma película. Si la base hace su trabajo, el segundo debe reventar. (Finge contraseñas por ahora: el hash bcrypt llega en la guía 4.)*

Crea **`verificar_rn01.py`** en la raíz (es desechable, lo borras al final):

```python
"""Demostración de RN-01: la base de datos rechaza el segundo voto.
Archivo desechable de verificación: correr y borrar.
"""
from sqlalchemy.exc import IntegrityError

from app.database import SesionLocal, crear_tablas
from app.modelos import Calificacion, Pelicula, Usuario

crear_tablas()

with SesionLocal() as sesion:
    # Dos socias y una película de prueba
    socia = Usuario(nombre="Socia de prueba", email="prueba@test.cl",
                    password_hash="hash-falso-solo-para-hoy")
    otra = Usuario(nombre="Otra socia", email="otra@test.cl",
                   password_hash="hash-falso-solo-para-hoy")
    pelicula = Pelicula(titulo="Coco", anio=2017,
                        sinopsis="Prueba", url_trailer="https://www.youtube.com/embed/x")
    sesion.add_all([socia, otra, pelicula])
    sesion.commit()
    print(f"Creados: socia id={socia.id}, otra id={otra.id}, película id={pelicula.id}")

    # Voto legítimo
    sesion.add(Calificacion(usuario_id=socia.id, pelicula_id=pelicula.id, estrellas=5))
    sesion.commit()
    print("1) Primer voto de la socia: guardado ✔")

    # El voto prohibido: misma socia, misma película
    sesion.add(Calificacion(usuario_id=socia.id, pelicula_id=pelicula.id, estrellas=3))
    try:
        sesion.commit()
        print("2) ¡El segundo voto se guardó! Algo está MAL ❌")
    except IntegrityError:
        sesion.rollback()
        print("2) Segundo voto de la MISMA socia: RECHAZADO por la base de datos ✔ (RN-01)")

    # Control: otra socia SÍ puede votar la misma película
    sesion.add(Calificacion(usuario_id=otra.id, pelicula_id=pelicula.id, estrellas=4))
    sesion.commit()
    print("3) Primera votación de OTRA socia: guardado ✔ (el UNIQUE no molesta)")
```

✅ **Verificación final:** con el servidor **detenido**, ejecuta:

```powershell
python verificar_rn01.py
```

Debes ver los tres mensajes con ✔. Después, borra el script y el archivo de prueba:

```powershell
del verificar_rn01.py
del cartelera.db
```

(Borramos `cartelera.db` para que el proyecto real parta con la base limpia: se recreará sola en el próximo arranque.)

---

## 📝 Punto de control

1. ¿Por qué la restricción UNIQUE protege mejor RN-01 que un `if` en Python? Describe el escenario de dos clics simultáneos.
2. ¿Qué guarda la columna `password_hash` y por qué no guardamos la contraseña misma?
3. `crear_tablas()` se ejecuta cada vez que parte la app. ¿Por qué eso no borra los datos existentes?

## Lo que acabas de aprender

- El patrón ORM: clases que se convierten en tablas (ADR-002)
- La diferencia entre una regla en el código y una **restricción en la base**
- Relaciones entre tablas y la eliminación en cascada
- A probar tu base de datos con un script, antes de que exista la interfaz

**Siguiente:** `guia-03-repositorios.md` — los almacenes del DFD: la capa que habla con estas tablas para que nadie más tenga que hacerlo.
