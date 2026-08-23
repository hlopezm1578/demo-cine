# ADR-001 — Arquitectura en capas (rutas → servicios → repositorios → modelos)

- **Estado:** Aceptada
- **Fecha:** 2026-08-23
- **Resuelve:** cómo organizar el código que implementa el diseño (fase 3)

## Contexto

Cartelera atiende dos "caras" con la misma lógica: las páginas HTML (catálogo, ficha, login, panel) y la API JSON (`/docs`, RF-10). Si todo vive mezclado, cada cambio toca todo y es imposible razonar dónde debe ir cada regla. El DFD del diseño (procesos 1.0–6.0, almacenes D1–D3) ya sugiere la separación; este ADR la hace obligatoria en el código.

## Opciones consideradas

| Opción | A favor | En contra |
|---|---|---|
| **A. Un solo archivo** con todo | Rápido de arrancar | HTML, validaciones y SQL pegados; imposible de testear por partes; crece mal |
| **B. Capas: rutas → servicios → repositorios → modelos** | Una responsabilidad por capa; la API y la web comparten lógica; testeable | Más archivos y "saltos" para seguir un flujo |
| **C. Framework MVC completo** (Django) | Estructura impuesta, admin incluido | Demasiado peso para 4 pantallas; esconde justo lo que el ramo quiere mostrar |

## Decisión

**Opción B**, con una regla por capa:

```
rutas/          → reciben la petición, validan forma, eligen respuesta (HTML o JSON)
servicios/      → lógica de negocio: los procesos 1.0–6.0 del DFD
repositorios/   → solo acceso a datos: los almacenes D1–D3 del DFD
modelos/        → tablas y relaciones: las entidades del diseño §2
```

**Regla de dependencia:** una capa solo habla con la capa inmediatamente inferior. Una ruta nunca escribe SQL; un repositorio nunca decide si un socio puede calificar.

## Consecuencias

**Positivas**
- El mismo servicio alimenta el widget HTML y el endpoint JSON: la lógica existe una sola vez.
- La lógica se prueba sin navegador ni servidor.
- El alumno ubica cualquier regla preguntando "¿qué capa es?".

**Negativas (honestas)**
- Para ~10 endpoints, el enjambre de carpetas es **overkill** y hay que decirlo en voz alta: el costo de las capas se paga cuando el sistema crece o hay que testear.
- Seguir un flujo completo requiere saltar entre archivos.

## Para conversar en clase

1. ¿En qué capa validarías "el año no puede ser futuro"? ¿Y "el email es único"? (pista: no es la misma capa).
2. ¿Qué pasaría si RN-01 viviera solo en la ruta HTML y no en el servicio?
3. ¿Cuándo dirían que un proyecto SÍ necesita capas y cuándo un script plano alcanza?
