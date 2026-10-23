# Estado del proyecto — punto de retorno

**Última actualización:** 23/10/2026
**Directorio:** `/home/leonardo/Mi_Negocio`
**Estado:** aplicación funcional conectada a Supabase alojado; pruebas locales en
verde y smoke test real aprobado.
**Pendiente para producción:** desplegar en Vercel, actualizar el dominio en
`.env`, Supabase y Google, y probar OAuth con la cuenta real.

> **Para una sesión nueva:** leer primero este archivo completo. Resume el estado,
> las credenciales configuradas (sin exponerlas), los comandos y lo pendiente.

---

## 0. Cambios de la sesión del 23/10/2026

### Flujo de producto simplificado

- **Dos tipos de usuario:** `admin` y `customer`. El rol histórico `registrar` se
  conserva en el enum pero ya no da acceso al panel.
- **Administrador inicial:** `drleocortex@gmail.com` recibe `admin` automáticamente al
  registrarse (`private.bootstrap_admins` + trigger `handle_new_user`).
- **Panel exclusivo de administración** (`/admin`), con cuatro pestañas:
  - **Generar QR:** importe en COP, previsualización de puntos, QR, descarga,
    impresión y enlace.
  - **Clientes:** búsqueda, filtros, orden por puntos descendente por defecto;
    tarjetas en móvil y tabla en escritorio.
  - **Detalle del cliente:** saldo grande, datos, compras, movimientos, canjes y
    formulario para aplicar una recompensa.
  - **Compras:** historial y cancelación de QR pendientes/expirados.
  - **Configuración:** reglas, catálogo, auditoría, historial y vencimiento manual.
- **Canje aplicado por caja:** `admin_redeem_reward` descuenta los puntos de
  inmediato y deja el canje como `redeemed`. `admin_cancel_redemption` anula y
  devuelve los puntos. El cliente ya no genera códigos.
- **Cliente:** login con Google o correo/contraseña (registro simple sin
  verificación), saldo en grande, correo visible, catálogo con recompensas
  disponibles resaltadas, historial de compras con puntos ganados y canjes.
- **Diseño móvil primero:** navegación inferior, tarjetas en lugar de tablas,
  tipografía grande y objetivos táctiles amplios.

### Infraestructura

- Repositorio GitHub: `https://github.com/leoCortes123/fidelizacion01` (rama `main`).
- `.env` creado con permisos `600` y claves públicas del proyecto alojado
  (gitignored).
- Proyecto Supabase vinculado: `gxfytcquyobhhbjipbea` (`fidelizacion01`,
  `us-east-2`).
- Migraciones y seed aplicados, incluida `20261023000100_admin_flow.sql`.
- Google OAuth habilitado en Supabase; redirect URI confirmada en Google Cloud.
- Registro por correo habilitado con `mailer_autoconfirm = true`.
- Smoke test real con usuarios temporales: registro, rol admin, QR, reclamo, canje,
  historial y devolución. Datos temporales eliminados.

### Pruebas

- TypeScript sin errores.
- **29** unitarias/componentes.
- **89** aserciones SQL + concurrencia.
- **6** flujos E2E.
- Build correcto.

---

## 1. Credenciales y configuración

### Archivo `.env` (no versionado)

```dotenv
VITE_SUPABASE_URL=https://gxfytcquyobhhbjipbea.supabase.co
VITE_SUPABASE_ANON_KEY=sb_publishable_…
VITE_BRAND_NAME=Mi Negocio
VITE_BRAND_TAGLINE=Puntos por tus compras
VITE_PUBLIC_APP_URL=          # pendiente: URL de Vercel
```

### Supabase

| Dato | Valor |
| --- | --- |
| Project ref | `gxfytcquyobhhbjipbea` |
| Región | `us-east-2` |
| URL base | `https://gxfytcquyobhhbjipbea.supabase.co` |
| Clave usada en frontend | `publishable` |
| Site URL actual | `http://127.0.0.1:5173` |
| Redirect URLs actuales | `http://127.0.0.1:5173/**`, `http://localhost:5173/**` |
| Google provider | Habilitado |
| Confirmación de correo | Desactivada |

**Pendiente al tener Vercel:** añadir `https://<dominio>/**` y
`https://<dominio>/auth/callback`, cambiar Site URL y el origin de Google Cloud.

### Secretos

- La contraseña de la base y el Personal Access Token (`sbp_…`) se usaron solo para
  `supabase link` / `db push` y el smoke test. **Revocar el token** cuando termine la
  configuración.
- Google Client Secret está solo en el panel de Supabase; no está en el repo.
- `service_role` / `sb_secret_…` nunca se usó en el frontend.

---

## 2. Qué quedó implementado

### Base de datos

- `supabase/migrations/20251021000100_schema.sql`
  - Tablas: `profiles`, `user_roles`, `program_settings`, `purchases`, `rewards`,
    `redemptions`, `points_ledger`, `audit_log`, `config_history`,
    `private.rate_limits`.
  - Enums, FKs, `CHECK`, índices, unicidad, idempotencia, snapshot de reglas, libro
    inmutable y trigger diferido de saldo no negativo.
- `supabase/migrations/20251021000200_functions.sql`
  - RPC de cliente y administración.
  - Reclamo atómico con `FOR UPDATE`, rate limiting, auditoría, cancelaciones.
- `supabase/migrations/20251021000300_policies.sql`
  - RLS, matriz de permisos, revocación de escrituras directas y de ejecución
    pública.
- `supabase/migrations/20261023000100_admin_flow.sql`
  - `private.bootstrap_admins` y rol admin automático.
  - `is_staff()` = admin.
  - Listado de clientes sin administradores.
  - `admin_redeem_reward`, `admin_cancel_redemption`.
  - `customer_redeem_reward` deshabilitada.
  - Restricción de entrega ajustada para permitir anulación con trazabilidad.
- `supabase/seed.sql`: configuración inicial + 5 recompensas.

### Frontend

- `src/app/router.tsx`: rutas lazy, `/customer/*` y `/admin/*`, redirecciones
  heredadas.
- `src/components/layout/AdminShell.tsx`: panel de administración responsive.
- `src/components/layout/CustomerShell.tsx`: tres pestañas + acceso admin.
- `src/features/admin/`:
  - `QrPage.tsx`
  - `CustomersPage.tsx`
  - `CustomerDetailPage.tsx` (canje y anulación)
  - `PurchasesPage.tsx`
  - `SettingsPage.tsx`
  - `hooks.ts`
- `src/features/customer/`: inicio, compras y canjes.
- `src/features/auth/`: login Google + correo/contraseña, callback y sesión.
- `src/lib/errors.ts`: mensajes de auth y canje administrativo.

### Documentación

- `README.md`
- `docs/ARQUITECTURA.md`
- `docs/BASE_DE_DATOS.md`
- `docs/SEGURIDAD.md`
- `docs/CONFIGURACION.md`
- `docs/PRUEBAS.md`
- `docs/DESPLIEGUE.md`
- `docs/INFORME_FINAL.md`
- `vercel.json`, `public/_headers` (fallback SPA y cabeceras/CSP)

---

## 3. Última ejecución verificada

```bash
NODE_OPTIONS=--no-network-family-autoselection npm run test:all
```

| Suite | Resultado |
| --- | --- |
| TypeScript (`tsc -b`) | Sin errores |
| Unitarias/componentes | **29/29** |
| Base de datos | **89 aserciones** + concurrencia |
| E2E Playwright | **6/6** |
| Build de producción | Correcto (`dist/`); chunks separados |

Evidencia de concurrencia: dos reclamos simultáneos del mismo QR; solo uno gana y se
acredita un único movimiento.

Smoke test real (23/10/2026): registro por correo, rol admin, `create_purchase(50000)`,
`claim_purchase`, `admin_redeem_reward` (40), `admin_cancel_redemption`, saldo de 50 →
10 → 50. Limpieza completa.

---

## 4. Cómo retomar

```bash
cd /home/leonardo/Mi_Negocio

npm run typecheck
npm test
NODE_OPTIONS=--no-network-family-autoselection npm run test:all
npm run dev
```

Si Playwright no tiene navegador:

```bash
npx playwright install chromium
```

Supabase local:

```bash
npx supabase start
npx supabase db reset
```

Proyecto alojado:

```bash
npx supabase link --project-ref gxfytcquyobhhbjipbea
npx supabase db push
```

---

## 5. Pendientes que bloquean producción

1. **Crear repositorio en GitHub** y desplegar en Vercel.
2. **Obtener la URL de Vercel** y actualizar:
   - `.env` → `VITE_PUBLIC_APP_URL`.
   - Supabase → Site URL y Redirect URLs.
   - Google Cloud → Authorized JavaScript origin.
3. Reconstruir y desplegar con la URL final.
4. **Probar Google OAuth real** con `drleocortex@gmail.com`.
5. Verificar que la cuenta recibe `admin` y que un cliente no entra a `/admin`.
6. Probar un QR impreso desde un segundo teléfono con el dominio final.
7. Revisar CSP/cabeceras en el hosting definitivo.
8. Revocar el Personal Access Token de Supabase usado en la configuración.

---

## 6. Limitaciones conocidas

- Una moneda (COP), un local y una zona horaria de presentación.
- El vencimiento de puntos es manual desde `/admin/settings`; no hay cron.
- El vencimiento no implementa FIFO exacto por compra: expira créditos antiguos
  hasta el saldo disponible.
- No hay tiempo real: el cliente ve el saldo al reabrir o enfocar la app.
- El registro por correo no verifica el correo (decisión de esta etapa).
- El E2E usa un backend en memoria; RLS/transacciones se prueban en PostgreSQL real.
- El QR es una credencial portadora hasta que se reclama.
- La marca es provisional/configurable.
- La cuenta admin aplica canjes sin segunda aprobación; se mitiga con auditoría y
  anulación con devolución.

---

## 7. Próximos pasos sugeridos

1. Completar el despliegue en Vercel y el dominio.
2. Smoke test manual con dos dispositivos reales (QR impreso).
3. Añadir E2E real contra staging (sin datos personales).
4. Decidir política de vencimiento y, si se automatiza, crear una función de
   servicio en `private` con `SECURITY DEFINER` y auditoría.
5. Decidir si se exige verificación de correo o se deja solo Google.
6. Opcional: imágenes reales de recompensas.
7. Opcional: escaneo de QR con cámara para el canje (hoy el admin selecciona al
   cliente en una lista).

---

## 8. Archivos clave para revisar primero

- `src/app/router.tsx` — rutas, guards y lazy loading.
- `src/components/layout/AdminShell.tsx` — navegación del panel.
- `src/features/admin/CustomerDetailPage.tsx` — canje y anulación.
- `src/features/customer/DashboardPage.tsx` — saldo y catálogo del cliente.
- `src/features/auth/LoginPage.tsx` — Google + correo/contraseña.
- `supabase/migrations/20261023000100_admin_flow.sql` — nuevo flujo y permisos.
- `supabase/migrations/20251021000200_functions.sql` — lógica de negocio.
- `supabase/migrations/20251021000300_policies.sql` — RLS.
- `tests/db/run-tests.sql` — aserciones SQL.
- `tests/e2e/loyalty.spec.ts` — flujos de navegador.
- `vercel.json` / `public/_headers` — despliegue y CSP.

---

## 9. Handoff para la próxima sesión

> “Lee `ESTADO_PROYECTO.md` y `docs/INFORME_FINAL.md`. El proyecto ya está conectado
> a Supabase; continúa con el despliegue en Vercel, la URL final y la prueba de
> Google OAuth. No rehagas el flujo de canje ni el modelo de roles.”

Orden recomendado:

1. `ESTADO_PROYECTO.md` (este archivo).
2. `docs/INFORME_FINAL.md` y `docs/PRUEBAS.md`.
3. `docs/DESPLIEGUE.md` para Vercel y `docs/CONFIGURACION.md` para Auth.
4. Ejecutar `npm run test:all` antes de cambiar código.
5. Completar la sección 5 (pendientes de producción).
