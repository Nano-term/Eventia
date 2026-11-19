# 04 · Diagramas de estados

Complementan `03-diagramas-secuencia.md`. Las transiciones no listadas están **prohibidas** y la BD debe rechazarlas (trigger o función de transición).

## Evento / actividad (`eventos.estado`) — RN-04, RN-06
```mermaid
stateDiagram-v2
    [*] --> borrador: creado (al aprobar solicitud o INSERT actividad)
    borrador --> publicado: publicar (padre publicado)
    publicado --> borrador: despublicar (solo si no hay inscripciones)
    publicado --> cerrado: cerrar
    borrador --> cancelado: cancelar
    publicado --> cancelado: cancelar (notifica inscritos)
    cerrado --> [*]
    cancelado --> [*]
```
- `cerrado`: ya no admite inscripciones ni escaneo, pero permite reportes y emisión de constancias.
- `cancelado`: terminal. Solo lectura.

## Solicitud (`solicitudes_evento.estado`) — RN-10
```mermaid
stateDiagram-v2
    [*] --> pendiente: enviada
    pendiente --> aprobada: admin aprueba (crea evento)
    pendiente --> rechazada: admin rechaza (observaciones obligatorias)
    aprobada --> [*]
    rechazada --> [*]
```

## Invitación (`invitaciones.estado`) — RN-22
```mermaid
stateDiagram-v2
    [*] --> activa
    activa --> revocada: revocar
    activa --> activa: canje (usos++)
    note right of activa
        Vencida o agotada = activa pero no canjeable.
        Se calcula con expira_en y usos_maximos.
    end note
```

## Credencial (`credenciales.estado`) — RN-42
```mermaid
stateDiagram-v2
    [*] --> activa: creada con el perfil
    activa --> revocada: regenerar o revocar
    revocada --> activa: nuevo codigo emitido
```

## Inscripción (`inscripciones.estado`) — RN-30, RN-36
```mermaid
stateDiagram-v2
    [*] --> confirmada: inscribirse
    confirmada --> cancelada: cancelar (si no hay asistencia)
    cancelada --> confirmada: reinscribirse (si hay cupo)
```

## Resultado de un escaneo (UI P25) — RN-52
```mermaid
stateDiagram-v2
    [*] --> Leyendo
    Leyendo --> Verde: OK
    Leyendo --> Rojo: QR_INVALIDO / CREDENCIAL_REVOCADA / YA_REGISTRADO / EVENTO_NO_ACTIVO
    Leyendo --> Amarillo: SIN_INSCRIPCION
    Amarillo --> Excepcional: Entrada excepcional (si permitida y con capacidad)
    Excepcional --> Verde: Autorizar
    Excepcional --> Leyendo: Cancelar
    Verde --> Leyendo: Siguiente escaneo
    Rojo --> Leyendo: Siguiente escaneo
```

## Elemento de la cola offline (IndexedDB) — `06-offline.md`
```mermaid
stateDiagram-v2
    [*] --> pendiente: escaneo guardado
    pendiente --> sincronizando: enviar lote
    sincronizando --> sincronizada: OK o DUPLICADO
    sincronizando --> rechazada: RECHAZADO (requiere revisión)
    sincronizando --> pendiente: error de red (reintento)
    rechazada --> [*]: personal la descarta o resuelve
    sincronizada --> [*]: purgada tras N días
```
