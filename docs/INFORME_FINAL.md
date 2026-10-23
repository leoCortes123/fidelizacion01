# Informe final

## Resumen

Aplicación web de fidelización lista para operar en local y conectada a un proyecto
Supabase alojado. El flujo se simplificó el 23/10/2026:

- **Cliente:** entra con Google o correo/contraseña; consulta saldo, recompensas e
  historial de compras. No aplica canjes desde la app.
- **Administrador:** genera el QR de la compra, consulta clientes ordenados por
  puntos, aplica el canje de una recompensa y puede anularlo con devolución.

## Entregado

### Base de datos

- Esquema con perfiles, roles, configuración, compras, recompensas, canjes, libro de
  puntos inmutable, auditoría, historial de configuración y rate limiting.
- RLS en todas las tablas públicas y escrituras solo por RPC `SECURITY DEFINER`.
- Reclamo atómico con `FOR UPDATE` e idempotencia.
- Canje administrativo idempotente con bloqueo por cliente.
- Anulación de canje con devolución y trazabilidad.
- Administrador inicial automático por correo (`private.bootstrap_admins`).
- Migraciones aplicadas al proyecto `gxfytcquyobhhbjipbea`.

### Frontend

- Portal del cliente móvil-first: saldo en tipografía grande, correo visible,
  recompensas con estado “Disponible/Faltan X”, historial con puntos por compra y
  canjes aplicados.
- Panel de administración con cuatro pestañas: Generar QR, Clientes, Compras y
  Configuración.
- Detalle de cliente con saldo, compras, movimientos, canjes y formulario para
  aplicar una recompensa.
- Login con Google y registro simple por correo (sin verificación).
- Rutas antiguas redirigidas; guards de admin en frontend y base de datos.
- Navegación inferior en móvil y tarjetas en lugar de tablas en pantallas pequeñas.

### Pruebas

| Suite | Resultado |
| --- | --- |
| TypeScript | Sin errores |
| Unitarias/componentes | 29/29 |
| Base de datos | 89 aserciones + concurrencia |
| E2E Playwright | 6/6 |
| Build | Correcto, chunks separados |

Smoke test real contra Supabase alojado: registro, rol admin, creación de QR,
reclamo, canje, historial y anulación verificados; datos temporales eliminados.

### Documentación

`README.md`, `ESTADO_PROYECTO.md` y `docs/` (arquitectura, base de datos, seguridad,
configuración, pruebas, despliegue).

## Pendiente para producción

1. Crear el repositorio en GitHub y desplegar en Vercel.
2. Copiar la URL final a `VITE_PUBLIC_APP_URL` y reconstruir.
3. Actualizar Site URL y Redirect URLs en Supabase, y el JavaScript origin en Google
   Cloud.
4. Probar el consentimiento real de Google con `drleocortex@gmail.com`.
5. Verificar que esa cuenta recibe el rol `admin` y que un cliente no entra a
   `/admin`.
6. Probar un QR impreso desde un segundo teléfono con el dominio final.
7. Revisar CSP y cabeceras en el hosting definitivo.

## Limitaciones conocidas

- Una moneda (COP), un local y una zona horaria de presentación.
- El vencimiento de puntos es manual desde `/admin/settings`; no hay cron.
- El vencimiento no implementa FIFO exacto por compra.
- No hay tiempo real: el cliente ve el saldo al reabrir o enfocar la app.
- El registro por correo no verifica el correo (decisión de esta etapa).
- El E2E usa un backend simulado; RLS y transacciones se prueban en PostgreSQL real.
- El QR sigue siendo una credencial portadora mientras no se reclame.
- La marca es provisional y configurable.

## Criterio de aceptación

Se considera listo para producción cuando los siete pasos pendientes estén
completados y el flujo completo se haya ejecutado en el dominio final con dos
dispositivos reales.
