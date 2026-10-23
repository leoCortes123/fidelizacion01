# Puntos & Fidelización

Sistema web de fidelización para un negocio de comida. Incluye un portal para
clientes y un panel de administración, con puntos, QR de compra, canje de
recompensas y auditoría.

> **Identidad provisional.** El nombre y el lema se configuran con
> `VITE_BRAND_NAME` y `VITE_BRAND_TAGLINE`. No se asume una marca definitiva.

## Qué hace

- **Cliente:** entra con Google o correo/contraseña, ve su nombre, correo y saldo en
  grande, consulta las recompensas disponibles y el historial de compras con los
  puntos ganados. No canjea desde la app: lo pide en caja.
- **Administrador:** genera el QR de una compra, consulta clientes (ordenados por
  puntos), abre el detalle, aplica el canje de una recompensa con descuento
  inmediato y puede anularlo con devolución. También configura reglas y catálogo.
- **Tipos de usuario:** `admin` y `customer`. Los usuarios y roles se administran
  por base de datos en esta etapa. `drleocortex@gmail.com` recibe el rol `admin`
  automáticamente en su primer registro.

## Stack

| Capa | Tecnología |
| --- | --- |
| Frontend | React 19 + TypeScript + Vite 6 |
| Estilos | Tailwind CSS 3 + componentes estilo shadcn/ui |
| Rutas | React Router 7 |
| Datos y estado | TanStack Query 5 |
| Backend | Supabase (PostgreSQL, Auth, PostgREST, RPC) |
| Autenticación | Supabase Auth: Google OAuth (PKCE) y correo/contraseña |
| QR | `qrcode.react` |
| Pruebas | Vitest + Testing Library + Playwright |
| Migraciones | Supabase CLI |

## Inicio rápido

```bash
cp .env.example .env
# Edita VITE_SUPABASE_URL y VITE_SUPABASE_ANON_KEY

npm ci
npm run dev          # http://127.0.0.1:5173
```

Con Supabase CLI:

```bash
npx supabase start
npx supabase db reset   # aplica migraciones + seed
```

Para un proyecto alojado:

```bash
npx supabase link --project-ref <project-ref>
npx supabase db push
npx supabase db push --include-seed   # o ejecuta supabase/seed.sql
```

Configuración de OAuth, dominio y primer administrador:
[`docs/CONFIGURACION.md`](docs/CONFIGURACION.md).

## Scripts

```bash
npm run dev          # servidor de desarrollo
npm run build        # typecheck + build de producción
npm run preview      # sirve dist/
npm run typecheck    # tsc -b
npm test             # Vitest + Testing Library
npm run test:db      # PostgreSQL efímero en Docker: migraciones, RLS, RPC y concurrencia
npm run test:e2e     # Playwright (backend Supabase simulado)
npm run test:all     # typecheck + unit + DB + E2E
```

## Estructura

```text
src/
  app/                  providers, router, query client
  components/
    layout/             shells (cliente/admin) y guards de autenticación/roles
    shared/             marca, estados, badges, diálogos
    ui/                 primitivas reutilizables
  features/
    auth/               sesión, Google OAuth, correo/contraseña, callback
    customer/           inicio con saldo y catálogo, compras, canjes
    purchases/          reclamación del QR
    admin/              QR, clientes, detalle/canje, compras, configuración
    points/ rewards/    hooks de datos
  lib/                  Supabase, formato COP/fechas, errores, config
  types/                tipos compartidos
supabase/
  migrations/           esquema, RPC y RLS
  seed.sql              configuración y catálogo inicial
  config.toml           configuración local
tests/
  unit/                 Vitest + Testing Library
  db/                   pruebas SQL de integración
  e2e/                  Playwright + backend simulado
docs/                   arquitectura, seguridad, configuración, pruebas
```

## Rutas principales

| Ruta | Acceso | Contenido |
| --- | --- | --- |
| `/` | Público | Presentación y acceso |
| `/login` | Público | Google, correo/contraseña y registro simple |
| `/claim/:token` | Autenticado | Reclamar una compra escaneando el QR |
| `/customer` | Cualquier usuario | Saldo, recompensas y compras recientes |
| `/customer/purchases` | Cualquier usuario | Historial con puntos por compra |
| `/customer/redemptions` | Cualquier usuario | Canjes aplicados por caja |
| `/admin` | Solo admin | Generar QR de compra |
| `/admin/customers` | Solo admin | Clientes con filtros y orden por puntos |
| `/admin/customers/:id` | Solo admin | Detalle y canje de recompensa |
| `/admin/purchases` | Solo admin | Compras y cancelaciones |
| `/admin/settings` | Solo admin | Reglas, catálogo y auditoría |

Las rutas antiguas `/registrar/*` y `/customer/rewards` redirigen a las nuevas.

## Documentación

- [Arquitectura y flujos](docs/ARQUITECTURA.md)
- [Modelo de datos, RPC y RLS](docs/BASE_DE_DATOS.md)
- [Seguridad y amenazas](docs/SEGURIDAD.md)
- [Configuración local, Supabase y Google OAuth](docs/CONFIGURACION.md)
- [Pruebas y evidencia](docs/PRUEBAS.md)
- [Despliegue](docs/DESPLIEGUE.md)
- [Informe final](docs/INFORME_FINAL.md)
- [Estado del proyecto](ESTADO_PROYECTO.md)

## Estado verificable

- `npm run build`: compila sin errores.
- `npm test`: **29 pruebas** unitarias/de componentes aprobadas.
- `npm run test:db`: **89 aserciones SQL** aprobadas, incluida la prueba de
  concurrencia con dos reclamos simultáneos.
- `npm run test:e2e`: **6 flujos** Playwright aprobados con backend simulado.
- Smoke test real contra Supabase alojado: registro, rol admin, QR, reclamo,
  canje, historial y devolución verificados; datos temporales eliminados.

Pendiente: dominio de Vercel/Netlify para actualizar `VITE_PUBLIC_APP_URL` y las
Redirect URLs de Supabase/Google. No se declara producción hasta completar ese
paso y validar el OAuth en el dominio final.
