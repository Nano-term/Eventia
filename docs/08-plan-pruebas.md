# 08 · Plan de pruebas y criterios de aceptación

Formato: Dado / Cuando / Entonces. Cada criterio cita su `RN`/`RF`. Las IA deben escribir una prueba por criterio **antes** de implementar la regla (TDD).

## Niveles
| Nivel | Herramienta sugerida | Qué cubre |
|---|---|---|
| Base de datos | `pgTAP` o scripts SQL con `supabase test db` | Constraints, triggers, RPC, **RLS** (probar como cada rol) |
| Unitarias | Vitest | Utilidades, validaciones, algoritmo de escaneo local |
| Componentes | Testing Library | Formularios y estados de pantallas |
| E2E | Playwright | Flujos de `03-diagramas-secuencia.md`, incluido modo offline (`context.setOffline(true)`) |

## Criterios de aceptación

### Auth y perfil (RF-01)
- **A1** Dado un correo nuevo, cuando se registra, entonces existen `perfiles` y `credenciales` activas para ese usuario.
- **A2** Dado un correo existente, cuando se registra, entonces error y no se crea nada.
- **A3** Dado un PART, cuando intenta `UPDATE perfiles SET rol_plataforma='administrador'`, entonces falla.

### Solicitudes (RF-02/03, RN-10…13)
- **S1** Aprobar crea evento `borrador` + membresía `organizador_principal` + notificación, atómicamente.
- **S2** Rechazar sin observaciones devuelve `OBSERVACIONES_REQUERIDAS` y no cambia estado.
- **S3** Resolver dos veces devuelve `SOLICITUD_YA_RESUELTA`.
- **S4** Un no administrador no puede resolver ni ver solicitudes ajenas.

### Eventos (RF-04/05, RN-01…08)
- **E1** Crear actividad con padre no agrupador → `PADRE_NO_AGRUPADOR`.
- **E2** Crear actividad cuyas fechas exceden las del padre → `FECHAS_FUERA_DE_RANGO`.
- **E3** Publicar actividad con padre en `borrador` → `TRANSICION_INVALIDA`.
- **E4** Cancelar un agrupador cancela sus actividades y notifica a inscritos.
- **E5** Un ANON solo ve eventos `publicado`.

### Invitaciones (RF-06, RN-20…23)
- **I1** Código con `usos_maximos=2`: tercer canje distinto → `INVITACION_AGOTADA`.
- **I2** Código vencido o revocado no canjea. Las membresías previas permanecen.
- **I3** El mismo usuario canjeando dos veces no duplica membresía ni consume uso.
- **I4** No se puede crear invitación con rol `organizador_principal`.

### Inscripción y aforo (RF-07/08, RN-30…36)
- **N1** Actividad con aforo 1, dos usuarios inscribiéndose a la vez (concurrente) → exactamente 1 `OK` y 1 `ACTIVIDAD_LLENA`.
- **N2** Raíz con aforo general 2 y 3 usuarios distintos en distintas actividades → el tercero recibe `EVENTO_LLENO`.
- **N3** Un usuario ya contado en el general se inscribe en otra actividad → no consume cupo general.
- **N4** Inscribirse dos veces → `YA_INSCRITO`.
- **N5** Cancelar libera cupo. Cancelar con asistencia → `YA_ASISTIO`.
- **N6** Inscribirse en evento `borrador`, `cerrado` o finalizado → `INSCRIPCION_CERRADA`.
- **N7** Reinscribirse tras cancelar reactiva el registro sin crear uno nuevo.
- **N8** `aforo_maximo` nulo = sin límite.

### Credencial y asistencia (RF-09…11, RN-40…57)
- **C1** El QR contiene solo `codigo` (no UUIDs de perfil).
- **C2** Regenerar revoca el código anterior: escanearlo da `CREDENCIAL_REVOCADA` o `QR_INVALIDO`.
- **C3** Escaneo de inscrito → `OK`, 1 fila `asistencias` `tipo_registro='inscrito'`.
- **C4** Segundo escaneo → `YA_REGISTRADO` sin fila nueva.
- **C5** No inscrito + `permite_entrada_excepcional=false` → `SIN_INSCRIPCION` sin `puede_excepcional`.
- **C6** Excepcional sin motivo → `MOTIVO_REQUERIDO`.
- **C7** Excepcional cuando ya se llenó (inscritos + excepcionales = aforo) → `SIN_CAPACIDAD`.
- **C8** Un PART sin membresía llamando `registrar_asistencia` → `NO_AUTORIZADO`.
- **C9** STAFF del evento A no puede registrar en el evento B.
- **C10** Reintentar el mismo `id_cliente` es idempotente (mismo resultado, 1 fila).
- **C11** Escaneo fuera de la ventana → `EVENTO_NO_ACTIVO`.
- **C12** Verificar que el escaneo en línea responde < 3 s (RNF-03) con 200 inscritos.

### Offline (RF-12, `06-offline.md`)
- **O1** Con snapshot cargado y red cortada, escanear un inscrito → verde y 1 elemento `pendiente` en cola.
- **O2** Recargar la app sin red: la cola persiste y el QR propio sigue visible.
- **O3** Volver la red → sincroniza solo y la cola queda `sincronizada` con `origen='sin_conexion'`.
- **O4** Dos dispositivos offline escanean al mismo participante → tras sincronizar hay **1** asistencia y el otro recibe `DUPLICADO`.
- **O5** Credencial revocada entre snapshot y sincronización → `rechazada` visible en P27.
- **O6** Cortar la red a mitad de un lote → sin pérdida ni duplicados al reintentar.
- **O7** Reloj del dispositivo con > 24 h de desfase → `rechazada` con detalle.

### Constancias (RF-13, RN-60…65)
- **K1** Activar plantilla desactiva la anterior del mismo tipo.
- **K2** Emitir dos veces a la misma persona no duplica (mismo folio).
- **K3** Los folios son únicos bajo emisión concurrente.
- **K4** Constancia `participante` solo a quien asistió.
- **K5** `verificar_folio` no expone correo ni teléfono.
- **K6** Descarga usa URL firmada con expiración.

### Reportes y notificaciones (RF-14/15)
- **R1** Totales de `reporte_evento` coinciden con conteos manuales. Las excepcionales salen aparte.
- **R2** RESP solo ve reportes de sus actividades.
- **R3** Un usuario no puede leer ni marcar notificaciones ajenas.

### Seguridad / RLS (RNF-01, 02, 06, 09)
Para cada tabla de `02-matriz-permisos.md`, una prueba por celda ❌ (debe fallar) y por celda ✅ (debe funcionar), ejecutadas como `anon`, PART, STAFF, RESP, ORG y ADM. Las celdas ⚙️ se prueban verificando que el `INSERT` directo falla.

## Definición de terminado (DoD) por funcionalidad
1. Criterios de aceptación anteriores con pruebas en verde.
2. Migración SQL versionada y reversible (`supabase/migrations`).
3. RLS probada con roles.
4. Tipos regenerados, sin `any`.
5. Textos de error en español según `05-contratos-rpc.md`.
6. Probado en celular (viewport 375 px) y en la PWA instalada.
7. Documentación actualizada (si cambió una regla, cambiar su `RN` primero).
