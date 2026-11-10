# Sistema de Captura de Informes IPACOOP — Valoración, stack y plan de desarrollo (v3)

Oficina de Planificación · documento de trabajo · v2 del 9 de noviembre de 2026
Base del análisis: `referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html` (8.117 líneas, ~233 funciones, 9 vistas, ~59 iteraciones de requisitos visibles en los comentarios «Task N»).

**v3 (10-nov):** stack simplificado a una sola aplicación Django por tratarse de una plataforma administrativa no crítica (§4).

**Cambios de la v2 respecto a la v1** (según las respuestas de la Oficina):
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
- **Arquitectura propuesta (v3, simplificada):** una sola aplicación web **Django** (Python) con su **web service REST** y **PostgreSQL**, en dos máquinas virtuales Linux. Al no ser un sistema de misión crítica se eligió lo más simple de construir y mantener. Todo software libre; costo de licencias = 0.
- **Seguridad:** autenticación de Django con contraseñas protegidas (argon2), sesiones seguras, **perfiles (roles) con alcance por provincia / departamento / programa aplicado en el servidor**, bitácora de auditoría que no se puede alterar desde la aplicación, TLS interno.
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

**Regla de implementación:** las fórmulas viven en un único módulo (`planificacion/calculos.py`) y se verifican con **pruebas de paridad contra la maqueta** (§4.3). Como la Oficina las considera definitivas, cualquier diferencia entre el sistema y la maqueta es un defecto, no una decisión.

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
| Impresión/Excel | `window.open` + ZIP escrito a mano | Impresión del navegador con la misma CSS + `openpyxl` |
| Catálogos | Constantes en el código; programas por departamento cambiaron tres veces | Tablas administrables por Planificación, versionadas por año |
| Calidad de datos | `Panampa Sede Central`, `Dirección de Fomento de Cooperativo`; cooperativas sin identificador (427 liquidadas, 276 inactivas de 1.381) | Depuración antes de cargar las semillas; identificador interno |

---

## 4. Stack tecnológico recomendado (v3 — simplificado)

**Criterio rector (decisión de la Oficina, 10-nov):** es una plataforma **administrativa** de planificación, **no de misión crítica**. Se prioriza lo más simple de construir, operar y mantener: **una sola aplicación, un solo lenguaje, el mínimo de piezas**, todo software libre.

### 4.0 Qué cambió frente a la v2
| v2 (descartado) | v3 (recomendado) | Por qué |
|---|---|---|
| Aplicación de una página (React) + API separada (Node/Fastify) | **Una sola aplicación web Django** con pantallas generadas en el servidor | Una pieza en vez de dos; sin compilación del frontend ni gestión de estado en el navegador |
| Pantallas de usuarios, perfiles y catálogos hechas a mano | **Administración de Django** (incluida) | Usuarios, grupos/perfiles, permisos y los ~10 catálogos quedan resueltos casi sin código |
| Autenticación propia (sesiones, CSRF, argon2, bloqueo) | **Autenticación de Django** + `django-axes` | Probado y mantenido; menos código de seguridad propio |
| Bitácora con *triggers* SQL | **`django-simple-history`** (historial por registro) + registro de ingresos | Configuración, no desarrollo |
| Monorepo pnpm/Turborepo, 3 paquetes | **Un solo proyecto** Django | Más fácil de entender y mantener |
| Respaldo con archivado continuo (WAL) | **`pg_dump` diario** con retención | Suficiente para un sistema no crítico (pérdida máxima: 1 día) |

### 4.1 Componentes
| Capa | Elección | Justificación |
|---|---|---|
| Lenguaje | **Python 3.12** | Legible, muy usado en el sector público, abundan desarrolladores para el mantenimiento |
| Framework | **Django 5.2 LTS** (soporte hasta abril de 2028) | Incluye ORM, migraciones, autenticación, permisos, administración, formularios, protección CSRF/XSS/SQL y plantillas |
| Interactividad | **HTMX + Alpine.js** (sin compilación) | Suficiente para la matriz del plan (totales automáticos, filas que se agregan/duplican, ventana de datos específicos por mes) sin una SPA |
| Estilos | **Bootstrap 5** (o la CSS de la maqueta reutilizada) | Componentes listos; la CSS de impresión de la maqueta se reutiliza tal cual |
| Base de datos | **PostgreSQL 16** | Gratuita, robusta; `JSONB` para datos específicos y `unaccent` para búsquedas sin acentos |
| **Web service** | **Django REST Framework**, acotado: consulta de catálogos, planes, informes y avance (JSON) | Cumple el requisito de web service para integraciones futuras sin que la interfaz dependa de él |
| Fórmulas | Módulo `planificacion/calculos.py` (funciones puras) | Una sola ubicación para todas las reglas de §3.2 |
| Excel | **openpyxl** | Planificado / Ejecutado |
| Impresión | Navegador con la **CSS de impresión de la maqueta** | Sin generar PDF en el servidor |
| Gráficos | **Chart.js** | Barras y matriz de los indicadores |
| Seguridad adicional | `django-axes` (bloqueo por intentos), hash **argon2**, cookies seguras, cierre de sesión por inactividad | |
| Auditoría | `django-simple-history` + registro de ingresos/salidas | Quién cambió qué y cuándo, con valores anteriores |
| Pruebas | **pytest-django** + pruebas de paridad contra la maqueta | §4.3 |
| Servidor | **Nginx** (TLS, archivos estáticos) → **Gunicorn** (servicio `systemd`) | Configuración estándar y documentada |

Descartado por innecesario para este alcance: frontend separado, contenedores obligatorios, colas de trabajos, generación de PDF en servidor, Redis, CI complejo.

### 4.2 Topología en la granja de servidores
| Servidor | Contenido | Dimensionamiento estimado* |
|---|---|---|
| **VM-App** | Linux + Nginx + Gunicorn/Django | 2 vCPU · 4 GB RAM · 40 GB |
| **VM-BD** | Linux + PostgreSQL 16 + respaldo diario | 2 vCPU · 4–8 GB RAM · 60 GB |

\*Estimación; el volumen es pequeño. Solo la VM-App se conecta a la BD (puerto 5432); los usuarios entran por HTTPS a la VM-App. El entorno de **pruebas/aceptación** es un segundo sitio en las mismas VM con su propia base de datos.

### 4.3 Paridad con la maqueta
Las fórmulas son definitivas, así que se verifican contra la propia maqueta:
1. Una sola vez, se ejecutan las funciones de la maqueta (`avancePlan`, `consolFilas`, `indicadoresCalcular`) sobre ~30 escenarios con casos límite y se guardan entradas y resultados como archivos JSON.
2. Las pruebas de `calculos.py` deben reproducir esos resultados exactamente. Cualquier diferencia es un defecto.

### 4.4 Costo
Licencias: **0**. Costos reales: los recursos de las 2 VM (ya disponibles en la granja), el uso de Claude Code y el tiempo del responsable técnico y del dueño de producto.

### 4.5 Perfiles propuestos (a validar con Planificación)

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

### 4.6 Modelo de datos (esquema inicial)
- **Catálogos (versionados por año):** `provincia`, `direccion`, `departamento`, `programa`, `departamento_programa(año)`, `actividad`, `actividad_codigo(depto, programa)`, `plantilla_datos`, `actividad_plantilla`, `cooperativa`.
- **Seguridad:** `usuario`, `rol`, `usuario_alcance(provincia, departamento, programa)`, `sesion`, `audit_log`.
- **Plan:** `plan_anual(direccion, departamento, año, estado)` **único** → `plan_seccion(programa)` → `plan_actividad(actividad, necesidad, objetivo, meta, costo_viaticos, costo_alquiler, costo_transporte)` → `plan_actividad_mes(mes, cantidad, datos JSONB)`.
- **Mensual:** `informe_mensual(direccion, año, mes)` **único** → `informe_aporte(autor, departamento, programa, logros, dificultades)` → `informe_fila(actividad, cooperativa, programada, resumen, estado, mujeres, hombres, jovenes, adultos, mayores, coop_atendidas, capacitaciones, presupuesto_ejec, datos JSONB)`. Las cifras que entran en fórmulas son **columnas reales**.
- **Técnico:** `informe_tecnico`, `tecnico_pendiente(estatus)`, `respuesta_coordinador`, `secuencia_control` (atómica, única).

---

## 5. Plan de desarrollo (inicio: lunes 9 de noviembre de 2026)

### 5.1 Forma de trabajo con Claude Code
- **Desarrollo guiado por especificación:** este documento y la maqueta son la fuente; se crea un `CLAUDE.md` con convenciones (estructura del proyecto Django, reglas de seguridad, «las fórmulas no se tocan»).
- **Rebanadas verticales** (modelo → vista/pantalla → prueba) con pruebas automáticas en cada una; sesiones paralelas en ramas separadas para módulos independientes.
- **Roles humanos mínimos:** (1) *responsable técnico* que revisa el código, administra los servidores y despliega; (2) *dueño de producto de Planificación* con tiempo diario para validar pantallas y reglas; (3) *administrador de infraestructura* que entrega las VM.
- **Congelamiento de alcance** una vez iniciada cada entrega; cambios nuevos entran a la siguiente.

### 5.2 Semana 0 — arranque (9–13 nov; martes 10 feriado)
| Día | Entregable |
|---|---|
| Lun 9 | Plan v2 acordado; **solicitud formal de las 2 VM** y accesos (fecha tope de entrega: **miércoles 18-nov**) |
| Mié 11 | Proyecto Django, `CLAUDE.md`, pruebas automáticas, modelos de catálogos, carga de semillas (depuradas), escenarios de paridad |
| Jue 12 | Ingreso, perfiles (grupos) y alcance, administración de usuarios y catálogos, historial de cambios |
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
