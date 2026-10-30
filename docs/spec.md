# Finanzas personales — Spec v1

Stack: Python + FastAPI + PostgreSQL.
Decisión tomada: **reemplaza el sistema anterior (hoja de cálculo + aplicación de finanzas)**. No convive con él.
Estado: spec + diseño API + ER listos en revisión. **No hay código subido, no hay issues, no hay cron ni deploy todavía.** La fase de construcción empieza solo cuando se apruebe este documento (con o sin correcciones).

## 1. Problema que resuelve

Las finanzas del hogar viven hoy en dos piezas que hay que mantener a mano: una hoja de cálculo y una aplicación anterior. Cada gasto nuevo exige doble cuidado y la deduplicación es manual. La nueva app debe ser **una sola fuente de verdad**, con las reglas del hogar ya incorporadas.

## 2. Alcance v1: qué entra

1. Cuentas: efectivo, banco, PayPal, cripto (USDT, BNB y otras si las hay).
2. Transacciones: ingreso, gasto y transferencia entre cuentas propias.
3. Categorías con marca necesario / innecesario y jerarquía (padre / subcategoría).
4. Estados de verificación: `confirmado`, `por_confirmar`, `deuda_pendiente` (un gasto puede quedar en DEUDAS para el mes siguiente, no como gasto pagado del mes actual).
5. Reparto de facturas compartidas: una compra se reparte entre los fondos del hogar.
6. Presupuestos mensuales por categoría, con real contra plan. Monto a definir en la migración.
7. Suscripciones recurrentes (cantidad y monto a definir en la migración) que generan su gasto en su fecha.
8. Fondo familiar y alcancía de emergencia: saldos y meta (saldos y meta a definir en la migración).
9. Informes con el formato siguiente: página 1 resumen por categorías (necesarios vs innecesarios), luego **detalle completo sin recortar** (fecha, monto, estado, cuenta, categoría).
10. Tasas de cambio con fecha y fuente: cada movimiento en Bs, EUR o cripto guarda la tasa usada, no se recalcula después.
11. Migración completa desde el sistema anterior, con informe de conciliación (ver §8).

## 3. Fuera de v1 (propuesto, no se construye ahora)

- Multi-tenant para terceros (esta app es de tu hogar, no un SaaS).
- Inversiones, trading y cálculo de impuestos.
- Recibos con subida de archivos: en v1 solo se guarda el **enlace** al recibo; la subida de adjuntos queda para v1.1.
- App móvil nativa: v1 es API + panel web.
- Importación bancaria automática: registro manual + importación por archivo en una fase posterior.

## 4. Reglas de negocio heredadas (reglas del proyecto)

- `aporte de otro miembro del hogar` es **ingreso del hogar, nunca gasto ni deuda**. Los aportes son ingresos, nunca deuda.
- Una transferencia entre cuentas propias, convertida a otra moneda, **no es gasto** (doble conteo). Solo cuenta el pago final al servicio.
- Un cargo queda `por_confirmar` hasta su verificación. Los informes distinguen confirmado de pendiente, nunca los mezclan en silencio.
- El salario mensual entra como **saldo inicial del mes siguiente**, con la tasa guardada en el momento, no recalculada.
- Dinero sin decimales flotantes: montos en unidades menores enteras (centavos), nunca `float`. Ver `er.md`.
- Una transacción nunca se borra en duro: se anula (`voided`) con motivo, y queda trazabilidad.
- Deduplicación: mismo origen, descripción, monto, fecha y cuenta no se registra dos veces. La API rechaza el duplicado y devuelve el registro existente.

## 5. Usuarios y roles [SUPUESTO A CONFIRMAR]

Un solo hogar (`household`), no multi-tenant:

- `admin`: usuario administrador. Todo: cuentas, categorías, presupuestos, migración, anulaciones.
- `registro`: segundo usuario del hogar. Registrar y ver movimientos, sin tocar configuración, presupuestos ni migración.

Si la v1 es de un solo usuario, se deja un solo usuario y el rol `registro` se activa después sin cambiar el modelo.

## 6. Stack y producción [DECIDIDO stack, PENDIENTE producción]

- **Stack decidido: Python + FastAPI + PostgreSQL.** Validación con Pydantic v2, modelos con SQLAlchemy, migraciones con Alembic, tests con pytest, despliegue con Docker. Motivo: Decimal nativo y validación estricta de dinero, velocidad de desarrollo para una app de un hogar, y análisis de patrones de gasto en el mismo lenguaje.
- **Producción propuesta: VPS propio, con dominio a definir.** No se crea nada hasta confirmarlo.
- Idioma: español en toda la interfaz y los mensajes de la API. Zona horaria: America/Caracas.
- Autenticación: OAuth2 con JWT (access token corto + refresh token), contraseñas con Argon2/bcrypt. Sin registro público: los usuarios los crea el admin.

## 7. Flujos clave

1. **Registrar gasto**: fecha, cuenta origen, categoría, monto + moneda, tasa si aplica, estado inicial, descripción, enlace de recibo opcional, reparto opcional. Si el hash de deduplicación ya existe, se devuelve el existente.
2. **Confirmar un pendiente**: `por_confirmar` → `confirmado`, con fecha de confirmación. Un `deuda_pendiente` se mueve al mes indicado, no se reescribe su mes original en silencio.
3. **Reparto compartido**: una transacción tiene 1..n repartos (`splits`) que suman el 100% (FUND_MAIN / FUND_CONTRIBUTOR u otros fondos definidos en la configuración).
4. **Cierre de mes**: el informe mensual se genera desde transacciones, no se edita a mano. El saldo inicial del mes siguiente es una transacción de ingreso marcada como tal.
5. **Suscripción**: plantilla con día de cobro; al llegar la fecha genera la transacción en `por_confirmar` (nunca se da por pagada sola) y queda pendiente de confirmación.
6. **Transferencia entre cuentas propias**: un solo movimiento con cuenta origen y destino, no dos registros sueltos. No cuenta como ingreso ni como gasto.

## 8. Migración y corte (condición para apagar sistema anterior y Excel)

1. Exportar sistema anterior por API (cuentas, categorías, transacciones) y exportar el hoja de cálculo (pestañas por mes).
2. Normalizar: cuentas sistema anterior → `accounts`; categorías sistema anterior + categorías del Excel → `categories` (sin duplicar nombres); cada fila → `transaction` con su estado original.
3. Deduplicar entre ambas fuentes por descripción + monto + fecha + cuenta.
4. Importación en seco (`dry-run`) con informe: totales por mes y por categoría en origen contra destino. **El corte solo se autoriza si los totales cuadran en la fecha de corte**; no se fija una cifra vieja como verdad.
5. Congelar sistema anterior y hoja de cálculo en solo lectura, importar en firme, verificar el informe mensual (resumen + detalle) contra la última hoja de cálculo.
6. Solo entonces se apagan sistema anterior y hoja de cálculo como registro activo. Se conservan sus respaldos.

## 9. Seguridad, respaldo y pruebas (mínimos antes de cualquier deploy)

- Backups automáticos de la base de datos antes de cada deploy y antes de la migración.
- Ningún secreto en el código; tokens JWT solo por variables de entorno, con expiración y revocación de refresh tokens.
- Pruebas obligatorias: deduplicación, reparto que suma 100%, transferencia que no infla ingresos/gastos, cambio de estado sin reescribir historia, informe mensual contra datos conocidos.
- Rollback: cada release debe poder volver a la versión anterior sin perder transacciones.

## 10. Qué falta decidir para cerrar la spec

1. ~~Stack: decidido — Python + FastAPI + PostgreSQL.~~
2. Producción: ¿qué VPS y qué dominio?
3. Usuarios: ¿dos usuarios del hogar desde v1, o solo uno?
4. Recibos: ¿enlace en v1, o subida de archivos desde el inicio?
5. Fecha de corte objetivo: ¿qué mes quieres que sea el primer mes 100% en la app nueva?

Cuando se confirmen estos 4 puntos, se pasa a issues. El cron de trabajo, pruebas, deploy e informes es la fase siguiente y se define ahí, no antes.
