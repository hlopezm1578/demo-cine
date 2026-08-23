# Guía 7 — Panel de administración y carátulas

> **Qué construirás hoy:** el panel de la coordinadora: publicar películas con carátula de verdad (subida de archivos) y eliminarlas. El cierre del circuito del catálogo.
> **Al terminar tendrás:** `/admin` protegido por rol, subida de imágenes validada (RN-05) y la carpeta `uploads/` servida al navegador.
> **Necesitas:** la guía 6 terminada y verificada.

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Multipart** | El formato con que HTTP viaja formularios que mezclan texto y archivos binarios |
| **Autorización** | Ya sabiéndote QUIÉN eres (autenticación), decidir SI PUEDES (aquí: rol admin, RN-04) |
| **Montar (mount)** | Registrar una carpeta para que el servidor entregue sus archivos tal cual |

---

## Paso 0 — Dependencia y la coordinadora

```powershell
pip install python-multipart
pip freeze > requirements.txt
```

🧠 **El desarrollador piensa:** *para probar el panel necesito a la coordinadora (rol admin), y su creación formal (la `semilla.py`) recién llega en la guía 8. Solución honesta: la creo a mano ahora con un mini-script desechable — hash de verdad incluido, que para eso ya tenemos el servicio — y en la guía 8 lo convierto en algo permanente y reproducible.*

Crea **`crear_coordinadora.py`** en la raíz (desechable):

```python
from app.database import SesionLocal, crear_tablas
from app.modelos import Usuario
from app.repositorios import RepositorioUsuarios
from app.servicios import autenticacion

crear_tablas()
with SesionLocal() as sesion:
    repo = RepositorioUsuarios(sesion)
    if repo.por_email("coordinadora@cineclub.cl") is None:
        repo.crear(Usuario(
            nombre="Macarena",
            email="coordinadora@cineclub.cl",
            password_hash=autenticacion.hashear("admin1234"),
            rol="admin",
        ))
        print("Coordinadora creada: coordinadora@cineclub.cl / admin1234")
```

```powershell
python crear_coordinadora.py
```

## Paso 1 — `rutas/admin.py`: todo el router, blindado de una vez

🧠 **El desarrollador piensa (la decisión del día):** *puedo poner `Depends(requiere_admin)` ruta por ruta… y olvidarme de una. Mejor: la protección va en el **router entero** — una línea que cubre presente y futuro del panel. Sin sesión → 401; con sesión de socia → 403. Es la regla RN-04 hecha infraestructura. (Sí, la socia que toque `/admin` a mano verá un JSON de error y no una linda página: defecto aceptado y documentado; arreglarlo sería cosmética, no seguridad.)*

Crea **`app/rutas/admin.py`**:

```python
"""Panel de administración: publica y elimina películas (RN-04).

El router ENTERO queda protegido por requiere_admin: sin sesión → 401;
con sesión de socia → 403.
"""
from pathlib import Path

from fastapi import APIRouter, Depends, File, Form, HTTPException, Request, UploadFile
from fastapi.responses import RedirectResponse
from fastapi.templating import Jinja2Templates
from sqlalchemy.orm import Session

from app.database import obtener_sesion
from app.dependencias import requiere_admin, usuario_obligatorio
from app.modelos import Usuario
from app.repositorios import RepositorioPeliculas
from app.rutas.web import _extras_vista
from app.servicios import peliculas as servicio_peliculas

plantillas = Jinja2Templates(
    directory=str(Path(__file__).resolve().parent.parent / "plantillas")
)

router = APIRouter(
    prefix="/admin", tags=["Administración"], dependencies=[Depends(requiere_admin)]
)


@router.get("")
@router.get("/")
def panel(
    request: Request,
    usuario: Usuario = Depends(usuario_obligatorio),
    sesion: Session = Depends(obtener_sesion),
):
    repo = RepositorioPeliculas(sesion)
    promedios = repo.promedios()
    peliculas = []
    for pelicula in repo.listar():
        datos = servicio_peliculas.a_resumen(pelicula, *promedios.get(pelicula.id, (0.0, 0)))
        datos.update(_extras_vista(*promedios.get(pelicula.id, (0.0, 0))))
        peliculas.append(datos)
    return plantillas.TemplateResponse(
        request=request, name="admin/panel.html",
        context={"usuario": usuario, "peliculas": peliculas},
    )


@router.post("/peliculas")
def crear_pelicula(
    titulo: str = Form(...),
    anio: int = Form(...),
    sinopsis: str = Form(""),
    url_trailer: str = Form(...),
    caratula: UploadFile | None = File(None),
    usuario: Usuario = Depends(usuario_obligatorio),
    sesion: Session = Depends(obtener_sesion),
):
    # Las reglas (YouTube, formato, tamaño) las valida el servicio (RN-03, RN-05)
    servicio_peliculas.crear_pelicula(sesion, titulo, anio, sinopsis, url_trailer, caratula)
    return RedirectResponse("/admin", status_code=303)


@router.post("/peliculas/{pelicula_id}/eliminar")
def eliminar_pelicula(
    pelicula_id: int,
    usuario: Usuario = Depends(usuario_obligatorio),
    sesion: Session = Depends(obtener_sesion),
):
    pelicula = RepositorioPeliculas(sesion).por_id(pelicula_id)
    if pelicula is None:
        raise HTTPException(status_code=404, detail="Película no encontrada")
    servicio_peliculas.eliminar_pelicula(sesion, pelicula)
    return RedirectResponse("/admin", status_code=303)
```

Actualiza **`app/rutas/__init__.py`**:

```python
from app.rutas import admin, api, web

__all__ = ["admin", "api", "web"]
```

## Paso 2 — La plantilla del panel

Crea **`app/plantillas/admin/panel.html`**:

```html
{% extends "base.html" %}

{% block titulo %}Panel admin — Cartelera{% endblock %}

{% block contenido %}
<h1>Panel de administración</h1>
<p class="sub">Publica nuevas películas y retira las que ya no correspondan (RN-04: solo la coordinadora puede).</p>

<section class="seccion">
  <h2>Nueva película</h2>
  <form class="tarjeta-form" method="post" action="/admin/peliculas" enctype="multipart/form-data">
    <label>Título *
      <input type="text" name="titulo" required maxlength="120">
    </label>
    <div class="fila">
      <label>Año *
        <input type="number" name="anio" min="1900" max="2100" required>
      </label>
      <label>Link del tráiler (YouTube) *
        <input type="url" name="url_trailer" required placeholder="https://www.youtube.com/watch?v=…">
      </label>
    </div>
    <label>Sinopsis
      <textarea name="sinopsis" rows="4" maxlength="1000" placeholder="De qué se trata…"></textarea>
    </label>
    <label>Carátula (JPG, PNG o WebP — máximo 5 MB)
      <input type="file" name="caratula" accept=".jpg,.jpeg,.png,.webp">
    </label>
    <button type="submit">Publicar película</button>
  </form>
</section>

<section class="seccion">
  <h2>Películas publicadas ({{ peliculas|length }})</h2>
  <table class="tabla">
    <thead>
      <tr><th></th><th>Título</th><th>Año</th><th>Calificación</th><th></th></tr>
    </thead>
    <tbody>
      {% for p in peliculas %}
      <tr>
        <td><img class="mini" src="{{ p.imagen }}" alt=""></td>
        <td><a href="/pelicula/{{ p.id }}">{{ p.titulo }}</a></td>
        <td>{{ p.anio }}</td>
        <td>{{ p.texto }}</td>
        <td>
          <form method="post" action="/admin/peliculas/{{ p.id }}/eliminar"
                onsubmit="return confirm('¿Eliminar esta película y todas sus calificaciones?')">
            <button class="peligro" type="submit">Eliminar</button>
          </form>
        </td>
      </tr>
      {% else %}
      <tr><td colspan="5">Aún no hay películas publicadas.</td></tr>
      {% endfor %}
    </tbody>
  </table>
</section>
{% endblock %}
```

🧠 **Dos detalles de la plantilla:** *`enctype="multipart/form-data"` no es opcional: sin él, el navegador envía el nombre del archivo pero **no el archivo**. Y el botón Eliminar pide confirmación con `confirm(...)`: es una acción destructiva e irreversible (se lleva las calificaciones puestas), y un clic accidental no debe costarle el historial al cineclub.*

Agrega al final de **`app/static/estilos.css`** (estilos propios del panel):

```css
/* ---------- Panel admin ---------- */

.seccion {
  background: var(--panel); border: 1px solid var(--borde);
  border-radius: 12px; padding: 1.2rem 1.5rem; margin-bottom: 1.5rem;
}
.tarjeta-form { max-width: 640px; }
.fila { display: grid; grid-template-columns: 1fr 3fr; gap: 1rem; }

.tabla { width: 100%; border-collapse: collapse; }
.tabla th, .tabla td {
  text-align: left; padding: 0.5rem 0.6rem; border-bottom: 1px solid var(--borde);
}
.tabla .mini {
  width: 42px; height: 63px; object-fit: cover;
  border-radius: 4px; display: block;
}

.peligro {
  background: transparent; color: var(--peligro);
  border: 1px solid var(--peligro); border-radius: 8px;
  padding: 0.3rem 0.7rem; cursor: pointer;
}
.peligro:hover { background: var(--peligro); color: #fff; }

@media (max-width: 640px) {
  .fila { grid-template-columns: 1fr; }
}
```

## Paso 3 — Conectar y montar `uploads/`

Edita **`app/main.py`**:

```python
"""Punto de composición: arma la aplicación y conecta las piezas (ADR-001)."""
from pathlib import Path

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

from app.config import UPLOAD_DIR
from app.database import crear_tablas
from app.rutas import admin, api, web

app = FastAPI(
    title="Cartelera",
    description=(
        "Catálogo de películas del CineClub Barrio con calificación por "
        "estrellas. La misma lógica alimenta la web y esta API."
    ),
    version="1.0.0",
)

crear_tablas()

# El disco de carátulas debe existir antes de montarlo (ADR-004)
Path(UPLOAD_DIR).mkdir(parents=True, exist_ok=True)

# Carpetas servidas tal cual al navegador (ADR-006)
app.mount("/static", StaticFiles(directory=Path(__file__).parent / "static"), name="static")
app.mount("/uploads", StaticFiles(directory=UPLOAD_DIR), name="uploads")

app.include_router(web.router)    # páginas HTML
app.include_router(api.router)    # API JSON (/docs)
app.include_router(admin.router)  # panel de la coordinadora


@app.get("/salud")
def salud() -> dict:
    """Chequeo mínimo para saber que el servicio está vivo."""
    return {"estado": "ok"}
```

---

## ✅ Verificación de la guía 7

Con `python run.py` y una película de la guía 6 (o el catálogo vacío, también sirve):

1. **Sin sesión:** abre **/admin** → error 401 en JSON. **Como socia:** entra con una cuenta normal → `/admin` → **403** (RN-04 cerrado por ambos lados).
2. **Como coordinadora:** inicia sesión con `coordinadora@cineclub.cl` / `admin1234`. La barra ahora muestra **Panel admin**. Entra: formulario + tabla con el conteo de películas.
3. **Publica con carátula:** llena el formulario con una película real (por ejemplo *Interestelar*, 2014, un link de YouTube de `watch?v=`) y **elige una imagen cualquiera JPG/PNG de tu computador**. Publicar → la tabla la lista, el catálogo la muestra… **con tu carátula**. Mira dentro de la carpeta `uploads/`: el archivo está, con nombre aleatorio.
4. **Las reglas en acción (RN-03 y RN-05):** intenta publicar con un link de Vimeo → rechazado con el mensaje del servicio. Intenta con una imagen `.gif` → rechazada por formato.
5. **HU-07:** elimina la película de prueba (confirma el diálogo) → desaparece del catálogo, de la tabla, y su archivo sale de `uploads/`.
6. **Cronómetro (CS1):** publica una película completa midiendo el tiempo. ¿Menos de 5 minutos? El cliente firma contento.

Detén el servidor. La base con la coordinadora creada déjala: la guía 8 la reconstruirá de forma reproducible.

---

## 📝 Punto de control

1. ¿Dónde exactamente se decidió que TODO `/admin` exige rol admin? ¿Cuántas líneas habría que tocar para agregar un tercer rol con acceso?
2. ¿Por qué el nombre del archivo guardado es aleatorio y no "caratula.jpg"?
3. La socia que visita `/admin` ve un JSON feo, no una página bonita de error. ¿Es un problema de seguridad o de estética? ¿Cómo lo arreglarías sin debilitar la regla?

## Lo que acabas de aprender

- Subida de archivos multipart con validación de formato y tamaño
- Protección de un router completo con una dependencia
- Montar carpetas estáticas y de uploads
- Eliminación en cascada visible en la interfaz

**Siguiente:** `guia-08-datos-iniciales.md` — la semilla reproducible, el repaso general y la gran verificación final contra el contrato.
