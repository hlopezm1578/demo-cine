# Guía 1 — El esqueleto: que la aplicación viva

> **Qué construirás hoy:** la estructura mínima para que exista una aplicación web llamada Cartelera que responde en tu computador.
> **Al terminar tendrás:** tu navegador mostrando `{"estado": "ok"}` y una documentación interactiva automática.
> **Necesitas:** Python 3.11 o superior instalado (`python --version` para comprobarlo).

---

## Los términos de hoy (antes de copiar nada)

| Término | Qué es, en una frase |
|---|---|
| **Servidor web** | Un programa que espera pedidos (de navegadores) y les responde |
| **Framework** | Un conjunto de piezas ya construidas para no escribir el servidor desde cero. Nosotros usaremos **FastAPI** |
| **Endpoint** | Una "dirección" del servidor que responde algo: `/salud` responderá `{"estado": "ok"}` |
| **Entorno virtual** | Una carpeta con su propia copia aislada de librerías, para que cada proyecto tenga lo suyo sin ensuciar el computador |
| **Puerto** | El "numero de departamento" del servidor en tu máquina. Nosotros usaremos el 8000 |

---

## Paso 1 — Crear el proyecto y su entorno virtual

🧠 **El desarrollador piensa:** *voy a crear una carpeta exclusiva del proyecto. Todo lo de Cartelera vive adentro: código, configuración y librerías. Si mañana lo borro, no queda rastro en el sistema. Y el entorno virtual me garantiza que "en mi computador funciona" también funcionará en el de mi compañero, porque las versiones exactas quedan anotadas en `requirements.txt`.*

Abre una terminal (PowerShell en Windows) y ejecuta, línea por línea:

```powershell
mkdir cartelera
cd cartelera
python -m venv venv
venv\Scripts\activate
```

Si usas **Git Bash** o **Linux/macOS**, la activación cambia:

```bash
python -m venv venv
source venv/Scripts/activate    # Git Bash
source venv/bin/activate        # Linux / macOS
```

> Si PowerShell reclama por "scripts deshabilitados", no luchues con él: usa directamente `venv\Scripts\python` en vez de `python` en cada comando que sigue.

Instala las dos primeras dependencias y déjalas registradas:

```powershell
pip install fastapi "uvicorn[standard]"
pip freeze > requirements.txt
```

Crea el archivo **`requirements.txt`** con ese último comando (ábrelo: debe listar `fastapi`, `uvicorn` y sus librerías amigas). Cualquier persona podrá reproducir tu entorno con `pip install -r requirements.txt`.

---

## Paso 2 — La carpeta `app`: el hogar del código

🧠 **El desarrollador piensa:** *todo el código vivirá en un paquete llamado `app`, organizado por capas como dice el documento de arquitectura (§4). Hoy solo creo la estructura mínima; las carpetas de `modelos/`, `servicios/` y las demás irán apareciendo en las próximas guías, **cuando tengan contenido**, porque una carpeta vacía no le enseña nada a nadie.*

Crea estas carpetas y archivos (a mano o con tu editor, por ejemplo VS Code):

```
cartelera/
├── venv/                ← ya existe (la creó el paso 1)
├── requirements.txt     ← ya existe
├── run.py               ← lo creamos en el paso 5
└── app/
    ├── __init__.py      ← marca app como "paquete" importable
    ├── config.py        ← lo creamos en el paso 3
    └── main.py          ← lo creamos en el paso 4
```

El archivo **`app/__init__.py`** puede quedar vacío; su sola existencia le dice a Python "esta carpeta se importa como un paquete":

```python
# app/__init__.py
# (vacío a propósito: su presencia es lo que importa)
```

---

## Paso 3 — `config.py`: TODO lo configurable, en un solo lugar

🧠 **El desarrollador piensa:** *esta es la primera decisión de arquitectura real del proyecto (RNF-03, ADR-007). Cada cosa que pueda variar entre "mi computador" y "el servidor de producción" —claves, direcciones de base de datos— vive en **variables de entorno**, con valores por defecto cómodos para desarrollo. Así el mismo código sirve para ambos mundos, y jamás habrá una contraseña pegada en un archivo que suba a GitHub. Concentro todo en `config.py`: una sola fuente de la verdad.*

Crea **`app/config.py`**:

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

# Firma de los tokens de sesión (ADR-003). La usaremos en la guía 4.
# En producción: variable de entorno con un valor largo y secreto.
SECRET_KEY = os.getenv("SECRET_KEY", "dev-secreto-cambiar-en-produccion")

# URL de la base de datos (ADR-002). La usaremos en la guía 2.
# Desarrollo: SQLite, un archivo, cero instalación. Producción: Postgres en Neon.
DATABASE_URL = os.getenv(
    "DATABASE_URL", f"sqlite:///{(BASE_DIR / 'cartelera.db').as_posix()}"
)
```

❌ **El error que este archivo evita** (conversa muy común en la industria):

```python
# ❌ NUNCA: el secreto queda pegado en el código, viaja a GitHub y es público
SECRET_KEY = "sk-prod-9f8e7d6c5b4a"

# ✅ SIEMPRE: el valor viene del entorno; el código solo lo lee
SECRET_KEY = os.getenv("SECRET_KEY", "valor-solo-para-desarrollo")
```

El día que este archivo se suba a GitHub, en la versión ❌ acabas de regalar la llave del sistema. En la ✅ no hay nada que regalar.

---

## Paso 4 — `main.py`: el punto de composición

🧠 **El desarrollador piensa:** *`main.py` no va a contener lógica de negocio en su vida (ADR-001): su trabajo es **componer** —decir qué existe y en qué orden—. Hoy compone muy poco: una aplicación FastAPI y un endpoint de salud. Las próximas guías irán agregando aquí las rutas, los archivos estáticos y la base de datos, siempre como líneas de composición, nunca como lógica. Y el endpoint `/salud` lo pongo desde el día 1: es la forma más barata de responder "¿está vivo el servidor?" — nos salvará en el despliegue.*

Crea **`app/main.py`**:

```python
"""Punto de composición: arma la aplicación y conecta las piezas (ADR-001).

Este archivo no tiene lógica de negocio: decide QUÉ existe y EN QUÉ orden.
"""
from fastapi import FastAPI

app = FastAPI(
    title="Cartelera",
    description=(
        "Catálogo de películas del CineClub Barrio con calificación por "
        "estrellas. La misma lógica alimenta la web y esta API."
    ),
    version="1.0.0",
)


@app.get("/salud")
def salud() -> dict:
    """Chequeo mínimo para saber que el servicio está vivo."""
    return {"estado": "ok"}
```

Dos cosas que acabas de escribir, explicadas:

- `@app.get("/salud")` es un **decorador**: le dice a FastAPI "cuando pidan GET /salud, ejecuta la función de abajo". En la guía 5 verás que el `/docs` de Swagger se arma solo con estos decoradores.
- `-> dict` es una **anotación de tipo**: documenta que la función devuelve un diccionario. FastAPI las usa para validar y documentar.

---

## Paso 5 — `run.py`: el botón de encendido

🧠 **El desarrollador piensa:** *podría pedirle a los alumnos que escriban `uvicorn app.main:app` cada vez, pero un `run.py` de dos líneas elimina el error de tipeo y documenta cómo se parte el proyecto. `reload=True` recarga el servidor cada vez que guardo un archivo: imprescindible para desarrollar; jamás se usa en producción, donde Render arranca el servidor por su cuenta.*

Crea **`run.py`** en la raíz del proyecto:

```python
"""Arranque en desarrollo: python run.py → http://127.0.0.1:8000"""
import uvicorn

if __name__ == "__main__":
    uvicorn.run("app.main:app", host="127.0.0.1", port=8000, reload=True)
```

---

## ✅ Verificación de la guía 1

Con el entorno virtual activo y parado en la raíz del proyecto:

```powershell
python run.py
```

Deberías ver algo como `Uvicorn running on http://127.0.0.1:8000`. Ahora, en tu navegador:

1. Abre **http://127.0.0.1:8000/salud** → debes ver `{"estado":"ok"}`
2. Abre **http://127.0.0.1:8000/docs** → debes ver la documentación interactiva de Swagger UI, ya mostrando tu endpoint `/salud`, con el nombre "Cartelera" y tu descripción

> 🤯 **Date ese segundo:** con ~30 líneas ya tienes un servidor web documentado. Esa documentación automática será protagonista de la guía 5, cuando la comparemos contra el contrato OpenAPI que diseñamos en la fase 4.

Para detener el servidor: `Ctrl+C` en la terminal.

---

## 📝 Punto de control (respóndelas sin mirar la guía)

1. ¿Por qué `SECRET_KEY` se lee del entorno y no se escribe directo en `config.py`?
2. ¿Qué contiene `requirements.txt` y para qué le sirve a tu compañero que quiere correr tu proyecto?
3. Según ADR-001, ¿qué tipo de código está *prohibido* en `main.py`?

## Lo que acabas de aprender

- Entorno virtual y por qué cada proyecto tiene el suyo
- Variables de entorno como lugar de los secretos (RNF-03)
- El patrón "punto de composición": `main.py` compone, no piensa
- Tu primer endpoint y la documentación automática de FastAPI

**Siguiente:** `guia-02-modelos-y-base-de-datos.md` — le damos memoria al sistema: las tablas de usuarios, películas y calificaciones, con la regla RN-01 instalada en la base de datos.
