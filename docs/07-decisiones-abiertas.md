# 07 · Decisiones abiertas (responder antes de implementar)

Cada una tiene una recomendación ya reflejada como **[PROPUESTA]** en los demás documentos. Marca aquí tu decisión y actualiza el documento citado.

| # | Decisión | Recomendación | Impacta | Estado |
|---|---|---|---|---|
| D1 | **Aforo general (RN-33):** ¿cuenta participantes distintos en el raíz, o suma de todas las inscripciones? | Participantes distintos | `01`, `inscribirse` | ⬜ |
| D2 | **Eventos sin inscripción (RN-37):** ¿cómo se registra asistencia si `requiere_inscripcion=false`? | Permitir `tipo_registro='inscrito'` con `inscripcion_id` nulo en esos eventos y ajustar el CHECK | esquema, `registrar_asistencia` | ⬜ |
| D3 | **Gafetes (RN-64):** ¿se registran o solo se generan al vuelo? | Al vuelo, sin tabla ni folio | esquema, P20 | ⬜ |
| D4 | **Dónde se genera el PDF** (cliente con `pdf-lib` vs Edge Function) | Cliente para gafete propio. Edge Function para emisión masiva (evita bloquear el navegador) | `03` §11 | ⬜ |
| D5 | **Ventana de acceso (RN-53):** ¿cuánto antes de `inicia_en` se admite el escaneo? | 2 h antes, hasta `termina_en` | `registrar_asistencia` | ⬜ |
| D6 | **Herencia de roles (RN-24):** ¿membresía del raíz aplica a las actividades? | Sí | RLS | ⬜ |
| D7 | **Reinscripción tras cancelar (RN-30)** | Reactivar el mismo registro | `inscribirse` | ⬜ |
| D8 | **Cancelación en cascada (RN-06)** y notificar a inscritos | Sí | `cambiar_estado_evento` | ⬜ |
| D9 | **Excepcional offline que se pasa del aforo** (`sobrecupo`) | Aceptar y marcar para revisión (columna nueva `sobrecupo`) | esquema, `06` | ⬜ |
| D10 | **Constancia del raíz:** ¿requiere asistir a ≥1 actividad, a un mínimo %, o a todas? | ≥ 1 actividad, parametrizable después | `emitir_constancias` | ⬜ |
| D11 | **Nombre de la PWA / identidad visual** (logo, colores) | Definir antes de construir la UI | UI | ⬜ |
| D12 | **Datos personales:** aviso de privacidad y retención | Redactar aviso básico y borrar credencial/foto al eliminar cuenta | legal | ⬜ |
| D13 | **Zona horaria:** almacenar UTC, mostrar `America/Merida` | Sí | UI | ⬜ |
| D14 | **Región/plan de Supabase** y límites de Storage | Plan gratuito para curso. Verificar límites de Storage y de conexiones | despliegue | ⬜ |

## Vacíos conocidos de la propuesta original
1. El campo `tipo` de `eventos` incluye `semana cultural, feria, congreso…`, pero un agrupador y una actividad comparten valores. Definir catálogo exacto (CHECK).
2. Falta un `slug` o URL amigable del evento. Opcional.
3. No hay `descripcion` de actividad separada del evento (misma tabla). Se asume la columna `descripcion`.
4. No se define qué ve un participante inscrito en un evento `cerrado`. Propuesta: solo lectura.
