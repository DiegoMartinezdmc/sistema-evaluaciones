# DOC-03 Modelo de datos

**Estado: pendiente de definición y aprobación.** No existen tablas del dominio en este esqueleto.

Posibles conceptos por investigar en el análisis funcional: evaluados, períodos, preguntas, opciones, intentos, respuestas, resultados e informes.

El equipo documentador debe acordar entidades, atributos, relaciones, restricciones, trazabilidad y protección de datos **antes** de generar SQL o clases JPA.

## Decisiones registradas en el repositorio

| Fecha | Tarea | Decisión |
| --- | --- | --- |
| 2026-11-17 | GEN-01-BE | `subprueba.factor` (`VARCHAR(80)`, nulo mientras no se cargue): código del factor al que aporta la subprueba según el Documento BFA §5.3. Varias subpruebas pueden compartir factor (S1 y S1b → S1; N1-A y N1-B → N1). Migración `V202611170600__comun_agregar_factor_subprueba.sql`. |
| 2026-11-17 | GEN-01-BE | E2 no guarda `opciones` ni `numeroPreguntas`: los calcula desde `pregunta` y `opcion_respuesta` de la configuración publicada. |

## Endpoints y tablas

**Tarea:** DOC-41 · **Estado:** borrador. Muestra qué tablas lee y escribe cada endpoint del contrato (`docs/api/openapi.yaml`). Esta sección cubre **E1–E15** (Diego); **E16–E31** los completa Daysi en este mismo PR.

**Cómo leer los nombres de tabla**

- **Físico:** la tabla existe en las migraciones del repositorio (`test`, `subprueba`, `configuracion_subprueba`, `version_formulario`, `pregunta`, `opcion_respuesta`) o en la PR #89 (`cuenta`, `cuenta_rol`, `sesion`, aún en borrador).
- *Lógico:* entidad del Documento general (ENT-COM) cuya tabla todavía no está en el repositorio. Se escribe en cursiva y su nombre físico está **por confirmar** al fusionarse la migración correspondiente. Las lecturas y escrituras de estos casos son una **propuesta** derivada del propósito del endpoint en el contrato, no un comportamiento ya implementado.

| Endpoint | Método y ruta | Tablas que lee | Tablas que escribe | Comentario |
| --- | --- | --- | --- | --- |
| E1 | `GET /api/v1/tests` | `test` | — | Lista los tests activos (`activo`). |
| E2 | `GET /api/v1/tests/{testId}` | `test`, `subprueba`, `version_formulario`, `configuracion_subprueba`, `pregunta`, `opcion_respuesta` | — | Lee la edición publicada. `opciones` y `numeroPreguntas` se calculan desde `pregunta` y `opcion_respuesta`, no se guardan (decisión de GEN-01-BE). |
| E3 | `POST /api/v1/evaluaciones` | *sujeto*, *periodo*, `test`, `version_formulario`, `configuracion_subprueba` | *evaluacion*, *evaluacion_subprueba* | Fija sujeto, periodo y edición publicada al iniciar. Puede registrar un evento de auditoría. |
| E4 | `GET /api/v1/evaluaciones/{evaluacionId}` | *evaluacion*, *evaluacion_subprueba*, *respuesta* | — | Estado y avance. Si el límite venció, muestra la subprueba CERRADA con `cerradaEn` igual al límite (según el contrato común, comentario de Iván en DOC-02). |
| E5 | `POST /api/v1/evaluaciones/{evaluacionId}/subpruebas/{codigo}/iniciar` | *evaluacion*, `configuracion_subprueba`, *evaluacion_subprueba* | *evaluacion_subprueba*, *evento de auditoría* | Registra el inicio real y, si hay duración autorizada, el límite. |
| E6 | `GET /api/v1/evaluaciones/{evaluacionId}/subpruebas/{codigo}/preguntas` | *evaluacion_subprueba*, `configuracion_subprueba`, `pregunta`, `opcion_respuesta`, *respuesta* | — | Devuelve los reactivos y la selección previa; no expone la clave. |
| E7 | `PUT /api/v1/evaluaciones/{evaluacionId}/subpruebas/{codigo}/respuestas` | *evaluacion_subprueba*, `pregunta`, `opcion_respuesta` | *respuesta*, *evento de auditoría* | Valida que la opción pertenezca a la pregunta y que la ejecución esté activa. Una fila por ejecución y pregunta. |
| E8 | `POST /api/v1/evaluaciones/{evaluacionId}/finalizar` | *evaluacion*, *evaluacion_subprueba*, *respuesta*, *insumos de cálculo (clave y baremo)* | *evaluacion*, *evaluacion_subprueba*, *resultado*, *revisión de cálculo*, *evento de auditoría* | Cierra y consolida. El cálculo depende de decisiones abiertas de Psicología. |
| E9 | `GET /api/v1/evaluaciones/{evaluacionId}/resultados` | *resultado*, *revisión de cálculo* | — | Solo el resultado publicado, según permisos. |
| E10 | `PUT /api/v1/evaluaciones/{evaluacionId}/calificaciones/{calificacionId}/palabras/{respuestaId}/revision` | *respuesta*, *calificación* | *revisión de palabra*, *evento de auditoría* | Fluidez Verbal (BFA). No aplica a Weill. |
| E11 | `POST /api/v1/evaluaciones/{evaluacionId}/recalculos` | *evaluacion*, *resultado*, *respuesta*, *insumos de cálculo* | *revisión de cálculo*, *evento de auditoría* | Crea una revisión nueva; el historial se conserva. |
| E12 | `PUT /api/v1/evaluaciones/{evaluacionId}/resultados/vigente` | *resultado*, *revisión de cálculo* | *resultado*, *evento de auditoría* | Actualiza la revisión vigente y el estado. Actualización y bitácora en la misma transacción. |
| E13 | `POST /api/v1/auth/registro` | `cuenta` | `cuenta`, `cuenta_rol` | Comprueba que el correo sea único; guarda solo el resumen de la contraseña. El rol inicial lo define la regla de registro (por confirmar). |
| E14 | `POST /api/v1/auth/sesion` | `cuenta`, `cuenta_rol` | `sesion` | Abre sesión; guarda el resumen del token, el rol activo y `expira_en`. |
| E15 | `DELETE /api/v1/auth/sesion` | `sesion` | `sesion` | Cierra la sesión actual escribiendo `cerrada_en`; no borra la fila. |

## Criterios de aceptación

- [ ] Los 31 endpoints están: **E1–E15** aquí; **E16–E31** pendientes de Daysi.
- [x] En E1–E15 solo aparecen nombres de tablas físicas del repositorio o de la PR #89, o entidades lógicas marcadas en cursiva por confirmar.
- [ ] Reemplazar los nombres en cursiva por los físicos cuando se fusionen las migraciones de evaluación, resultado y auditoría.
