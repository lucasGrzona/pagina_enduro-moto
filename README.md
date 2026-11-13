# Aventuras Raulito · Enduro en San Rafael

Plataforma web de rutas de enduro en San Rafael, Mendoza. Mapa interactivo satelital/relieve,
track GPX descargable, clima simulado, solicitudes de guía, cuentas de rider y comunidad con
comentarios y valoraciones.

![Next.js](https://img.shields.io/badge/Next.js-15-black) ![React](https://img.shields.io/badge/React-19-149eca) ![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6) ![Tailwind](https://img.shields.io/badge/Tailwind-4-38bdf8)

## Contenido

- [Funcionalidades](#funcionalidades)
- [Stack](#stack)
- [Puesta en marcha](#puesta-en-marcha)
- [Cuentas demo](#cuentas-demo)
- [Roles y visibilidad](#roles-y-visibilidad)
- [API](#api)
- [Variables de entorno](#variables-de-entorno)
- [Scripts](#scripts)
- [Pruebas ejecutadas](#pruebas-ejecutadas)
- [Datos y persistencia](#datos-y-persistencia)
- [Despliegue](#despliegue)
- [Créditos](#créditos)

## Funcionalidades

- **Mapa inmersivo** con capas de satélite (Esri World Imagery) y relieve (OpenTopoMap),
  polyline por dificultad, waypoints y encuadre automático. Carga diferida (`ssr: false`).
- **Catálogo oficial** de 6 rutas (Hard Enduro, Trail y Arena/Dunas) con descripción, desnivel,
  altitud, duración, temporada, highlights y perfil de elevación.
- **Explorador** con búsqueda (nombre, zona, tag), filtros por dificultad y alcance
  (todas / oficiales / comunidad / mías).
- **Panel lateral** con clima simulado determinista, perfil de elevación SVG, waypoints,
  descarga GPX, comentarios con estrellas y solicitud de guía contextual.
- **Cuentas**: registro/login modal, sesión en cookie HttpOnly firmada, tema claro/oscuro persistente.
- **Rutas de usuarios**: públicas, privadas (sólo dueño, admins y emails invitados) u oficiales
  si quien publica es administrador.
- **Comunidad**: muro de últimas opiniones, rutas de riders y contadores reales.
- **Guía local**: formulario con validación, fecha futura, honeypot antispam y rate limiting.
- **Seguridad y UX**: skeletons topográficos, guard clauses, sanitización, validación zod,
  cabeceras de seguridad, rate limiting y bloqueo de fuerza bruta.
- **Mobile-first**: drawer a pantalla completa, mapa primero, menú colapsable, imágenes lazy.

## Stack

| Capa | Tecnología |
| --- | --- |
| Framework | Next.js 15 (App Router, Route Handlers) |
| UI | React 19, TypeScript 5.9, Tailwind CSS 4 |
| Mapa | Leaflet 1.9 + react-leaflet 5 |
| Validación | zod 4 |
| Persistencia | JSON local atómico (`.data/db.json`) |
| Contraseñas | `scrypt` con salt aleatorio (Node crypto) |
| Sesiones | HMAC-SHA256 sobre cookie HttpOnly |

No requiere claves de API externas.

## Puesta en marcha

Requiere Node.js 20+ (probado con v24).

```bash
npm install
npm run dev
```

Abrir <http://localhost:3000>. En el primer arranque se crea `.data/db.json` con usuarios,
2 rutas de comunidad y 4 comentarios semilla.

Producción local:

```bash
npm run build
npm run start
```

## Cuentas demo

Sembradas automáticamente. **Sólo para evaluación local**, cambiarlas o desactivarlas en producción.

| Rol | Email | Contraseña | Uso |
| --- | --- | --- | --- |
| Administrador | `admin@aventurasraulito.com` | `AdminRaulito2026!` | Publicar rutas oficiales, ver solicitudes, moderar/eliminar |
| Rider | `raul@aventurasraulito.com` | `GuiaRaul2026!` | Dueño de una ruta pública de comunidad |
| Rider | `demo@aventurasraulito.com` | `DemoRider2026!` | Cuenta de prueba estándar |

En el modal de login hay un desplegable **“Cuentas demo para probar”** que completa las
credenciales con un clic. El administrador también se puede recrear desde `ADMIN_EMAIL` y
`ADMIN_PASSWORD` al inicializar la base.

## Roles y visibilidad

| Acción | Anónimo | Rider | Dueño | Admin |
| --- | --- | --- | --- | --- |
| Ver rutas oficiales y públicas | ✅ | ✅ | ✅ | ✅ |
| Ver ruta privada | ❌ | Invitado | ✅ | ✅ |
| Descargar GPX de ruta visible | ✅ | ✅ | ✅ | ✅ |
| Comentar ruta visible | ❌ | ✅ | ✅ | ✅ |
| Crear ruta | ❌ | ✅ | ✅ | ✅ |
| Eliminar ruta de comunidad | ❌ | ❌ | ✅ | ✅ |
| Publicar como oficial (`isOfficial`) | ❌ | ❌ | ❌ | ✅ |
| Ver solicitudes de guía | ❌ | ❌ | ❌ | ✅ |

Una ruta privada es visible para: dueño, administradores y emails de `sharedWith` que tengan
cuenta. Los endpoints de detalle, comentarios y GPX aplican la misma regla (404 si no corresponde).

## API

Todas las respuestas son JSON salvo el GPX. Los POST requieren `Content-Type: application/json`.

| Método | Ruta | Descripción |
| --- | --- | --- |
| POST | `/api/auth/register` | Crear cuenta (nombre, email, contraseña) |
| POST | `/api/auth/login` | Iniciar sesión |
| POST | `/api/auth/logout` | Cerrar sesión |
| GET | `/api/auth/me` | Usuario de la sesión actual |
| GET | `/api/routes?q=&difficulty=&scope=` | Listar rutas visibles |
| POST | `/api/routes` | Crear ruta (auth) |
| GET | `/api/routes/:id` | Detalle si es visible |
| DELETE | `/api/routes/:id` | Eliminar (dueño/admin) |
| GET | `/api/routes/:id/gpx` | Descargar GPX |
| GET/POST | `/api/routes/:id/comments` | Listar/crear comentarios |
| GET | `/api/community` | Feed de comentarios y contadores |
| POST | `/api/guide-requests` | Crear solicitud de guía |
| GET | `/api/guide-requests` | Listar solicitudes (admin) |
| GET | `/api/health` | Estado, versión y conteos |

## Variables de entorno

Copiar `.env.example` a `.env.local` (opcional en desarrollo):

| Variable | Default | Descripción |
| --- | --- | --- |
| `AUTH_SECRET` | secreto de desarrollo | Clave HMAC de sesiones. **Obligatoria en producción.** |
| `ADMIN_EMAIL` | `admin@aventurasraulito.com` | Email que recibe rol admin al sembrar |
| `ADMIN_PASSWORD` | `AdminRaulito2026!` | Contraseña inicial del admin |
| `DATA_DIR` | `./.data` | Carpeta del JSON persistente |
| `IP_PEPPER` | pepper de desarrollo | Sal para hashear IPs de solicitudes |

## Scripts

```bash
npm run dev        # desarrollo
npm run build      # build de producción
npm run start      # servir build
npm run typecheck  # TypeScript sin emitir
npm run loadtest   # carga + rate limit (requiere servidor)
```

Prueba de carga:

```bash
PORT=3100 npm run start
BASE_URL=http://localhost:3100 DURATION=8 CONCURRENCY=20 npm run loadtest
```

## Pruebas ejecutadas

Fecha: noviembre 2026. Entorno local Windows, Node v24.21.0, servidor `next start` en el puerto 3100.

- `npm run typecheck`: sin errores.
- `npm run build`: compilado y 11 páginas/rutas generadas sin errores.
- **Flujo API (19 comprobaciones)**: login admin/rider, sesión, ruta privada oculta a anónimos,
  visible para invitado, bloqueo de borrado ajeno, comentario, GPX, validación 422, content-type
  415, borrado admin, solicitud de guía, feed y rate limit 429.
- **Carga (8 s, 20 workers, tráfico mixto home + APIs)**:
  - 2.190 peticiones, 272,7 req/s, 0 errores de red, 0 respuestas 5xx.
  - Latencia media 73 ms; p50 28 ms, p95 230 ms, p99 350 ms.
  - Ráfaga de 10 logins inválidos: 5 × 401 y luego 5 × 429.
  - Cabeceras verificadas: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`,
    `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` restrictiva.
- **Rate limit de login** verificado además en el flujo API (6.º intento → 429).

Los números son de una máquina de desarrollo y no representan un SLA. Ver `SECURITY.md` para el
detalle de controles y límites.

## Datos y persistencia

- Rutas oficiales: estáticas en `lib/data/official-routes-*.ts` (no editables desde la web).
- Usuarios, rutas de comunidad, comentarios y solicitudes: `.data/db.json`.
- Escritura atómica (`tmp` + `rename`) y mutaciones serializadas por cola en proceso.
- El archivo se crea con seeds si no existe o si el esquema no coincide (versión 1).
- `.data/` está en `.gitignore`: contiene hashes de contraseñas.
- Para reiniciar la demo: detener el servidor, borrar `.data/db.json` y volver a arrancar.

## Despliegue

Para producción se recomienda:

1. Definir `AUTH_SECRET` fuerte, `ADMIN_EMAIL`, `ADMIN_PASSWORD` e `IP_PEPPER`.
2. Servir siempre por HTTPS (la cookie es `Secure` en producción).
3. Usar un volumen persistente para `DATA_DIR` si se mantiene el JSON.
4. Reemplazar el rate limiting en memoria por Redis si hay más de una instancia.
5. Migrar la persistencia a Postgres/SQLite gestionado para concurrencia real.
6. Configurar backups y rotación de credenciales.

En plataformas serverless con filesystem efímero (por ejemplo Vercel Functions) el JSON local no
persiste: usar base gestionada o volumen persistente.

## Estructura

```
app/
  api/…            Route Handlers (auth, rutas, GPX, comentarios, comunidad, guía, health)
  globals.css      Tokens de tema, componentes y estilos Leaflet
  layout.tsx       Metadatos, fuentes, providers, shell
  page.tsx         Home server-rendered con datos iniciales
components/        Navbar, Hero, Explorer, mapa, drawer, modales, comunidad, guía
lib/
  data/            Rutas oficiales y seeds
  server/          DB, auth, rate limit, validación, servicio de rutas
  types.ts         Contrato de datos
scripts/           Prueba de carga
```

## Créditos

- Diseño y contenido: Aventuras Raulito, San Rafael, Mendoza.
- Fotografías: [Unsplash](https://unsplash.com) (hotlinking permitido por la licencia Unsplash).
- Mapas: © Esri, Maxar, Earthstar Geographics; © OpenStreetMap contributors, SRTM, OpenTopoMap (CC-BY-SA).
- Tipografías: Bebas Neue y Manrope vía Google Fonts.

> Las rutas y el clima son orientativos. Verificá siempre el terreno, el estado de los accesos y
> las condiciones meteorológicas reales antes de salir.
# pagina_mercer-a
