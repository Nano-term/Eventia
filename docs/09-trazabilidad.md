# 09 · Matriz de trazabilidad

Requisito → pantallas (`P#` del PDF *Interfaces de Eventia*) → tablas → RPC → secuencia (`03`) → reglas → pruebas (`08`).

| RF | Descripción corta | Pantallas | Tablas | RPC / acceso | Seq. | Reglas | Pruebas |
|---|---|---|---|---|---|---|---|
| RF-01 | Cuenta y sesión | P1, P2, P3 | `perfiles`, `credenciales` | Supabase Auth | §1 | RN-40 | A1–A3 |
| RF-02 | Solicitar evento | P12, P13 | `solicitudes_evento`, `organizaciones` | `crear_solicitud` | §2 | RN-13 | S1–S4 |
| RF-03 | Resolver solicitud | P28, P29 | `solicitudes_evento`, `eventos`, `membresias_evento` | `resolver_solicitud` | §2 | RN-10…12 | S1–S4 |
| RF-04 | CRUD y estados de evento | P14, P15, P16 | `eventos` | `cambiar_estado_evento` | §3, §15 | RN-03…08 | E1–E5 |
| RF-05 | Independiente o agrupador | P15, P16, P6 | `eventos` | triggers | §3 | RN-01, 02, 07 | E1, E2 |
| RF-06 | Invitaciones | P17 | `invitaciones`, `membresias_evento` | `crear_/revocar_/canjear_invitacion` | §4 | RN-20…25 | I1–I4 |
| RF-07 | Inscribirse / cancelar | P5–P9 | `inscripciones` | `inscribirse`, `cancelar_inscripcion` | §5, §6 | RN-30…31, 36 | N1–N8 |
| RF-08 | Aforo | P5, P6, P7, P18 | `eventos`, `inscripciones` | `inscribirse` | §5 | RN-32…35 | N1–N3 |
| RF-09 | Credencial QR | P10 | `credenciales` | `regenerar_credencial` | §7 | RN-40…43 | C1, C2 |
| RF-10 | Escaneo y asistencia | P23, P24, P25 | `asistencias`, `credenciales`, `inscripciones` | `registrar_asistencia` | §8 | RN-50…53, 57 | C3–C12 |
| RF-11 | Entrada excepcional | P25, P26 | `asistencias` | `capacidad_actividad`, `registrar_asistencia` | §9 | RN-54…56 | C5–C7 |
| RF-12 | Sin conexión | P27, P10, P24 | IndexedDB, `asistencias` | `snapshot_actividad`, `sincronizar_asistencias` | §10 | `06-offline.md` | O1–O7 |
| RF-13 | Plantillas, gafetes, constancias | P19, P20, P10, P11 | `plantillas`, `constancias` | `activar_plantilla`, `emitir_constancias`, `verificar_folio` | §11, §12 | RN-60…65 | K1–K6 |
| RF-14 | Reportes | P21, P18, P22 | vistas sobre `inscripciones`, `asistencias` | `reporte_evento` | §13 | RN-80…82 | R1, R2 |
| RF-15 | Notificaciones | P4 | `notificaciones` | `marcar_notificacion_leida` | §14 | RN-70…72 | R3 |

## Requisitos no funcionales
| RNF | Cómo se cumple | Dónde se verifica |
|---|---|---|
| RNF-01 Seguridad (RLS) | `02-matriz-permisos.md` | `08` §Seguridad |
| RNF-02 QR aleatorio y revocable | RN-41…43 | C1, C2 |
| RNF-03 < 3 s en validación | RPC única, índices en `credenciales.codigo`, `asistencias(evento_id,perfil_id)` | C12 |
| RNF-04 Offline | `06-offline.md` | O1–O7 |
| RNF-05 Usabilidad/PWA | Mobile-first, manifest, SW | DoD punto 6 |
| RNF-06 Integridad | CHECK/UNIQUE del diccionario de datos | pruebas de BD |
| RNF-07 Compatibilidad | Chrome, Edge, Firefox, Safari (la cámara/QR en Safari iOS requiere HTTPS y gesto del usuario) | E2E multi-navegador |
| RNF-08 Mantenibilidad | `10-guia-desarrollo.md` | revisión de código |
| RNF-09 Privacidad | RN-90, 91 | R3, K5 |

## Interfaces ↔ rol
Ver `02-matriz-permisos.md` §Por acción/pantalla.

## Índices de BD recomendados (RNF-03)
- `credenciales(codigo)` único (ya existe como UK).
- `asistencias(evento_id, perfil_id)` único (ya existe), `asistencias(id_cliente)` único.
- `inscripciones(evento_id, estado)` para conteos de aforo.
- `eventos(evento_padre_id)`, `eventos(estado, inicia_en)` para catálogo.
- `membresias_evento(usuario_id)`, `membresias_evento(evento_id, rol)` para RLS.
- `notificaciones(perfil_id, leida, creado_en DESC)`.
