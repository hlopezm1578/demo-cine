# Proyecto Cartelera — Documentos del ciclo de vida

> **Módulos:** ISI601 Taller de Diseño de Sistemas · ISI602 Arquitectura de Software
> **Producto:** Cartelera — catálogo de películas con calificación por estrellas
> **Método:** el proyecto avanza **paso a paso por el ciclo de vida del software**,
> un documento por fase. Cada fase usa la anterior como insumo y la referencia
> de forma trazable.

| # | Fase del ciclo de vida | Documento | Estado |
|---|---|---|---|
| 1 | Necesidad del cliente | `01_necesidad_del_cliente.md` | ✅ Listo |
| 2 | Análisis de requerimientos | `02_requerimientos.md` | ✅ Listo |
| 3 | Diseño (modelo de datos, procesos, pantallas) | `03_diseno.md` | ✅ Listo |
| 4 | Arquitectura y decisiones (ADRs) + contrato API-first | `04_arquitectura/` (documento + 8 ADRs + `contrato_api.yaml`) | ✅ Listo |
| 5 | Desarrollo | `05_desarrollo/` (8 guías paso a paso con razonamiento y código) | ✅ Listo |
| 6 | Pruebas | `06_pruebas.md` | Pendiente |
| 7 | Despliegue | `07_despliegue.md` | Pendiente |
| 8 | Mantenimiento | `08_mantenimiento.md` | Pendiente |

**Regla del proyecto:** ninguna fase se escribe sin aprobar la anterior. Así se
vive el ciclo: la necesidad aprueba el cliente, los requerimientos los firma el
analista, el diseño se valida contra los requerimientos, y así hasta producción.
