# 10 · Guía de desarrollo

## Stack (sección 1.9 de la propuesta)
React + TypeScript + Vite · Supabase (PostgreSQL, Auth, Storage) · PWA (Service Worker, Workbox/`vite-plugin-pwa`) · biblioteca de QR (generar y leer) · biblioteca de PDF.

**Bibliotecas sugeridas [PROPUESTA]**: `@supabase/supabase-js`, `react-router`, `@tanstack/react-query`, `react-hook-form` + `zod`, `qrcode` (generar), `@yudiel/react-qr-scanner` o `html5-qrcode` (leer), `pdf-lib`, `idb` (IndexedDB), `vite-plugin-pwa`, `vitest`, `@playwright/test`. Elegir una por necesidad y fijar versión.

## Estructura de carpetas sugerida
```
eventia/
├─ docs/                      ← este directorio (fuente de verdad)
├─ supabase/
│  ├─ migrations/             ← SQL versionado (tablas, triggers, RPC, RLS)
│  ├─ tests/                  ← pgTAP
│  └─ seed.sql                ← datos de ejemplo (1 admin, 1 organizador, eventos)
├─ src/
│  ├─ features/               ← un módulo por funcionalidad
│  │  ├─ auth/  solicitudes/  eventos/  invitaciones/  inscripciones/
│  │  ├─ credencial/  acceso/ (escáner + offline)  constancias/  reportes/  notificaciones/  admin/
│  │  └─ cada una: components/ hooks/ api.ts types.ts
│  ├─ lib/ (supabase client, idb, pdf, qr, errores)
│  ├─ routes/  (según 05-contratos-rpc.md)
│  └─ sw/     (service worker)
└─ e2e/
```

## Convenciones
- Nombres de BD en **español sin acentos y snake_case** (como el diccionario de datos). Código y componentes en inglés o español, pero consistentes: **[PROPUESTA]** componentes/archivos en inglés, textos de UI en español.
- Toda regla de negocio vive en la BD (constraints, triggers, RPC). El cliente solo valida para UX y **nunca** es la única barrera.
- Sin `service_role` en el cliente. Solo `anon key` + JWT del usuario.
- Fechas en UTC (`timestamptz`) y mostradas en `America/Merida`.
- Errores: el cliente mapea `MESSAGE` a texto (tabla en `05`). No mostrar mensajes crudos de Postgres.
- Los cambios de esquema solo por migración; nunca editar en el panel sin migración.
- Commits pequeños y convencionales (`feat:`, `fix:`, `docs:`). Rama por funcionalidad, PR con checklist de DoD (`08`).

## Variables de entorno
`.env.example` (versionado) y `.env.local` (ignorado):
```
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
```
Nunca comitear llaves `service_role` ni `.env.local`.

## Puesta en marcha (cuando exista el código)
```
npm install
npx supabase start          # BD local con Docker
npx supabase db reset       # aplica migraciones + seed
npm run dev
npm run test                # unitarias
npx supabase test db        # pruebas de BD y RLS
npx playwright test         # E2E
```

## Orden de implementación sugerido (alineado al cronograma de 12 semanas)
1. Esquema + migraciones + seed + trigger de perfil/credencial.
2. Auth + perfil (P1–P3).
3. RLS base y funciones auxiliares (`es_admin`, `es_miembro`) con pruebas.
4. Solicitudes y aprobación (P12, P13, P28, P29).
5. Eventos, actividades, catálogo (P5, P6, P14–P16).
6. Invitaciones (P17).
7. Inscripción y aforo (P7, P8, P9, P18).
8. Credencial y escáner en línea (P10, P23–P26).
9. Modo offline y cola (P27).
10. Plantillas, constancias, verificación (P11, P19, P20).
11. Reportes, notificaciones, supervisión (P4, P21, P22, P30).
12. Pulido, pruebas E2E, despliegue.

## Cómo trabajar con IA en este proyecto
1. Antes de cada tarea, indicar a la IA qué documentos leer (`CLAUDE.md` lo hace por defecto).
2. Pedir **una funcionalidad por vez**, citando RF, RN y la sección de secuencia.
3. Exigir pruebas primero (criterios de `08`).
4. Si la IA propone una regla nueva, debe agregarla a `01` como `[PROPUESTA]`, no solo al código.
5. Revisar siempre migraciones y RLS a mano: son lo más crítico (seguridad y aforo).

## Riesgos principales
| Riesgo | Mitigación |
|---|---|
| Sobrecupo por concurrencia | RPC con bloqueo (RN-35) + prueba N1 |
| Fuga de datos por RLS mal escrita | Pruebas por rol (08) y denegar por defecto |
| Duplicados al sincronizar | `id_cliente` único + prueba O4 |
| Cámara/QR no funciona en algunos navegadores | HTTPS, gesto del usuario, plan B de ingreso manual de código **[PROPUESTA]** |
| Reloj del dispositivo incorrecto | Validación en servidor (`06`) |
| PDF masivo bloquea el navegador | Edge Function o generación por lotes (D4) |
| Alcance excesivo en 12 semanas | Priorizar RF de prioridad Alta; Media/Baja al final |

## Privacidad (RNF-09)
Datos personales guardados: nombre, correo, teléfono (opcional), foto (opcional). El QR no contiene datos personales. Los miembros de un evento ven solo nombre, correo y estado de inscritos (RN-90). Redactar aviso de privacidad (D12).
