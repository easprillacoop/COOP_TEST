# Sistema de Captura de Informes IPACOOP — Valoración, stack y plan de desarrollo

Oficina de Planificación · documento de trabajo · elaborado el 9 de noviembre de 2026
Base del análisis: `referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html` (8.117 líneas, ~233 funciones, 9 vistas, 59 iteraciones de requisitos visibles en los comentarios «Task N»).

---

## 1. Resumen ejecutivo

- **La maqueta es una excelente especificación funcional, pero no es una base de código reutilizable.** Es un único HTML con estado global, datos en `localStorage` del navegador, sin usuarios ni servidor. Se reutilizan **las reglas, los catálogos, las plantillas de datos y los formatos de impresión**; el código se reescribe.
- **El valor real está en las fórmulas y reglas de negocio**, ya validadas durante ~59 rondas con los usuarios. El riesgo principal no es técnico: es que **hoy conviven tres definiciones distintas de «cumplimiento»** (ver §3.2) y hay que unificarlas antes de fijarlas en código.
- **Recomendación de stack:** TypeScript de extremo a extremo — Next.js + PostgreSQL, con un paquete `domain` de funciones puras donde viven las fórmulas, validado contra la propia maqueta como «oráculo» (pruebas de paridad).
- **Calendario:** el cuello de botella es estacional. El **Plan Anual 2027 debe poder capturarse en diciembre** y los primeros mensuales llegan en **febrero de 2027**. Por eso el plan entrega primero **Plan Anual (11-dic)**, luego **Informe Mensual + avance (22-ene)**, y el resto hasta el cierre del T1 (**20-mar-2027**).
- Esfuerzo orientativo: **~14 persona-mes** (equipo de ~3,5 FTE × 19 semanas). Estimación gruesa; se afina tras la semana 0.

---

## 2. Qué contiene la maqueta (inventario funcional)

| # | Módulo | Qué hace | Complejidad |
|---|---|---|---|
| 1 | **Plan Anual** | Matriz por actividad: necesidad/objetivo/meta, cantidades ENE–DIC, costos (viáticos, alquiler/alimentación, transporte), subformulario de datos específicos **por mes** según el tipo de actividad. Un plan por *dirección provincial + departamento + año*; cada programa se integra como «sección». | Alta |
| 2 | **Informe Mensual** | Filas actividad + cooperativa (catálogo de 1.381), detección automática *Programada Sí/No*, modal de impacto (mujeres/hombres/edades/capacitaciones/presupuesto ejecutado). **Un solo informe por provincia + mes + año**, con aportes de varios técnicos. | Alta |
| 3 | **Informe Técnico** | Informe de visita a cooperativa; número de control homologado `DAT-BDT-CYF-001-2026`; pendientes con estatus; **respuesta del coordinador** con su propio control `DAT-COOR-CYF-001-2026` y certificación de recomendaciones. | Media-alta |
| 4 | **Consolidado Mensual / Trimestral** | Por departamento + programa + mes: programadas / realizadas / no programadas, técnicas vs administrativas, % de cumplimiento; versión imprimible y versión de captura manual. | Media |
| 5 | **Repositorio** (5 pestañas) | Planes, mensuales, consolidados, trimestral, seguimiento técnico, con filtros y avance plan-vs-ejecutado por mes y trimestre. | Media-alta |
| 6 | **Dirección de Fomento** | Monitoreo de solo lectura acotado por departamento. | Baja |
| 7 | **Indicadores** | KPIs con semáforo, matriz programa×mes, alertas de atraso, gráficos, bitácora, respaldo JSON/CSV, hallazgos con IA (Gemini). | Media-alta |
| 8 | **Exportaciones** | Excel de 2 hojas (Planificado / Ejecutado) con ZIP escrito a mano; 6 documentos imprimibles con `window.open` + CSS. | Media |

Datos de referencia embebidos: 12 direcciones, 8 departamentos, 13 programas, 72 actividades con códigos (`04-xx` Promoción, `2xx–6xx` Asistencia Técnica, `Técnicas-01`, `Admva-02..06`), 1.381 cooperativas, 25 tipos, 41 cargos. Extraídos a `referencia/seeds/`.

---

## 3. Valoración

### 3.1 Fortalezas

1. **Reglas de negocio maduras y explícitas** (comentadas con su origen): clasificación de actividades por departamento/programa/serie de código, técnica vs administrativa, familias de plantilla de datos, tope del 100 %, separación de actividades no programadas.
2. **UX validada:** flujos de captura por secciones, auto-detección de «programada», números de control autogenerados, impresión fiel a los modelos oficiales.
3. **Modelo de aporte multi-técnico bien pensado:** cada mención de una actividad programada = 1 acción; los aportes de todos los técnicos se suman.
4. **Ya existe la vía de migración de datos:** el «Exportar respaldo» JSON puede ser el formato de importación al sistema real.

### 3.2 Hallazgos sobre las fórmulas (decisiones de negocio pendientes)

> Las fórmulas están en `avancePlan`, `consolFilas`, `ccCumpl`, `indicadoresCalcular`. Se listan tal como están implementadas.

| Fórmula | Implementación actual |
|---|---|
| Total de fila del plan | `Σ m1..m12` |
| Costo total de fila | `viáticos + alquiler/alimentación + transporte` |
| Acciones planificadas (ACC plan) | `total` de la fila, o `datosExtra.cantidad` si el total es 0 (salvo familia *organización*) |
| Acciones ejecutadas (ACC) | **1 por cada fila de mensual** que menciona una actividad programada (no usa el campo «cantidad») |
| **Avance / cumplimiento del plan** | `min(100, round(ACC_ejecutadas × 100 / ACC_planificadas))`; `null` si el plan no promete acciones; 0 % si no hay mensuales |
| % presupuesto | `min(100, round(Σ presupuestoEjec × 100 / Σ costoTotal))` |
| Total de asociados | `mujeres + hombres` (si no, `asociadosTotal` o `participantes`); alerta si la suma por edades ≠ total |
| *Programada* | `true` si la actividad está (por nombre sin acentos) en el plan de misma provincia + año + departamento; la bandera guardada prevalece |
| **Consolidado: cumplimiento por provincia** | `min(100, round(realizadas_técnicas × 100 / programadas_técnicas))`; *programadas* = actividades con cantidad > 0 en la columna del mes; *realizadas* = actividades **distintas** reportadas ese mes |
| **Consolidado (captura manual)** | `SI(O(B="",D=""),0,(D+E)/(B+C))` → `(realTéc+realAdm)/(progTéc+progAdm)`, y 0 si falta progTéc o realTéc |
| Cobertura de mensuales | `programas con ≥1 mensual del mes / programas con actividades planificadas` |
| Semáforo | `≥75 % verde · ≥40 % ámbar · <40 % rojo` |
| Alerta técnico sin respuesta | `> 15 días` desde la fecha de la actividad |

**Puntos que requieren decisión de la Oficina antes de codificar** (cada uno cambia números oficiales):

1. **Tres definiciones de «cumplimiento»:** (a) menciones/ACC planificadas, (b) actividades *distintas* técnicas, (c) fórmula (D+E)/(B+C) del consolidado manual. Un mismo programa-mes puede dar tres porcentajes. Definir **una** métrica canónica y nombrar las demás.
2. **Tope del 100 % solo en el agregado:** sobre-ejecutar una actividad compensa sub-ejecutar otra dentro del mismo programa. ¿Se desea tope por actividad?
3. **ACC ejecutada ignora «cantidad»:** un técnico que reporta «3 charlas» en una fila cuenta 1 acción, pero el plan sí promete 3. ¿Es intencional (Task 48)?
4. **Redondeo/null:** `Math.round` y `null` vs `0`. Fijar redondeo (half-up) y representación de «sin dato» en los reportes.
5. **Consolidados manuales vs calculados:** los de captura «no alimentan nada». ¿Se mantienen o el sistema genera el consolidado y se permite ajuste con justificación?
6. **Congelamiento (cierre mensual):** hoy todo se recalcula en vivo; un informe impreso puede cambiar después. Se recomienda **cierre de mes con instantánea** de cifras.

### 3.3 Debilidades técnicas y de seguridad (no deben trasladarse)

| Tema | Situación en la maqueta | Consecuencia |
|---|---|---|
| Almacenamiento | `localStorage` (≈5 MB, por navegador) | Sin repositorio central real; cada PC ve datos distintos; pérdida al limpiar el navegador |
| Identidad | **No hay autenticación ni roles.** Las vistas «exclusivas» son solo entradas de menú; el «usuario» de la bitácora es texto libre | Cualquiera ve/edita todo; auditoría falsificable |
| IA | Llave de Gemini **guardada en el navegador** y llamada directa desde el cliente; envía informes técnicos de cooperativas a un tercero | Fuga de llave; datos sensibles fuera de control institucional |
| Claves de datos | Se relaciona por **nombre sin acentos** (actividad, programa, cooperativa, departamento con 2 convenciones: `Promoción y Educación` / `DEPARTAMENTO_DE_PROMOCIÓN_Y_EDUCACIÓN`) | Un cambio de nombre rompe históricos; hubo que escribir `renProgLegado`, `dividirPlanesLegacy`, `fusionarPlanes`, `fusionarInformes` |
| Duplicados | Deduplicación por «huella» de texto de la fila | En BD se resuelve con restricciones únicas y filas con autor, no con heurísticas |
| Plantillas de datos | Se infieren con **regex sobre el nombre** de la actividad (`obtenerPlantilla`) | Frágil; debe ser una relación explícita actividad→plantilla en catálogo |
| Control de números | Secuencia 001–500 elegida a mano | Colisiones con usuarios concurrentes; tope artificial |
| Impresión/Excel | `window.open + document.write`; ZIP/XLSX escrito a mano | Salida dependiente del navegador; mantenimiento costoso |
| Calidad de catálogo | `Panampa Sede Central`, `Dirección de Fomento de Cooperativo`, cooperativas sin identificador (nombre como clave, mayúsculas inconsistentes, 427 «Liquidada» + 276 «Inactiva» de 1.381) | Requiere depuración y un identificador estable (n.º de registro) |
| Reglas cambiantes | Los programas por departamento cambiaron en Task 54/55/56 | Catálogos deben ser **editables por Planificación y versionados por año**, no constantes en código |

### 3.4 Qué se reutiliza y qué no

- **Reutilizar (portar):** catálogos y códigos (`referencia/seeds/`), plantillas de datos (`PLANTILLAS`), funciones de cálculo (como especificación + oráculo de pruebas), CSS y estructura de los documentos imprimibles.
- **No reutilizar:** estado global, render por `innerHTML`, almacenamiento, deduplicación heurística, generador XLSX, llamada a IA desde el cliente.
- **Nota:** los comentarios mencionan `lib/actividad-forms.ts` y `plan-anual.tsx` y se usa `use-toast`/iconos Lucide, lo que sugiere una app React/Next ya existente de la que la maqueta es «espejo». **Confirmar si ese repositorio existe**; si es así, se evalúa reutilizarlo como punto de partida.

---

## 4. Stack tecnológico recomendado

Criterios: equipo pequeño, un solo lenguaje, lógica de negocio pesada y compartida entre servidor y pantalla, datos tabulares con agregaciones, documentos oficiales imprimibles, sostenibilidad en una institución pública.

| Capa | Elección | Justificación |
|---|---|---|
| Lenguaje | **TypeScript** en todo | Un solo lenguaje; las fórmulas actuales son JS y se portan casi directo |
| Frontend | **Next.js (React, App Router) + Tailwind + shadcn/ui** | Coincide con las pistas de la app original; ecosistema amplio, accesible |
| Tablas/formularios | **TanStack Table + React Hook Form + Zod** | La matriz del plan (12 meses × costos) y los subformularios dinámicos por plantilla se validan con el mismo esquema Zod que usa el servidor |
| Backend | **Next.js (Route Handlers/Server Actions)** + paquete `domain` | Menos piezas móviles para un equipo pequeño; separable luego |
| Dominio | **`packages/domain`** (TS puro, sin I/O) | Aquí viven `avancePlan`, `consolidado`, `semáforos`, `clasificación de actividades`; 100 % probado |
| Base de datos | **PostgreSQL 16** | Integridad (únicos parciales, FK), `JSONB` para datos específicos, extensión `unaccent`/`pg_trgm` (reemplaza `quitarAcentos`), secuencias atómicas, vistas, auditoría por *triggers* |
| Acceso a datos | **Drizzle ORM + migraciones SQL** | SQL-first, tipado, buen soporte de índices parciales/JSONB |
| Autenticación | **OIDC** (Keycloak autoalojado, o el directorio institucional si es Microsoft 365/Google) con Auth.js | Roles: Técnico, Coordinador/Jefe, Director Provincial, Planificación (admin), Fomento (lectura), Auditor; alcance por provincia/departamento/programa aplicado **en servidor** |
| Documentos PDF | **Chromium headless (Playwright)** renderizando las mismas plantillas HTML/CSS de impresión | Fidelidad inmediata con los modelos oficiales; evita rediseñarlos |
| Excel | **exceljs** | Reemplaza el ZIP manual |
| Trabajos asíncronos | **pg-boss** (cola sobre PostgreSQL) | PDF masivos, exportaciones, IA; sin Redis |
| Gráficos | **Recharts** (o ECharts) | Los gráficos de la maqueta son barras/matriz |
| IA (opcional, fase final) | **Pasarela de IA en servidor** (proveedor intercambiable), llave en secretos, redacción de datos, resultado como *sugerencia* revisable | Resuelve la fuga de llave y habilita decidir proveedor tras revisión de gobernanza de datos |
| Pruebas | **Vitest** (dominio), **Playwright** (E2E), **pruebas de paridad contra la maqueta** | Ver §4.2 |
| CI/CD | **GitHub Actions** → contenedores Docker | Entornos dev / staging / prod |
| Infra | **Docker Compose** en VM institucional o nube aprobada; PostgreSQL con respaldo diario + PITR; proxy TLS (Caddy/Nginx) | Sujeto a política de alojamiento (§7) |
| Observabilidad | Logs estructurados (pino) + Sentry (o equivalente aprobado) | Detectar fallos en provincias |
| Monorepo | **pnpm + Turborepo**: `apps/web`, `packages/domain`, `packages/db`, `packages/ui` | |

### 4.1 Modelo de datos (esquema inicial)

- **Catálogos (versionados por año):** `provincia`, `direccion`, `departamento`, `programa`, `departamento_programa(año)`, `actividad`, `actividad_codigo(depto, programa)`, `plantilla_datos(versión, esquema JSON)`, `actividad_plantilla`, `cooperativa(id, n.º registro, nombre, siglas, tipo, provincia, estado)`.
- **Seguridad:** `usuario`, `rol`, `usuario_alcance(provincia, departamento, programa)`, `audit_log` (solo-inserción, por *triggers*, con usuario real).
- **Plan:** `plan_anual(direccion, departamento, año, estado)` **único** por esos tres campos → `plan_seccion(programa)` → `plan_actividad(actividad, necesidad, objetivo, meta, costo_viaticos, costo_alquiler, costo_transporte)` → `plan_actividad_mes(mes, cantidad, datos JSONB)`. Los totales son columnas generadas/vistas, no datos duplicados.
- **Mensual:** `informe_mensual(direccion, año, mes)` **único** → `informe_aporte(autor, departamento, programa, fecha, logros, dificultades)` → `informe_fila(actividad, cooperativa_id, programada, resumen, estado, mujeres, hombres, jovenes, adultos, mayores, coop_atendidas, capacitaciones, presupuesto_ejec, datos JSONB)`. Las cifras que entran en fórmulas son **columnas reales**, no JSON.
- **Técnico:** `informe_tecnico`, `tecnico_pendiente(estatus)`, `respuesta_coordinador`, `secuencia_control(depto, provincia, programa, año)` con generación atómica y restricción única.
- **Cierres:** `cierre_mensual` / `consolidado_snapshot` (cifras congeladas, quién y cuándo cerró).
- **IA:** `hallazgo(origen, severidad, estado)` ligados al informe técnico.

### 4.2 Estrategia de paridad con la maqueta

Como la maqueta contiene «las fórmulas reales», se usa como **oráculo ejecutable**:
1. Extraer `avancePlan`, `consolFilas`, `ccCumpl`, `indicadoresCalcular` en un arnés Node (con *stubs* mínimos de DOM).
2. Generar ~30 escenarios (JSON) con casos límite: plan sin acciones, sobre-ejecución, actividad no programada, dos técnicos, filas sin sección, departamento con convención antigua.
3. Guardar entradas/salidas como **pruebas doradas**; el paquete `domain` en TS debe reproducirlas bit a bit — salvo donde la Oficina decida cambiar la regla (§3.2), y entonces la diferencia queda documentada y aprobada.

---

## 5. Plan de desarrollo (inicio: lunes 9 de noviembre de 2026)

**Supuestos:** equipo de 1 líder técnico, 2 desarrolladores full-stack, 0,5 QA/analista y un dueño de producto de la Oficina de Planificación con disponibilidad semanal. Sprints de 2 semanas. Feriados (verificar calendario oficial): martes 10-nov, martes 8-dic, 25-dic, 1-ene; capacidad reducida en el sprint de fin de año.

### 5.1 Semana 0 — arranque (9–13 nov)

| Día | Entregable |
|---|---|
| Lun 9 | Kickoff con Planificación; confirmar las decisiones de §7; crear monorepo, CI y ADR de stack |
| Mar 10 | *Feriado* |
| Mié 11 | Arnés oráculo (§4.2) + semillas de catálogos; ER v1 |
| Jue 12 | Entornos dev/staging; migración 0001 de catálogos; decidir proveedor de identidad |
| Vie 13 | **Demo del esqueleto** (login + lectura de catálogos); backlog de S1 refinado; sesión agendada para reconciliar fórmulas |

### 5.2 Hitos

| Sprint | Fechas | Alcance | Hito de salida |
|---|---|---|---|
| **S1** | 16–27 nov | Autenticación y roles con alcance; administración de catálogos (años, programas por departamento); importador de semillas; **Plan Anual: captura de matriz y costos**, guardado, integración por sección | Planificación carga un plan completo |
| **S2** | 30 nov–11 dic | Plan Anual: subformularios por mes y por familia de plantilla, validaciones, repositorio de planes con filtros, **PDF del plan**, bitácora real; piloto con 2 provincias | **R1 (11-dic): Plan Anual 2027 en producción** |
| **S3** | 14 dic–8 ene | **Informe Mensual:** filas, catálogo de cooperativas con búsqueda, *programada* automática, modal de impacto, aporte multi-técnico; paquete `domain` con avance y pruebas de paridad | Mensual capturable |
| **S4** | 11–22 ene | Repositorio de mensuales; **avance plan-vs-ejecutado** por mes/trimestre; PDF del mensual; capacitación a técnicos | **R2 (22-ene): Plan + Mensual** antes de los primeros reportes de febrero |
| **S5** | 25 ene–5 feb | **Informe Técnico:** control homologado atómico, pendientes, respuesta del coordinador y su control, seguimiento | Flujo técnico → coordinador completo |
| **S6** | 8–19 feb | Consolidados mensual y trimestral (calculados + cierre/instantánea); Excel Planificado/Ejecutado; vista de Fomento | **R3:** consolidados antes del cierre del T1 |
| **S7** | 22 feb–5 mar | **Indicadores:** KPIs/semáforos, matriz programa×mes, alertas, gráficos, bitácora consultable, exportes CSV | Panel de Planificación |
| **S8** | 8–19 mar | Pasarela de IA (si se aprueba), importador del respaldo JSON, endurecimiento (pruebas de seguridad, carga, respaldos/restauración), manuales y capacitación final | **Go-live completo: 20-mar-2027** |

### 5.3 Definición de «terminado» por entrega
Pruebas de dominio dorado en verde · E2E del flujo principal · roles verificados en servidor (prueba de acceso cruzado entre provincias) · bitácora con usuario real · PDF idéntico al modelo oficial · respaldo restaurado en staging.

---

## 6. Riesgos principales

| Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|
| Fórmulas sin definición única (§3.2) | Alta | Alto | Sesión de reconciliación en S1; matriz de decisiones firmada por el dueño de producto |
| Cambios de reglas continuos (59 iteraciones) | Alta | Medio | Catálogos administrables y versionados; reglas en `domain` con pruebas |
| Conectividad en provincias (Darién, comarcas) | Media | Alto | Autoguardado de borradores local (IndexedDB), reintentos, formularios ligeros |
| Hosting / identidad no definidos | Media | Alto | Decidir en semana 0 (§7); diseño agnóstico (contenedores + OIDC) |
| Datos sensibles en IA externa | Media | Alto | IA al final y opcional; redacción; aprobación formal previa |
| Calidad de catálogo de cooperativas | Alta | Medio | Depuración con Registro Cooperativo; identificador estable |
| Fecha de captura del Plan 2027 | Media | Alto | R1 priorizado el 11-dic; plan B: capturar en la maqueta y migrar con el JSON |
| Adopción por técnicos | Media | Medio | Piloto en S2, capacitación en S4 y S8, soporte en la primera semana de cada mes |

---

## 7. Decisiones que necesito de usted (semana 0)

1. ¿Existe el repositorio de la app web original (Next/React) del que la maqueta es «espejo»? ¿Se reutiliza?
2. **Equipo y presupuesto reales** (el plan supone ~3,5 FTE) y fecha límite firme para el Plan Anual 2027.
3. **Alojamiento:** ¿servidor institucional, nube aprobada por la AIG u otra? ¿Qué directorio de usuarios usa IPACOOP (Microsoft 365, Google, Active Directory)?
4. **Fórmulas:** fecha para la sesión de reconciliación de §3.2 (6 puntos).
5. ¿Los consolidados de captura manual se mantienen?
6. ¿Se autoriza enviar texto de informes técnicos a un proveedor de IA externo? (Tratamiento de datos personales según la Ley 81 de 2019; validar con Asesoría Legal.)
7. ¿Hay datos reales ya capturados en navegadores de las provincias que deban migrarse?

---

## 8. Archivos de referencia en este repositorio

- `referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html` — maqueta original (oráculo de fórmulas).
- `referencia/seeds/catalogos.json` — direcciones, departamentos, programas, tipos, estados, cargos, 1.381 cooperativas.
- `referencia/seeds/codigos-actividad.json` — 72 actividades con sus códigos.
