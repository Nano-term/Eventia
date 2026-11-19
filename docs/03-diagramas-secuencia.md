# 03 · Diagramas de secuencia

Cada flujo indica requisitos (`RF`), pantallas (`P#` del PDF de interfaces), reglas (`RN`) y RPC (`05-contratos-rpc.md`).

Participantes comunes: **UI** (React PWA) · **SW** (Service Worker + IndexedDB) · **API** (Supabase PostgREST/RPC con RLS) · **Auth** (Supabase Auth) · **DB** (PostgreSQL) · **ST** (Storage).

Renderizar con https://mermaid.live, `mmdc`, o directo en GitHub/VS Code.

---

## 1. Registro e inicio de sesión
RF-01 · P1, P2, P3 · RN-40

```mermaid
sequenceDiagram
    actor U as Usuario
    participant UI
    participant Auth
    participant DB
    U->>UI: Completa registro (nombre, correo, telefono?, contraseña)
    UI->>UI: Valida formato y que contraseñas coincidan
    UI->>Auth: signUp(correo, contraseña, metadata nombre/telefono)
    Auth-->>UI: sesión o error (correo ya usado)
    Auth->>DB: Trigger on auth.users INSERT
    DB->>DB: INSERT perfiles (id = auth.uid, rol_plataforma = usuario)
    DB->>DB: INSERT credenciales (perfil_id, codigo aleatorio, estado = activa)
    UI-->>U: Redirige al catálogo (P5)
    Note over U,DB: Inicio de sesión
    U->>UI: Correo y contraseña
    UI->>Auth: signInWithPassword
    alt credenciales válidas
        Auth-->>UI: sesión (JWT + refresh)
        UI->>API: SELECT perfiles WHERE id = uid
        UI-->>U: Navega según rol y destino previo
    else inválidas
        Auth-->>UI: error
        UI-->>U: Mensaje genérico sin revelar cuál campo falló
    end
```

## 2. Solicitud de evento y resolución por administrador
RF-02, RF-03 · P12, P13, P28, P29 · RN-08, RN-10…13, RN-70

```mermaid
sequenceDiagram
    actor O as Solicitante
    actor A as Administrador
    participant UI
    participant API
    participant DB
    O->>UI: Llena formulario (organización, nombre, tipo, descripción, fechas)
    UI->>API: rpc crear_solicitud(...)
    API->>DB: INSERT organizaciones si es nueva
    API->>DB: INSERT solicitudes_evento (estado = pendiente)
    API->>DB: INSERT notificaciones para cada administrador (solicitud_nueva)
    API-->>UI: solicitud_id
    UI-->>O: Muestra P13 con estado pendiente

    A->>UI: Abre bandeja P28 filtrando por pendiente
    UI->>API: SELECT solicitudes_evento (RLS: solo ADM)
    A->>UI: Abre detalle P29
    alt Aprueba
        UI->>API: rpc resolver_solicitud(id, aprobar, observaciones?)
        API->>DB: Verifica ADM y estado = pendiente
        API->>DB: INSERT eventos (raiz, estado = borrador, creado_por = solicitante)
        API->>DB: INSERT membresias_evento (organizador_principal)
        API->>DB: UPDATE solicitudes_evento (aprobada, evento_id, revisado_por, revisado_en)
        API->>DB: INSERT notificaciones (solicitud_resuelta al solicitante)
        API-->>UI: evento_id
    else Rechaza
        UI->>API: rpc resolver_solicitud(id, rechazar, observaciones)
        API->>DB: Exige observaciones no vacías
        API->>DB: UPDATE solicitudes_evento (rechazada)
        API->>DB: INSERT notificaciones (solicitud_resuelta)
        API-->>UI: ok
    end
    Note over API,DB: Todo en una sola transacción. Si falla algo, nada se guarda.
```

## 3. Crear evento, actividades y publicar
RF-04, RF-05 · P14, P15, P16 · RN-01…07

```mermaid
sequenceDiagram
    actor O as Organizador
    participant UI
    participant API
    participant DB
    participant ST as Storage
    O->>UI: Edita datos del evento raíz (P15)
    opt Sube portada
        UI->>ST: upload cover (ruta eventos/{id}/portada)
        ST-->>UI: ruta
    end
    UI->>API: UPDATE eventos SET ... (RLS: ORG del evento)
    API->>DB: Valida CHECK (fechas, aforo) y triggers
    DB-->>API: ok o error de constraint
    API-->>UI: evento actualizado

    O->>UI: Nueva actividad (P16)
    UI->>API: INSERT eventos (evento_padre_id = raíz, estado = borrador)
    API->>DB: Trigger valida padre agrupador (RN-02) y fechas dentro del padre (RN-07)
    alt válido
        DB-->>UI: actividad creada
    else inválido
        DB-->>UI: error con código (PADRE_NO_AGRUPADOR, FECHAS_FUERA_DE_RANGO)
    end

    O->>UI: Asigna responsable (usuario ya miembro)
    UI->>API: INSERT membresias_evento (rol = responsable_actividad, evento_id = actividad)

    O->>UI: Publicar
    UI->>API: rpc cambiar_estado_evento(id, publicado)
    API->>DB: Verifica transición válida (RN-04) y padre publicado (RN-06)
    DB-->>UI: ok
    Note over UI,DB: Ahora aparece en el catálogo P5 para ANON
```

## 4. Invitación y canje de código
RF-06 · P17 · RN-20…23

```mermaid
sequenceDiagram
    actor O as Organizador
    actor C as Colaborador
    participant UI
    participant API
    participant DB
    O->>UI: Genera código (rol, usos máximos, vencimiento)
    UI->>API: rpc crear_invitacion(evento_id, rol, usos?, expira?)
    API->>DB: Verifica ORG y rol distinto de organizador_principal
    API->>DB: INSERT invitaciones (codigo aleatorio, estado = activa)
    API-->>UI: codigo
    O-->>C: Comparte el código fuera de la app

    C->>UI: Ingresa código
    UI->>API: rpc canjear_invitacion(codigo)
    API->>DB: SELECT invitacion FOR UPDATE
    alt activa, no vencida, con usos disponibles
        API->>DB: INSERT membresias_evento ON CONFLICT DO NOTHING
        API->>DB: INSERT notificaciones (colaborador_unido al organizador)
        API-->>UI: evento_id y rol
    else inválida
        API-->>UI: error (INVITACION_REVOCADA, INVITACION_VENCIDA, INVITACION_AGOTADA, INVITACION_NO_EXISTE)
    end

    O->>UI: Revoca código
    UI->>API: rpc revocar_invitacion(id)
    API->>DB: UPDATE estado = revocada, revocada_en = now()
    Note over API,DB: Membresías ya concedidas se conservan (RN-23)
```

## 5. Inscripción con control de aforo
RF-07, RF-08 · P5, P6, P7, P8 · RN-30…35

```mermaid
sequenceDiagram
    actor P as Participante
    participant UI
    participant API
    participant DB
    P->>UI: Pulsa Inscribirme en la actividad (P7)
    UI->>API: rpc inscribirse(evento_id)
    API->>DB: BEGIN
    API->>DB: SELECT actividad y raíz FOR UPDATE
    API->>DB: Verifica publicado, requiere_inscripcion, no terminado (RN-31)
    API->>DB: Busca inscripción previa (evento_id, perfil_id)
    alt ya confirmada
        API-->>UI: error YA_INSCRITO
    else sin cupo en actividad (RN-32)
        API-->>UI: error ACTIVIDAD_LLENA
    else participante nuevo en el raíz y raíz lleno (RN-33)
        API-->>UI: error EVENTO_LLENO
    else hay cupo
        API->>DB: INSERT o reactiva inscripciones (confirmada)
        API->>DB: INSERT notificaciones (inscripcion_confirmada)
        API->>DB: COMMIT
        API-->>UI: inscripcion y cupos_restantes
        UI-->>P: Botón cambia a Cancelar inscripción
    end
```

## 6. Cancelar inscripción
RF-07 · P7, P8 · RN-36

```mermaid
sequenceDiagram
    actor P as Participante
    participant UI
    participant API
    participant DB
    P->>UI: Cancelar inscripción
    UI->>UI: Confirmación
    UI->>API: rpc cancelar_inscripcion(evento_id)
    API->>DB: SELECT inscripción propia FOR UPDATE
    alt existe asistencia ligada
        API-->>UI: error YA_ASISTIO
    else confirmada
        API->>DB: UPDATE estado = cancelada, cancelada_en = now()
        API->>DB: INSERT notificaciones (inscripcion_cancelada)
        API-->>UI: ok
        UI-->>P: Cupo liberado, actividad sale de calendario
    end
```

## 7. Mostrar gafete QR (disponible sin conexión)
RF-09 · P10 · RN-40…43

```mermaid
sequenceDiagram
    actor P as Participante
    participant UI
    participant SW
    participant API
    participant ST as Storage
    P->>UI: Abre Mi gafete
    UI->>SW: Lee credencial en caché
    alt hay caché
        SW-->>UI: codigo, nombre
    else no hay y hay conexión
        UI->>API: SELECT credenciales WHERE perfil_id = uid
        API-->>UI: codigo y estado
        UI->>SW: Guarda en IndexedDB
    end
    UI->>UI: Genera QR local desde codigo
    UI-->>P: QR grande + nombre + aviso funciona sin conexión
    opt Descargar PDF
        UI->>ST: Obtiene plantilla gafete activa
        UI->>UI: Compone PDF (plantilla + nombre + QR)
        UI-->>P: Descarga gafete.pdf
    end
    Note over UI,SW: El QR contiene solo el código (RN-41)
```

## 8. Escaneo de asistencia en línea
RF-10 · P23, P24, P25 · RN-50…57

```mermaid
sequenceDiagram
    actor S as Personal
    participant UI
    participant API
    participant DB
    S->>UI: Elige evento y actividad (P23) e inicia escaneo
    UI->>UI: Cámara lee QR, obtiene codigo
    UI->>UI: Genera id_cliente UUID y escaneado_en = ahora
    UI->>API: rpc registrar_asistencia(codigo, evento_id, id_cliente, escaneado_en)
    API->>DB: Verifica que el usuario sea STAFF/RESP/ORG autorizado (RN-51)
    API->>DB: SELECT credencial por codigo
    alt no existe
        API-->>UI: QR_INVALIDO
    else revocada
        API-->>UI: CREDENCIAL_REVOCADA
    else actividad no activa o fuera de ventana
        API-->>UI: EVENTO_NO_ACTIVO
    else id_cliente ya existe
        API-->>UI: resultado previo (idempotente)
    else ya hay asistencia (evento, perfil)
        API-->>UI: YA_REGISTRADO
    else tiene inscripción confirmada
        API->>DB: INSERT asistencias (inscrito, origen = en_linea)
        API-->>UI: OK + nombre del participante
    else sin inscripción
        API-->>UI: SIN_INSCRIPCION + puede_excepcional
    end
    UI-->>S: Pantalla verde, roja o amarilla (P25)
    S->>UI: Siguiente escaneo
```

## 9. Entrada excepcional
RF-11 · P25, P26 · RN-54…56

```mermaid
sequenceDiagram
    actor S as Personal
    participant UI
    participant API
    participant DB
    Note over S,UI: Parte de resultado amarillo SIN_INSCRIPCION
    S->>UI: Pulsa Entrada excepcional
    UI->>API: rpc capacidad_actividad(evento_id)
    API-->>UI: permite_excepcional, capacidad_disponible
    alt no permitido o sin capacidad
        UI-->>S: Botón Autorizar deshabilitado con motivo
    else permitido
        S->>UI: Escribe motivo (obligatorio) y pulsa Autorizar
        UI->>API: rpc registrar_asistencia(codigo, evento_id, id_cliente, escaneado_en, excepcional, motivo)
        API->>DB: Revalida capacidad con bloqueo (RN-55)
        alt hay capacidad
            API->>DB: INSERT asistencias (excepcional, inscripcion_id NULL, motivo)
            API-->>UI: OK
            UI-->>S: Verde, acceso registrado
        else se llenó entre tanto
            API-->>UI: SIN_CAPACIDAD
            UI-->>S: Rojo, no se puede autorizar
        end
    end
```

## 10. Escaneo sin conexión y sincronización
RF-12 · P23, P24, P27 · `06-offline.md`

```mermaid
sequenceDiagram
    actor S as Personal
    participant UI
    participant SW as SW + IndexedDB
    participant API
    participant DB
    Note over S,API: Con conexión, al iniciar escaneo
    UI->>API: rpc snapshot_actividad(evento_id)
    API-->>UI: lista codigo, perfil, nombre, inscrito, ya_asistio, aforo, permite_excepcional
    UI->>SW: Guarda snapshot

    Note over S,SW: Se pierde la conexión
    S->>UI: Escanea QR
    UI->>SW: Busca codigo en snapshot y en cola local
    alt codigo desconocido
        UI-->>S: Rojo QR_INVALIDO (solo local)
    else ya en snapshot ya_asistio o en cola
        UI-->>S: Rojo YA_REGISTRADO
    else inscrito
        UI->>SW: Encola asistencia (id_cliente, escaneado_en, origen = sin_conexion)
        UI-->>S: Verde
    else no inscrito
        UI-->>S: Amarillo, excepcional si permitido y capacidad local
    end
    UI-->>S: Contador y P27 muestran pendientes

    Note over S,DB: Regresa la conexión o pulsa Sincronizar ahora
    UI->>SW: Lee cola pendiente (por lotes)
    UI->>API: rpc sincronizar_asistencias(lote)
    loop cada elemento
        API->>DB: Revalida como registrar_asistencia usando escaneado_en original
        API->>DB: INSERT ON CONFLICT (id_cliente) DO NOTHING
    end
    API-->>UI: resultado por id_cliente (OK, DUPLICADO, RECHAZADO + motivo)
    UI->>SW: Marca sincronizadas y conserva rechazadas para revisión
    UI-->>S: Mensaje de resultado (n ok, m con problema)
```

## 11. Plantillas y emisión de constancias
RF-13 · P19, P20 · RN-60…64

```mermaid
sequenceDiagram
    actor O as Organizador
    participant UI
    participant ST as Storage
    participant API
    participant DB
    O->>UI: Sube imagen o PDF base (P19)
    UI->>ST: upload plantillas/{evento_id}/...
    O->>UI: Posiciona campos (nombre, evento, folio, QR) y guarda
    UI->>API: INSERT plantillas (configuracion jsonb, activa = false)
    O->>UI: Activar plantilla
    UI->>API: rpc activar_plantilla(id)
    API->>DB: Desactiva la anterior del mismo tipo y activa esta

    O->>UI: Emisión (P20): tipo, destinatarios (todos o los que asistieron)
    UI->>API: rpc destinatarios_constancia(evento_id, filtro)
    API-->>UI: lista
    O->>UI: Vista previa y Generar
    UI->>API: rpc emitir_constancias(evento_id, plantilla_id, perfiles[], concepto)
    loop cada destinatario
        API->>DB: Si existe constancia, reutilizar (RN-63)
        API->>DB: Si no, genera folio único e INSERT constancias
        API->>DB: INSERT notificaciones (constancia_emitida)
    end
    API-->>UI: constancias con folio
    UI->>UI: Genera PDF de cada una (plantilla + datos + folio)
    UI->>ST: upload constancias/{evento}/{folio}.pdf
    UI->>API: UPDATE constancias SET archivo_url
    Note over UI,ST: Alternativa: generar PDF en Edge Function. Decisión pendiente en 07-decisiones-abiertas.md
```

## 12. Descarga de constancia y verificación de folio
RF-13 · P11

```mermaid
sequenceDiagram
    actor P as Participante
    actor T as Tercero
    participant UI
    participant API
    participant ST as Storage
    P->>UI: Abre Mis constancias
    UI->>API: SELECT constancias WHERE perfil_id = uid
    API-->>UI: lista con folio y fecha
    P->>UI: Descargar PDF
    UI->>ST: createSignedUrl(archivo_url, 60s)
    ST-->>UI: URL temporal
    UI-->>P: Descarga
    T->>UI: Abre /verificar/{folio}
    UI->>API: rpc verificar_folio(folio)
    API-->>UI: valido, nombre, evento, fecha (sin correo ni teléfono)
```

## 13. Reportes
RF-14 · P21 · RN-80…82

```mermaid
sequenceDiagram
    actor O as Organizador o responsable
    participant UI
    participant API
    participant DB
    O->>UI: Elige evento y actividad
    UI->>API: rpc reporte_evento(evento_id, actividad_id?)
    API->>DB: Verifica membresía con alcance
    API->>DB: Agrega inscritos, asistentes, excepcionales, % asistencia
    API-->>UI: totales + filas de detalle
    UI-->>O: Tarjetas, tabla y gráfica
    O->>UI: Exportar
    UI->>UI: Genera CSV en cliente
    UI-->>O: Descarga reporte.csv
```

## 14. Notificaciones
RF-15 · P4 · RN-70…72

```mermaid
sequenceDiagram
    actor U as Usuario
    participant UI
    participant API
    participant DB
    Note over DB: Una función de BD inserta la notificación junto al hecho que la origina
    UI->>API: SELECT notificaciones WHERE perfil_id = uid ORDER BY creado_en DESC
    API-->>UI: lista con leida y no leidas
    UI-->>U: Insignia con contador
    U->>UI: Marcar como leída
    UI->>API: UPDATE notificaciones SET leida = true WHERE id = ?
    Note over UI,API: Sin push ni correo. Para frescura, suscripción Realtime a perfil_id = uid o refetch al enfocar la app
```

## 15. Cancelar o cerrar un evento (cascada)
RN-04, RN-06 · P14, P15

```mermaid
sequenceDiagram
    actor O as Organizador
    participant UI
    participant API
    participant DB
    O->>UI: Cambia estado a cancelado
    UI->>UI: Confirmación con número de inscritos afectados
    UI->>API: rpc cambiar_estado_evento(id, cancelado)
    API->>DB: Verifica transición válida
    API->>DB: UPDATE eventos del raíz y sus actividades
    API->>DB: INSERT notificaciones (evento_cancelado) a cada inscrito confirmado
    API-->>UI: ok
    Note over API,DB: Las inscripciones se conservan por historial. El catálogo deja de mostrar el evento
```
