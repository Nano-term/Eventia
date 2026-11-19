# 05 · Contratos de funciones RPC y rutas

Todas las funciones son `SECURITY DEFINER` en PostgreSQL (schema `public`), validan `auth.uid()` y lanzan errores con `RAISE EXCEPTION USING ERRCODE='P0001', MESSAGE='<CODIGO>'`. El cliente traduce `<CODIGO>` a texto en español (tabla de errores abajo).

Tipos de TypeScript se generan con `supabase gen types typescript`; no escribirlos a mano.

## Funciones

| RPC | Entradas | Salida | Valida (RN) | Rol |
|---|---|---|---|---|
| `crear_solicitud` | organizacion (id o datos nuevos), nombre_evento, tipo_evento, descripcion, fecha_inicio_estimada, fecha_fin_estimada | `solicitud_id` | RN-13 | PART |
| `resolver_solicitud` | solicitud_id, decision (`aprobar`/`rechazar`), observaciones | `evento_id?` | RN-10…12 | ADM |
| `cambiar_estado_evento` | evento_id, nuevo_estado | evento | RN-04, 06 | ORG |
| `crear_invitacion` | evento_id, rol, usos_maximos?, expira_en? | `codigo` | RN-21 | ORG |
| `revocar_invitacion` | invitacion_id | ok | RN-23 | ORG |
| `canjear_invitacion` | codigo | `{evento_id, rol}` | RN-22 | PART |
| `inscribirse` | evento_id | `{inscripcion, cupos_restantes}` | RN-30…35 | PART |
| `cancelar_inscripcion` | evento_id | ok | RN-36 | PART |
| `regenerar_credencial` | — | `codigo` | RN-42 | PART |
| `snapshot_actividad` | evento_id | roster para offline | RN-51 | STAFF/RESP/ORG |
| `capacidad_actividad` | evento_id | `{permite_excepcional, capacidad_disponible}` | RN-55 | STAFF/RESP/ORG |
| `registrar_asistencia` | codigo, evento_id, id_cliente, escaneado_en, tipo (`inscrito`/`excepcional`), motivo? | `{resultado, perfil_nombre?, puede_excepcional?}` | RN-50…57 | STAFF/RESP/ORG |
| `sincronizar_asistencias` | lote[] de entradas de `registrar_asistencia` | `[{id_cliente, resultado, detalle?}]` | RN-57, `06-offline.md` | STAFF/RESP/ORG |
| `activar_plantilla` | plantilla_id | ok | RN-60 | ORG |
| `destinatarios_constancia` | evento_id, filtro (`todos`/`asistieron`), concepto | perfiles[] | RN-62 | ORG |
| `emitir_constancias` | evento_id, plantilla_id, perfil_ids[], concepto | `[{perfil_id, folio, constancia_id}]` | RN-61…63 | ORG |
| `verificar_folio` | folio | `{valido, nombre, evento, fecha}` | RN-65 | ANON |
| `reporte_evento` | evento_id, actividad_id? | totales + filas | RN-80…81 | ORG/RESP |
| `marcar_notificacion_leida` | id (o todas) | ok | RN-72 | PART |

### Resultado de `registrar_asistencia`
```ts
type ResultadoEscaneo =
  | 'OK' | 'QR_INVALIDO' | 'CREDENCIAL_REVOCADA' | 'YA_REGISTRADO'
  | 'SIN_INSCRIPCION' | 'EVENTO_NO_ACTIVO' | 'SIN_CAPACIDAD' | 'NO_AUTORIZADO';
```
Color de UI: `OK` verde · `SIN_INSCRIPCION` amarillo (con botón excepcional si `puede_excepcional`) · el resto rojo.

## Códigos de error (`MESSAGE`)
| Código | Texto UI |
|---|---|
| `NO_AUTENTICADO` | Inicia sesión para continuar. |
| `SIN_PERMISO` | No tienes permiso para esta acción. |
| `ACTIVIDAD_LLENA` | La actividad ya no tiene cupo. |
| `EVENTO_LLENO` | El evento alcanzó su aforo general. |
| `YA_INSCRITO` | Ya estás inscrito. |
| `YA_ASISTIO` | No puedes cancelar: ya registraste asistencia. |
| `INSCRIPCION_CERRADA` | Las inscripciones están cerradas. |
| `TRANSICION_INVALIDA` | Ese cambio de estado no está permitido. |
| `PADRE_NO_AGRUPADOR` | El evento padre debe ser agrupador. |
| `FECHAS_FUERA_DE_RANGO` | Las fechas deben estar dentro del evento. |
| `INVITACION_NO_EXISTE` / `_REVOCADA` / `_VENCIDA` / `_AGOTADA` | Código de invitación no válido (con motivo). |
| `OBSERVACIONES_REQUERIDAS` | Escribe el motivo del rechazo. |
| `MOTIVO_REQUERIDO` | El motivo de la entrada excepcional es obligatorio. |
| `SOLICITUD_YA_RESUELTA` | Esta solicitud ya fue resuelta. |

## Rutas de la PWA (React Router)
| Ruta | Pantalla | Acceso |
|---|---|---|
| `/login`, `/registro` | P1, P2 | ANON |
| `/` , `/eventos` | P5 Catálogo | ANON |
| `/eventos/:id` | P6 Detalle | ANON |
| `/actividades/:id` | P7 | PART |
| `/perfil` | P3 | PART |
| `/notificaciones` | P4 | PART |
| `/mis-inscripciones` | P8 | PART |
| `/calendario` | P9 | PART |
| `/gafete` | P10 | PART |
| `/constancias` | P11 | PART |
| `/solicitar-evento`, `/mis-solicitudes` | P12, P13 | PART |
| `/organizar` | P14 | miembro |
| `/organizar/nuevo`, `/organizar/:id/editar` | P15 | ORG |
| `/organizar/:id/actividades` | P16 | ORG |
| `/organizar/:id/colaboradores` | P17 | ORG |
| `/organizar/:id/inscritos` | P18 | STAFF/RESP/ORG |
| `/organizar/:id/plantillas` | P19 | ORG |
| `/organizar/:id/emision` | P20 | ORG |
| `/organizar/:id/reportes` | P21 | RESP/ORG |
| `/mis-actividades` | P22 | RESP |
| `/acceso` | P23 | STAFF/RESP/ORG |
| `/acceso/escaner` | P24 + P25 (resultado) + P26 (modal) | idem |
| `/acceso/cola` | P27 | idem |
| `/admin/solicitudes`, `/admin/solicitudes/:id` | P28, P29 | ADM |
| `/admin/supervision` | P30 | ADM |
| `/verificar/:folio` | verificación pública | ANON |
| `/unirse/:codigo` | canje de invitación | PART |
