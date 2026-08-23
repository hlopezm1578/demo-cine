# Fase 8 — Guía de Mantenimiento: el sistema vive

> **Módulos:** ISI601 / ISI602 · **Fase del ciclo de vida:** 8. Mantenimiento
> **Qué construirás hoy:** nada de código nuevo — construirás **la disciplina**: la bitácora del sistema y los procedimientos para cuando algo se rompa o el cliente pida cambios.
> **Al terminar tendrás:** un `CHANGELOG.md`, tres escenarios de mantenimiento resueltos paso a paso, y el mapa de lo que queda abierto.
> **Necesitas:** la fase 7 terminada (la aplicación publicada).
> **La idea de fondo:** el desarrollo termina; el software no. Esta fase es, en tiempo y dinero, la más larga de la vida de cualquier sistema real.
> **Fecha:** 2026-08-23

---

## Los términos de hoy

| Término | Qué es, en una frase |
|---|---|
| **Mantenimiento correctivo** | Arreglar lo que se rompió (bugs reportados) |
| **Mantenimiento adaptativo** | Ajustar a lo que cambió alrededor (nueva versión de una librería, nuevo requisito legal) |
| **Mantenimiento perfectivo** | Mejorar lo que funciona (rendimiento, experiencia) |
| **Mantenimiento preventivo** | Evitar lo que va a romperse (deuda técnica, dependencias viejas) |
| **Deuda técnica** | El costo futuro de las decisiones rápidas de hoy — se paga en interés compuesto |
| **Incidente** | El sistema fallando en producción, con usuarios reales mirando |

🧠 **El desarrollador piensa:** *la clasificación anterior es la ISO/IEC 14764, y la buena noticia es que Cartelera ya acumuló un poco de cada una — documentada. Repasarla con ejemplos propios es la mejor forma de aprenderla.*

---

## Paso 1 — La bitácora: `CHANGELOG.md`

🧠 **El desarrollador piensa:** *dentro de tres meses, "¿cuándo agregamos la película X?" o "¿qué cambió en el último deploy?" no deben depender de la memoria ni de leer el git log completo. Un changelog es barato y se escribe mientras los cambios están frescos.*

Crea **`CHANGELOG.md`** en la raíz del repo:

```markdown
# Changelog — Cartelera

Formato: fecha, versión corta, cambios agrupados por tipo de mantenimiento
(C=correctivo, A=adaptativo, P=perfectivo, Pv=preventivo).

## [1.0.0] — 2026-08-__
### Lanzamiento inicial
- Catálogo público, ficha con tráiler embebido, calificación por estrellas (P)
- Panel de administración con subida de carátulas (P)
- API JSON según contrato_api.yaml (P)
- Publicación en Render + Neon (P)
- Limitaciones conocidas: disco efímero para carátulas (ADR-004),
  sin migraciones (ADR-002), sin CSRF en formularios (README).
```

La regla: **cada commit que cambia comportamiento visible suma una línea**. Cada línea lleva su letra (C/A/P/Pv) — así, al final del semestre, la distribución de letras de tu changelog **es tu perfil de mantenimiento** (y una discusión de clase: ¿qué mezcla es sana?).

## Paso 2 — Escenario 1: CORRECTIVO — "el promedio no cambia"

> *Mensaje de Macarena:* "Ana me dice que calificó Coco de nuevo y el número de la portada no cambió."

🧠 **El desarrollador piensa:** *antes de tocar código, el ritual: **reproducir**. Si no puedo reproducirlo, estoy adivinando. Y fíjate lo que el sistema me regala: la suite de la fase 6 con `test_calificar_y_recalificar` en verde. ¿El promedio en la API o solo en la pantalla? Pregunta clave para acotar la capa.*

El procedimiento (memorizable — sirve para cualquier bug):

```
1. REPRODUCIR   → ¿dónde, cuándo, quién? (local vs producción)
2. ACOTAR       → ¿qué capa? (prueba la API en /docs: ¿el promedio cambia ahí?)
3. ESCRIBIR LA PRUEBA QUE FALLA → el bug se convierte en un caso rojo
4. CORREGIR     → el fix más pequeño que ponga la prueba en verde
5. VERIFICAR EN PRODUCCIÓN → deploy + comprobación
6. REGISTRAR    → changelog (C) + commit referenciando el reporte
```

En este caso la reproducción revela (casi seguro) que el promedio **sí** cambió en la base — Ana recalificó 2★ sobre su 5★ y el promedio bajó de 5,0 a 2,0: el sistema hizo lo correcto y el "bug" era una expectativa. **Caso cerrado sin tocar código** — y ese desenlace es tan válido y frecuente que merece nombre: *el bug del usuario*. Detectarlo sin quemar un día en "fixes" fantasmas es mantenimiento de alto nivel.

## Paso 3 — Escenario 2: ADAPTATIVO/PERFECTIVO — "queremos medias estrellas (3,5)"

> *Mensaje de Macarena:* "¿podría ser media estrella? La gente pelea por el 3 o el 4."

🧠 **El desarrollador piensa (la decisión del día):** el instinto del principiante es abrir el editor y cambiar `Integer` por `Float`. El instinto que este proyecto entrenó es otro: **el cambio de pedido del cliente no empieza en el código — empieza en la fase 1**. Sigo el hilo hacia atrás:

| Fase | Documento a tocar | El cambio |
|---|---|---|
| 1 | Necesidad | nueva petición P6 con cita de Macarena |
| 2 | Requerimientos | RN-02 v2: estrellas de 0,5 a 5,0 en pasos de 0,5; nueva HU |
| 3 | Diseño | diccionario de datos: `estrellas` pasa de entero a decimal; el widget pinta medias |
| 4 | Arquitectura | ¿nuevo ADR? (no: la decisión UNIQUE no cambia) — el **contrato sí**: `minimum: 0.5, maximum: 5, type: number` → **versión 1.1.0 del contrato, aprobada antes de codificar** (ADR-008) |
| 5-7 | Guías/código/pruebas | el cambio en orden: modelo → esquema → plantilla → pruebas → changelog (A) |

¿Por qué el rodeo si el código son 20 líneas? Porque **cada documento es un contrato con alguien**: la BD guarda enteros, el contrato prometió enteros, las pruebas defienden enteros. Cambiar el código directo deja tres documentos mintiendo — y los documentos que mienten son peores que los que no existen.

## Paso 4 — Escenario 3: PREVENTIVO — pagar la deuda del ADR-004

La carátula que desapareció tras el redeploy (fase 7) es deuda técnica **documentada**: el ADR-004 dejó la puerta abierta a propósito. El ejercicio de pago:

```
1. Crear AlmacenamientoCloudinary(Almacenamiento) — guardar, eliminar, url_publica
2. Configurar credenciales como variables de entorno (nunca en el código)
3. Cambiar UNA línea en app/servicios/almacenamiento.py
4. Correr la suite → verde (¡ninguna prueba sabía que existía el disco local!)
```

Ese cuarto paso es la demostración más elocuente del DIP del ramo: **intercambiaste la infraestructura de archivos completa y el sistema no se enteró**. Changelog: (Pv).

## Paso 5 — Monitoreo mínimo

- **`/salud`** es tu chequeo de vida: Render también lo puede usar como health check.
- **Logs de Render** (pestaña Logs): cada petición y cada error, con fecha y hora. Primera parada en cualquier incidente real.
- **Las pruebas como termómetro**: cualquier duda de salud → `python -m pytest pruebas -v` y verde/rojo en segundos.

---

## ✅ Verificación de la guía

1. `CHANGELOG.md` creado con el lanzamiento 1.0.0 registrado.
2. **Simulacro de incidente** (en parejas, en clase): uno reporta un bug inventado ("el tráiler de X no carga", "no puedo registrarme con email tal"), el otro ejecuta el procedimiento de los 6 pasos **en voz alta**, llegando a la capa correcta. Inviértense los roles.
3. **Simulacro de cambio**: elegir otra petición real de Macarena ("quiero buscar películas por título") y recorrer las 5 fases del hilo hacia atrás **en papel**, sin escribir código — exactamente como se evalúa en los informes del ramo.
4. Abrir los logs de Render y encontrar la última petición tuya.

---

## 📝 Punto de control

1. De los tres escenarios, ¿cuál NO requirió cambiar código y por qué fue igual de valioso?
2. Tu changelog de fin de semestre tiene 80 % de letra C. ¿Qué te está diciendo? (pista: hablemos de calidad en la fase 5).
3. "Los documentos que mienten son peores que los que no existen." Relaciónalo con el escenario de las medias estrellas.

## Lo que acabas de aprender

- Los 4 tipos de mantenimiento, con ejemplos de tu propio proyecto
- El procedimiento de 6 pasos ante un bug (y el "bug del usuario")
- Que los cambios del cliente recorren el ciclo COMPLETO, hacia atrás
- Pagar deuda técnica documentada sin tocar el resto del sistema

---

## El ciclo se cierra (y se reabre)

Con las 8 fases recorridas, el proyecto queda **vivo y documentado**: cada decisión con su porqué, cada regla con su defensa, cada evolución futura con su puerta abierta. Lo que queda en el mapa son las escaleras que conversamos: **migraciones (Alembic)** para evolucionar la base sin borrarla, **Cloudinary** para las carátulas, **React separado** cuando la interactividad lo exija (ADR-009 revirtiendo el 006), **pruebas de navegador** cuando la UI pese más.

> La pregunta de cierre del curso, para llevar a casa: *¿qué tendría que pasarle a Cartelera —usuarios, equipo, dinero— para justificar cada uno de esos escalones?* Quien sabe responderla, no memorizó arquitectura: **la piensa**.
