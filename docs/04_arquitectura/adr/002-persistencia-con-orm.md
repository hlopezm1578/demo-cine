# ADR-002 — Persistencia con SQLAlchemy ORM: SQLite en desarrollo, Postgres en producción

- **Estado:** Aceptada
- **Fecha:** 2026-08-23
- **Resuelve:** cómo persisten los almacenes D1–D3 del diseño

## Contexto

El sistema necesita guardar usuarios, películas y calificaciones (diseño §2). En desarrollo se trabaja en cualquier notebook sin instalar nada; en producción hay un Postgres gratis disponible (ADR-007). El cliente exige presupuesto $0 y funcionamiento desatendido (C1, CS4). Queremos **el mismo código** en ambos mundos.

## Opciones consideradas

| Opción | A favor | En contra |
|---|---|---|
| **A. SQL crudo** a mano | Se ve el SQL, cero abstracción | SQL distinto por motor; mapear filas a mano; propenso a errores |
| **B. SQLAlchemy ORM** | Mismo código para SQLite y Postgres; modelos = tablas; restricciones declaradas una vez | Los alumnos ven menos SQL; otra API que aprender |
| **C. ORM de Django / base documental** | Incluido en Django / esquema flexible | Cambia todo el stack / pierde las restricciones que justo necesitamos (UNIQUE de RN-01) |

## Decisión

**Opción B.** SQLAlchemy 2.0 con tres modelos (Usuario, Pelicula, Calificacion). El motor se elige por una sola variable de entorno:

- Desarrollo: SQLite en un archivo (cero instalación — valor por defecto de la configuración).
- Producción: Postgres en Neon (ADR-007), solo cambia la URL.

Las tablas se crean al arrancar si no existen. **Sin migraciones** por ahora (ver consecuencias).

## Consecuencias

**Positivas**
- Migrar de SQLite a Postgres es cambiar una variable de entorno: demo en vivo de por qué abstraer el motor (RNF-05).
- La restricción UNIQUE(usuario, película) se declara una vez en el modelo y la base la hace cumplir en ambos motores (RN-01).
- En clase se puede activar el eco de SQL para **mostrar el SQL real** que genera el ORM.

**Negativas**
- El ORM esconde el SQL: hay que complementar mostrando la consulta generada, o el alumno nunca la ve.
- Crear tablas al arrancar no modifica tablas existentes: al cambiar un modelo en producción hay que borrar y recrear (aquí aceptable) o adoptar **Alembic** (migraciones versionadas) — el siguiente paso natural del curso.

## Para conversar en clase

1. ¿Por qué la restricción UNIQUE en la base y no solo un `if` en Python? (dos peticiones simultáneas).
2. Activen el eco de SQL: ¿qué consulta dispara el promedio del catálogo? ¿Cuántas son? (problema N+1).
3. El modelo cambió en producción: ¿qué hacen los equipos reales? (migraciones).
