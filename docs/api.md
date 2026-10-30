# Diseño API — Finanzas personales v1

Stack: Python + FastAPI + PostgreSQL (decidido 2026-10-30). Esquemas con Pydantic v2, modelos con SQLAlchemy, migraciones con Alembic.
Base: `/api/v1`
Idioma de respuestas y errores: español. Zona horaria: America/Caracas.
Autenticación: OAuth2 con JWT, `Authorization: Bearer <access_token>`. Sin token → 401. Sin rol suficiente → 403.
Formato de error: `{ "detail": "..." }` en errores simples y `{ "detail": [{ "loc": ["campo"], "msg": "..." }] }` en validación (422, formato estándar de FastAPI).
Paginación: `?page=&per_page=` (máx. 100). Respuesta lista: `{ "data": [...], "meta": { "current_page", "per_page", "total" } }`.

## Reglas transversales

- **Idempotencia**: en `POST /transactions` y `POST /imports` se acepta `Idempotency-Key`. Repetir la misma clave devuelve el resultado original, no crea otro registro.
- **Deduplicación**: hash SHA-256 de `cuenta + descripción normalizada + monto en unidades menores + moneda + fecha`. Si ya existe una transacción activa con ese hash, la API devuelve 409 con la transacción existente, salvo que se envíe `force_duplicate=true` con motivo (queda registrado).
- **Dinero**: siempre entero en unidades menores (`amount_minor`, ej. 6680 = $66.80) + `currency` (USD, VES, EUR, USDT, BNB). Nunca `float` en request ni response.
- **Filtros comunes**: `?month=2026-11&category_id=&account_id=&status=confirmado|por_confirmar|deuda_pendiente&type=gasto|ingreso|transferencia&q=`.
- **Fechas**: `YYYY-MM-DD` para fecha de la transacción; timestamps en ISO 8601.

## Endpoints

### Auth y hogar
- `POST /auth/login` (form OAuth2: `username` + `password`) → `{ access_token, refresh_token, token_type: "bearer", user }` (solo usuarios creados por el admin; no hay registro público)
- `POST /auth/refresh` → nuevo access token con el refresh token vigente
- `POST /auth/logout` → revoca el refresh token
- `GET /me`
- `GET /household` (admin y registro: datos del hogar, sin secretos)

### Cuentas
- `GET /accounts` · `POST /accounts` (admin) · `GET /accounts/{id}` · `PATCH /accounts/{id}` (admin) · `DELETE /accounts/{id}` (admin, solo si no tiene transacciones; si las tiene, se archiva con `archived_at`)
- Campos: `name` (ej. Efectivo, Cuenta bancaria, PayPal, USDT), `type`: efectivo|banco|paypal|cripto|tarjeta, `currency`, `opening_balance_minor`, `is_main_fund` (fondo principal), `archived_at`
- `GET /accounts/{id}/balance?at=2026-10-31` → saldo calculado desde transacciones, no guardado a mano

### Categorías
- `GET /categories` · `POST /categories` (admin) · `PATCH /categories/{id}` (admin)
- Campos: `name`, `parent_id` (nullable), `kind`: necesario|innecesario, `is_income_category` (para aporte de otro miembro del hogar y salarios), `archived_at`
- Semilla v1 desde la migración: Educación, Supermercado, Consumo, Transferencia, Pago móvil, Hosting, Vehículo, Comisiones, Internet/TV, más las que salgan de la migración sin duplicar.

### Transacciones (núcleo)
- `GET /transactions` (filtros comunes + `status`, `type`)
- `POST /transactions` (admin y registro)
- `GET /transactions/{id}` · `PATCH /transactions/{id}` (corregir descripción, categoría, fecha; el monto y la cuenta solo los corrige el admin y queda en auditoría)
- `POST /transactions/{id}/confirm` → `por_confirmar` → `confirmado`
- `POST /transactions/{id}/move-to-month` (deuda_pendiente → mes destino, guarda `original_month`)
- `POST /transactions/{id}/void` (admin, con `reason`; no borra)
- Campos de creación:
```json
{
  "type": "gasto",
  "date": "2026-11-03",
  "account_id": 1,
  "destination_account_id": null,
  "category_id": 4,
  "description": "Compra de ejemplo",
  "amount_minor": 10000,
  "currency": "USD",
  "exchange_rate": null,
  "base_amount_minor": 10000,
  "base_currency": "USD",
  "status": "deuda_pendiente",
  "due_month": "2026-11",
  "receipt_url": null,
  "is_opening_balance": false,
  "splits": [
    { "fund": "FUND_MAIN", "amount_minor": 6000 },
    { "fund": "FUND_CONTRIBUTOR", "amount_minor": 4000 }
  ]
}
```
- Reglas: gasto exige `account_id` + `category_id`; transferencia exige `account_id` + `destination_account_id` y no admite categoría de gasto; ingreso con `is_opening_balance=true` marca saldo inicial; `splits`, si vienen, deben sumar `amount_minor`; aporte de otro miembro del hogar es ingreso con categoría de ingreso, nunca gasto.

### Presupuestos
- `GET /budgets?month=2026-11` · `POST /budgets` (admin) · `PATCH /budgets/{id}` (admin)
- Campos: `month` (YYYY-MM), `category_id`, `amount_minor`, `currency`. Único por hogar + mes + categoría.
- `GET /budgets/{month}/vs-real` → por categoría: plan, real confirmado, real pendiente, diferencia.

### Suscripciones
- `GET /subscriptions` · `POST /subscriptions` (admin) · `PATCH /subscriptions/{id}` (admin)
- Campos: `name`, `account_id`, `category_id`, `amount_minor`, `currency`, `charge_day` (1-31), `status`: activa|pausada|cancelada
- Al llegar `charge_day`, el sistema genera la transacción en `por_confirmar` enlazada por `subscription_id`. Nunca se marca pagada sola.

### Tasas de cambio
- `GET /exchange-rates?from=EUR&to=USD&date=2026-11-01`
- `POST /exchange-rates` (admin): `from_currency`, `to_currency`, `rate` (string decimal, ej. "1.16"), `date`, `source` (ej. "conversión aplicada al salario", "tasa BCV del día")
- Cada transacción en moneda distinta de USD guarda su tasa en el momento del registro.

### Informes (mismo formato del Excel)
- `GET /reports/monthly?month=2026-11` →
```json
{
  "month": "2026-11",
  "summary": {
    "by_category": [{ "category": "Supermercado", "kind": "necesario", "confirmed_minor": 0, "pending_minor": 0 }],
    "totals": { "income_confirmed_minor": 0, "expense_confirmed_minor": 0, "expense_pending_minor": 0 }
  },
  "detail": [ { "date": "2026-11-03", "description": "...", "amount_minor": 10000, "currency": "USD", "status": "deuda_pendiente", "account": "Efectivo", "category": "Supermercado" } ]
}
```
- El detalle es completo, sin recorte por páginas. Confirmado y pendiente van separados, nunca sumados en silencio.
- `GET /reports/funds` → saldos de los fondos del hogar y de la alcancía (con su meta).

### Migración e importaciones
- `POST /imports/previous-system/dry-run` y `POST /imports/spreadsheet/dry-run` → informe de totales origen vs destino, duplicados detectados, categorías nuevas.
- `POST /imports/commit` (admin, solo tras un dry-run cuadrado) → importa en firme y deja constancia de la fecha de corte.
- `GET /imports/{id}` → resultado de la importación (creados, duplicados omitidos, errores).

## Qué NO hace la API en v1

No hay registro público, no hay subida de archivos de recibos (solo `receipt_url`), no hay conexión bancaria automática, no hay endpoints para terceros ni multi-tenant.
