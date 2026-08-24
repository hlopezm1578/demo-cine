# Guía 6 — La web: pantallas y estrellas

> **Qué construirás hoy:** las 4 pantallas del diseño (§4): catálogo, ficha, registro/login, y el inicio de sesión con cookie. Más el único JavaScript del sitio: el widget de estrellas.
> **Al terminar tendrás:** el sitio navegable en el navegador, con registro, login y calificación funcionando de punta a punta.
> **Necesitas:** la guía 5 terminada y verificada.

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **SSR / plantilla** | El servidor arma el HTML final con los datos y lo manda listo (ADR-006) |
| **Jinja2** | El motor de plantillas: `{{ variable }}` imprime valores, `{% if %}` decide estructura |
| **Cookie de sesión** | La "entréena" que el navegador guarda y reenvía en cada petición; la nuestra es HttpOnly (ADR-003) |
| **Estáticos** | Archivos que se sirven tal cual: CSS, JS, imágenes (no pasan por plantillas) |

---

## Paso 0 — Dependencias y carpetas

```powershell
pip install jinja2 python-multipart
pip freeze > requirements.txt
```

🧠 **El desarrollador piensa:** *`jinja2` es el motor de plantillas. Y `python-multipart` es la sorpresa del día: uno cree que solo sirve para subir archivos, pero **todo formulario HTML** —aunque solo tenga campos de texto, como nuestro login— viaja como *form data*, y FastAPI le delega el parseo a esa librería. Sin ella, la aplicación **ni siquiera parte**: revienta al arrancar en cuanto se registra la primera ruta con `Form(...)`. (Pregunta de clase: ¿por qué el error aparece al arrancar y no al enviar el formulario? Pista: FastAPI analiza las firmas de las rutas al registrarlas.)*

Crea esta estructura:

```
app/
├── rutas/web.py
├── plantillas/
│   ├── base.html
│   ├── index.html
│   ├── detalle.html
│   ├── login.html
│   └── registro.html
└── static/
    ├── estilos.css
    ├── estrellas.js
    └── img/placeholder.svg
```

## Paso 1 — `base.html`: el molde común

🧠 **El desarrollador piensa:** *todas las páginas comparten barra superior, pie y estructura. Eso va en UN molde (`base.html`) y cada página solo declara su bloque de contenido — si mañana cambio la barra, la cambio una vez. La barra además se adapta a quién mira: anónimo ve "Iniciar sesión", la socia ve su nombre, la coordinadora además ve el link al panel (que construiremos en la guía 7).*

**`app/plantillas/base.html`**:

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{% block titulo %}Cartelera{% endblock %}</title>
  <link rel="stylesheet" href="/static/estilos.css">
</head>
<body>
<header class="barra">
  <a class="logo" href="/">🎬 Cartelera</a>
  <nav>
    {% if usuario %}
      {% if usuario.rol == 'admin' %}<a href="/admin">Panel admin</a>{% endif %}
      <span class="usuario">Hola, {{ usuario.nombre }}</span>
      <a href="/salir">Salir</a>
    {% else %}
      <a href="/login">Iniciar sesión</a>
      <a class="boton" href="/registro">Crear cuenta</a>
    {% endif %}
  </nav>
</header>

<main>
{% block contenido %}{% endblock %}
</main>

<footer>
  CineClub Barrio — demo educativa ·
  <a href="/docs" target="_blank">API en /docs</a>
</footer>
</body>
</html>
```

## Paso 2 — `rutas/web.py`: las páginas

🧠 **El desarrollador piensa:** *decisión clave: el formulario web valida con **los mismos esquemas Pydantic de la API** (guía 5). Construyo `UsuarioCrear(...)` a mano con lo que llegó del formulario; si Pydantic rechaza, muestro el error en español sobre el formulario **conservando lo escrito** (no castigar al usuario). El login, en cambio, no necesita validación fina: el servicio ya responde con el mensaje genérico correcto. Y al entrar bien, seteo la cookie **HttpOnly**: el JavaScript del sitio jamás podrá leerla (ADR-003) — el widget de estrellas enviará la petición y el navegador adjuntará la cookie solo.*

Otro detalle: `_extras_vista` formatea el promedio **al estilo chileno** (coma decimal) y calcula cuántas estrellas van llenas — la plantilla solo pinta.

Crea **`app/rutas/web.py`**:

```python
"""Rutas web: las páginas HTML del sitio (ADR-001 capa de rutas, ADR-006 SSR)."""
from pathlib import Path

from fastapi import APIRouter, Depends, Form, HTTPException, Request
from fastapi.responses import RedirectResponse
from fastapi.templating import Jinja2Templates
from pydantic import ValidationError
from sqlalchemy.orm import Session

from app.config import DIAS_TOKEN, NOMBRE_COOKIE
from app.database import obtener_sesion
from app.dependencias import obtener_usuario_actual
from app.esquemas import UsuarioCrear
from app.modelos import Usuario
from app.repositorios import RepositorioPeliculas
from app.servicios import autenticacion
from app.servicios import calificaciones as servicio_calificaciones
from app.servicios import peliculas as servicio_peliculas

plantillas = Jinja2Templates(
    directory=str(Path(__file__).resolve().parent.parent / "plantillas")
)

router = APIRouter(tags=["Web"])

# Mensajes en español para los errores que Pydantic reporta en inglés
MENSAJES_CAMPO = {
    "nombre": "El nombre debe tener al menos 2 caracteres.",
    "email": "El email no tiene un formato válido.",
    "contrasena": "La contraseña debe tener al menos 8 caracteres.",
}


def _extras_vista(promedio: float, total: int) -> dict:
    """Presentación del bloque de estrellas (formato chileno: coma decimal)."""
    if total > 0:
        texto_promedio = f"{promedio:.1f}".replace(".", ",")
        return {
            "llenas": round(promedio),
            "texto": f"{texto_promedio} · {total} calificación{'es' if total != 1 else ''}",
        }
    return {"llenas": 0, "texto": "Sin calificaciones"}


# ---------- Catálogo público ----------

@router.get("/")
def inicio(
    request: Request,
    usuario: Usuario | None = Depends(obtener_usuario_actual),
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
        request=request, name="index.html",
        context={"usuario": usuario, "peliculas": peliculas},
    )


@router.get("/pelicula/{pelicula_id}")
def ficha(
    pelicula_id: int,
    request: Request,
    usuario: Usuario | None = Depends(obtener_usuario_actual),
    sesion: Session = Depends(obtener_sesion),
):
    repo = RepositorioPeliculas(sesion)
    pelicula = repo.por_id(pelicula_id)
    if pelicula is None:
        raise HTTPException(status_code=404, detail="Película no encontrada")

    promedio, total = repo.promedio(pelicula_id)
    datos = servicio_peliculas.a_detalle(pelicula, promedio, total)
    datos.update(_extras_vista(promedio, total))

    return plantillas.TemplateResponse(
        request=request,
        name="detalle.html",
        context={
            "usuario": usuario,
            "pelicula": datos,
            "mi_calificacion": servicio_calificaciones.mi_calificacion(sesion, usuario, pelicula_id),
        },
    )


# ---------- Cuentas ----------

@router.get("/registro")
def formulario_registro(request: Request):
    return plantillas.TemplateResponse(
        request=request, name="registro.html",
        context={"usuario": None, "error": None, "valores": {}},
    )


@router.post("/registro")
def registrar(
    request: Request,
    nombre: str = Form(...),
    email: str = Form(...),
    contrasena: str = Form(...),
    sesion: Session = Depends(obtener_sesion),
):
    valores = {"nombre": nombre, "email": email}
    try:
        # El MISMO esquema de la API valida el formulario web: una sola
        # definición de "cómo se ve un registro correcto" (ADR-001 en acción)
        UsuarioCrear(nombre=nombre.strip(), email=email.strip().lower(), contrasena=contrasena)
    except ValidationError as error:
        campo = str(error.errors()[0]["loc"][-1])
        return plantillas.TemplateResponse(
            request=request, name="registro.html",
            context={"usuario": None,
                     "error": MENSAJES_CAMPO.get(campo, "Datos inválidos."),
                     "valores": valores},
        )

    try:
        autenticacion.registrar(sesion, nombre, email, contrasena)
    except ValueError as error:  # email duplicado u otra regla del servicio
        return plantillas.TemplateResponse(
            request=request, name="registro.html",
            context={"usuario": None, "error": str(error), "valores": valores},
        )

    return RedirectResponse("/login?registrado=1", status_code=303)


@router.get("/login")
def formulario_login(request: Request, registrado: int = 0):
    return plantillas.TemplateResponse(
        request=request, name="login.html",
        context={"usuario": None, "error": None, "registrado": bool(registrado)},
    )


@router.post("/login")
def iniciar_sesion(
    request: Request,
    email: str = Form(...),
    contrasena: str = Form(...),
    sesion: Session = Depends(obtener_sesion),
):
    try:
        token = autenticacion.entrar(sesion, email, contrasena)
    except ValueError as error:
        return plantillas.TemplateResponse(
            request=request, name="login.html",
            context={"usuario": None, "error": str(error), "registrado": False},
        )

    respuesta = RedirectResponse("/", status_code=303)
    # HttpOnly: el JavaScript de la página no puede leer la cookie (ADR-003)
    respuesta.set_cookie(
        NOMBRE_COOKIE, token,
        max_age=DIAS_TOKEN * 24 * 3600, httponly=True, samesite="lax",
    )
    return respuesta


@router.get("/salir")
def salir():
    respuesta = RedirectResponse("/", status_code=303)
    respuesta.delete_cookie(NOMBRE_COOKIE)
    return respuesta
```

## Paso 3 — Las plantillas de páginas

**`app/plantillas/index.html`** (catálogo):

```html
{% extends "base.html" %}

{% block contenido %}
<section class="encabezado">
  <h1>Cartelera</h1>
  <p>Califica con estrellas las películas que ya viste.</p>
</section>

<div class="grilla">
  {% for p in peliculas %}
    <a class="tarjeta" href="/pelicula/{{ p.id }}">
      <img src="{{ p.imagen }}" alt="Carátula de {{ p.titulo }}" loading="lazy">
      <div class="cuerpo">
        <h3>{{ p.titulo }} <span class="anio">({{ p.anio }})</span></h3>
        <div class="estrellas" aria-label="{{ p.texto }}">
          {%- for i in range(1, 6) -%}
          <span class="{{ 'llena' if i <= p.llenas else 'vacia' }}">★</span>
          {%- endfor -%}
        </div>
        <small>{{ p.texto }}</small>
      </div>
    </a>
  {% else %}
    <p class="vacio">Aún no hay películas. La coordinadora publicará la primera desde su panel.</p>
  {% endfor %}
</div>
{% endblock %}
```

**`app/plantillas/detalle.html`** (ficha + widget):

```html
{% extends "base.html" %}

{% block titulo %}{{ pelicula.titulo }} — Cartelera{% endblock %}

{% block contenido %}
<section class="ficha">
  <img class="caratula" src="{{ pelicula.imagen }}" alt="Carátula de {{ pelicula.titulo }}">

  <div class="datos">
    <h1>{{ pelicula.titulo }} <span class="anio">({{ pelicula.anio }})</span></h1>

    <div class="estrellas grandes" aria-label="{{ pelicula.texto }}">
      {%- for i in range(1, 6) -%}
      <span class="{{ 'llena' if i <= pelicula.llenas else 'vacia' }}">★</span>
      {%- endfor -%}
      <span class="detalle-promedio">{{ pelicula.texto }}</span>
    </div>

    <p class="sinopsis">{{ pelicula.sinopsis }}</p>

    {% if usuario %}
      <div class="calificar" data-pelicula="{{ pelicula.id }}">
        <span class="etiqueta">Tu calificación:</span>
        <span class="selector">
          {%- for i in range(1, 6) -%}
          <button type="button" class="estrella {{ 'activa' if i <= mi_calificacion }}"
                  data-valor="{{ i }}" title="{{ i }} de 5">★</button>
          {%- endfor -%}
        </span>
        <small>Haz clic en una estrella para calificar o cambiar tu voto.</small>
      </div>
    {% else %}
      <p class="aviso"><a href="/login">Inicia sesión</a> o
        <a href="/registro">crea una cuenta</a> para calificar esta película.</p>
    {% endif %}
  </div>
</section>

<section class="trailer">
  <h2>Tráiler</h2>
  <div class="marco">
    <iframe src="{{ pelicula.url_trailer }}" title="Tráiler de {{ pelicula.titulo }}"
            allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
            allowfullscreen></iframe>
  </div>
</section>

<script src="/static/estrellas.js"></script>
{% endblock %}
```

**`app/plantillas/login.html`**:

```html
{% extends "base.html" %}

{% block titulo %}Iniciar sesión — Cartelera{% endblock %}

{% block contenido %}
<section class="formulario">
  <h1>Iniciar sesión</h1>

  {% if registrado %}<p class="exito">Cuenta creada. Ya puedes iniciar sesión.</p>{% endif %}
  {% if error %}<p class="error">{{ error }}</p>{% endif %}

  <form method="post" action="/login">
    <label>Email
      <input type="email" name="email" required autofocus autocomplete="email">
    </label>
    <label>Contraseña
      <input type="password" name="contrasena" required autocomplete="current-password">
    </label>
    <button type="submit">Entrar</button>
  </form>

  <p class="pie">¿Sin cuenta? <a href="/registro">Créala aquí</a>, toma un minuto.</p>
</section>
{% endblock %}
```

**`app/plantillas/registro.html`**:

```html
{% extends "base.html" %}

{% block titulo %}Crear cuenta — Cartelera{% endblock %}

{% block contenido %}
<section class="formulario">
  <h1>Crear cuenta</h1>
  <p class="sub">Con una cuenta puedes calificar películas con estrellas.</p>

  {% if error %}<p class="error">{{ error }}</p>{% endif %}

  <form method="post" action="/registro">
    <label>Nombre
      <input type="text" name="nombre" required minlength="2" maxlength="80"
             value="{{ valores.get('nombre', '') }}" autofocus>
    </label>
    <label>Email
      <input type="email" name="email" required maxlength="120"
             value="{{ valores.get('email', '') }}" autocomplete="email">
    </label>
    <label>Contraseña (mínimo 8 caracteres)
      <input type="password" name="contrasena" required minlength="8" autocomplete="new-password">
    </label>
    <button type="submit">Crear cuenta</button>
  </form>

  <p class="pie">¿Ya tienes cuenta? <a href="/login">Inicia sesión</a>.</p>
</section>
{% endblock %}
```

## Paso 4 — Estáticos: CSS, imagen genérica y el widget

**`app/static/img/placeholder.svg`**:

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="300" height="450" viewBox="0 0 300 450">
  <rect width="300" height="450" fill="#1f2430"/>
  <rect x="1" y="1" width="298" height="448" fill="none" stroke="#2a3140" stroke-width="2"/>
  <text x="150" y="220" font-size="64" text-anchor="middle">🎬</text>
  <text x="150" y="275" font-size="16" fill="#9aa3b5" text-anchor="middle" font-family="sans-serif">Sin carátula</text>
</svg>
```

**`app/static/estilos.css`** (tema oscuro de cine — copia completo):

```css
/* Cartelera — tema oscuro de cine. CSS plano, sin frameworks (ADR-006). */

:root {
  --fondo: #0e1116;
  --panel: #171b23;
  --panel-2: #1f2430;
  --borde: #2a3140;
  --texto: #e8ebf1;
  --texto-suave: #9aa3b5;
  --acento: #f5c518;       /* amarillo de estrellas */
  --peligro: #e5484d;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: "Segoe UI", system-ui, -apple-system, sans-serif;
  background: var(--fondo);
  color: var(--texto);
  line-height: 1.5;
}

a { color: var(--acento); text-decoration: none; }
a:hover { text-decoration: underline; }

h1, h2, h3 { line-height: 1.2; }

/* ---------- Barra superior ---------- */

.barra {
  display: flex; align-items: center; justify-content: space-between;
  gap: 1rem; padding: 0.8rem 1.5rem;
  background: var(--panel); border-bottom: 1px solid var(--borde);
}
.logo { font-size: 1.25rem; font-weight: 700; color: var(--texto); }
.logo:hover { text-decoration: none; }
.barra nav { display: flex; align-items: center; gap: 1rem; flex-wrap: wrap; }
.barra .usuario { color: var(--texto-suave); }

.boton {
  background: var(--acento); color: #1a1a1a; font-weight: 600;
  padding: 0.4rem 0.9rem; border-radius: 8px;
}
.boton:hover { text-decoration: none; filter: brightness(1.1); }

/* ---------- Contenido ---------- */

main { max-width: 1080px; margin: 0 auto; padding: 1.5rem; }
.encabezado h1 { margin: 0.5rem 0 0; font-size: 2rem; }
.encabezado p { margin: 0.25rem 0 1.5rem; color: var(--texto-suave); }
.sub { color: var(--texto-suave); }

/* ---------- Grilla del catálogo ---------- */

.grilla {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 1.2rem;
}
.tarjeta {
  background: var(--panel); border: 1px solid var(--borde);
  border-radius: 12px; overflow: hidden; color: var(--texto);
  transition: transform 0.15s ease, border-color 0.15s ease;
}
.tarjeta:hover { transform: translateY(-3px); border-color: var(--acento); text-decoration: none; }
.tarjeta img { width: 100%; aspect-ratio: 2 / 3; object-fit: cover; display: block; }
.tarjeta .cuerpo { padding: 0.7rem 0.8rem 0.9rem; }
.tarjeta h3 { margin: 0 0 0.3rem; font-size: 0.95rem; }
.tarjeta .anio { color: var(--texto-suave); font-weight: 400; }
.tarjeta small { color: var(--texto-suave); }
.vacio { grid-column: 1 / -1; color: var(--texto-suave); padding: 2rem 0; }

/* ---------- Estrellas ---------- */

.estrellas { letter-spacing: 2px; font-size: 1rem; }
.estrellas .llena { color: var(--acento); }
.estrellas .vacia { color: var(--borde); }
.estrellas.grandes { font-size: 1.5rem; }
.detalle-promedio {
  color: var(--texto-suave); letter-spacing: normal;
  margin-left: 0.5rem; font-size: 0.9rem;
}

.calificar {
  margin-top: 1rem; padding: 0.9rem 1rem;
  background: var(--panel); border: 1px solid var(--borde); border-radius: 10px;
}
.calificar .etiqueta { display: block; color: var(--texto-suave); margin-bottom: 0.3rem; }
.calificar small { color: var(--texto-suave); display: block; margin-top: 0.3rem; }
.selector { display: inline-flex; gap: 2px; }

.estrella {
  background: none; border: none; cursor: pointer;
  font-size: 1.9rem; line-height: 1; color: var(--borde);
  padding: 0 2px; transition: color 0.1s ease, transform 0.1s ease;
}
.estrella:hover, .estrella.brillo { color: var(--acento); transform: scale(1.12); }
.estrella.activa { color: var(--acento); }

.aviso { margin-top: 1rem; padding: 0.8rem 1rem; background: var(--panel); border-radius: 10px; }

/* ---------- Ficha de película ---------- */

.ficha { display: flex; gap: 2rem; margin-bottom: 2rem; }
.ficha .caratula {
  width: 260px; aspect-ratio: 2 / 3; object-fit: cover;
  border-radius: 12px; border: 1px solid var(--borde);
}
.ficha .datos { flex: 1; }
.ficha .anio { color: var(--texto-suave); font-weight: 400; }
.sinopsis { max-width: 60ch; }

.marco {
  position: relative; width: 100%; max-width: 720px;
  aspect-ratio: 16 / 9; border-radius: 12px;
  overflow: hidden; border: 1px solid var(--borde);
}
.marco iframe { position: absolute; inset: 0; width: 100%; height: 100%; border: 0; }

/* ---------- Formularios ---------- */

.formulario {
  max-width: 420px; margin: 2rem auto;
  background: var(--panel); border: 1px solid var(--borde);
  border-radius: 12px; padding: 1.5rem 1.8rem;
}
.formulario label {
  display: block; margin-bottom: 0.9rem;
  color: var(--texto-suave); font-size: 0.9rem;
}
input, textarea {
  display: block; width: 100%; margin-top: 0.3rem;
  padding: 0.55rem 0.7rem;
  background: var(--panel-2); border: 1px solid var(--borde);
  border-radius: 8px; color: var(--texto); font: inherit;
}
input:focus, textarea:focus { outline: 2px solid var(--acento); border-color: transparent; }

button[type="submit"] {
  background: var(--acento); color: #1a1a1a; font-weight: 600;
  border: none; border-radius: 8px; padding: 0.6rem 1.1rem;
  cursor: pointer; font-size: 1rem;
}
button[type="submit"]:hover { filter: brightness(1.1); }

.exito { background: #14352a; border: 1px solid #1f6b4a; color: #7ee2b8;
         padding: 0.6rem 0.9rem; border-radius: 8px; }
.error { background: #3a1a1e; border: 1px solid #7a2e35; color: #ff9aa2;
         padding: 0.6rem 0.9rem; border-radius: 8px; }
.pie { color: var(--texto-suave); font-size: 0.9rem; }

/* ---------- Pie de página ---------- */

footer {
  margin-top: 3rem; padding: 1rem 1.5rem 2rem; text-align: center;
  color: var(--texto-suave); font-size: 0.85rem;
  border-top: 1px solid var(--borde);
}

/* ---------- Pantallas chicas (C4: celular primero) ---------- */

@media (max-width: 640px) {
  .ficha { flex-direction: column; }
  .ficha .caratula { width: 200px; }
}
```

**`app/static/estrellas.js`** — el único JavaScript del sitio:

```javascript
// Widget de estrellas: la única pieza interactiva del sitio (ADR-006).
// Llama a la API JSON con la cookie de sesión y recarga para reflejar
// el nuevo promedio.

document.querySelectorAll(".calificar").forEach(function (bloque) {
  var idPelicula = bloque.dataset.pelicula;
  var botones = Array.prototype.slice.call(bloque.querySelectorAll(".estrella"));

  botones.forEach(function (boton) {
    boton.addEventListener("click", function () {
      var estrellas = Number(boton.dataset.valor);
      fetch("/api/calificaciones", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        credentials: "same-origin", // envía la cookie de sesión
        body: JSON.stringify({ pelicula_id: Number(idPelicula), estrellas: estrellas })
      })
        .then(function (respuesta) {
          if (respuesta.ok) {
            window.location.reload();
          } else {
            alert("No se pudo guardar la calificación (¿se venció tu sesión?).");
          }
        })
        .catch(function () {
          alert("Error de conexión al calificar.");
        });
    });

    // Vista previa al pasar el mouse: se iluminan las estrellas hasta el cursor
    boton.addEventListener("mouseenter", function () {
      var valor = Number(boton.dataset.valor);
      botones.forEach(function (b) {
        b.classList.toggle("brillo", Number(b.dataset.valor) <= valor);
      });
    });
  });

  bloque.querySelector(".selector").addEventListener("mouseleave", function () {
    botones.forEach(function (b) { b.classList.remove("brillo"); });
  });
});
```

🧠 **El desarrollador piensa:** *¿por qué `window.location.reload()` en vez de actualizar el promedio con JavaScript? Porque el promedio lo calcula y lo formatea el **servidor** (una sola fuente de la verdad — ADR-005). Recargar es honesto y simple; construir el promedio en el navegador sería duplicar lógica. Cuando eso duela por rendimiento, será la señal para pensar en SPA (ADR-006, pregunta 1).*

## Paso 5 — Conectar la web a la aplicación

Edita **`app/main.py`**:

```python
"""Punto de composición: arma la aplicación y conecta las piezas (ADR-001)."""
from pathlib import Path

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

from app.database import crear_tablas
from app.rutas import api, web

app = FastAPI(
    title="Cartelera",
    description=(
        "Catálogo de películas del CineClub Barrio con calificación por "
        "estrellas. La misma lógica alimenta la web y esta API."
    ),
    version="1.0.0",
)

crear_tablas()

# Archivos servidos tal cual al navegador (ADR-006)
app.mount("/static", StaticFiles(directory=Path(__file__).parent / "static"), name="static")

app.include_router(web.router)   # páginas HTML
app.include_router(api.router)   # API JSON (/docs)


@app.get("/salud")
def salud() -> dict:
    """Chequeo mínimo para saber que el servicio está vivo."""
    return {"estado": "ok"}
```

---

## ✅ Verificación de la guía 6 (recorrido completo de historias de usuario)

Necesitas una película y una socia. Crea **`datos_demo.py`** (desechable):

```python
from app.database import SesionLocal, crear_tablas
from app.modelos import Pelicula
from app.repositorios import RepositorioPeliculas
from app.servicios import autenticacion

crear_tablas()
with SesionLocal() as sesion:
    RepositorioPeliculas(sesion).crear(
        Pelicula(titulo="Coco", anio=2017,
                 sinopsis="Miguel viaja por accidente a la Tierra de los Muertos.",
                 url_trailer="https://www.youtube.com/embed/Ga6RYejo6Hk")
    )
    autenticacion.registrar(sesion, "Ana", "ana@test.cl", "clave-segura-123")
    print("Datos demo listos")
```

```powershell
python datos_demo.py
python run.py
```

Abre **http://127.0.0.1:8000** y recorre las historias de usuario:

1. **HU-01:** ves el catálogo con Coco, su placeholder de carátula y "Sin calificaciones". Prueba también en la vista de móvil del navegador (F12 → teléfono): la grilla se ordena en una columna (C4).
2. **HU-02:** entra a la ficha: sinopsis, tráiler reproducible, y —sin sesión— el aviso *"Inicia sesión o crea una cuenta"* (HU-05).
3. **HU-03:** crea tu propia cuenta desde el formulario. Prueba el error: contraseña de 3 caracteres → mensaje en español, y el formulario **conserva tu nombre**. Prueba el email duplicado de Ana → "Ya existe una cuenta con ese email".
4. **Login:** entra con tu cuenta. La barra te saluda ("Hola, …") y la ficha ahora muestra **Tu calificación** con las 5 estrellas.
5. **HU-04:** haz clic en la 5ª estrella → la página se recarga con tu voto y el promedio "5,0 · 1 calificación". Cambia a 2 estrellas → el total **sigue en 1** y el promedio baja (RN-01 frente a tus ojos).
6. **Salir:** vuelve al estado anónimo.

Detén el servidor y limpia (`del datos_demo.py` y `del cartelera.db`).

---

## 📝 Punto de control

1. El formulario web y la API usan la misma clase `UsuarioCrear`. ¿Qué pasaría si mañana la regla fuera "contraseña mínima de 10" y solo actualizaras la API?
2. ¿Por qué el widget recarga la página en vez de recalcular el promedio en JavaScript?
3. ¿Qué le impide al JavaScript del sitio robar la cookie de sesión y llevarse el token?

## Lo que acabas de aprender

- Plantillas Jinja2 con herencia (`base.html` como molde)
- Formularios web validados con los mismos esquemas que la API
- Cookie HttpOnly como puente entre login web y API
- El recorrido completo HU-01…HU-05 como verificación

**Siguiente:** `guia-07-panel-admin.md` — el panel de la coordinadora: subida de carátulas de verdad y el cierre del circuito.
