# Sistema de Captura de Informes — IPACOOP

Oficina de Planificación · Instituto Panameño Autónomo Cooperativo

Sistema web interno para que cada dirección provincial capture su **Plan Anual** de forma autónoma y se lleve de forma centralizada la **ejecución** (informes mensuales y técnicos), los **consolidados** y los **indicadores**.

## Documentación

| Documento | Contenido |
|---|---|
| [`docs/valoracion-y-plan-de-desarrollo.md`](docs/valoracion-y-plan-de-desarrollo.md) | Valoración de la maqueta, stack tecnológico, arquitectura en 2 servidores virtuales, plan de entregas (R1 1-dic-2026 … R4 30-dic-2026) y riesgos |
| [`docs/especificacion-de-requisitos.md`](docs/especificacion-de-requisitos.md) | Perfiles y permisos, requisitos funcionales por módulo, reglas de negocio y fórmulas, datos, requisitos no funcionales, supuestos y preguntas abiertas |

## Referencia

| Ruta | Contenido |
|---|---|
| `referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html` | Maqueta funcional original (fuente de reglas y fórmulas; se usa como oráculo de pruebas) |
| `referencia/seeds/catalogos.json` | Direcciones, departamentos, programas, tipos, estados, cargos y 1.381 cooperativas |
| `referencia/seeds/codigos-actividad.json` | 72 actividades con sus códigos |

## Stack previsto

Django 5.2 LTS (Python) con HTMX · Django REST Framework (web service) · PostgreSQL 16 · Nginx + Gunicorn · Linux — todo software libre, en una sola aplicación.
