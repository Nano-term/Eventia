# Documentación de Eventia

Complementa la propuesta original (`../Eventia_Propuesta_Completa_Corregida.docx`) y las interfaces (`../Interfaces de Eventia_015057.pdf`, `../eventia-interfaces.html`). Está escrita para que una persona o una IA pueda implementar sin adivinar.

| # | Documento | Para qué sirve |
|---|---|---|
| 01 | [Reglas de negocio](01-reglas-de-negocio.md) | Reglas `RN-xx`: aforo, asistencia, constancias, etc. |
| 02 | [Matriz de permisos](02-matriz-permisos.md) | Quién puede qué (base de RLS) |
| 03 | [Diagramas de secuencia](03-diagramas-secuencia.md) | 15 flujos de la app (Mermaid) |
| 04 | [Diagramas de estados](04-estados.md) | Ciclo de vida de evento, solicitud, invitación, inscripción, etc. |
| 05 | [Contratos RPC y rutas](05-contratos-rpc.md) | Funciones, errores, rutas |
| 06 | [Modo sin conexión](06-offline.md) | Snapshot, cola, sincronización, conflictos |
| 07 | [Decisiones abiertas](07-decisiones-abiertas.md) | Lo que hay que confirmar |
| 08 | [Plan de pruebas](08-plan-pruebas.md) | Criterios de aceptación y DoD |
| 09 | [Trazabilidad](09-trazabilidad.md) | RF ↔ pantalla ↔ tabla ↔ RPC ↔ prueba |
| 10 | [Guía de desarrollo](10-guia-desarrollo.md) | Estructura, convenciones, orden, riesgos |

## Orden de lectura recomendado
Propuesta (secciones 1–3) → `01` → `07` (confirmar decisiones) → `02` → `03`/`04` → `05` → `06` → `09` → `08` → `10`.

## Estado
- Las reglas marcadas **[PROPUESTA]** son decisiones tomadas para cubrir vacíos de la propuesta y **aún no están aprobadas**. Revisarlas con `07-decisiones-abiertas.md`.
- Ante conflicto entre documentos, gana el de numeración menor (`01` > `03` > …) y se corrige el otro.
- Los IDs (`RF-xx` de la propuesta, `RN-xx`, `P#`, `A1…O7`) son estables: no renumerar.
