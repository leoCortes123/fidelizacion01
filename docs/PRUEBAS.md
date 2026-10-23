# Pruebas

## Comandos

```bash
npm run typecheck   # TypeScript
npm test            # Vitest + Testing Library
npm run test:db     # PostgreSQL 16 efímero en Docker
npm run test:e2e    # Playwright (backend Supabase simulado)
npm run test:all    # todo lo anterior en orden
```

## Unitarias y de componentes

`tests/unit` (29 pruebas):

| Archivo | Cubre |
| --- | --- |
| `navigation.test.ts` | Rutas internas de OAuth, rechazo de URLs externas, destino por rol |
| `format.test.ts` | COP, entradas numéricas, fechas de Bogotá, puntos |
| `errors.test.ts` | Traducción de errores de negocio |
| `LoginPage.test.tsx` | Google con `next`, login por correo, registro simple, destino seguro |
| `QrPage.test.tsx` | Validación del importe, previsualización y QR |
| `ClaimPage.test.tsx` | Reclamo, QR inválido y sesión ausente |
| `CustomerDashboard.test.tsx` | Saldo grande, correo, recompensas disponibles y puntos por compra |

## Base de datos

`tests/db/run-tests.sql` y `scripts/test-db.sh` levantan PostgreSQL 16 en Docker,
crean un stub de `auth` y aplican las migraciones y la semilla.

Cubre:

- trigger de perfiles y roles (incluido el admin inicial);
- autorización: cliente vs. admin, incluido `admin_redeem_reward`;
- generación de compras, snapshot de reglas y validación de importes;
- reclamo atómico, QR expirado, cancelación y doble reclamo;
- canje aplicado por administrador, idempotencia y anulación con devolución;
- libro inmutable, saldo no negativo y trigger diferido;
- RLS por tabla y denegación de escrituras directas;
- vencimiento de puntos configurable e idempotente;
- concurrencia real: dos reclamos simultáneos del mismo QR; solo uno gana y se
  acredita un único movimiento.

Resultado actual: **89 aserciones** y la prueba de concurrencia en verde.

## E2E

`tests/e2e` usa Playwright con un backend Supabase en memoria
(`fake-supabase.ts`). No sustituye a Supabase: valida la interfaz y el contrato de
las RPC.

Flujos (6):

1. Admin genera QR → cliente reclama → admin aplica canje en el detalle →
   cliente ve saldo e historial → admin anula y devuelve puntos.
2. Rechazo de reclamo duplicado.
3. Rechazo de QR expirado.
4. Administrador actualiza reglas y crea recompensas.
5. Un cliente no puede entrar a `/admin`.
6. Portal del cliente y panel de administración en viewport móvil (390×844).

## Smoke test real

Ejecutado contra el proyecto alojado el 23/10/2026 con usuarios temporales:

- registro por correo/contraseña;
- concesión de rol `admin`;
- `create_purchase(50000)` → 50 puntos;
- `claim_purchase` → saldo 50;
- listado de clientes por puntos;
- `admin_redeem_reward` (Taco individual, 40) → saldo 10;
- `list_my_redemptions` → estado `redeemed`;
- `admin_cancel_redemption` → saldo 50;
- limpieza de usuarios, compras, canjes, libro y auditoría temporal.

Los datos temporales se eliminaron al finalizar.

## Qué no cubren las pruebas

- Google OAuth contra Google real (requiere interacción del titular).
- Un E2E contra Supabase alojado con datos reales.
- Pruebas de carga o de accesibilidad automatizada con axe.
- Envío real de correos (la confirmación está desactivada).
