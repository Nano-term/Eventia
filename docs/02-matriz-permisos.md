# 02 · Matriz de permisos (base de las políticas RLS)

Roles: **ADM** administrador general (`perfiles.rol_plataforma='administrador'`) · **ORG** organizador_principal · **RESP** responsable_actividad · **STAFF** personal_apoyo · **PART** cualquier usuario autenticado (participante) · **ANON** sin sesión.

Herencia (RN-24): ORG/STAFF en el raíz aplican a todas sus actividades. RESP solo a sus actividades asignadas.
"Propio" = fila cuyo `perfil_id`/`usuario_id` = `auth.uid()`.

Convención: ✅ permitido · 🔒 solo propio · 🎯 solo si es miembro del evento con ese rol · ❌ denegado · ⚙️ solo vía RPC (nunca escritura directa).

## Por tabla
| Tabla | ANON | PART | STAFF | RESP | ORG | ADM |
|---|---|---|---|---|---|---|
| `perfiles` | ❌ | SELECT/UPDATE 🔒 (sin cambiar `rol_plataforma`) | SELECT mínimo de inscritos de su evento | igual | igual | SELECT todos |
| `organizaciones` | ❌ | SELECT las propias/relacionadas, INSERT ⚙️ | SELECT la del evento | SELECT la del evento | SELECT/UPDATE la suya | ✅ |
| `solicitudes_evento` | ❌ | INSERT ⚙️, SELECT 🔒 | ❌ | ❌ | SELECT 🔒 | SELECT todas, UPDATE ⚙️ (resolver) |
| `eventos` (publicado) | SELECT ✅ | SELECT ✅ | SELECT ✅ | SELECT ✅ | ✅ | ✅ |
| `eventos` (borrador/cerrado/cancelado) | ❌ | ❌ | SELECT 🎯 | SELECT 🎯 | SELECT/UPDATE 🎯 | SELECT |
| `eventos` INSERT | ❌ | ❌ | ❌ | ❌ | actividades del raíz que administra 🎯 | ⚙️ (al aprobar) |
| `eventos` DELETE | ❌ | ❌ | ❌ | ❌ | solo `borrador` sin inscripciones | ✅ |
| `membresias_evento` | ❌ | SELECT 🔒 | SELECT 🔒 | SELECT del evento 🎯 | SELECT/DELETE 🎯 | SELECT |
| `invitaciones` | ❌ | ❌ (canje ⚙️) | ❌ | ❌ | CRUD 🎯 | SELECT |
| `credenciales` | ❌ | SELECT/regenerar 🔒 ⚙️ | lectura por código ⚙️ | ⚙️ | ⚙️ | ❌ |
| `inscripciones` | ❌ | SELECT 🔒, INSERT/UPDATE ⚙️ | SELECT 🎯 | SELECT 🎯 (sus actividades) | SELECT 🎯 | SELECT |
| `asistencias` | ❌ | SELECT 🔒 | INSERT ⚙️ SELECT 🎯 | INSERT ⚙️ SELECT 🎯 | SELECT 🎯, INSERT ⚙️ | SELECT |
| `plantillas` | ❌ | ❌ | ❌ | ❌ | CRUD 🎯 | SELECT |
| `constancias` | ❌ | SELECT 🔒 | ❌ | SELECT 🎯 | SELECT 🎯, INSERT ⚙️ | SELECT |
| `notificaciones` | ❌ | SELECT 🔒, UPDATE `leida` 🔒 | igual | igual | igual | igual |

Regla general: **ningún cliente inserta directamente** en `inscripciones`, `asistencias`, `constancias`, `notificaciones`, `membresias_evento`, `solicitudes_evento` (resolución) ni `credenciales`. Todo pasa por RPC `SECURITY DEFINER` que validan reglas (`05-contratos-rpc.md`).

## Por acción / pantalla
| # | Pantalla | ANON | PART | STAFF | RESP | ORG | ADM |
|---|---|---|---|---|---|---|---|
| 1-2 | Login / Registro | ✅ | — | — | — | — | — |
| 3-4 | Perfil / Notificaciones | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 5-6 | Catálogo / Detalle evento | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 7-11 | Actividad, inscripciones, calendario, gafete, constancias | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 12-13 | Solicitar evento / Estado | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 14 | Mis eventos (panel) | ❌ | ❌ | solo si tiene membresía | ✅ 🎯 | ✅ 🎯 | ❌ |
| 15 | Crear/editar evento | ❌ | ❌ | ❌ | ❌ (solo su actividad: editar horario/descripción) | ✅ | ❌ |
| 16 | Gestión de actividades | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| 17 | Colaboradores/invitaciones | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| 18 | Lista de inscritos | ❌ | ❌ | ✅ | ✅ (sus actividades) | ✅ | ❌ |
| 19-20 | Plantillas / Emisión | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| 21 | Reportes | ❌ | ❌ | ❌ | ✅ (sus actividades) | ✅ | ❌ |
| 22 | Mis actividades asignadas | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| 23-27 | Control de acceso (selección, escáner, resultado, excepcional, cola offline) | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |
| 28-30 | Solicitudes admin / Supervisión | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

## Reglas transversales para RLS
1. Función auxiliar `es_miembro(evento_id, roles[])` que considera herencia raíz→actividad.
2. Función auxiliar `es_admin()` = `perfiles.rol_plataforma='administrador'` del `auth.uid()`.
3. Un usuario no puede auto-asignarse `rol_plataforma='administrador'` (columna protegida; solo cambia vía SQL manual/service role).
4. Las lecturas de inscritos/asistentes para miembros devuelven solo `nombre_completo`, `correo`, estado y fechas (RN-90). Usar vista dedicada, no `SELECT` sobre `perfiles`.
5. `anon` solo tiene `SELECT` sobre la vista pública de eventos `publicado` (sin campos internos como `creado_por`).
6. Por defecto todas las tablas con `ENABLE ROW LEVEL SECURITY` y sin políticas = denegado.
