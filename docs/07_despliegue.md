# Fase 7 — Guía de Despliegue: el gran final (gratis)

> **Módulos:** ISI601 / ISI602 · **Fase del ciclo de vida:** 7. Despliegue
> **Qué construirás hoy:** la publicación real: tu aplicación con URL pública en internet, con base de datos Postgres persistente, sin pagar nada ni entregar tarjeta.
> **Al terminar tendrás:** tu Cartelera accesible desde el celular de cualquiera, y una lección de arquitectura que ningún slide enseña: el disco efímero.
> **Necesitas:** las guías 1 a 8 terminadas, el proyecto en un repo de GitHub, y una cuenta de correo.
> **Decisión de fondo:** ADR-007 (Render + Neon).
> **Fecha:** 2026-08-23

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Build** | El momento en que la plataforma instala tus dependencias (`pip install -r requirements.txt`) |
| **Start command** | El comando que arranca tu app en sus servidores (nosotros: `uvicorn ...`) |
| **Variable de entorno (en producción)** | La misma de siempre, pero configurada en el panel de la plataforma, no en tu computador |
| **Cold start / hibernación** | El plan gratis "duerme" tu app tras ~15 min sin visitas; el próximo visitante espera 30–60 s mientras despierta |
| **Disco efímero** | Los archivos que tu app guarda en el servidor **desaparecen** al reiniciar o re-desplegar. ADR-004, protagonizada hoy |

---

## Paso 0 — Requisitos en el repo

🧠 **El desarrollador piensa:** *Render va a construir desde cero: clona tu repo, instala requirements, arranca el comando. Solo necesita tres cosas de tu parte: el `requirements.txt` completo y correcto, el código en GitHub, y **nada de secretos pegados en el código** (que ya cumplimos — todo vive en `config.py` leyendo el entorno, RNF-03). Falta una pieza: el driver de Postgres, que en desarrollo no se usa pero en producción sí. Si falta, la app parte feliz con SQLite… hasta que la configuras con la URL de Neon y revienta.*

```powershell
pip install psycopg2-binary
pip freeze > requirements.txt
git add requirements.txt
git commit -m "Despliegue: driver de Postgres para produccion"
git push
```

## Paso 1 — La base de datos: Neon (Postgres gratis)

1. Entra a **[neon.tech](https://neon.tech)** y créate una cuenta (puede ser con GitHub).
2. Crea un proyecto (nombre sugerido: `cartelera`). Región: la más cercana (ej. AWS US East o la que ofrezca).
3. En el dashboard busca el **connection string**: empieza con `postgresql://` y es largo, con usuario y contraseña adentro. Cópialo y guárdalo en un lugar seguro — **es un secreto**.

🧠 **El desarrollador piensa:** *detalle que rompe a todos: SQLAlchemy necesita saber QUÉ driver usar para hablar Postgres, así que a la URL de Neon hay que anteponerle el driver: queda `postgresql+psycopg2://usuario:clave@ep-xxxx.neon.tech/...?sslmode=require`. Sin ese `+psycopg2`, la app no parte. Es exactamente el mecanismo del ADR-002: la misma base de código, distinto motor, **una variable de entorno de diferencia** — la promesa de la fase de arquitectura se cumple ante tus ojos.*

## Paso 2 — El servicio web: Render

1. Entra a **[render.com](https://render.com)** y créate una cuenta (con GitHub, para que pueda leer tus repos).
2. **New + → Web Service**, y conecta tu repositorio `cartelera`/`demo-cine`.
3. Configura:

| Campo | Valor |
|---|---|
| Name | `cartelera` (será parte de la URL) |
| Region | la misma que Neon |
| Branch | `main` |
| Runtime | **Python 3** |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `uvicorn app.main:app --host 0.0.0.0 --port $PORT` |
| Instance Type | **Free** |

4. Antes de crear, abre **Environment** y agrega las variables:

| Clave | Valor |
|---|---|
| `DATABASE_URL` | la URL de Neon **con** `+psycopg2` (paso 1) |
| `SECRET_KEY` | un secreto largo y aleatorio — genera uno con el comando de abajo |
| `ADMIN_EMAIL` | el email que quieras para la coordinadora en producción |
| `ADMIN_PASSWORD` | una contraseña digna (no `admin1234`…) |

```powershell
python -c "import secrets; print(secrets.token_hex(32))"
```

5. **Create Web Service** y espera el primer build (unos minutos, con log en pantalla). Al terminar, Render te da la URL: **`https://cartelera-xxxx.onrender.com`**.

🧠 **El desarrollador piensa:** *`--host 0.0.0.0` y `--port $PORT`: en mi computador yo elijo el puerto 8000; en Render el puerto me lo asignan ellos y me lo pasan EN esa variable. `run.py` era para mí; este comando es para la nube. Dos arranques, un mismo código.*

## Paso 3 — Sembrar la base de producción (una sola vez)

Desde tu computador, en la raíz del proyecto, apuntando a la base de Neon:

```bash
# Git Bash / Linux / macOS
DATABASE_URL="postgresql+psycopg2://usuario:clave@ep-xxxx.neon.tech/bd?sslmode=require" python semilla.py

# PowerShell
$env:DATABASE_URL="postgresql+psycopg2://usuario:clave@ep-xxxx.neon.tech/bd?sslmode=require"; python semilla.py
```

El mismo `semilla.py` de la guía 8, **cero cambios**: solo cambia el entorno. Debe imprimir que creó la coordinadora y las 4 películas.

## Paso 4 — Deploy continuo (el regalo de GitHub)

🧠 **El desarrollador piensa:** *acabo de conectar mi repo con Render: desde ahora, cada `git push` a `main` dispara un build y un despliegue automáticos. Eso que las empresas llaman CI/CD, en chico y gratis. Haz la prueba: cambia el saludo de la home en la plantilla, commitea, pushea, y mira el log de Render hacer el trabajo.*

---

## ✅ Verificación de la guía (en tu URL pública)

| # | Verificación | Origen |
|---|---|---|
| 1 | `https://TU-URL.onrender.com/salud` → `{"estado":"ok"}` | Guía 1 |
| 2 | El catálogo muestra las 4 películas de la semilla (con placeholder) | Guía 8 |
| 3 | Login como coordinadora → panel → publica una película **con carátula** | Guía 7 |
| 4 | Regístrate desde el celular, califica, recalifica: el total no sube | RN-01 |
| 5 | `/docs` funciona igual que en local — la API vive en internet | Guía 5 |
| 6 | **La lección del disco efímero** (abajo) | ADR-004 |

**La lección (hazla literalmente, vale la pena):** sube una carátula real desde el panel y confírmala en el catálogo. Ahora ve a Render → **Manual Deploy → Clear build cache & deploy** (o simplemente toca cualquier cambio y pushea). Espera el redeploy, entra de nuevo… **la película está, la calificación está, la carátula NO está** (volviste al placeholder). La base de Neon sobrevivió; el disco de Render se reseteó. Ese contraste, sentido en carne propia, ES la clase de arquitectura de ADR-004 — y la motivación perfecta para el ejercicio pendiente: escribir `AlmacenamientoCloudinary` sin tocar ninguna otra capa.

---

## 📝 Punto de control

1. Borras el servicio en Render: ¿qué sobrevive y qué se pierde? ¿Y si borras el proyecto en Neon?
2. ¿Por qué la URL lleva `+psycopg2` y qué pasa si lo olvidas?
3. Tu app está "lenta" solo en la primera visita de la mañana. Diagnóstico: ¿bug, hibernación o mala arquitectura? ¿Qué plan de Render lo arregla y qué cuesta la diferencia?
4. La carátula desapareció tras el redeploy pero la calificación no. Explica la diferencia con precisión de arquitectura.

## Lo que acabas de aprender

- Publicar un servicio Python real con variables de entorno como secretos
- El mismo código, dos mundos (SQLite/Postgres) por una variable
- Deploy automático desde GitHub (CI en chico)
- Hibernación y disco efímero: las dos lecciones del plan gratis

**Última parada:** `08_mantenimiento.md` — el sistema ya vive: qué pasa cuando algo se rompe, cuando el cliente pide cambios, y cómo se cierra (y se reabre) el ciclo.
