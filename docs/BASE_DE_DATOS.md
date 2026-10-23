# Base de datos, RPC y RLS

Migraciones:

1. `20251021000100_schema.sql`: tipos, tablas, restricciones, índices, triggers.
2. `20251021000200_functions.sql`: lógica transaccional y consultas.
3. `20251021000300_policies.sql`: RLS, grants y revocación de funciones.
4. `20261023000100_admin_flow.sql`: canje por administrador, anulación con
   devolución, admin inicial y panel exclusivo de administración.

Semilla: `supabase/seed.sql` (configuración y catálogo, sin usuarios).

## Tipos

| Tipo | Valores |
| --- | --- |
| `app_role` | `customer`, `admin` (el valor histórico `registrar` ya no da acceso) |
| `purchase_status` | `pending_claim`, `claimed`, `expired`, `cancelled` |
| `points_movement_type` | `purchase_credit`, `redemption_debit`, `redemption_refund`, `expiry_debit`, `admin_adjustment` |
| `redemption_status` | `pending_delivery`, `redeemed`, `cancelled` |

## Tablas

### `profiles`

Perfil mínimo asociado a `auth.users` (1:1). Se crea por trigger; el cliente solo
puede actualizar `full_name` (grant a nivel de columna). El correo y el id no son
editables desde el navegador.

### `user_roles`

`(user_id, role)` como clave primaria. Toda cuenta recibe `customer` al
registrarse. El correo listado en `private.bootstrap_admins` recibe además `admin`
en el alta; el resto de roles se gestiona por base de datos.

### `program_settings`

Fila única (`id = true`) con:

- `cop_per_point_unit` (por defecto 1000),
- `min_purchase_amount`,
- `max_points_per_purchase`,
- `qr_expiry_hours` (24 por defecto),
- `points_expiry_days` (nullable = sin vencimiento),
- `history_page_size`.

Los cambios se registran en `config_history` y `audit_log`. Aplican solo a compras
nuevas.

### `purchases`

- `amount_cop integer` (`> 0`, `<= 50.000.000`).
- `registrar_id` obligatorio.
- `token_hash char(64)` único; restricción de formato hexadecimal.
- `status` con restricciones de consistencia para `claimed` y `cancelled`.
- `claimed_by` es una sola columna: una compra no puede asociarse a dos clientes.
- `expires_at > created_at`.
- `settings_snapshot jsonb` con las reglas vigentes al crear la compra.
- `points_awarded` se llena al reclamar.

Índices: cliente/fecha, estado/expiración, registrador/fecha.

### `rewards`

Nombre único (sin distinguir mayúsculas), descripción, imagen opcional
(`http/https`), costo en puntos, disponibilidad.

### `redemptions`

- Cliente, recompensa y **snapshot** del nombre/costo.
- `code` único, formato alfanumérico en mayúsculas, índice único.
- `(customer_id, idempotency_key)` único.
- Restricciones de consistencia para `redeemed` y `cancelled`. Un canje cancelado
  conserva `delivered_at`/`delivered_by` como trazabilidad.
- `delivered_by` y `cancelled_by` registran al responsable. El canje del panel nace
  como `redeemed`.

### `points_ledger`

Libro mayor inmutable:

- `points` con signo; la restricción `points_ledger_type_sign` exige signo y
  referencias coherentes por tipo.
- `idempotency_key` único.
- Índices únicos parciales:
  - un crédito por compra,
  - un débito por canje,
  - una devolución por canje.
- Trigger `points_ledger_immutable` bloquea `UPDATE` y `DELETE` para todos.
- Trigger de restricción diferido `points_ledger_nonnegative` comprueba al final de
  la transacción que el saldo no sea negativo.

### `audit_log`

Actor, correo histórico, operación, entidad, metadatos y fecha. Solo lectura para
administradores. No se guardan tokens completos ni secretos.

### `config_history`

Snapshot `before`/`after` de cada cambio de configuración. Solo lectura para
administradores.

### `private.rate_limits`

Buckets por usuario/operación con ventana temporal. No se expone por la API.

### `private.bootstrap_admins`

Lista de correos que reciben `admin` automáticamente al registrarse. Incluye
`drleocortex@gmail.com`. Solo se modifica por SQL; el esquema `private` no está
expuesto y no tiene grants para `anon`/`authenticated`.

## Funciones transaccionales (RPC)

### Cliente

| Función | Propósito |
| --- | --- |
| `get_my_roles()` | Roles del usuario autenticado. |
| `get_my_profile()` | Perfil propio. |
| `get_program_settings()` | Reglas públicas para la interfaz. |
| `get_my_dashboard()` | Saldo, compras, próxima recompensa y pendientes. |
| `list_my_purchases(limit, offset)` | Historial paginado propio (importe/fecha). |
| `list_active_rewards()` | Catálogo con progreso desde el saldo real. |
| `list_my_redemptions()` | Canjes propios aplicados por caja. |
| `claim_purchase(token)` | Reclamo atómico + acreditación. |

`customer_redeem_reward` quedó deshabilitada: su `EXECUTE` fue revocado y el cuerpo
responde `redemption_requires_admin`.

### Administración / operación

Todas exigen rol `admin`. Los nombres `staff_*` se conservan por compatibilidad.

| Función | Propósito |
| --- | --- |
| `create_purchase(amount)` | Genera compra, token y snapshot. |
| `staff_cancel_purchase(id, reason)` | Cancela pendientes/expiradas; registra responsable. |
| `staff_list_customers(…)` | Búsqueda, orden y paginación; excluye administradores. |
| `staff_get_customer(id)` | Ficha con compras, movimientos y canjes. |
| `staff_list_purchases(…)` | Historial operativo; reconcilia expiraciones. |
| `admin_redeem_reward(customer, reward, key, reason)` | Canje inmediato, idempotente y auditado. |
| `admin_cancel_redemption(id, reason)` | Anula un canje y devuelve los puntos. |
| `staff_list_redemptions(status, limit)` | Canjes entregados/cancelados (histórico). |
| `staff_lookup_redemption(code)` | Verifica un código heredado. |
| `staff_confirm_redemption(code)` | Compatibilidad con canjes pendientes antiguos. |
| `staff_cancel_redemption(id, reason)` | Cancela pendientes heredados y devuelve puntos. |

### Administración

| Función | Propósito |
| --- | --- |
| `admin_get_settings()` / `admin_update_settings(jsonb)` | Configuración validada + historial. |
| `admin_list_rewards()` / `admin_upsert_reward(…)` | Catálogo. |
| `admin_list_staff()` / `admin_search_profiles(query)` | Gestión de equipo. |
| `admin_grant_role(user, role)` / `admin_revoke_role(…)` | Roles con protección del último admin. |
| `admin_get_audit_log(limit, offset)` | Auditoría. |
| `admin_get_config_history(limit)` | Historial de configuración. |
| `admin_expire_points()` | Vencimiento explícito e idempotente. |

## Matriz RLS

| Tabla | Cliente | Registrador | Admin |
| --- | --- | --- | --- |
| Tabla | Cliente | Admin |
| --- | --- | --- |
| `profiles` | SELECT/UPDATE solo propio (`full_name`) | SELECT todos |
| `user_roles` | SELECT propio | SELECT todos |
| `purchases` | SELECT reclamadas propias | SELECT todas |
| `points_ledger` | SELECT propio | SELECT todos |
| `rewards` | SELECT activas | SELECT todas |
| `redemptions` | SELECT propios | SELECT todos |
| `program_settings` | SELECT | SELECT |
| `audit_log` | — | SELECT |
| `config_history` | — | SELECT |

No existen políticas de `INSERT/UPDATE/DELETE` para clientes. Las escrituras pasan
por funciones `SECURITY DEFINER` que validan el rol. Además se revocaron los grants
directos de escritura y la ejecución pública de funciones; solo se concedió
`EXECUTE` a `authenticated` para las RPC necesarias.

## Concurrencia e idempotencia

- **Dos reclamos del mismo QR:** `SELECT … FOR UPDATE` + índice único de crédito;
  solo una transacción gana.
- **Dos canjes simultáneos (admin):** bloqueo de asesoría por cliente + verificación
  de saldo + trigger diferido de saldo no negativo.
- **Reintento de canje:** `(customer_id, idempotency_key)` único; devuelve el mismo
  canje sin volver a descontar.
- **Anulación:** único parcial de devolución por canje; el estado pasa a `cancelled`.
- **Doble crédito:** `idempotency_key = 'purchase:<id>'` y único parcial por compra.
- **Doble devolución:** único parcial por canje.

## Vencimiento

- La vigencia de cada QR se evalúa al reclamar. Los listados del personal reconcilian
  compras vencidas (`private.expire_pending_purchases`).
- `points_expiry_days` es configurable. `admin_expire_points()` expira créditos
  antiguos hasta donde el saldo actual lo permita, con clave
  `expiry:<credit_id>`. Es idempotente y debe ejecutarse explícitamente (manual o
  `pg_cron`). Si el saldo no alcanza, el crédito se marca como vencido por el monto
  disponible; no se implementa un consumo FIFO por compra individual.
