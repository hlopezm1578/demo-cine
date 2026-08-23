# Guía 3 — Repositorios: los almacenes del DFD

> **Qué construirás hoy:** la capa de acceso a datos: cuatro clases que concentran TODA la conversación con la base de datos.
> **Al terminar tendrás:** consultas listas para listar el catálogo con promedios (sin el problema N+1) y verificarlo con un script.
> **Necesitas:** la guía 2 terminada y verificada.

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Repositorio** | Una clase que encapsula el acceso a un "almacén" de datos: el resto del programa le pide cosas, sin saber cómo las consigue |
| **N+1 (problema)** | La trampa de hacer 1 consulta para listar N películas y luego 1 consulta de promedio **por cada una**: N+1 viajes a la base |
| **Consulta agrupada** | Un solo `GROUP BY` que resuelve todos los promedios de una vez |

---

## Paso 1 — La carpeta y la regla

🧠 **El desarrollador piensa:** *en el DFD del diseño (§3.2) hay tres almacenes: D1 Usuarios, D2 Películas, D3 Calificaciones. Voy a crear **un repositorio por almacén** — la correspondencia es literal. Y la regla que juro respetar (ADR-001): un repositorio ejecuta consultas y nada más. El día que alguien quiera ponerle "no dejar calificar dos veces", lo mando a la capa de servicios: eso es una regla de negocio disfrazada.*

Otra decisión: **el repositorio recibe la sesión por constructor; jamás la crea**. Quien llama (FastAPI, un script, una prueba) decide la duración de la conversación. Eso hará que en la guía 5 las pruebas sean casi gratis.

Crea la carpeta `app/repositorios/`.

---

## Paso 2 — `usuarios.py`

```python
"""Repositorio de usuarios: SOLO acceso a datos, sin lógica de negocio (ADR-001)."""
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.modelos import Usuario


class RepositorioUsuarios:
    def __init__(self, sesion: Session):
        self.sesion = sesion

    def por_id(self, usuario_id: int) -> Usuario | None:
        return self.sesion.get(Usuario, usuario_id)

    def por_email(self, email: str) -> Usuario | None:
        consulta = select(Usuario).where(Usuario.email == email.lower().strip())
        return self.sesion.scalars(consulta).first()

    def crear(self, usuario: Usuario) -> Usuario:
        self.sesion.add(usuario)
        self.sesion.commit()
        self.sesion.refresh(usuario)
        return usuario
```

🧠 **Detalle fino:** `por_email` normaliza (minúsculas, sin espacios) **antes** de buscar. Si no, `Ana@test.cl` y `ana@test.cl` pasarían por cuentas distintas… hasta que `unique=True` reviente en el peor momento. Normalizar en la entrada es más barato que disculparse después.

## Paso 3 — `peliculas.py` (con los promedios)

```python
"""Repositorio de películas: SOLO acceso a datos (ADR-001).

Incluye las consultas de promedio (ADR-005): AVG y COUNT se calculan en la
base de datos, no trayendo todas las filas a Python.
"""
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from app.modelos import Calificacion, Pelicula


class RepositorioPeliculas:
    def __init__(self, sesion: Session):
        self.sesion = sesion

    def listar(self) -> list[Pelicula]:
        return list(
            self.sesion.scalars(select(Pelicula).order_by(Pelicula.creado_en.desc()))
        )

    def por_id(self, pelicula_id: int) -> Pelicula | None:
        return self.sesion.get(Pelicula, pelicula_id)

    def crear(self, pelicula: Pelicula) -> Pelicula:
        self.sesion.add(pelicula)
        self.sesion.commit()
        self.sesion.refresh(pelicula)
        return pelicula

    def eliminar(self, pelicula: Pelicula) -> None:
        # las calificaciones asociadas se eliminan en cascada (guía 2)
        self.sesion.delete(pelicula)
        self.sesion.commit()

    def promedio(self, pelicula_id: int) -> tuple[float, int]:
        """(promedio, cantidad) de una película. (0.0, 0) si nadie ha votado."""
        consulta = select(
            func.coalesce(func.avg(Calificacion.estrellas), 0.0),
            func.count(),
        ).where(Calificacion.pelicula_id == pelicula_id)
        fila = self.sesion.execute(consulta).one()
        return float(fila[0]), int(fila[1])

    def promedios(self) -> dict[int, tuple[float, int]]:
        """Promedios de TODAS las películas en una sola consulta.

        Evita el problema N+1: una consulta para el catálogo completo en vez
        de una de promedio por cada película.
        """
        consulta = select(
            Calificacion.pelicula_id,
            func.avg(Calificacion.estrellas),
            func.count(),
        ).group_by(Calificacion.pelicula_id)
        filas = self.sesion.execute(consulta).all()
        return {fila[0]: (float(fila[1]), int(fila[2])) for fila in filas}
```

🧠 **El desarrollador piensa (la decisión del día):** *el catálogo necesita el promedio de cada película. Mi primer impulso: un `for` que llame a `promedio()` por película. Con 4 películas nadie nota; con 400 son 401 viajes a la base por cada visita al home. El método `promedios()` resuelve todo en **una** consulta agrupada. `promedio()` lo mantengo para la ficha de una sola película. Elegir la consulta correcta **antes** de que duela es la diferencia entre diseñar y parchear. (Y `coalesce`: sin votos, `AVG` devuelve "nada", no 0 — y "nada" rompe la aritmética posterior.)*

## Paso 4 — `calificaciones.py`

```python
"""Repositorio de calificaciones: SOLO acceso a datos (ADR-001, ADR-005)."""
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.modelos import Calificacion


class RepositorioCalificaciones:
    def __init__(self, sesion: Session):
        self.sesion = sesion

    def por_usuario_y_pelicula(self, usuario_id: int, pelicula_id: int) -> Calificacion | None:
        consulta = select(Calificacion).where(
            Calificacion.usuario_id == usuario_id,
            Calificacion.pelicula_id == pelicula_id,
        )
        return self.sesion.scalars(consulta).first()

    def guardar(self, calificacion: Calificacion) -> Calificacion:
        self.sesion.add(calificacion)
        self.sesion.commit()
        self.sesion.refresh(calificacion)
        return calificacion
```

Y **`app/repositorios/__init__.py`**:

```python
from app.repositorios.calificaciones import RepositorioCalificaciones
from app.repositorios.peliculas import RepositorioPeliculas
from app.repositorios.usuarios import RepositorioUsuarios

__all__ = [
    "RepositorioCalificaciones",
    "RepositorioPeliculas",
    "RepositorioUsuarios",
]
```

---

## ✅ Verificación de la guía 3

Crea **`verificar_repositorios.py`** en la raíz (desechable, como en la guía 2):

```python
"""Verificación de la guía 3: repositorios + consulta de promedios agrupada."""
from app.database import SesionLocal, crear_tablas
from app.modelos import Calificacion, Pelicula, Usuario
from app.repositorios import (
    RepositorioCalificaciones,
    RepositorioPeliculas,
    RepositorioUsuarios,
)

crear_tablas()

with SesionLocal() as sesion:
    repo_peliculas = RepositorioPeliculas(sesion)
    repo_usuarios = RepositorioUsuarios(sesion)
    repo_votos = RepositorioCalificaciones(sesion)

    peli1 = repo_peliculas.crear(Pelicula(titulo="Coco", anio=2017, sinopsis="…",
                                          url_trailer="https://www.youtube.com/embed/x"))
    peli2 = repo_peliculas.crear(Pelicula(titulo="Interestelar", anio=2014, sinopsis="…",
                                          url_trailer="https://www.youtube.com/embed/y"))
    ana = repo_usuarios.crear(Usuario(nombre="Ana", email="ana@test.cl",
                                       password_hash="hash-provisorio"))
    bo = repo_usuarios.crear(Usuario(nombre="Bo", email="bo@test.cl",
                                      password_hash="hash-provisorio"))

    repo_votos.guardar(Calificacion(usuario_id=ana.id, pelicula_id=peli1.id, estrellas=5))
    repo_votos.guardar(Calificacion(usuario_id=bo.id, pelicula_id=peli1.id, estrellas=3))
    repo_votos.guardar(Calificacion(usuario_id=ana.id, pelicula_id=peli2.id, estrellas=4))

    print("Promedio de una película:", repo_peliculas.promedio(peli1.id))   # (4.0, 2)
    print("Promedios agrupados:    ", repo_peliculas.promedios())
    print("Voto de Ana por Coco:   ", repo_votos.por_usuario_y_pelicula(ana.id, peli1.id))
    print("Catálogo (más nueva primero):", [p.titulo for p in repo_peliculas.listar()])

    # Eliminación en cascada: borrar Coco se lleva sus 2 votos
    repo_peliculas.eliminar(peli1)
    print("Tras eliminar Coco, promedios:", repo_peliculas.promedios())     # solo Interestelar
```

Ejecuta con el servidor **detenido**:

```powershell
python verificar_repositorios.py
```

Debes ver: promedio `(4.0, 2)` para Coco, el diccionario agrupado con ambas películas, el catálogo ordenado con Interestelar primero, y — tras eliminar Coco — solo queda Interestelar en los promedios (la cascada funcionó). Después limpia:

```powershell
del verificar_repositorios.py
del cartelera.db
```

---

## 📝 Punto de control

1. Un compañero propone agregar al repositorio de calificaciones el método `puede_votar(usuario, pelicula)` que verifique restricciones. ¿Va ahí? ¿Dónde va y por qué?
2. ¿Por qué el repositorio recibe la sesión en vez de crearla él mismo?
3. Explica el problema N+1 con tus palabras y cómo lo evita `promedios()`.

## Lo que acabas de aprender

- El patrón Repositorio: un almacén del DFD = una clase
- Consultas agregadas en la base (`AVG`, `COUNT`, `GROUP BY`, `COALESCE`)
- Por qué las sesiones se reciben, no se crean
- Verificar una capa con un script, sin interfaz todavía

**Siguiente:** `guia-04-servicios.md` — las reglas del negocio: contraseñas, tokens, tráilers de YouTube y la abstracción de almacenamiento (el DIP del ramo, en vivo).
