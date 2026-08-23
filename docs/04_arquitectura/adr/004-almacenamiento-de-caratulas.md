# ADR-004 — Carátulas: interfaz de almacenamiento con implementación local (y disco efímero documentado)

- **Estado:** Aceptada (revisar al pasar a producción seria)
- **Fecha:** 2026-08-23
- **Resuelve:** cómo guardar y servir el almacén A1 (archivos de carátulas)

## Contexto

La coordinadora sube la carátula de cada película: archivos binarios (JPG/PNG/WebP) de hasta 5 MB (RN-05). El diseño ya decidió que la base de datos guarda solo la **ruta** (fase 3, §2.3.3). Falta decidir dónde viven los archivos. Detalle conocido del plan gratuito: **el disco de Render free es efímero** — todo archivo desaparece cuando el servicio se reinicia o se vuelve a desplegar (ADR-007).

## Opciones consideradas

| Opción | A favor | En contra |
|---|---|---|
| **A. La imagen dentro de la BD** (BLOB) | Todo viaja junto, respaldos simples | La BD engorda; servirla la satura; mala práctica generalizada |
| **B. Disco local** servido por la propia app | Cero dependencias, funciona perfecto en desarrollo | En producción gratis los archivos **se pierden** en cada reinicio |
| **C. Servicio externo** (Cloudinary/S3 con capa gratis) | Persistente y escalable | Cuenta extra, API key, complejidad nueva para el alumno |

## Decisión

**B, con una trampa deliberada y una puerta abierta:**

1. **Trampa deliberada (lección en vivo):** usamos disco local y lo dejamos por escrito. El día que la clase suba carátulas en Render y el servicio se vuelva a desplegar, las imágenes desaparecerán y quedarán los placeholders. Ese momento vale más que cualquier slide sobre almacenamiento.
2. **Puerta abierta (DIP en acción):** el resto del código **no conoce** la implementación local. Depende de una interfaz con tres métodos: *guardar* (devuelve la ruta para la BD), *eliminar*, y *URL pública*. Agregar una implementación de Cloudinary mañana no toca ninguna otra capa. Es el principio del ejemplo `08_dip_inyeccion.py` del ramo (ISI602), aplicado a un problema real — y el caso emblemático de Open/Cerrado en el documento de arquitectura (§5).

## Consecuencias

**Positivas**
- Desarrollo sin cuentas ni claves; la interfaz deja el salto a producción como ejercicio real y acotado.
- La BD guarda solo la ruta: liviana (diseño §2.3.3).

**Negativas**
- En producción gratis las carátulas se pierden al reiniciar (documentado y convertido en clase).
- Sin deduplicación: subir dos veces el mismo archivo ocupa doble espacio.

## Para conversar en clase

1. Suban una carátula en producción y fuercen un redeploy: ¿qué ven? ¿Por qué la base de datos SÍ conservó la película?
2. Diseñen la implementación de Cloudinary: ¿qué cambia en *guardar* y qué cambia en *URL pública*? ¿Qué capa se toca?
3. ¿Por qué no guardar la imagen como BLOB en la tabla? Discutan tamaño, respaldo y caché del navegador.
