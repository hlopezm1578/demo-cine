# Guía 8 — Datos iniciales y la gran revisión final

> **Qué construirás hoy:** la semilla reproducible (coordinadora + películas de ejemplo) y la verificación final del proyecto completo.
> **Al terminar tendrás:** Cartelera completa, corriendo con datos, y la comparación contrato ↔ implementación como evidencia de cierre de la fase 5.
> **Necesitas:** las guías 1 a 7 terminadas.

---

## Paso 1 — La configuración completa

🧠 **El desarrollador piensa:** *la coordinadora dejó de ser un dato suelto de prueba: ahora es configuración del sistema, con su email y contraseña manejables por variables de entorno — en producción nadie debe quedarse pegado con "admin1234" (ADR-007, RNF-03).*

Agrega al final de **`app/config.py`**:

```python
# Cuenta de coordinadora que crea semilla.py
ADMIN_EMAIL = os.getenv("ADMIN_EMAIL", "coordinadora@cineclub.cl")
ADMIN_PASSWORD = os.getenv("ADMIN_PASSWORD", "admin1234")
```

## Paso 2 — `semilla.py`: datos iniciales reproducibles

🧠 **El desarrollador piensa:** *tres decisiones: (1) **idempotente** — correrlo dos veces no duplica nada (por eso los `if`); (2) usa los **servicios**, no inserts crudos: si mañana la regla del hash cambia, la semilla no se queda vieja; (3) las películas de ejemplo van **sin carátula** a propósito: subir carátulas reales es parte del ejercicio desde el panel, y así estrenamos el placeholder. Además, este mismo script correrá contra la base de producción cambiando solo la variable de entorno — sembrar la nube no será un procedimiento distinto.*

Crea **`semilla.py`** en la raíz del proyecto:

```python
"""Datos iniciales: la cuenta de la coordinadora y películas de ejemplo.

Uso (desde la raíz del proyecto):
    python semilla.py

Idempotente: si el admin ya existe o ya hay películas, no duplica nada.
En PRODUCCIÓN lo corre el build de Render en cada deploy (ver fase 7);
a mano también se puede, apuntando la base de Neon desde una red que
permita el puerto 5432:

    # Git Bash / Linux / macOS
    DATABASE_URL="postgresql+psycopg2://usuario:clave@host/bd" python semilla.py

    # PowerShell
    $env:DATABASE_URL="postgresql+psycopg2://usuario:clave@host/bd"; python semilla.py
"""
from app.config import ADMIN_EMAIL, ADMIN_PASSWORD
from app.database import SesionLocal, crear_tablas
from app.modelos import Pelicula, Usuario
from app.repositorios import RepositorioPeliculas, RepositorioUsuarios
from app.servicios import autenticacion

# Películas clásicas con tráilers oficiales en YouTube. Sin carátula a
# propósito: se muestra la imagen genérica y subir una real es parte del
# ejercicio desde el panel de administración.
PELICULAS_EJEMPLO = [
    {
        "titulo": "El Padrino",
        "anio": 1972,
        "sinopsis": "La saga de la familia Corleone y el ascenso de Michael, el hijo que nunca quiso pertenecer a los negocios de su padre.",
        "url_trailer": "https://www.youtube.com/embed/sY1S34973zA",
    },
    {
        "titulo": "Interestelar",
        "anio": 2014,
        "sinopsis": "Con la Tierra agonizando, un grupo de astronautas cruza un agujero negro en busca de un nuevo hogar para la humanidad.",
        "url_trailer": "https://www.youtube.com/embed/zSWdZVtXT7E",
    },
    {
        "titulo": "Coco",
        "anio": 2017,
        "sinopsis": "Miguel viaja por accidente a la Tierra de los Muertos y descubre la verdad sobre la historia de su familia.",
        "url_trailer": "https://www.youtube.com/embed/Ga6RYejo6Hk",
    },
    {
        "titulo": "Spider-Man: Un nuevo universo",
        "anio": 2018,
        "sinopsis": "Miles Morales descubre que no es el único Spider-Man cuando héroes de dimensiones paralelas llegan a la suya.",
        "url_trailer": "https://www.youtube.com/embed/g4Hbz2jLxvQ",
    },
]


def main() -> None:
    crear_tablas()

    with SesionLocal() as sesion:
        repo_usuarios = RepositorioUsuarios(sesion)

        if repo_usuarios.por_email(ADMIN_EMAIL) is None:
            repo_usuarios.crear(
                Usuario(
                    nombre="Macarena",
                    email=ADMIN_EMAIL,
                    password_hash=autenticacion.hashear(ADMIN_PASSWORD),
                    rol="admin",
                )
            )
            print(f"[+] Coordinadora creada: {ADMIN_EMAIL} (contraseña: {ADMIN_PASSWORD})")
        else:
            print("[=] La coordinadora ya existía")

        if RepositorioPeliculas(sesion).listar():
            print("[=] Ya hay películas en el catálogo, no agrego ejemplos")
        else:
            for datos in PELICULAS_EJEMPLO:
                sesion.add(Pelicula(**datos))
            sesion.commit()
            print(f"[+] {len(PELICULAS_EJEMPLO)} películas de ejemplo (sin carátula: súbelas desde el panel)")


if __name__ == "__main__":
    main()
```

## Paso 3 — Repaso de `requirements.txt`

Ábrelo: debe contener (entre librerías relacionadas que pip agrega):

```
bcrypt
email-validator
fastapi
jinja2
PyJWT
python-multipart
sqlalchemy
uvicorn[standard]
```

Si falta algo, las dependencias por guía fueron: `fastapi` y `uvicorn[standard]` (guía 1), `sqlalchemy` (guía 2), `bcrypt` y `PyJWT` (guía 4), `email-validator` (guía 5), `jinja2` y `python-multipart` (guía 6). El `psycopg2-binary` (driver de Postgres) se agregará en la guía de despliegue, cuando exista una base real a la que conectarse.

---

## Paso 4 — El mapa completo de lo construido

```
cartelera/
├── venv/                  ← entorno (fuera del repo: .gitignore)
├── uploads/               ← carátulas subidas (fuera del repo: .gitignore)
├── cartelera.db           ← base de desarrollo (fuera del repo: .gitignore)
├── requirements.txt
├── run.py                 ← arranque en desarrollo
├── semilla.py             ← datos iniciales reproducibles
├── .gitignore
└── app/
    ├── main.py            ← composición: routers + carpetas servidas
    ├── config.py          ← TODA la configuración, variables de entorno
    ├── database.py        ← motor, sesión, crear_tablas (ADR-002)
    ├── dependencias.py    ← ¿quién llama? ¿puede? (ADR-003)
    ├── modelos/           ← usuarios, peliculas, calificaciones (guía 2)
    ├── repositorios/      ← los almacenes D1–D3 del DFD (guía 3)
    ├── esquemas/          ← frontera Pydantic = schemas del contrato (guía 5)
    ├── servicios/         ← reglas RN + DIP de almacenamiento (guía 4)
    ├── rutas/             ← web.py, api.py, admin.py (guías 5–7)
    ├── plantillas/        ├── base, index, detalle, login, registro, admin/panel
    └── static/            ← estilos.css, estrellas.js, img/placeholder.svg
```

---

## ✅ La gran verificación final

**Desde cero, como lo hará quien clone el repo:**

```powershell
del cartelera.db
python semilla.py
python run.py
```

Y recorre la lista de cierre, marcando:

| # | Verificación | Origen |
|---|---|---|
| 1 | `/salud` responde `{"estado": "ok"}` | Guía 1 |
| 2 | El catálogo muestra las 4 películas con placeholder y "Sin calificaciones" | Guías 2, 6 |
| 3 | Login como `coordinadora@cineclub.cl` / `admin1234` → barra con "Panel admin" | Guías 4, 6, 8 |
| 4 | `/admin` publica una película con carátula; aparece en el catálogo con su imagen | Guía 7 |
| 5 | Link de Vimeo rechazado; imagen GIF rechazada | RN-03, RN-05 |
| 6 | Crear una cuenta de socia (con error de contraseña corta y de email duplicado en el camino) | HU-03 |
| 7 | Calificar 5★ desde la ficha; recalificar a 2★ → total sigue en 1 | RN-01, guía 6 |
| 8 | Eliminar una película con confirmación → desaparece con sus votos | HU-07 |
| 9 | En `/docs`: registro → sesión → Authorize → `/api/yo` → calificar | Guía 5 |
| 10 | **Comparación contrato ↔ `/docs`**: las 7 operaciones coinciden en rutas, parámetros, códigos y esquemas con `contrato_api.yaml` | ADR-008 |

La fila 10 es la evidencia formal de que la fase cumple su contrato (guía de desarrollo, §4): sin desvíos, la fase 5 está cerrada.

**Sugerencia de commit para cerrar la fase:**

```bash
git add -A
git commit -m "Fase 5 completa: implementación de Cartelera según guías 1-8

Cumple el contrato OpenAPI (verificación /docs vs contrato_api.yaml sin
desvios) y respeta los 8 ADRs de la fase 4."
```

---

## 📝 Punto de control (el del cierre)

1. Corre `python semilla.py` **dos veces**. ¿Qué pasa y qué habría pasado sin las guardas?
2. Repasa el mapa de carpetas: nombra una regla de negocio y señala en qué archivo vive. Nombra una consulta SQL y señala la suya.
3. Si mañana el cliente pidiera "que las estrellas se muestren con medio punto (3,5)", ¿qué documentos cambian primero: el código o los de las fases 1–4? ¿Y por qué ese orden no es burocracia?

## Lo que acabas de aprender

- Datos iniciales idempotentes y configurables por entorno
- El mismo script sirve para desarrollo y producción (una variable de entorno de diferencia)
- A cerrar una fase verificando contra el contrato, no contra la memoria

**Fin de la fase 5.** El ciclo continúa: la fase 6 (pruebas) formalizará en `pytest` lo que hoy verificamos a mano, y la fase 7 (despliegue) llevará esto a Render + Neon con URL pública. El proyecto ya es real: solo falta que el mundo lo vea.
