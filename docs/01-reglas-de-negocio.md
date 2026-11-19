# 01 · Reglas de negocio y decisiones

Fuente de verdad para la lógica. Cada regla tiene ID (`RN-xx`) para citarla en código, tests y prompts.

Leyenda: **[PROPUESTA]** = decisión nueva tomada para cerrar un vacío de la propuesta. Revisar y cambiar a **[CONFIRMADA]** o corregir antes de implementar. Todo lo no marcado viene de `Eventia_Propuesta_Completa_Corregida.docx`.

## Glosario (no confundir)
| Término | Tabla | Significado |
|---|---|---|
| Inscripción | `inscripciones` | Reserva de lugar en una actividad. Ocupa aforo. |
| Credencial | `credenciales` | QR personal (1 por usuario). Solo identifica a la persona. |
| Asistencia | `asistencias` | Acceso efectivo registrado tras escanear. |
| Evento raíz | `eventos` con `evento_padre_id IS NULL` | Evento independiente o agrupador. |
| Actividad | `eventos` con `evento_padre_id` no nulo | Hija de un agrupador. Misma tabla. |

## Eventos y jerarquía
- **RN-01** Solo hay 1 nivel: un agrupador (`es_agrupador=true`) contiene actividades; una actividad no puede ser agrupador ni tener hijos.
- **RN-02** `evento_padre_id` debe apuntar a un evento con `es_agrupador=true` y sin padre. Se valida con trigger.
- **RN-03** Un evento no puede ser su propio padre. `termina_en > inicia_en`. `aforo_maximo` es nulo (sin límite) o `> 0`.
- **RN-04** Estados: `borrador → publicado → cerrado`, y `cancelado` desde `borrador` o `publicado`. Ver `04-estados.md`.
- **RN-05** Solo los eventos `publicado` aparecen en el catálogo público. `borrador` solo lo ven miembros del evento y administradores.
- **RN-06** Una actividad no puede estar `publicado` si su padre no lo está. Cancelar/cerrar un agrupador cancela/cierra en cascada sus actividades. **[PROPUESTA]**
- **RN-07** Las fechas de una actividad deben quedar dentro del rango de su evento padre. **[PROPUESTA]**
- **RN-08** Solo un evento con solicitud aprobada puede crearse/publicarse (OE2). El evento nace al aprobar la solicitud (`solicitudes_evento.evento_id`), en estado `borrador`.

## Solicitudes
- **RN-10** Estados: `pendiente → aprobada | rechazada`. Resolución irreversible. Para reintentar, se crea una nueva solicitud.
- **RN-11** Solo `rol_plataforma='administrador'` resuelve. Al resolver se guardan `revisado_por`, `revisado_en`, `observaciones`.
- **RN-12** Al aprobar, en una sola transacción: crear el evento raíz (`creado_por` = solicitante, `organizacion_id`), crear la membresía `organizador_principal` del solicitante, vincular `evento_id` y notificar. Al rechazar, `observaciones` es obligatorio. **[PROPUESTA]** (obligatoriedad)
- **RN-13** Cualquier usuario autenticado puede solicitar. Si la organización no existe, se crea en el mismo paso (`creado_por` = solicitante). **[PROPUESTA]**

## Membresías e invitaciones
- **RN-20** Roles de gestión por evento: `organizador_principal`, `responsable_actividad`, `personal_apoyo`. El participante no requiere membresía.
- **RN-21** `(evento_id, usuario_id, rol)` es único. Una invitación nunca otorga `organizador_principal`.
- **RN-22** Canjear un código válido = `estado='activa'` + no vencido (`expira_en`) + usos restantes (`usos_maximos` nulo = ilimitado). Se cuenta con `COUNT(membresias WHERE invitacion_id)`. Canjear dos veces el mismo código por el mismo usuario no duplica ni consume uso.
- **RN-23** Revocar una invitación no elimina membresías (`invitacion_id` queda nulo según el modelo).
- **RN-24** Una membresía en el evento raíz se hereda a todas sus actividades. Una membresía en una actividad aplica solo a ella. **[PROPUESTA]**
- **RN-25** `responsable_actividad` solo se asigna sobre actividades (no sobre el raíz). **[PROPUESTA]**

## Inscripción y aforo
- **RN-30** Un participante tiene como máximo 1 inscripción por actividad: `(evento_id, perfil_id)` único. Si canceló y vuelve a inscribirse, se reactiva el mismo registro (`estado='confirmada'`, `cancelada_en=NULL`). **[PROPUESTA]**
- **RN-31** Solo se inscribe en eventos `publicado` con `requiere_inscripcion=true`, antes de `termina_en`.
- **RN-32** **Aforo por actividad:** `COUNT(inscripciones confirmadas) < aforo_maximo` (nulo = sin límite). **[PROPUESTA]** `cupos_restantes = aforo_maximo − confirmadas`.
- **RN-33** **Aforo general (evento raíz):** si el raíz tiene `aforo_maximo`, limita el número de **participantes distintos** con al menos una inscripción confirmada en cualquiera de sus actividades. Si el participante ya cuenta en el general, inscribirse a otra actividad no consume cupo general. **[PROPUESTA]**
- **RN-34** Para inscribirse a una actividad deben cumplirse RN-32 **y** RN-33. Si falla alguna, se rechaza con el código de error correspondiente (`ACTIVIDAD_LLENA` / `EVENTO_LLENO`).
- **RN-35** La inscripción se hace dentro de una transacción con bloqueo (`SELECT … FOR UPDATE` sobre la fila del evento/actividad y del raíz) para evitar sobrecupo por concurrencia. Debe ser una función RPC, nunca un `INSERT` directo desde el cliente.
- **RN-36** Cancelar inscripción: pone `estado='cancelada'` y libera cupo. No se puede cancelar si ya existe asistencia para esa inscripción. **[PROPUESTA]**
- **RN-37** Si `requiere_inscripcion=false`, no hay inscripción y cualquier credencial activa puede registrarse. Se guarda como `tipo_registro='inscrito'` con `inscripcion_id` nulo. **[POR CONFIRMAR]** (D2: el modelo solo permite `inscripcion_id` nulo en excepcionales, hay que relajar ese CHECK).

## Credencial y QR
- **RN-40** Cada perfil tiene exactamente 1 credencial, creada automáticamente al crearse el perfil (trigger). **[PROPUESTA]**
- **RN-41** `codigo` es aleatorio (≥ 32 bytes de entropía, base64url/hex), distinto de cualquier ID interno. El QR contiene **solo** el `codigo`, sin datos personales.
- **RN-42** El participante puede regenerar su código (revoca el anterior y emite uno nuevo con el mismo `perfil_id`: se actualiza el registro). Una credencial `revocada` nunca valida accesos. **[PROPUESTA]**
- **RN-43** El QR es estático (no rota) para poder mostrarse sin conexión. La seguridad se apoya en la entropía del código y la revocación.

## Asistencia
- **RN-50** Una asistencia por `(evento_id, perfil_id)`. Un segundo escaneo se responde como `YA_REGISTRADO` (pantalla roja) sin insertar.
- **RN-51** Asiste solo si el escáner pertenece al personal autorizado: membresía `personal_apoyo`, `responsable_actividad` de esa actividad u `organizador_principal` del raíz (RN-24).
- **RN-52** Resultado de escanear (ver `03-diagramas-secuencia.md` §8). Códigos: `OK`, `QR_INVALIDO`, `CREDENCIAL_REVOCADA`, `YA_REGISTRADO`, `SIN_INSCRIPCION`, `EVENTO_NO_ACTIVO`.
- **RN-53** Ventana de acceso: se acepta escaneo desde `inicia_en − 2 h` hasta `termina_en`. **[PROPUESTA]** (parámetro configurable `VENTANA_ACCESO_MIN`).
- **RN-54** `SIN_INSCRIPCION` ofrece "Entrada excepcional" solo si `permite_entrada_excepcional=true` **y** hay capacidad.
- **RN-55** **Capacidad para excepcional:** `aforo_maximo IS NULL OR (inscripciones confirmadas + asistencias excepcionales) < aforo_maximo`, y respeta además RN-33 si el raíz tiene aforo general. **[PROPUESTA]**
- **RN-56** Excepcional: `tipo_registro='excepcional'`, `inscripcion_id` nulo, `motivo_excepcion` obligatorio (no vacío).
- **RN-57** `escaneado_en` = hora del dispositivo; `sincronizado_en` = hora del servidor al insertar; `origen` = `en_linea` o `sin_conexion`; `id_cliente` = UUID generado en el dispositivo (idempotencia).

## Modo sin conexión
Ver `06-offline.md`. Regla clave: el servidor es la autoridad final; el dispositivo valida contra un *snapshot* descargado.

## Constancias, gafetes y plantillas
- **RN-60** Solo hay una plantilla `activa` por `(evento_id, tipo)`. Activar una desactiva la anterior. **[PROPUESTA]**
- **RN-61** Constancia: única por `(evento_id, perfil_id, concepto)`. Folio único con formato `EVT-AAAA-XXXXXX` (6 caracteres alfanuméricos aleatorios sin ambiguos). **[PROPUESTA]**
- **RN-62** Destinatarios de `concepto='participante'`: quienes tienen asistencia en ese evento/actividad. `concepto='responsable'`: miembros `responsable_actividad`. Las constancias del raíz requieren asistencia a al menos 1 actividad. **[PROPUESTA]**
- **RN-63** Emitir es idempotente: si ya existe la constancia, se devuelve la existente, no se duplica.
- **RN-64** **Gafetes [PROPUESTA]:** no se persisten. Se generan en PDF bajo demanda a partir de la plantilla `gafete` activa + datos del perfil + su QR. No tienen folio. (El modelo no tiene tabla de gafetes; la pantalla 20 listará solo constancias emitidas con folio.)
- **RN-65** Verificación pública de folio (`/verificar/:folio`) para validar autenticidad. **[PROPUESTA]**

## Notificaciones
- **RN-70** Solo dentro de la app (sin correo/push). Se crean desde funciones de BD (no desde el cliente), en la misma transacción del hecho que las origina.
- **RN-71** Catálogo de `tipo`: `solicitud_resuelta`, `solicitud_nueva` (a administradores), `inscripcion_confirmada`, `inscripcion_cancelada`, `evento_cancelado`, `evento_modificado`, `constancia_emitida`, `colaborador_unido` (al organizador). **[PROPUESTA]**
- **RN-72** El usuario solo puede marcar como leída las propias (`leida=true`); no puede editar título/mensaje.

## Reportes
- **RN-80** Sin tabla propia: se calculan con vistas/RPC (`reporte_evento`, `reporte_actividad`). Ver `05-contratos-rpc.md`.
- **RN-81** `porcentaje_asistencia = asistentes / inscritos_confirmados × 100`. Las excepcionales se muestran aparte (no entran al denominador). **[PROPUESTA]**
- **RN-82** Exportar: CSV generado en el cliente a partir del resultado del reporte.

## Privacidad
- **RN-90** Un usuario solo ve su perfil, inscripciones, constancias, credencial y notificaciones. Los miembros ven datos de inscritos/asistentes **solo** de los eventos que gestionan, y solo nombre, correo y estado (no teléfono). **[PROPUESTA]**
- **RN-91** El administrador general ve organizaciones, eventos y usuarios (pantalla 30) con datos mínimos.

## Cambios al esquema que estas reglas implican
1. Trigger: perfil creado → crear `credenciales`.
2. Trigger: `evento_padre_id` válido (RN-02), fechas (RN-07), cascada de estado (RN-06).
3. Índice único parcial en `plantillas (evento_id, tipo) WHERE activa` (RN-60).
4. Definir RN-37 (eventos sin inscripción) y, si se desea, un campo `sobrecupo boolean` en `asistencias` para marcar excepcionales offline que superaron aforo al sincronizar (ver `06-offline.md`).
5. Funciones RPC listadas en `05-contratos-rpc.md`.
