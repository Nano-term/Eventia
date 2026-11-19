# 06 · Especificación del modo sin conexión (PWA)

Cubre RF-12, RNF-04, OE6. Estado: **[PROPUESTA]** salvo lo tomado de la propuesta (`id_cliente`, `origen`, `escaneado_en`, `sincronizado_en`).

## Qué funciona sin conexión
| Función | Sin conexión | Requisito previo |
|---|---|---|
| Ver mi QR (P10) | ✅ | Haber abierto la app con conexión alguna vez (credencial en caché) |
| Ver mi calendario/inscripciones | ✅ solo lectura | Datos cacheados |
| Escanear y registrar asistencia | ✅ | Haber pulsado "Iniciar escaneo" con conexión (descarga el *snapshot*) |
| Inscribirse, cancelar, emitir, reportes | ❌ | Requieren servidor (aforo) |

## Almacenamiento (IndexedDB)
| Store | Clave | Contenido |
|---|---|---|
| `credencial` | `perfil_id` | `codigo`, `nombre`, `estado`, `cacheado_en` |
| `snapshot_actividad` | `evento_id` | `descargado_en`, aforo, `permite_excepcional`, `inscritos_confirmados`, filas `{codigo, perfil_id, nombre, inscrito, ya_asistio}` |
| `cola_asistencias` | `id_cliente` (UUID) | `evento_id`, `codigo`, `perfil_id?`, `tipo_registro`, `motivo?`, `escaneado_en`, `estado_local` (`pendiente`/`sincronizando`/`sincronizada`/`rechazada`), `intentos`, `resultado?` |

Service Worker: precache de la app shell (Workbox) y `registerType: 'autoUpdate'`. Los datos van en IndexedDB, no en la caché HTTP.

## Validación local (algoritmo)
```
al escanear codigo:
  fila = snapshot.filas.find(codigo)
  si no existe                     → ROJO QR_INVALIDO (local)
  si fila.ya_asistio o en cola      → ROJO YA_REGISTRADO
  si fila.inscrito                  → encolar(tipo=inscrito) → VERDE
  si no inscrito:
      si permite_excepcional y capacidad_local > 0 → AMARILLO (permitir excepcional)
      si no                                         → ROJO SIN_INSCRIPCION
capacidad_local = aforo - (inscritos_confirmados + excepcionales_en_cola + excepcionales_del_snapshot)
```
Una credencial emitida después del snapshot no estará en él: se informa "no encontrada, sincroniza cuando tengas conexión" (rojo). Aceptable.

## Sincronización
- Disparadores: evento `online`, botón "Sincronizar ahora" (P27), Background Sync si el navegador lo soporta, y al abrir la app.
- Envío por lotes (≤ 50) a `sincronizar_asistencias`. Orden por `escaneado_en` ascendente.
- **Idempotencia:** `id_cliente` único. Reenviar el mismo elemento nunca duplica; devuelve `DUPLICADO` y se marca sincronizada.
- Reintentos con *backoff* exponencial (5 s, 15 s, 45 s, 2 min…). Errores de red no cambian el estado a `rechazada`.

## Reglas de conflicto (el servidor manda)
El servidor re-valida cada elemento **con la hora original** (`escaneado_en`) y el estado actual:

| Situación al sincronizar | Resultado | Acción |
|---|---|---|
| Todo válido | `OK` | INSERT con `origen='sin_conexion'`, `sincronizado_en=now()` |
| `id_cliente` ya existe | `DUPLICADO` | Marcar sincronizada |
| Ya hay asistencia `(evento, perfil)` de otro dispositivo | `DUPLICADO` | No insertar. Marcar sincronizada |
| Inscripción cancelada después del snapshot | `OK` | El hecho físico ocurrió. Se registra como `excepcional` con motivo automático "Inscripción cancelada antes de sincronizar" **[POR CONFIRMAR]** |
| Credencial revocada después del snapshot | `RECHAZADO (CREDENCIAL_REVOCADA)` | Cola `rechazada` para revisión |
| Excepcional que supera aforo al sincronizar | `OK` con `sobrecupo=true` | El acceso ya ocurrió. Se registra y se marca para revisión (requiere columna nueva, ver `01` cambios al esquema) |
| Escáner sin autorización vigente | `RECHAZADO (NO_AUTORIZADO)` | Cola `rechazada` |
| Evento cancelado | `RECHAZADO (EVENTO_NO_ACTIVO)` | Cola `rechazada` |

Los rechazos nunca se borran solos: aparecen en P27 con motivo para que el personal u organizador decida.

## Reloj del dispositivo
`escaneado_en` es la hora del dispositivo y puede estar mal. El servidor guarda también `sincronizado_en`. Si `|escaneado_en − now()| > 24 h` al sincronizar, o el escaneo es anterior a `inicia_en − ventana`, se rechaza como `EVENTO_NO_ACTIVO` con detalle "hora del dispositivo incorrecta".

## Pruebas obligatorias
Ver `08-plan-pruebas.md` §Offline. Mínimo: escanear sin red, recargar la app, volver la red y verificar 0 duplicados con 2 dispositivos escaneando al mismo participante.
