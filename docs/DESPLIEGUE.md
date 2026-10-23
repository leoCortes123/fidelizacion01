# Despliegue

## 1. Preparar Supabase

Proyecto actual: `gxfytcquyobhhbjipbea` (región `us-east-2`).

```bash
npx supabase login
npx supabase link --project-ref gxfytcquyobhhbjipbea
npx supabase db push
```

Verificar:

```sql
select count(*) from public.rewards;          -- 5
select * from public.program_settings;
select relname, relrowsecurity
from pg_class
where relnamespace = 'public'::regnamespace and relkind = 'r'
order by relname;
```

Si aplicaste las migraciones antes del 23/10/2026, vuelve a ejecutar `db push` para
incluir `20261023000100_admin_flow.sql`.

## 2. Auth y Google

Seguir [`CONFIGURACION.md`](CONFIGURACION.md). En producción:

- Google Authorized redirect URI:
  `https://gxfytcquyobhhbjipbea.supabase.co/auth/v1/callback`
- Supabase Site URL: `https://<dominio-de-vercel>`
- Supabase Redirect URLs:
  - `https://<dominio-de-vercel>/**`
  - `https://<dominio-de-vercel>/auth/callback`

## 3. Primer administrador

`drleocortex@gmail.com` recibe `admin` automáticamente al registrarse. Después del
primer login, comprobar en SQL:

```sql
select p.email, r.role
from public.profiles p
join public.user_roles r on r.user_id = p.id
where p.email = 'drleocortex@gmail.com';
```

## 4. Variables de build

```bash
VITE_SUPABASE_URL="https://gxfytcquyobhhbjipbea.supabase.co" \
VITE_SUPABASE_ANON_KEY="sb_publishable_…" \
VITE_PUBLIC_APP_URL="https://<dominio-de-vercel>" \
VITE_BRAND_NAME="Mi Negocio" \
VITE_BRAND_TAGLINE="Puntos por tus compras" \
npm run build
```

Subir `dist/` a Vercel, Netlify, Cloudflare Pages o un servidor estático.

## 5. Vercel

Cuando exista el repositorio en GitHub:

1. Importar el repositorio en Vercel.
2. Framework preset: **Vite**.
3. Build command: `npm run build`; output: `dist`.
4. Configurar las variables de entorno del paso 4 (Production y Preview).
5. Añadir `vercel.json` (incluido) para el fallback SPA.
6. Desplegar y copiar la URL final (por ejemplo
   `https://puntos-negocio.vercel.app`).
7. Actualizar `VITE_PUBLIC_APP_URL` con esa URL y volver a desplegar.
8. Actualizar Site URL y Redirect URLs en Supabase, y el JavaScript origin en Google
   Cloud.

Mientras el dominio sea provisional, los QR impresos apuntarán a él. Si más adelante
cambia el dominio, conserva una redirección desde el anterior.

### Reglas del hosting

- Fallback SPA para `/claim/*`, `/auth/callback`, `/customer/*` y `/admin/*`.
- HTTPS obligatorio.
- No cachear `index.html`; los assets con hash sí pueden cachearse.
- Cabeceras recomendadas (ver `vercel.json` y `public/_headers`):
  - CSP con `connect-src` al proyecto Supabase,
  - `X-Content-Type-Options: nosniff`,
  - `Referrer-Policy: strict-origin-when-cross-origin`,
  - `Permissions-Policy` sin cámara/micrófono (no se usan).

Ejemplo de CSP:

```text
default-src 'self';
connect-src 'self' https://gxfytcquyobhhbjipbea.supabase.co;
img-src 'self' data: https:;
style-src 'self' 'unsafe-inline';
font-src 'self' data:;
script-src 'self';
frame-ancestors 'none';
base-uri 'self';
form-action 'self';
```

Vite sirve el JS como módulo desde el propio origen; no hace falta `unsafe-inline`
para scripts. Tailwind inyecta estilos en un archivo, pero algunos componentes usan
estilos inline, por eso `style-src` incluye `'unsafe-inline'`.

## 6. Verificación posterior

- [ ] La landing carga con la marca elegida.
- [ ] El botón de Google abre el consentimiento.
- [ ] El registro con correo entra sin verificación.
- [ ] Tras el login aparece `/customer` con el saldo en grande.
- [ ] `drleocortex@gmail.com` entra a `/admin` y ve las cuatro pestañas.
- [ ] Un administrador genera un QR y el cliente lo abre en **otro dispositivo**.
- [ ] El cliente reclama una sola vez y ve los puntos y el historial.
- [ ] El administrador aplica un canje desde el detalle del cliente.
- [ ] El saldo baja de inmediato y el cliente ve el canje en **Canjes**.
- [ ] Anular el canje devuelve los puntos.
- [ ] Un cliente no entra a `/admin` (redirige a `/unauthorized`).
- [ ] Los cambios de configuración quedan en auditoría.
- [ ] El QR impreso usa el dominio público, no `localhost`.

## 7. Vencimiento de puntos (opcional)

`admin_expire_points()` exige sesión de administrador, por lo que se ejecuta desde
`/admin/settings` → **Ejecutar vencimiento**. Para automatizarlo con `pg_cron` hace
falta una función de servicio en `private` con `SECURITY DEFINER`, sin `EXECUTE`
para `authenticated`, y auditoría propia. La implementación actual no programa el
vencimiento.

## 8. Rollback

- Frontend: volver al deployment anterior. Los QR siguen válidos mientras no cambie
  el dominio ni la base.
- Base de datos: las migraciones son aditivas. Para revertir datos usa backups de
  Supabase; no borres tablas con información de clientes.
- Si se revoca un administrador por error, vuelve a insertar el rol con el SQL del
  primer administrador.

## 9. Operación

- Revisar `audit_log` periódicamente.
- Mantener al menos dos administradores activos.
- Configurar alertas de errores del hosting.
- Rotar credenciales de Google si se comprometen.
- No ejecutar `supabase db reset` en producción.
- Probar los cambios en un proyecto de staging antes de aplicarlos.
