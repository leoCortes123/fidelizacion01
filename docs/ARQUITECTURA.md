# Arquitectura

## Vista general

```text
┌──────────────────────────────┐        ┌──────────────────────────────┐
│ Portal del cliente (React)   │        │ Panel de administración      │
│ /customer, /claim/:token     │        │ /admin/*                     │
└──────────────┬───────────────┘        └──────────────┬───────────────┘
               │  Supabase JS (clave publicable, JWT)   │
               └───────────────────┬────────────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │ Supabase Auth                │
                    │ Google OAuth + correo/clave  │
                    ├──────────────────────────────┤
                    │ PostgREST + RPC PostgreSQL   │
                    │  · RLS en todas las tablas   │
                    │  · funciones SECURITY DEFINER│
                    │  · transacciones y bloqueos  │
                    └──────────────────────────────┘
```

No hay una API propia: la interfaz llama a Supabase directamente y toda la lógica
sensible vive en funciones PostgreSQL invocadas por RPC. El navegador nunca recibe
la clave secreta (`service_role` / `sb_secret_…`).

## Modelo de usuarios

- `profiles`: nombre, correo y fechas.
- `user_roles`: enum `app_role` con `customer`, `admin` y el valor histórico
  `registrar`, que ya no otorga acceso al panel.
- Todo registro crea `customer`. El correo listado en
  `private.bootstrap_admins` recibe además `admin` en el alta.
- Los roles se gestionan por base de datos en esta etapa; la interfaz no permite
  crearlos ni revocarlos.

## Decisiones principales

### 1. El servidor es la única autoridad

- El importe lo envía el administrador autenticado; el cliente no puede modificarlo.
- Los puntos se calculan al reclamar, con la configuración congelada al crear la
  compra (`purchases.settings_snapshot`).
- El saldo es la suma del libro `points_ledger`; no hay un campo editable.
- Los roles se validan dentro de PostgreSQL (`has_role`, `is_staff`), no en el
  cliente. `is_staff()` equivale a `admin` desde la migración del 23/10/2026.

### 2. Dinero exacto

Los importes se almacenan como `integer` en pesos colombianos. El COP se maneja sin
centavos en este negocio; un entero evita errores de coma flotante. La interfaz usa
`Intl.NumberFormat('es-CO', { currency: 'COP' })`.

### 3. Tiempo

La base usa `timestamptz`. La interfaz presenta fechas en `America/Bogota` mediante
`Intl.DateTimeFormat`. Las reglas de vigencia se evalúan en el servidor con `now()`.

### 4. Tokens de compra

- El token se genera con 32 bytes criptográficos y se codifica en Base64 URL-safe.
- Solo se guarda `SHA-256(token)` en `purchases.token_hash`.
- El QR contiene únicamente `/claim/<token>`; nunca importe ni puntos.
- El token viaja al cliente en la respuesta de `create_purchase` y en el QR, no se
  registra en auditoría.

### 5. Canje aplicado en caja

El cliente ya no reserva recompensas ni genera códigos. El administrador elige la
recompensa en el detalle del cliente y `admin_redeem_reward`:

1. exige rol `admin`;
2. bloquea al cliente con `pg_advisory_xact_lock`;
3. valida saldo y recompensa activa;
4. inserta el canje como `redeemed` y el débito en el libro;
5. audita la operación.

Un canje equivocado se corrige con `admin_cancel_redemption`, que devuelve los
puntos y conserva la trazabilidad de la entrega y de la anulación.

## Flujo de compra y reclamación

```text
Administrador                   Servidor                        Cliente
     │                            │                               │
     │ create_purchase(importe)   │                               │
     ├───────────────────────────►│ valida rol, importe, límites  │
     │                            │ genera token + hash           │
     │                            │ guarda snapshot de reglas     │
     │◄───────────────────────────┤ devuelve token (una vez)      │
     │ muestra QR                 │                               │
     │                            │◄──────────────────────────────┤ escanea /claim/token
     │                            │                               │ inicia sesión (si falta)
     │                            │◄──────────────────────────────┤ claim_purchase(token)
     │                            │ bloquea la compra (FOR UPDATE)│
     │                            │ calcula puntos y acredita     │
     │                            │◄──────────────────────────────┤ saldo actualizado
```

## Flujo de canje

```text
Administrador                   Servidor                        Cliente
     │                            │                               │
     │ abre /admin/customers/:id  │                               │
     │ admin_redeem_reward(...)   │                               │
     ├───────────────────────────►│ valida admin y saldo          │
     │                            │ bloquea cliente y recompensa  │
     │                            │ inserta canje + débito        │
     │                            │ audita                        │
     │◄───────────────────────────┤ saldo nuevo                   │
     │                            │                               │
     │ (si hubo error)            │                               │
     │ admin_cancel_redemption    │                               │
     ├───────────────────────────►│ revierte y devuelve puntos    │
```

El cliente ve el cambio al volver a abrir o enfocar la app: TanStack Query
revalida al recuperar el foco. No hay suscripciones en tiempo real.

## Frontend

- **Rutas:** `src/app/router.tsx`, con carga diferida y `Suspense`.
- **Guards:** `RequireAuth` para el portal del cliente y `RequireAdmin` para todo
  `/admin`.
- **Shells:** `CustomerShell` (tres pestañas: Inicio, Compras, Canjes) y
  `AdminShell` (Generar QR, Clientes, Compras, Configuración), con navegación
  inferior en móvil.
- **Diseño móvil primero:** saldo en tipografía grande, tarjetas en lugar de tablas
  en pantallas pequeñas, objetivos táctiles amplios y navegación fija inferior.
- **Estado del servidor:** TanStack Query; las mutaciones invalidan cliente,
  clientes, catálogo, auditoría y panel del cliente.
