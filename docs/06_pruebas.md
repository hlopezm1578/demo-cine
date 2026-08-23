# Fase 6 — Guía de Pruebas: automatizar lo que verificábamos a mano

> **Módulos:** ISI601 / ISI602 · **Fase del ciclo de vida:** 6. Pruebas
> **Qué construirás hoy:** una suite de pruebas automáticas que ejecuta el flujo completo del sistema (registro → login → calificar → promedio) y las reglas de negocio, sin navegador y en segundos.
> **Al terminar tendrás:** `pruebas/test_api.py` con 17 pruebas que corren con un solo comando y te dicen en una línea si el sistema está sano.
> **Necesitas:** las guías 1 a 8 terminadas (la aplicación funcionando).
> **Fecha:** 2026-08-23

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Prueba automatizada** | Un programa que usa tu programa y compara lo que obtiene con lo que esperaba: verde pasa, rojo grita |
| **Prueba unitaria** | Prueba una función aislada (ej.: `normalizar_trailer` por sí sola) |
| **Prueba de integración** | Prueba varias capas juntas (ruta → servicio → base de datos) |
| **TestClient** | Un "navegador de mentira" que FastAPI ofrece para probar rutas sin levantar servidor |
| **Cobertura** | Qué porcentaje de tu código es ejecutado por las pruebas (aquí: aspiración, no obsesión) |

---

## Paso 1 — Qué probamos y qué no (la decisión más importante)

🧠 **El desarrollador piensa:** *no puedo probar todo, así que elijo con criterio: **se prueban las reglas**. Cada RN del documento de requerimientos merece al menos una prueba que la defienda — porque una regla que nadie defiende es una regla que se rompe en silencio en el segundo refactor. Lo que NO pruebo: el CSS se ve bonito, el iframe de YouTube carga —eso requiere un navegador real y pruebas de extremo a extremo (Playwright, Selenium) que están fuera del alcance de este curso. Decir qué no se prueba y por qué es parte del oficio.*

| Regla / flujo | Origen | La defensa |
|---|---|---|
| Tráiler solo YouTube, 3 formatos | RN-03 | Pruebas de `normalizar_trailer` |
| Una calificación por socio | RN-01 | Calificar + recalificar: el total NO sube |
| Estrellas 1–5 | RN-02 | 7 estrellas → 422 |
| Solo la coordinadora publica | RN-04 | Panel sin sesión → 401; socia → 403 |
| Carátula JPG/PNG/WebP | RN-05 | GIF → 400 |
| Registro y login | RF-01, RF-02 | 201 / 409 duplicado / 401 clave mala |
| El hash jamás sale | RNF-01 | La respuesta no contiene `password_hash` |
| La web funciona | HU-01…HU-05 | La home y la ficha renderizan; el login web setea cookie |

## Paso 2 — Instalar y aislar

```powershell
pip install pytest httpx
pip freeze > requirements.txt
```

🧠 **El desarrollador piensa:** *regla de oro de las pruebas: **nunca pruegas contra los datos de verdad**. Si pruebo contra `cartelera.db`, cada corrida registra usuarios falsos en mi catálogo. Solución: una base de pruebas aparte (`bd_prueba.db`), configurada como variable de entorno **antes de importar la aplicación** — porque `config.py` lee el entorno al importarse. Es el mismo mecanismo que separa desarrollo de producción (ADR-002), ahora separando "real" de "prueba". Y `httpx` no es para probar otra API: es la librería que el TestClient de FastAPI necesita por debajo.*

Crea la carpeta `pruebas/` y el archivo **`pruebas/test_api.py`**:

```python
"""Pruebas de Cartelera: el flujo completo y las reglas de negocio.

Correr desde la raíz del proyecto:
    python -m pytest pruebas -v
"""
import base64
import os
from pathlib import Path

import pytest
from sqlalchemy import select

# Base de datos de PRUEBAS aislada (un archivo propio que se borra al empezar).
# Debe configurarse ANTES de importar la aplicación: config.py lee el entorno
# en el momento de importarse.
ARCHIVO_BD = Path(__file__).with_name("bd_prueba.db")
if ARCHIVO_BD.exists():
    ARCHIVO_BD.unlink()
os.environ["DATABASE_URL"] = f"sqlite:///{ARCHIVO_BD.as_posix()}"

from fastapi.testclient import TestClient  # noqa: E402

from app.database import SesionLocal  # noqa: E402
from app.main import app  # noqa: E402
from app.modelos import Pelicula, Usuario  # noqa: E402
from app.servicios import autenticacion as auth  # noqa: E402
from app.servicios.peliculas import normalizar_trailer  # noqa: E402

cliente = TestClient(app)

# PNG de 1x1 píxel válido, para probar la subida de carátulas
PNG_MINIMO = base64.b64decode(
    "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg=="
)

CLAVES = {
    "coordinadora": ("coordinadora@test.cl", "clave-segura-123"),
    "fer": ("fer@test.cl", "clave-segura-123"),
}


def sembrar_datos() -> None:
    """Crea una película y la coordinadora directo en la base (sin HTTP)."""
    with SesionLocal() as sesion:
        sesion.add(Pelicula(
            titulo="La Prueba", anio=2020, sinopsis="Película sembrada",
            url_trailer="https://www.youtube.com/embed/xyz"))
        sesion.add(Usuario(
            nombre="Coordinadora Test", email=CLAVES["coordinadora"][0],
            password_hash=auth.hashear(CLAVES["coordinadora"][1]), rol="admin"))
        sesion.commit()


def obtener_token(email: str, contrasena: str) -> str:
    """Inicia sesión por la API y devuelve el token Bearer."""
    respuesta = cliente.post("/api/sesion", json={"email": email, "contrasena": contrasena})
    assert respuesta.status_code == 200, respuesta.text
    return respuesta.json()["token"]


sembrar_datos()


# ---------- RN-03: normalización del tráiler (unitarias) ----------

def test_normalizar_trailer_acepta_los_tres_formatos():
    assert normalizar_trailer("https://www.youtube.com/watch?v=abc123") == \
        "https://www.youtube.com/embed/abc123"
    assert normalizar_trailer("https://youtu.be/abc123") == \
        "https://www.youtube.com/embed/abc123"
    assert normalizar_trailer("https://www.youtube.com/shorts/abc123") == \
        "https://www.youtube.com/embed/abc123"


def test_normalizar_trailer_rechaza_otras_plataformas():
    with pytest.raises(ValueError):
        normalizar_trailer("https://vimeo.com/12345")


# ---------- Cuentas (RF-01, RF-02, RNF-01) ----------

def test_registrar_usuario_devuelve_201_y_nunca_el_hash():
    respuesta = cliente.post(
        "/api/usuarios",
        json={"nombre": "Fernanda", "email": CLAVES["fer"][0],
              "contrasena": CLAVES["fer"][1]},
    )
    assert respuesta.status_code == 201
    cuerpo = respuesta.json()
    assert cuerpo["email"] == CLAVES["fer"][0]
    assert "password_hash" not in cuerpo  # el hash jamás sale del servidor


def test_email_duplicado_da_409():
    respuesta = cliente.post(
        "/api/usuarios",
        json={"nombre": "Otra Fer", "email": CLAVES["fer"][0],
              "contrasena": "otra-clave-123"},
    )
    assert respuesta.status_code == 409


def test_login_con_clave_mala_da_401():
    respuesta = cliente.post(
        "/api/sesion", json={"email": CLAVES["fer"][0], "contrasena": "incorrecta!"}
    )
    assert respuesta.status_code == 401


# ---------- Calificaciones (RF-08, RF-09, RN-01, RN-02) ----------

def test_no_se_puede_calificar_sin_sesion():
    respuesta = cliente.post("/api/calificaciones",
                             json={"pelicula_id": 1, "estrellas": 5})
    assert respuesta.status_code == 401


def test_calificar_y_recalificar():
    token = obtener_token(*CLAVES["fer"])
    cabeceras = {"Authorization": f"Bearer {token}"}

    primera = cliente.post(
        "/api/calificaciones", json={"pelicula_id": 1, "estrellas": 5},
        headers=cabeceras,
    )
    assert primera.status_code == 200
    assert primera.json() == {
        "promedio": 5.0, "total_calificaciones": 1, "mi_calificacion": 5,
    }

    # RN-01: recalificar REEMPLAZA el voto, no agrega otro
    segunda = cliente.post(
        "/api/calificaciones", json={"pelicula_id": 1, "estrellas": 1},
        headers=cabeceras,
    )
    assert segunda.json()["total_calificaciones"] == 1
    assert segunda.json()["promedio"] == 1.0


def test_estrellas_fuera_de_rango_son_rechazadas():
    token = obtener_token(*CLAVES["fer"])
    respuesta = cliente.post(
        "/api/calificaciones", json={"pelicula_id": 1, "estrellas": 7},
        headers={"Authorization": f"Bearer {token}"},
    )
    assert respuesta.status_code == 422  # validación de Pydantic (RN-02)


# ---------- Autorización (RN-04) ----------

def test_sin_sesion_el_panel_admin_se_rechaza():
    assert cliente.post("/admin/peliculas").status_code == 401


def test_usuario_comun_no_puede_crear_peliculas():
    token = obtener_token(*CLAVES["fer"])
    respuesta = cliente.post(
        "/admin/peliculas", headers={"Authorization": f"Bearer {token}"},
        follow_redirects=False,
    )
    assert respuesta.status_code == 403


# ---------- Panel admin: carátula (RN-05) y tráiler (RN-03) ----------

def test_admin_crea_pelicula_con_caratula():
    token = obtener_token(*CLAVES["coordinadora"])
    respuesta = cliente.post(
        "/admin/peliculas",
        headers={"Authorization": f"Bearer {token}"},
        data={"titulo": "Con carátula", "anio": "2021",
              "sinopsis": "Subida por la prueba",
              "url_trailer": "https://youtu.be/g4Hbz2jLxvQ"},  # formato corto
        files={"caratula": ("tapa.png", PNG_MINIMO, "image/png")},
        follow_redirects=False,
    )
    assert respuesta.status_code == 303

    with SesionLocal() as sesion:
        pelicula = sesion.scalars(
            select(Pelicula).where(Pelicula.titulo == "Con carátula")
        ).first()
        assert pelicula is not None
        assert pelicula.ruta_caratula is not None          # el archivo quedó registrado
        assert pelicula.url_trailer == "https://www.youtube.com/embed/g4Hbz2jLxvQ"


def test_caratula_con_formato_prohibido_da_400():
    token = obtener_token(*CLAVES["coordinadora"])
    respuesta = cliente.post(
        "/admin/peliculas",
        headers={"Authorization": f"Bearer {token}"},
        data={"titulo": "Mala", "anio": "2021", "sinopsis": "",
              "url_trailer": "https://www.youtube.com/watch?v=ok"},
        files={"caratula": ("malo.gif", b"GIF89a", "image/gif")},
    )
    assert respuesta.status_code == 400


def test_trailer_que_no_es_youtube_da_400():
    token = obtener_token(*CLAVES["coordinadora"])
    respuesta = cliente.post(
        "/admin/peliculas",
        headers={"Authorization": f"Bearer {token}"},
        data={"titulo": "Mala", "anio": "2021", "sinopsis": "",
              "url_trailer": "https://vimeo.com/1"},
    )
    assert respuesta.status_code == 400


# ---------- Catálogo y web HTML (HU-01…HU-05) ----------

def test_el_catalogo_api_muestra_las_peliculas():
    respuesta = cliente.get("/api/peliculas")
    assert respuesta.status_code == 200
    titulos = [p["titulo"] for p in respuesta.json()]
    assert "La Prueba" in titulos


def test_la_home_html_muestra_el_catalogo():
    pagina = cliente.get("/")
    assert pagina.status_code == 200
    assert "La Prueba" in pagina.text


def test_la_ficha_html_existe():
    pagina = cliente.get("/pelicula/1")
    assert pagina.status_code == 200
    assert "youtube.com/embed/xyz" in pagina.text  # tráiler embebido (RN-03)


def test_flujo_web_con_cookie_de_sesion():
    """El formulario web usa cookie (no Bearer): TestClient la guarda solo."""
    respuesta = cliente.post(
        "/login",
        data={"email": CLAVES["fer"][0], "contrasena": CLAVES["fer"][1]},
        follow_redirects=False,
    )
    assert respuesta.status_code == 303

    inicio = cliente.get("/")
    assert "Fernanda" in inicio.text  # la barra saluda a la socia logueada
```

🧠 **El desarrollador piensa (dos trucos que parecen magia):** *primero, el orden importa: `sembrar_datos()` corre al importar, así la película con id 1 existe antes que cualquier prueba — las pruebas del archivo corren de arriba hacia abajo y comparten la base. Segundo, `test_flujo_web_con_cookie_de_sesion`: el TestClient **guarda cookies** como un navegador; por eso el login por formulario deja logueadas las peticiones siguientes. Es literalmente HU-04 en dos asserts.*

## Paso 3 — El ritual: integrar pruebas al flujo de trabajo

🧠 **El desarrollador piensa:** *una suite que solo corre cuando uno se acuerda es decoración. El hábito profesional: **correrlas antes de cada commit** (y en el futuro, dejar que GitHub las corra solo — eso será "integración continua" de verdad, cuando el ramo llegue ahí).*

---

## ✅ Verificación de la guía

Desde la raíz del proyecto, con el servidor **detenido**:

```powershell
python -m pytest pruebas -v
```

Esperado: **17 passed** en verde, en segundos. Después prueba el valor real de la suite — **rompe algo a propósito**: abre `app/servicios/calificaciones.py` y cambia el upsert para que siempre cree una calificación nueva (elimina el bloque del `if existente`). Corre las pruebas de nuevo: `test_calificar_y_recalificar` debe ponerse **en rojo** — la suite acaba de detectar una violación de RN-01 que a simple vista nadie notaría. Arregla el código, vuelve al verde, y quédate con esa sensación: *eso que acabas de sentir es por lo que existen las pruebas.*

---

## 📝 Punto de control

1. ¿Por qué la base de pruebas se configura ANTES del `from app.main import app`? ¿Qué pasaría si lo haces después?
2. `test_registrar_usuario_devuelve_201_y_nunca_el_hash` defiende RNF-01 sin leer la base de datos. ¿Cómo?
3. Tu compañero dice "yo pruebo a mano, más rápido". ¿Qué le respondes basándote en el experimento de romper el upsert?

## Lo que acabas de aprender

- Pruebas unitarias vs de integración, y qué merece prueba (las reglas)
- Aislar la base de pruebas con variables de entorno
- TestClient: probar rutas y hasta la cookie de sesión sin navegador
- El ritual: verde antes de cada commit

**Siguiente:** `07_despliegue.md` — el gran final: publicar en Render + Neon con URL pública, gratis, sin tarjeta. Y la lección del disco efímero, en vivo.
