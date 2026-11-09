# Sistema de Captura de Informes IPACOOP — Valoración, stack y plan de desarrollo (v2)

Oficina de Planificación · documento de trabajo · v2 del 9 de noviembre de 2026
Base del análisis: `referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html` (8.117 líneas, ~233 funciones, 9 vistas, ~59 iteraciones de requisitos visibles en los comentarios «Task N»).

**Cambios respecto a la v1** (según las respuestas de la Oficina):
- No existe aplicación previa: se construye desde cero (web + web service + base de datos).
- Presupuesto bajo → solo software libre, sin licencias; desarrollo con Claude Code.
- Hospedaje **interno**: dos servidores virtuales (aplicación y base de datos). Se elimina todo lo que dependa de nube o servicios externos.
- **Las fórmulas de la maqueta son definitivas**: se implementan tal cual; ya no hay sesión de reconciliación.
- **Sin IA externa:** el módulo de «hallazgos con Gemini» queda fuera del alcance.
- **Sin migración de datos:** el sistema arranca vacío; se elimina el importador de respaldos.
- Fechas duras: **Plan Anual disponible el martes 1 de diciembre de 2026**; resto del sistema **al cierre de diciembre (jueves 31)**.

---

## 1. Resumen ejecutivo

- La maqueta es una **especificación funcional muy completa** (reglas, fórmulas, catálogos, formatos de impresión), pero no es una base de código reutilizable: un solo HTML, datos en el navegador, sin usuarios ni servidor. Se reconstruye siguiendo la maqueta como especificación y como **oráculo de pruebas**.
- **Arquitectura propuesta:** aplicación de una página (React) + **web service REST** (Node.js/Fastify) + **PostgreSQL**, en dos máquinas virtuales Linux. Todo software libre; costo de licencias = 0.
- **Seguridad:** cuentas propias con contraseñas protegidas (argon2id), sesiones seguras, **perfiles (roles) con alcance por provincia / departamento / programa aplicado en el servidor**, bitácora de auditoría que no se puede alterar desde la aplicación, TLS interno.
- **Calendario en 4 entregas:** R1 Plan Anual (1-dic) → R2 Informe Mensual y avance (11-dic) → R3 Informe Técnico y consolidados (22-dic) → R4 Indicadores, Fomento y cierre (30-dic, con 31-dic de colchón).
- **El calendario es posible pero exigente.** Claude Code acelera la programación, no el aprovisionamiento de servidores, la revisión humana ni las pruebas de aceptación. El mayor riesgo para el 1-dic es que las máquinas virtuales y accesos no estén listos a tiempo (ver §6).

---

## 2. Qué contiene la maqueta

| # | Módulo | Qué hace | Estado en el alcance |
|---|---|---|---|
| 1 | **Plan Anual** | Matriz por actividad: necesidad/objetivo/meta, cantidades ENE–DIC, costos (viáticos, alquiler/alimentación, transporte), subformulario de datos específicos **por mes** según el tipo de actividad. Un plan por *dirección provincial + departamento + año*; cada programa se integra como «sección». | **R1** |
| 2 | **Informe Mensual** | Filas actividad + cooperativa (catálogo de 1.381), detección automática *Programada Sí/No*, modal de impacto (mujeres, hombres, edades, capacitaciones, presupuesto ejecutado). **Un solo informe por provincia + mes + año**, con aportes de varios técnicos. | **R2** |
| 3 | **Repositorio** | Planes, mensuales y seguimiento técnico con filtros; avance plan-vs-ejecutado por mes y trimestre. | R1 (planes) · R2 (mensuales/avance) |
| 4 | **Informe Técnico** | Visita a cooperativa; control homologado `DAT-BDT-CYF-001-2026`; pendientes con estatus; **respuesta del coordinador** con su control `DAT-COOR-CYF-001-2026`. | **R3** |
| 5 | **Consolidado Mensual / Trimestral** | Por departamento + programa + mes: programadas / realizadas / no programadas (técnicas y administrativas) y % de cumplimiento. | **R3** (calculados por el sistema) |
| 6 | **Dirección de Fomento** | Monitoreo de solo lectura por departamento. | R4 |
| 7 | **Indicadores** | KPIs con semáforo, matriz programa×mes, alertas de atraso, gráficos, bitácora, exportación CSV. | R4 |
| 8 | **Exportaciones** | Excel de 2 hojas (Planificado / Ejecutado) y documentos imprimibles. | R3–R4 |

**Fuera del alcance (decidido por la Oficina):** análisis con IA externa (Gemini), importación/restauración de respaldos JSON, migración de datos.
**Supuesto a confirmar:** los formularios de *captura manual* de consolidados (mensual/trimestral) **no se construyen**; el propósito es que cada dirección provincial capture su plan de forma autónoma y la ejecución se centralice en el sistema, de donde se calculan los consolidados e indicadores.

Datos de referencia (extraídos a `referencia/seeds/`): 12 direcciones, 8 departamentos, 13 programas, 72 actividades con códigos (`04-xx` Promoción, `2xx–6xx` Asistencia Técnica, `Técnicas-01`, `Admva-02..06`), 1.381 cooperativas, 25 tipos, 41 cargos.

---

## 3. Valoración

### 3.1 Fortalezas
1. Reglas de negocio maduras y documentadas en los comentarios (clasificación por departamento/programa/serie de código, técnica vs administrativa, tope del 100 %, separación de actividades no programadas).
2. UX validada con los usuarios en ~59 iteraciones; flujos de captura por secciones; números de control autogenerados; impresión fiel a los modelos oficiales.
3. Modelo de aporte multi-técnico claro: cada mención de una actividad programada = 1 acción; los aportes de todos los técnicos se suman.

### 3.2 Fórmulas vigentes (se implementan tal cual)

Origen: `avancePlan`, `consolFilas`, `ccCumpl`, `indicadoresCalcular`.

| Fórmula | Implementación |
|---|---|
| Total de fila del plan | `Σ m1..m12` |
| Costo total de fila | `viáticos + alquiler/alimentación + transporte` |
| Acciones planificadas (ACC plan) | `total` de la fila; si es 0, `datosExtra.cantidad` (excepto familia *organización*) |
| Acciones ejecutadas (ACC) | 1 por cada fila de mensual que menciona una actividad programada |
| **Avance / cumplimiento del plan** | `min(100, round(ACC_ejecutadas × 100 / ACC_planificadas))`; sin acciones planificadas = «—»; sin mensuales = 0 % |
| % de presupuesto | `min(100, round(Σ presupuestoEjec × 100 / Σ costoTotal))` |
| Total de asociados | `mujeres + hombres` (si no, `asociadosTotal` o `participantes`); advertencia si la suma por edades ≠ total |
| *Programada* | `true` si la actividad está en el plan de la misma provincia + año + departamento (comparación sin acentos); la bandera guardada prevalece |
| **Cumplimiento del consolidado por provincia** | `min(100, round(realizadas_técnicas × 100 / programadas_técnicas))`; *programadas* = actividades con cantidad > 0 en la columna del mes; *realizadas* = actividades **distintas** reportadas ese mes |
| Cobertura de mensuales | `programas con ≥1 mensual del mes / programas con actividades planificadas` |
| Semáforo | `≥ 75 % verde · ≥ 40 % ámbar · < 40 % rojo` |
| Alerta de informe técnico sin respuesta | `> 15 días` desde la fecha de la actividad |

**Regla de implementación:** las fórmulas viven en un único paquete (`packages/domain`) y se verifican con **pruebas de paridad contra la maqueta** (§4.2). Como la Oficina las considera definitivas, cualquier diferencia entre el sistema y la maqueta es un defecto, no una decisión.

*Nota informativa (no bloquea):* en la interfaz se rotulará cada porcentaje con su nombre exacto («Avance del plan por acciones», «Cumplimiento técnico del consolidado»), porque son métricas distintas y un mismo programa-mes puede mostrar valores diferentes en cada una.

### 3.3 Debilidades de la maqueta que NO se trasladan
| Tema | En la maqueta | En el sistema real |
|---|---|---|
| Almacenamiento | `localStorage` por navegador | PostgreSQL central |
| Identidad | Sin autenticación; «exclusivo» = solo menú; usuario de bitácora = texto libre | Cuentas, roles y alcance verificados en servidor; bitácora con usuario real |
| Claves de datos | Se enlaza por **nombre sin acentos** (actividad, programa, departamento con 2 convenciones de nombre) | Identificadores estables (llaves foráneas) |
| Duplicados | Deduplicación heurística por «huella» de texto (`fusionarInformes`, `fusionarPlanes`) | Restricciones únicas + filas con autor |
| Plantillas de datos | Inferidas con regex sobre el nombre de la actividad | Relación explícita actividad → plantilla en catálogo |
| Control de números | Secuencia 001–500 elegida a mano | Secuencia atómica por (departamento, provincia, programa, año) |
| Impresión/Excel | `window.open` + ZIP escrito a mano | Impresión del navegador con la misma CSS + `exceljs` |
| Catálogos | Constantes en el código; programas por departamento cambiaron tres veces | Tablas administrables por Planificación, versionadas por año |
| Calidad de datos | `Panampa Sede Central`, `Dirección de Fomento de Cooperativo`; cooperativas sin identificador (427 liquidadas, 276 inactivas de 1.381) | Depuración antes de cargar las semillas; identificador interno |

---

## 4. Stack tecnológico recomendado

**Criterios:** costo de licencias cero, dos servidores virtuales, uso interno, un solo lenguaje, lógica de negocio pesada, y desarrollo asistido por Claude Code (que rinde mejor con TypeScript tipado, pruebas automáticas y convenciones claras).

| Capa | Elección | Justificación |
|---|---|---|
| Lenguaje | **TypeScript** en todo | Una sola base de habilidades; las fórmulas actuales son JS y se portan casi directo; el tipado reduce errores del código generado |
| Frontend | **React + Vite** (aplicación de una página) + Tailwind + componentes shadcn/ui | Se publica como archivos estáticos servidos por Nginx: sin proceso de servidor extra |
| Tablas y formularios | **TanStack Table / Query**, **React Hook Form + Zod** | La matriz del plan (12 meses × costos) y los subformularios dinámicos validan con el mismo esquema que el servidor |
| **Web service** | **Node.js 22 LTS + Fastify**, REST/JSON con documentación **OpenAPI** | Es el «web service» solicitado; permite integraciones futuras |
| Dominio | **`packages/domain`** (TS puro, sin I/O) | Fórmulas, clasificación de actividades, semáforos; 100 % probado |
| Base de datos | **PostgreSQL 16** (servidor propio) | Gratuito; integridad (únicos parciales, llaves foráneas), `JSONB` para datos específicos, `unaccent` (reemplaza `quitarAcentos`), secuencias atómicas, auditoría por *triggers* |
| Acceso a datos | **Drizzle ORM** + migraciones SQL versionadas | SQL-first, tipado |
| Autenticación | **Cuentas propias**: argon2id, sesión en cookie `HttpOnly + Secure + SameSite`, sesiones en PostgreSQL, bloqueo por intentos, cambio obligatorio de contraseña inicial, protección CSRF, límite de peticiones. **Adaptador LDAP/Active Directory opcional** si IPACOOP lo tiene | Sin servicio adicional (se evita Keycloak); MFA TOTP como mejora posterior |
| Autorización | **Perfiles + alcance** (provincia, departamento, programa) aplicados en el servidor en cada consulta; pruebas automáticas de acceso cruzado | Ver §4.3 |
| Auditoría | Tabla `audit_log` solo-inserción alimentada por *triggers*; el usuario de la aplicación no puede modificarla | Reemplaza la bitácora de 500 registros de la maqueta |
| Reportes | Impresión/PDF del navegador con la CSS de impresión de la maqueta; **exceljs** en el servidor para Excel | Evita instalar un navegador sin cabeza en el servidor |
| Gráficos | **Recharts** | Los gráficos de la maqueta son barras y matriz |
| Pruebas | **Vitest** (dominio y API), **Playwright** (flujos principales), **pruebas de paridad contra la maqueta** | §4.2 |
| Servidor web | **Nginx** (TLS con certificado de la CA interna, compresión, cabeceras de seguridad) | |
| Ejecución | **systemd** para el servicio Node (Docker opcional si la granja ya lo usa) | Mínimo mantenimiento |
| Código y CI | Repositorio Git + **GitHub Actions** (o el CI interno) que genera un paquete desplegable | Despliegue con un script; se puede hacer sin internet en el servidor |
| Respaldos | `pg_dump` nocturno + archivado de WAL a otro servidor/almacenamiento; prueba de restauración antes del go-live | |

**Sin nube y sin servicios externos en tiempo de ejecución.**

### 4.1 Topología en la granja de servidores

| Servidor | Contenido | Dimensionamiento estimado* |
|---|---|---|
| **VM-App** | Nginx (archivos estáticos + proxy), servicio Node (API) | 2–4 vCPU · 4–8 GB RAM · 40 GB |
| **VM-BD** | PostgreSQL 16, tareas de respaldo | 2–4 vCPU · 8 GB RAM · 100 GB |

\*Estimación mía; el volumen es pequeño (decenas de miles de filas por año). Solo la VM-BD es accesible desde la VM-App (puerto 5432); los usuarios acceden únicamente a la VM-App por HTTPS. Un entorno de **pruebas/aceptación** puede ser un segundo sitio de Nginx + una segunda base de datos en las mismas VM.

### 4.2 Estrategia de paridad con la maqueta
Como la maqueta contiene las fórmulas definitivas, se usa como **oráculo ejecutable**:
1. Extraer `avancePlan`, `consolFilas`, `ccCumpl`, `indicadoresCalcular` a un arnés Node (con *stubs* mínimos de DOM).
2. Generar ~30 escenarios (JSON) con casos límite: plan sin acciones, sobre-ejecución, actividad no programada, dos técnicos, filas sin sección, departamento con convención antigua.
3. Guardar entradas y salidas como **pruebas doradas**; el paquete `domain` debe reproducirlas exactamente.

### 4.3 Perfiles propuestos (a validar con Planificación)

| Perfil | Alcance | Puede |
|---|---|---|
| **Administrador del sistema (TI)** | Todo | Usuarios, roles, bloqueos, parámetros; no edita contenido de negocio |
| **Oficina de Planificación** | Nacional | Administra catálogos; ve todo; indicadores; bitácora; reabre planes; marca recepción |
| **Director/a Provincial** | Su provincia | Captura y envía el plan anual de su provincia; ve mensuales y técnicos de su provincia |
| **Técnico/a** | Su provincia + departamento + programa | Captura informes mensuales y técnicos propios |
| **Coordinador/a de departamento** | Su departamento, nacional | Monitorea su departamento; responde informes técnicos; indicadores de su departamento |
| **Dirección de Fomento** | Lectura, por departamento | Monitoreo de solo lectura |
| **Auditor/a** | Lectura | Consulta la bitácora |

**Estados del plan (mínimo para la «recepción»):** *Borrador → Enviado/Recibido* (con fecha y usuario); Planificación puede reabrir. Es una adición pequeña a la maqueta que da sentido a «recepción del plan anual»; confirmar.

### 4.4 Modelo de datos (esquema inicial)
- **Catálogos (versionados por año):** `provincia`, `direccion`, `departamento`, `programa`, `departamento_programa(año)`, `actividad`, `actividad_codigo(depto, programa)`, `plantilla_datos`, `actividad_plantilla`, `cooperativa`.
- **Seguridad:** `usuario`, `rol`, `usuario_alcance(provincia, departamento, programa)`, `sesion`, `audit_log`.
- **Plan:** `plan_anual(direccion, departamento, año, estado)` **único** → `plan_seccion(programa)` → `plan_actividad(actividad, necesidad, objetivo, meta, costo_viaticos, costo_alquiler, costo_transporte)` → `plan_actividad_mes(mes, cantidad, datos JSONB)`.
- **Mensual:** `informe_mensual(direccion, año, mes)` **único** → `informe_aporte(autor, departamento, programa, logros, dificultades)` → `informe_fila(actividad, cooperativa, programada, resumen, estado, mujeres, hombres, jovenes, adultos, mayores, coop_atendidas, capacitaciones, presupuesto_ejec, datos JSONB)`. Las cifras que entran en fórmulas son **columnas reales**.
- **Técnico:** `informe_tecnico`, `tecnico_pendiente(estatus)`, `respuesta_coordinador`, `secuencia_control` (atómica, única).

---

## 5. Plan de desarrollo (inicio: lunes 9 de noviembre de 2026)

### 5.1 Forma de trabajo con Claude Code
- **Desarrollo guiado por especificación:** este documento y la maqueta son la fuente; se crea un `CLAUDE.md` con convenciones (estructura del monorepo, reglas de seguridad, «las fórmulas no se tocan»).
- **Rebanadas verticales** (base de datos → API → pantalla → prueba) con pruebas automáticas en cada una; sesiones paralelas en ramas separadas para módulos independientes.
- **Roles humanos mínimos:** (1) *responsable técnico* que revisa el código, administra los servidores y despliega; (2) *dueño de producto de Planificación* con tiempo diario para validar pantallas y reglas; (3) *administrador de infraestructura* que entrega las VM.
- **Congelamiento de alcance** una vez iniciada cada entrega; cambios nuevos entran a la siguiente.

### 5.2 Semana 0 — arranque (9–13 nov; martes 10 feriado)
| Día | Entregable |
|---|---|
| Lun 9 | Plan v2 acordado; **solicitud formal de las 2 VM** y accesos (fecha tope de entrega: **miércoles 18-nov**) |
| Mié 11 | Monorepo, `CLAUDE.md`, CI, migraciones base, semillas de catálogos (depuradas), arnés oráculo |
| Jue 12 | Autenticación + roles + alcance (esqueleto), `audit_log`, pantalla de ingreso |
| Vie 13 | Demo del esqueleto en entorno temporal; backlog de R1 listo |

### 5.3 Entregas
| Entrega | Fecha | Alcance | Criterio de salida |
|---|---|---|---|
| **R1 — Plan Anual** | **Mar 1-dic** | Usuarios, perfiles y alcance; administración de catálogos; **captura del Plan Anual** (matriz ENE–DIC, costos, catálogo de actividades filtrado por departamento/programa, códigos, subformularios por mes y plantilla, integración por sección); estados Borrador/Enviado; repositorio de planes con filtros; impresión; bitácora | Una dirección provincial captura y envía su plan; Planificación lo recibe y lo consulta |
| **R2 — Mensual y avance** | Vie 11-dic | Informe Mensual completo (cooperativas con búsqueda, *programada* automática, impacto, aporte multi-técnico); repositorio de mensuales; **avance plan vs ejecutado** por mes y trimestre | Pruebas de paridad del avance en verde |
| **R3 — Técnico y consolidados** | Mar 22-dic | Informe Técnico con control homologado atómico, pendientes y respuesta del coordinador; consolidados mensual y trimestral calculados; Excel Planificado/Ejecutado | Flujo técnico → coordinador completo |
| **R4 — Indicadores y cierre** | Mié 30-dic (colchón 31-dic) | Indicadores (KPIs, semáforos, matriz, alertas, gráficos, CSV); vista de Fomento; bitácora consultable; endurecimiento de seguridad, carga, respaldos/restauración; manuales y capacitación | **Sistema completo en producción** |

Feriados considerados: 10-nov, 8-dic, 24/25-dic; verificar calendario oficial.

### 5.4 Definición de «terminado»
Pruebas de dominio doradas en verde · prueba de acceso cruzado entre provincias/departamentos · bitácora con usuario real · respaldo restaurado en pruebas · validación por el dueño de producto en entorno de aceptación · documentación de despliegue.

---

## 6. Riesgos

| Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|
| **VM, red y certificados no listos a tiempo para el 1-dic** | Media-alta | Muy alto | Pedir las VM hoy con fecha tope 18-nov; desarrollar y hacer aceptación en un entorno temporal; plan B: ambas funciones en una sola VM |
| Plazo de 3 semanas para R1 | Alta | Alto | R1 limitado a seguridad + Plan Anual + repositorio de planes; todo lo demás después |
| Revisión humana como cuello de botella | Media | Alto | Dueño de producto con validación diaria; pruebas automáticas; entregas pequeñas |
| Cambios de reglas durante el desarrollo (históricamente frecuentes) | Alta | Medio | Catálogos administrables; congelar alcance por entrega |
| Conectividad en provincias | Media | Alto | Formularios ligeros; autoguardado de borrador en el navegador y reintento |
| Seguridad en intranet | Media | Alto | Roles aplicados en servidor, pruebas de acceso cruzado, TLS, bloqueo por intentos, auditoría; revisión de seguridad antes de R4 |
| Calidad del catálogo de cooperativas | Alta | Medio | Depuración mínima antes de cargar; mejora continua posterior |
| Adopción por técnicos | Media | Medio | Piloto con 1–2 provincias en R1; capacitación en R2 y R4 |

---

## 7. Pendientes menores (no bloquean el arranque)

1. Sistema operativo y versión de las VM (se asume Linux, p. ej. Ubuntu Server LTS o RHEL-compatible), y si el servidor de aplicación tiene salida a internet para instalar paquetes (si no, se despliega con paquetes ya construidos).
2. Nombre interno (DNS) y certificado TLS de la CA institucional.
3. ¿IPACOOP tiene Active Directory/LDAP que se deba integrar? (si no, cuentas propias).
4. Confirmar los perfiles de §4.3 y el estado «Enviado/Recibido» del plan.
5. Confirmar que los formularios de consolidado manual no se construyen (§2).
6. Nombre y disponibilidad del responsable técnico y del dueño de producto.

---

## 8. Archivos de referencia
- `referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html` — maqueta original (oráculo de fórmulas).
- `referencia/seeds/catalogos.json` — direcciones, departamentos, programas, tipos, estados, cargos, 1.381 cooperativas.
- `referencia/seeds/codigos-actividad.json` — 72 actividades con sus códigos.
