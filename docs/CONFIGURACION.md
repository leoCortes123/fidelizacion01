# Configuración

## Estado actual

| Elemento | Estado |
| --- | --- |
| Proyecto Supabase | `gxfytcquyobhhbjipbea` (`fidelizacion01`, región us-east-2) |
| Migraciones + seed | Aplicadas el 23/10/2026 |
| Google OAuth | Provider habilitado en Supabase; redirect URI confirmada en Google Cloud |
| Registro con correo/contraseña | Habilitado con autoconfirmación (sin verificación por correo) |
| Dominio público | **Pendiente**: se define al crear el proyecto en Vercel |
| Administrador inicial | `drleocortex@gmail.com` recibe `admin` al registrarse |

## Requisitos

- Node.js 22+ y npm 10+
- Docker (para Supabase local y las pruebas de base de datos)
- Supabase CLI (`npx supabase`)
- Un proyecto de Google Cloud con OAuth 2.0

## Variables de entorno

```dotenv
VITE_SUPABASE_URL=https://gxfytcquyobhhbjipbea.supabase.co
VITE_SUPABASE_ANON_KEY=sb_publishable_…
VITE_BRAND_NAME=Mi Negocio
VITE_BRAND_TAGLINE=Puntos por tus compras
VITE_PUBLIC_APP_URL=
```

- `VITE_SUPABASE_URL` es la URL base, **sin** `/rest/v1/` ni barra final.
- `VITE_SUPABASE_ANON_KEY` es la clave pública (`publishable` o `anon`). Es la única
  clave que puede llegar al navegador.
- `VITE_PUBLIC_APP_URL` construye la URL de los QR. **Debe** ser el dominio público.
  Mientras esté vacío se usa `window.location.origin` (útil en local).
- `VITE_BRAND_NAME` y `VITE_BRAND_TAGLINE` son provisionales.
- Vite incorpora estas variables en el build: hay que reconstruir si cambian.
- El archivo `.env` está en `.gitignore` y debe tener permisos `600`.

> Nunca agregues la clave secreta (`service_role` / `sb_secret_…`) al frontend ni a
> `.env`. Solo se usa en el panel de Supabase o en scripts administrativos.

## Supabase alojado

```bash
npx supabase login
npx supabase link --project-ref gxfytcquyobhhbjipbea
npx supabase db push
```

La semilla (`supabase/seed.sql`) crea la configuración y cinco recompensas:

```bash
# Opción A: CLI con seed habilitado en config.toml
npx supabase db push --include-seed

# Opción B: pegar supabase/seed.sql en el SQL Editor
```

No ejecutes `supabase db reset` contra producción: borra datos.

## Google OAuth

### 1. Google Cloud

1. APIs & Services → Credentials → **Create credentials → OAuth client ID**.
2. Tipo **Web application**.
3. **Authorized JavaScript origins:** `https://<dominio-final>`
4. **Authorized redirect URI:**
   `https://gxfytcquyobhhbjipbea.supabase.co/auth/v1/callback`
5. OAuth consent screen: tipo External, scopes `openid`, `email`, `profile`, estado
   **In production**.

### 2. Supabase

Authentication → Providers → Google:

- Habilitado.
- Client ID y Client Secret de Google Cloud (secreto solo del lado servidor).

Authentication → URL Configuration:

- **Site URL:** `https://<dominio-final>` (mientras tanto, `http://127.0.0.1:5173`).
- **Redirect URLs:**
  - `http://127.0.0.1:5173/**`
  - `http://localhost:5173/**`
  - `https://<dominio-final>/**`
  - `https://<dominio-final>/auth/callback`

El frontend redirige a `/auth/callback?next=…`; Supabase añade el código OAuth.

### 3. Verificar

1. Abrir `/login`.
2. Pulsar **Continuar con Google**.
3. Completar el consentimiento.
4. Confirmar que se crea `auth.users`, `profiles` y `user_roles` (`customer`).

## Registro con correo y contraseña

En Authentication → Providers → Email:

- Email habilitado.
- **Confirm email desactivado** (`mailer_autoconfirm = true`), para que el registro
  simple entre directo sin verificación.

La pantalla `/login` permite alternar entre **Entrar** y **Crear cuenta**. El alta
guarda el nombre en `raw_user_meta_data.full_name` y el trigger crea el perfil.

> Este modo es apropiado para la etapa inicial. Antes de abrir al público, decide si
> quieres exigir verificación de correo y activa un proveedor SMTP.

## Administrador inicial

`private.bootstrap_admins` contiene `drleocortex@gmail.com`. El trigger
`handle_new_user` concede `admin` a ese correo en el primer registro. Si la cuenta ya
existía, la migración del 23/10/2026 también la habilitó.

Comprobar tras el primer login:

```sql
select p.email, r.role
from public.profiles p
join public.user_roles r on r.user_id = p.id
where p.email = 'drleocortex@gmail.com';
```

Para habilitar a otra persona por base de datos:

```sql
insert into public.user_roles (user_id, role)
select id, 'admin'
from auth.users
where email = 'persona@dominio.com'
on conflict (user_id, role) do nothing;
```

Para retirar el rol:

```sql
delete from public.user_roles
where user_id = (select id from auth.users where email = 'persona@dominio.com')
  and role = 'admin';
```

La aplicación no crea ni modifica usuarios desde la interfaz.

## Ejecutar la aplicación

```bash
npm ci
npm run dev
```

Abrir `http://127.0.0.1:5173`.

## Solución de problemas

| Síntoma | Causa probable | Acción |
| --- | --- | --- |
| Aviso “Falta configurar Supabase” | `.env` ausente o vacío | Crear `.env` y reiniciar Vite |
| `Invalid API key` | Clave incorrecta o URL con `/rest/v1/` | Copiar la URL base y la clave pública correctas |
| Google devuelve `redirect_uri_mismatch` | URI no autorizada | Añadir la URI exacta en Google Cloud |
| El login vuelve a `/` | Redirect URL no permitida en Supabase | Añadir el dominio en Auth → URL Configuration |
| El registro por correo pide confirmación | `mailer_autoconfirm` desactivado | Activarlo en Authentication → Providers → Email |
| El QR abre `localhost` en otro teléfono | `VITE_PUBLIC_APP_URL` vacío | Definir el dominio público y reconstruir |
| `permission denied for table …` | Se intentó escribir sin RPC | Usar la operación de la interfaz; es el comportamiento esperado |
| `redemption_requires_admin` | Se llamó al canje antiguo del cliente | Aplicar el canje desde `/admin/customers/:id` |
| npm no conecta al registry | Resolución de red IPv6 del entorno | `NODE_OPTIONS=--no-network-family-autoselection npm install` |
| Docker no arranca para pruebas | Docker detenido | Iniciar Docker y reintentar `npm run test:db` |
