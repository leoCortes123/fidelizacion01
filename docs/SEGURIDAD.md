# Seguridad

## Principios

1. **Cero confianza en el navegador.** La identidad se obtiene de `auth.uid()` en
   PostgreSQL; nunca de un `user_id` enviado por el cliente.
2. **Autorización en el servidor.** Cada RPC valida rol y estado dentro de la misma
   transacción.
3. **Mínimo privilegio.** El cliente solo recibe la clave `anon`; no hay
   `service_role` en el frontend.
4. **Escrituras mediadas.** No hay grants de `INSERT/UPDATE/DELETE` para clientes en
   tablas críticas; se usan funciones `SECURITY DEFINER`.
5. **Trazabilidad.** Las operaciones administrativas y las transiciones de estado se
   registran en `audit_log`.

## Amenazas y mitigaciones

| Amenaza | Mitigación |
| --- | --- |
| Cliente intenta asignarse `admin` | `user_roles` sin políticas de escritura y sin grants; solo RPC de admin. Prueba SQL de escalada. |
| Cliente modifica su saldo | `points_ledger` sin escritura directa, trigger de inmutabilidad y saldo derivado. |
| Cliente altera el importe de una compra | Las compras no tienen política de escritura; el importe solo lo fija `create_purchase`. |
| Reclamar el mismo QR dos veces | `FOR UPDATE`, validación de estado y único parcial de crédito por compra. |
| Dos personas reclaman el mismo QR | La segunda transacción ve `claimed` y falla; prueba de concurrencia. |
| Cliente intenta canjear desde la API | `customer_redeem_reward` sin `EXECUTE` y cuerpo que responde `redemption_requires_admin`. |
| Administrador aplica un canje equivocado | `admin_cancel_redemption` devuelve los puntos y deja rastro en auditoría. |
| Canjear sin saldo | Bloqueo por cliente, verificación y trigger diferido de saldo no negativo. |
| Reintento de red duplica un canje | Clave de idempotencia única por cliente; devuelve el mismo código. |
| Fuerza bruta de tokens/códigos | Tokens de 256 bits; límites de frecuencia por usuario y operación. |
| Fuga del token del QR | Solo se guarda SHA-256; no se registra en auditoría ni logs; el QR no incluye importe/puntos. |
| Acceso a datos de otros clientes | RLS por `auth.uid()`; el personal accede solo con rol verificado. |
| Usuario sin rol admin intenta operar | `is_staff()` (equivale a `admin`) dentro de cada RPC y guard `RequireAdmin` en el frontend. |
| Eliminar al último admin | `admin_revoke_role` bloquea `last_admin` y la auto-revocación. |
| Manipulación de configuración | RPC de admin, validaciones `CHECK`, historial `config_history` y auditoría. |
| Inyección SQL en búsquedas | Consultas parametrizadas y orden por lista blanca en `staff_list_customers`. |
| XSS | React escapa contenido; no se usa `dangerouslySetInnerHTML`; URLs de imagen restringidas a `http/https`; CSP en el hosting. |
| CSRF | Autenticación por bearer token en cabecera, no cookies de sesión. |
| Abuso de la cuenta admin inicial | El correo debe estar verificado por Google/Supabase; se recomienda activar 2FA en esa cuenta y rotar el correo en `private.bootstrap_admins` cuando exista otro admin. |

## Tokens y secretos

- **Token de compra:** 32 bytes aleatorios, Base64 URL-safe. Se devuelve una vez al
  registrador y se guarda solo como hash. La URL `/claim/<token>` es la única
  credencial; quien la posee puede reclamar la compra antes que el cliente.
- **Código de canje:** aleatorio y único; se conserva por compatibilidad, pero el
  flujo nuevo no lo muestra al cliente. El canje nace entregado.
- **Clave `anon`:** única clave en el frontend. Es pública por diseño, pero RLS y
  RPC son la verdadera barrera.
- **`service_role`:** nunca se incluye en `.env` del frontend ni en el repositorio.
- **Logs:** la aplicación no imprime tokens. La auditoría guarda solo IDs y montos.
- **Errores:** los mensajes de negocio son códigos estables; no exponen datos de
  otras cuentas.

## Rate limiting

`private.consume_rate_limit(bucket, máximo, ventana)` se invoca en:

| Operación | Límite |
| --- | --- |
| Crear compra | 30/min por usuario |
| Reclamar compra | 20/min por usuario |
| Canje aplicado por admin | 60/min por administrador |
| Buscar código | 60/min por usuario |
| Confirmar entrega | 40/min por usuario |
| Cancelar compra/canje | 20/min por usuario |
| Cambios de configuración/roles | 20–30/min por usuario |

Al superar el límite se devuelve `rate_limited` y la interfaz muestra un mensaje
comprensible.

## RLS y privilegios

- RLS habilitado en todas las tablas públicas.
- Políticas explícitas de `SELECT`; sin políticas de escritura para clientes.
- `profiles` solo permite actualizar `full_name` mediante grant de columna.
- `audit_log` y `config_history` solo son legibles por administradores.
- `REVOKE ALL` sobre tablas y `REVOKE EXECUTE` sobre funciones para `public`,
  `anon` y `authenticated`; luego se conceden solo las RPC necesarias.
- Las funciones `SECURITY DEFINER` fijan `search_path = public, pg_temp` y validan
  rol/identidad dentro de la función.

La matriz completa está en [`BASE_DE_DATOS.md`](BASE_DE_DATOS.md).

## Primer administrador

`private.bootstrap_admins` contiene `drleocortex@gmail.com`. El trigger
`handle_new_user` concede `admin` a ese correo al registrarse, y la migración del
23/10/2026 también habilitó cuentas existentes.

Verificación:

```sql
select p.email, r.role
from public.profiles p
join public.user_roles r on r.user_id = p.id
where p.email = 'drleocortex@gmail.com';
```

La lista solo se modifica por SQL:

```sql
insert into private.bootstrap_admins (email) values ('otra@dominio.com')
on conflict do nothing;
```

Para habilitar una cuenta ya registrada:

```sql
insert into public.user_roles (user_id, role)
select id, 'admin' from auth.users
where email = 'persona@dominio.com'
on conflict (user_id, role) do nothing;
```

La interfaz no crea usuarios ni asigna roles: eso se hace por base de datos en esta
etapa. Para retirar un administrador debe existir al menos otro; `admin_revoke_role`
bloquea el último y la auto-revocación.

## Registro con correo y contraseña

La confirmación por correo está desactivada (`mailer_autoconfirm = true`) para que el
registro simple entre directo. Implicaciones:

- Cualquiera puede crear una cuenta con un correo que no controla; el correo se usa
  solo como identificador y para iniciar sesión.
- No se envían enlaces mágicos ni recuperación de contraseña automática.
- Antes de abrir al público, decide si activas verificación y un proveedor SMTP, o si
  mantienes solo Google como método de alta.

## Google OAuth

- El flujo usa PKCE y `detectSessionInUrl`.
- El destino se valida contra rutas internas (`safeNextPath`), evitando open
  redirects.
- El token de reclamación se conserva en el parámetro `next` y en `sessionStorage`.
- El provider está configurado contra el proyecto alojado, pero el consentimiento
  real debe probarse con la cuenta del titular y el dominio final. Pasos en
  [`CONFIGURACION.md`](CONFIGURACION.md).

## Recomendaciones de despliegue

- HTTPS obligatorio.
- Configurar CSP permitiendo solo el dominio de Supabase, Google Fonts y el propio
  origen; evitar `unsafe-inline` si se migra a estilos externos.
- Restringir las redirect URLs de Supabase al dominio real.
- Rotar la clave `anon` si se filtra (no compromete RLS, pero conviene).
- Revisar `audit_log` y ejecutar `admin_expire_points` según la política de
  vencimiento.
- Mantener las dependencias actualizadas y ejecutar `npm audit` en CI.
- No usar la base de producción para pruebas destructivas.

## Riesgos residuales conocidos

- La URL del QR es un portador: si se comparte antes de reclamar, otra cuenta puede
  usarla. Mitigación: vigencia corta configurable, rate limiting y confirmación
  explícita.
- La cuenta administradora puede aplicar canjes sin una segunda aprobación. La
  auditoría y la anulación con devolución son el control compensatorio.
- La expiración de compras se reconcilia de forma perezosa al listar; el reclamo
  siempre rechaza por fecha aunque el estado aún diga `pending_claim`.
- El vencimiento de puntos no se ejecuta solo por defecto; requiere cron/manual.
- El backend E2E es simulado; las garantías de RLS/transacciones se validan en las
  pruebas SQL contra PostgreSQL real, no en Playwright.
