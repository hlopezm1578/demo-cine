# ADR-007 — Despliegue gratuito: Render (servicio web) + Neon (Postgres)

- **Estado:** Aceptada
- **Fecha:** 2026-08-23
- **Resuelve:** cómo publicar con presupuesto $0 (P5, C1) y funcionamiento desatendido (CS4)

## Contexto

Objetivo del cliente sin vuelta: la página funciona sin que nadie cuide un servidor (CS4) y no hay presupuesto ni tarjeta (C1, P5). Objetivo pedagógico que se suma: que cada alumno termine con **su propia aplicación publicada, con URL pública**. El código es Python/FastAPI con SQLAlchemy (ADR-002) y archivos en disco local (ADR-004).

## Opciones consideradas

| Opción | A favor | En contra |
|---|---|---|
| **A. PythonAnywhere** | Muy amigable para principiantes | Despliegue manual (no desde git); capa gratis limitada para apps con base de datos |
| **B. Railway** | Experiencia muy pulida | Solo crédito de prueba: la app muere cuando se agota |
| **C. Render (web service free) + Neon (Postgres free)** | Deploy automático desde GitHub; capas gratis permanentes; sin tarjeta | El servicio **duerme** tras inactividad; el disco es efímero |
| **D. Vercel serverless** | Gama alta de DX | El modelo serverless cambia la forma del código (archivos en disco, conexiones a la base) |

## Decisión

**Opción C.**

- **GitHub** aloja el código (además: portafolio del alumno).
- **Render** ejecuta el servicio: instalación de dependencias en el build, arranque del servidor en el puerto de Render, secretos como variables de entorno en el dashboard (RNF-03).
- **Neon** entrega el Postgres gratis; su URL se pasa a SQLAlchemy anteponiendo el driver (ADR-002).
- Un script de datos iniciales crea la cuenta de coordinadora y películas de ejemplo, corrido una vez contra producción.

## Consecuencias

**Positivas**
- `git push` → Render construye y publica: el alumno vive por primera vez un pipeline, en chico.
- Capas gratis permanentes y componibles; sin tarjeta.

**Negativas (mostrarlas, no esconderlas)**

- **Hibernación:** tras ~15 min sin tráfico el servicio se duerme; el primer visitante espera 30–60 s. Excelente excusa para conversar arranque en frío y costos reales.
- **Disco efímero:** las carátulas subidas desaparecen en cada despliegue (ADR-004). La base de Neon SÍ persiste: el contraste es la clase.
- La capa gratis comparte recursos: no es para producción seria.

## Para conversar en clase

1. ¿Qué es una variable de entorno y por qué la clave secreta no puede ir en el código que suben a GitHub?
2. Se borra el servicio: ¿qué sobrevive y qué no? (repo: todo el código; Neon: los datos; Render: nada).
3. ¿Cuánto costaría tener esto siempre encendido? (planes de Render/Neon, dimensión del salto de gratis a pago).
