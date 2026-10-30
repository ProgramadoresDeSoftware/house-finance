# house-finance

API y base de datos para llevar las finanzas personales del hogar.

**Decisión: reemplaza el sistema anterior. No convive con ellos.**

## Documentos (fuente de verdad del diseño)

- [Spec v1](docs/spec.md): objetivos, alcance, reglas de negocio, migración y corte.
- [Diseño API](docs/api.md): recursos, endpoints, autenticación, idempotencia y deduplicación.
- [Modelo entidad-relación](docs/er.md): tablas, claves, índices y mapeo de migración.

## Estado

Spec v1 en revisión. Sin código, sin issues creadas, sin deploy.
Stack decidido: Python + FastAPI + PostgreSQL.
Los supuestos a confirmar están marcados en la spec: producción en un VPS propio,
roles admin/registro y recibos solo por enlace en v1.
