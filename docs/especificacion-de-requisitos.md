# Sistema de Captura de Informes IPACOOP — Especificación de Requisitos (v1)

Oficina de Planificación · 9 de noviembre de 2026
Documentos relacionados: `docs/valoracion-y-plan-de-desarrollo.md` (stack y calendario) · `referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html` (maqueta, fuente de reglas y fórmulas) · `referencia/seeds/` (catálogos).

---

## 1. Introducción

### 1.1 Propósito
Definir todos los requisitos funcionales, reglas de negocio, datos y requisitos no funcionales del sistema web que permitirá a cada dirección provincial del IPACOOP **capturar de forma autónoma su Plan Anual**, reportar su **ejecución mensual y técnica** de forma **centralizada**, y a la Oficina de Planificación **monitorear el avance e indicadores**.

### 1.2 Alcance
Aplicación web interna + web service (API REST) + base de datos, alojada en la granja de servidores del IPACOOP (un servidor de aplicación y un servidor de base de datos).

### 1.3 Convenciones
- **RF** requisito funcional · **RN** regla de negocio · **RNF** requisito no funcional · **SUP** supuesto adoptado mientras no haya respuesta (ver §10).
- **Prioridad / entrega:** R1 = 1-dic-2026 (Plan Anual) · R2 = 11-dic · R3 = 22-dic · R4 = 30-dic.
- «Debe» = obligatorio; «Debería» = deseable dentro del alcance si el tiempo lo permite.

### 1.4 Definiciones
| Término | Significado |
|---|---|
| Dirección | Unidad que captura: 10 direcciones provinciales (+ Dirección de Fomento y Sede Central en catálogo) |
| Departamento | Área funcional: Promoción y Educación (P&E), Asistencia Técnica (AT), Dirección Provincial, Administración Provincial, Planificación Provincial, Asesoría Legal, Registro Cooperativo, Dirección de Fomento |
| Programa | Línea de trabajo de un departamento (p. ej. Contabilidad y Finanzas) |
| Sección | Bloque de un plan o informe correspondiente a un departamento + programa |
| Actividad | Elemento del catálogo (72) con código: `04-xx` (P&E), `0201–0606` (AT por serie), `Técnicas-01`, `Admva-02..06` (transversales) |
| Familia / plantilla | Tipo de datos específicos que pide una actividad (evento educativo, libros contables, asesoría, asambleas, organización, diagnóstico, seguimiento, informes, material, genérica) |
| ACC | Acción: unidad de medida del avance del plan |
| Programada | Actividad del informe que existe en el plan de la misma dirección, departamento y año |

---

## 2. Actores y perfiles

### 2.1 Perfiles
| Código | Perfil | Alcance de datos |
|---|---|---|
| ADM | Administrador del sistema (TI) | Global, sin edición de contenido de negocio |
| PLA | Oficina de Planificación | Nacional |
| DIR | Director/a Provincial | Su dirección provincial |
| JEF | Jefe/a de departamento provincial (SUP-01) | Su dirección + departamento |
| TEC | Técnico/a | Su dirección + departamento + programa(s) |
| COO | Coordinador/a nacional de departamento (P&E o AT) | Su departamento, todas las provincias |
| FOM | Dirección de Fomento | Lectura, por departamento, todas las provincias |
| AUD | Auditor/a | Lectura de bitácora |

Un usuario puede tener más de un perfil y más de un alcance.

### 2.2 Matriz de permisos
C = crear · E = editar · V = ver · A = aprobar/recibir · — = sin acceso

| Función | ADM | PLA | DIR | JEF | TEC | COO | FOM | AUD |
|---|---|---|---|---|---|---|---|---|
| Usuarios y perfiles | C/E/V | V | — | — | — | — | — | V |
| Catálogos | V | C/E/V | V | V | V | V | V | — |
| Plan Anual (su alcance) | — | V/A (reabrir) | C/E/V, enviar | C/E/V (su depto) | V | V (su depto) | V | — |
| Informe Mensual | — | V | C/E/V | C/E/V | C/E/V (sus filas) | V (su depto) | V | — |
| Informe Técnico | — | V | V | V | C/E/V (propios) | V (su depto) | V | — |
| Respuesta del coordinador | — | V | V | V | V (de sus informes) | C/E/V | V | — |
| Repositorio y avance | — | V nacional | V provincia | V depto | V propio | V depto | V depto | — |
| Consolidados | — | V/imprimir | V provincia | V depto | — | V depto | V | — |
| Indicadores | — | V completo | V provincia (SUP-02) | — | — | V su depto | — | — |
| Bitácora | V | V | — | — | — | — | — | V |
| Exportar Excel/CSV | — | Sí | Sí (su alcance) | — | — | Sí (su depto) | — | — |

---

## 3. Requisitos funcionales

### 3.1 Seguridad y usuarios (SEG) — R1
| ID | Requisito | Criterio de aceptación |
|---|---|---|
| RF-SEG-01 | El sistema debe exigir autenticación con usuario y contraseña para todo acceso. | Ninguna pantalla ni endpoint (salvo ingreso) responde sin sesión válida. |
| RF-SEG-02 | ADM debe poder crear, editar, desactivar y reactivar usuarios con nombre, cédula (SUP-03), correo institucional, cargo (catálogo), perfiles y alcances. | No se eliminan usuarios con historial; solo se desactivan. |
| RF-SEG-03 | El usuario nuevo o con contraseña restablecida debe cambiarla en el primer ingreso. | |
| RF-SEG-04 | Política de contraseñas: mínimo 12 caracteres, no igual al usuario, no reutilizar las 5 últimas. | |
| RF-SEG-05 | Bloqueo temporal tras 5 intentos fallidos (15 min) y desbloqueo por ADM. | |
| RF-SEG-06 | Cierre de sesión manual y por inactividad (30 min; configurable). | |
| RF-SEG-07 | Restablecimiento de contraseña por ADM (y por correo si existe SMTP, SUP-07). | |
| RF-SEG-08 | Toda operación debe validarse en el servidor contra el perfil y el alcance del usuario. | Prueba automática: un TEC de Coclé no puede leer ni modificar datos de Veraguas por API. |
| RF-SEG-09 | Debería poder integrarse con Active Directory/LDAP si IPACOOP lo dispone (SUP-04). | |
| RF-SEG-10 | El usuario debe poder ver su perfil, alcances y último ingreso. | |

### 3.2 Catálogos (CAT) — R1
| ID | Requisito |
|---|---|
| RF-CAT-01 | PLA debe administrar: direcciones, provincias (con código de 3 letras: BDT, CCL, COL, CH, DAR, HER, LSA, PAN, POE, VER), departamentos (con abreviatura DAT/DPE), programas (con código de 3 letras), actividades, códigos, cargos, tipos y estados de cooperativa. |
| RF-CAT-02 | La relación **departamento → programas** debe ser configurable **por año** (vigente 2027 según la maqueta: P&E = Gestión Socio Educativa, Formación Empresarial, Cooperativismo Juvenil, Coordinación y Jefatura, Coordinación del programa; AT = Contabilidad y Finanzas, Asistencia en Agropecuaria, Asistencia en Mercadeo, Organización y gestión Empresarial, Formulación y ejecución de Proyectos, Coordinación y Jefatura, Coordinación del programa; Dirección Provincial = todos; demás departamentos = «N/A»). |
| RF-CAT-03 | La relación **actividad → código(s) → departamento/programa** debe ser configurable (RN-CLA). |
| RF-CAT-04 | La relación **actividad → plantilla de datos específicos** debe ser explícita en catálogo (reemplaza la inferencia por nombre de la maqueta) y precargarse con la clasificación actual. |
| RF-CAT-05 | Catálogo de **cooperativas**: nombre, siglas, tipo, provincia, estado (Activa, Intervenida, Liquidada, Comisión Liquidadora, Inactiva), número de registro (SUP-05). PLA puede agregar, editar y desactivar. |
| RF-CAT-06 | Los formularios deben ofrecer por defecto solo cooperativas Activas e Intervenidas, con opción «mostrar todas» (SUP-06). |
| RF-CAT-07 | Un elemento de catálogo usado en registros no se elimina: se desactiva; los registros históricos conservan su valor. |
| RF-CAT-08 | Carga inicial desde `referencia/seeds/` previa depuración (errores conocidos: «Panampa Sede Central», «Dirección de Fomento de Cooperativo»). |
| RF-CAT-09 | La Dirección de Fomento **no** se ofrece como departamento en los formularios de captura. |

### 3.3 Plan Anual (PLA) — R1
| ID | Requisito |
|---|---|
| RF-PLA-01 | Encabezado: dirección provincial (fija según alcance; PLA puede elegir), departamento, programa, año del plan. |
| RF-PLA-02 | Tabla de actividades con columnas: N.º, Necesidad, Objetivo, Metas, Actividad (lista filtrada por departamento + programa, RN-CLA), cantidades ENE–DIC, TOTAL (automático), costos en B/. — Viáticos, Alquiler/alimentación, Transporte — y TOTAL de costo (automático). |
| RF-PLA-03 | Al elegir la actividad, el sistema asigna el **código** según RN-CLA-04 y muestra el aviso «requiere datos específicos» si tiene plantilla. |
| RF-PLA-04 | Acciones por fila: agregar, duplicar, eliminar (el plan conserva al menos una fila). |
| RF-PLA-05 | Pie con totales por mes, total de cantidades, totales por tipo de costo y costo total. |
| RF-PLA-06 | **Datos específicos por mes**: al elegir la actividad se abre un formulario según su plantilla (§5.3); se captura **por mes**; la cantidad del mes del subformulario y la columna de ese mes deben mantenerse sincronizadas; el indicador de la fila muestra cuántos meses tienen datos. |
| RF-PLA-07 | Firmas (nombres): Elaborado por, Coordinador(a), Aprobado por. |
| RF-PLA-08 | Validación para guardar: dirección y programa obligatorios; al menos una fila con actividad. Las filas sin actividad se descartan. Validaciones adicionales en RN-PLA-05. |
| RF-PLA-09 | **Un plan por dirección + departamento + año** (RN-PLA-01). Guardar una sección (programa) **sustituye solo las filas de esa sección** y conserva las de los demás programas. |
| RF-PLA-10 | Autoguardado de borrador (local en el navegador y en servidor) para no perder la captura por desconexión. |
| RF-PLA-11 | **Estados**: Borrador → Enviado (Recibido por Planificación, con fecha, hora y usuario). Un plan enviado solo lo edita quien tenga perfil después de que PLA lo **reabra** con motivo (SUP-08). |
| RF-PLA-12 | PLA debe ver el **control de recepción**: por año, matriz dirección × departamento con estado (sin iniciar / borrador / enviado) y fecha. |
| RF-PLA-13 | Impresión del plan con encabezado institucional y logo, en formato horizontal, fiel al modelo de la maqueta; «Guardar e imprimir» nunca imprime sin guardar. |
| RF-PLA-14 | Edición de un plan existente desde el repositorio por usuarios con permiso. |
| RF-PLA-15 | Debería permitir copiar un plan de un año anterior como punto de partida. |

### 3.4 Informe Mensual (MEN) — R2
| ID | Requisito |
|---|---|
| RF-MEN-01 | Encabezado: dirección provincial, departamento (área), programa, mes, año, fecha del informe, preparado por. |
| RF-MEN-02 | Filas: Necesidad, Actividad (filtrada por departamento + programa), Programada (Sí/No/—), Cooperativa, Resumen del resultado, B/. ejecutado. Acciones agregar/duplicar/eliminar. |
| RF-MEN-03 | **Programada** se calcula automáticamente (RN-MEN-03) y se recalcula al cambiar dirección, departamento o año; el usuario puede ajustarla manualmente cuando no hay plan contra qué comparar. |
| RF-MEN-04 | **Cooperativa**: buscador sobre el catálogo (sin acentos, por nombre o siglas) que muestra tipo, provincia y estado; permite registrar un nombre no catalogado que queda marcado para revisión de PLA. |
| RF-MEN-05 | **Datos del resultado** (ventana por fila): bloque de impacto según RN-MEN-05 + datos específicos de la plantilla. Muestra el total de asociados (mujeres + hombres) y advierte si la suma por edades no coincide. |
| RF-MEN-06 | Logros del mes y Observaciones/dificultades, guardados **por departamento**. |
| RF-MEN-07 | Firmas: Responsable, Coordinador(a), Aprobado por. |
| RF-MEN-08 | **Un informe por dirección + mes + año** (RN-MEN-01). Cada técnico/sección aporta filas que se **suman** al informe; cada fila guarda autor, departamento y programa. Una fila idéntica a otra existente no se duplica (RN-MEN-02). |
| RF-MEN-09 | Cada autor edita o elimina solo sus filas; DIR/JEF pueden editar todas las de su alcance. |
| RF-MEN-10 | **Cierre del mes** (SUP-09): después de la fecha límite, PLA puede cerrar el mes; las correcciones posteriores requieren reapertura por PLA y quedan en bitácora. |
| RF-MEN-11 | Impresión del informe con encabezado institucional; «Guardar e imprimir» nunca imprime sin guardar. |
| RF-MEN-12 | Autoguardado de borrador. |

### 3.5 Informe Técnico y respuesta del coordinador (TEC) — R3
| ID | Requisito |
|---|---|
| RF-TEC-01 | Encabezado: dirección provincial, departamento (P&E o AT), programa, número de control (automático, RN-TEC-01), fecha, cooperativa (catálogo; tipo y ubicación se completan solos), actividad realizada (con código), nombre del técnico, fecha de la actividad. |
| RF-TEC-02 | **Plantilla por departamento**: AT y otros → Introducción, Objetivo de la actividad, Detalle de la actividad, Conclusión, Recomendaciones. P&E → 1. Antecedentes, 2. Objetivo, 3. Metodología, 4. Resultados, 5. Análisis técnico, 6. Conclusiones, 7. Recomendaciones. |
| RF-TEC-03 | Pie: Elaborado por (nombre y apellido), cargo (catálogo), unidad administrativa, c.c. |
| RF-TEC-04 | Anexos: archivos adjuntos (PDF, JPG, PNG; máx. 10 MB c/u, 10 por informe) (SUP-10). |
| RF-TEC-05 | La **secuencia** del control es el único elemento editable; el sistema propone la primera libre y no permite repetir una secuencia en el mismo alcance. |
| RF-TEC-06 | **Respuesta del coordinador** (COO): número de control de respuesta (automático, RN-TEC-02), fecha de revisión, coordinador(a)s, antecedentes, análisis, recomendaciones, pendientes de respuesta y seguimiento, elaborado por, cargo, c.c. |
| RF-TEC-07 | **Gestión de recomendaciones**: el coordinador agrega recomendaciones al informe (estatus inicial *Pendiente*) y certifica cada una como *Completado* con fecha y usuario. |
| RF-TEC-08 | **Seguimiento**: lista con filtros por provincia, departamento, programa, mes y año (de la actividad), columnas control, cooperativa, fecha de actividad, respondido (sí/no), recomendaciones pendientes; detalle desplegable; acciones ver, editar, responder, imprimir. |
| RF-TEC-09 | Impresión del informe técnico y de la respuesta, fieles a los modelos oficiales. |
| RF-TEC-10 | Debería permitir vincular el informe técnico con la fila del informe mensual que lo originó (SUP-11). |

### 3.6 Repositorio y monitoreo (REP) — R1 (planes) / R2 (resto)
| ID | Requisito |
|---|---|
| RF-REP-01 | Pestaña **Planes Anuales**: filtros provincia, año, departamento, programa (en cascada) y mes; por plan muestra actividades, presupuesto, estado, **avance** (RN-AVA) y barras de avance por departamento/programa. |
| RF-REP-02 | Detalle del plan: comparativo **planificado vs ejecutado** por actividad (ACC plan / ACC ejecutadas, asociados, cooperativas, capacitaciones, presupuesto), actividades **no programadas** separadas, vista **por mes** (columnas ENE–DIC) y **por trimestre**. |
| RF-REP-03 | Pestaña **Informes Mensuales**: filtros provincia, año, mes, departamento, programa; tarjetas por mes con número de filas, impacto y secciones; vista del informe segmentada por departamento/programa. |
| RF-REP-04 | Pestañas **Consolidado Mensual** y **Trimestral** (§3.7) y **Seguimiento a Informes Técnicos** (RF-TEC-08). |
| RF-REP-05 | Cada usuario ve únicamente lo que su alcance permite (RF-SEG-08). |

### 3.7 Consolidados (CON) — R3
| ID | Requisito |
|---|---|
| RF-CON-01 | **Consolidado mensual** calculado por el sistema: parámetros departamento, programa, mes, año, filtro opcional de provincia; una fila por provincia con Actividades Programadas (técnicas / administrativas), Realizadas (técnicas / administrativas), Cumplimiento %, No Programadas (técnicas / administrativas), y totales. Fórmula RN-CON. |
| RF-CON-02 | **Consolidado trimestral**: igual, sumando los 3 meses del trimestre (T1 ene–mar … T4 oct–dic), cumplimiento recalculado sobre la suma. |
| RF-CON-03 | **Resumen analítico** del trimestral: por programa, «Principales resultados» y «Limitantes de las actividades no realizadas» (texto), guardado por dirección/departamento/trimestre. |
| RF-CON-04 | Impresión con logo, encabezado y firmas al pie. |
| RF-CON-05 | No se construyen formularios de consolidado de **captura manual** (SUP-12). |

### 3.8 Indicadores (IND) — R4
| ID | Requisito |
|---|---|
| RF-IND-01 | Filtros: año y departamento. Modo **Planificación** (todo + bitácora) y modo **Jefatura/Coordinación** (solo su departamento, sin bitácora). |
| RF-IND-02 | KPIs con semáforo (RN-IND-01): avance físico (ACC), ejecución presupuestaria, cobertura de mensuales del mes en curso, informes técnicos sin respuesta, recomendaciones pendientes (y documentos con pendientes). |
| RF-IND-03 | **Matriz programa × mes**: estado por celda — recibido (≥1 mensual), atrasado (mes vencido sin mensual), mes en curso, futuro. |
| RF-IND-04 | **Alertas**: programas atrasados (más recientes primero), informes técnicos sin respuesta > 15 días, informes con recomendaciones rezagadas. |
| RF-IND-05 | **Gráficos por programa**: presupuesto planificado vs ejecutado y avance físico (ACC plan vs ejecutadas); informes recibidos por mes. |
| RF-IND-06 | Exportación CSV de los datos mostrados. |

### 3.9 Dirección de Fomento (FOM) — R4
| ID | Requisito |
|---|---|
| RF-FOM-01 | FOM elige un departamento y ve el repositorio (planes, mensuales, consolidados, seguimiento técnico) **solo de ese departamento**, en modo lectura, en todas las provincias; puede cambiar de departamento. |
| RF-FOM-02 | Desde este modo no se muestran indicadores ni bitácora. |

### 3.10 Exportación e impresión (EXP) — R3/R4
| ID | Requisito |
|---|---|
| RF-EXP-01 | Excel de **2 hojas** según filtros (año, dirección, departamento, programa): **Planificado** (detalle de cada actividad del plan) y **Ejecutado** (detalle de cada fila de los mensuales), con columnas: Año, Dirección Provincial, Departamento, Programa, Mes, N.º de actividad en el plan, Necesidad, Objetivo, Metas, Actividad, Código, Cooperativa, Siglas, Tipo, Provincia, Estatus, cifras de impacto y datos específicos. |
| RF-EXP-02 | Todos los documentos imprimibles usan formato carta/oficio (SUP-13), logo institucional, numeración de página y salto de página que no corta filas. |

### 3.11 Bitácora y auditoría (AUD) — R1
| ID | Requisito |
|---|---|
| RF-AUD-01 | Registrar automáticamente: ingreso, salida, intentos fallidos, creación/edición/eliminación/envío/reapertura de cualquier documento, cambios de catálogo y de usuarios; con fecha-hora, usuario real, IP, tipo, referencia y valores antes/después. |
| RF-AUD-02 | La bitácora no se puede modificar ni borrar desde la aplicación. |
| RF-AUD-03 | Consulta con filtros (fecha, usuario, tipo, documento) y exportación CSV para PLA, ADM y AUD. |

### 3.12 Notificaciones (NOT) — R4, «Debería» (SUP-07)
| ID | Requisito |
|---|---|
| RF-NOT-01 | Aviso dentro del sistema (y por correo si hay SMTP) a: PLA cuando se envía un plan; COO cuando se registra un informe técnico de su departamento; DIR/JEF cuando su programa tiene un mensual atrasado. |

---

## 4. Reglas de negocio

> Las fórmulas son **definitivas** (decisión de la Oficina) y deben reproducir exactamente la maqueta; la verificación se hace con pruebas de paridad automáticas.

### 4.1 Clasificación de actividades (RN-CLA)
- **RN-CLA-01** P&E usa códigos `04-01` a `04-21` en todos sus programas.
- **RN-CLA-02** AT usa series por programa: `0201–0209` Organización y gestión Empresarial · `0301–0314` Agropecuaria · `0401–0411` Contabilidad y Finanzas · `0501–0509` Proyectos · `0601–0606` Mercadeo.
- **RN-CLA-03** Transversales `Técnicas-01` y `Admva-02..06` están disponibles en **todos** los departamentos y programas; los departamentos distintos de P&E y AT solo usan transversales.
- **RN-CLA-04** Código asignado: el de la serie del programa; si no, el de P&E/AT según el departamento; si no, el primero.
- **RN-CLA-05** **Técnica vs administrativa**: administrativa si la familia es *informes* o *material*, o el código es `Admva-0x`; si no, técnica.

### 4.2 Plan Anual (RN-PLA)
- **RN-PLA-01** Unicidad: (dirección, departamento, año).
- **RN-PLA-02** TOTAL de fila = Σ cantidades ENE–DIC.
- **RN-PLA-03** Costo total de fila = viáticos + alquiler/alimentación + transporte.
- **RN-PLA-04** Resumen del plan: n.º de actividades, Σ totales, Σ costos totales.
- **RN-PLA-05** Cantidades y costos son números ≥ 0 (costos con 2 decimales).

### 4.3 Informe Mensual (RN-MEN)
- **RN-MEN-01** Unicidad: (dirección, año, mes).
- **RN-MEN-02** Fila duplicada = misma actividad + cooperativa + resumen + estado + programada (comparación sin acentos ni mayúsculas).
- **RN-MEN-03** Programada = la actividad está en el plan de la misma dirección y año **del mismo departamento** (si no hay plan del departamento, se usan los planes de la dirección); sin planes → sin determinar.
- **RN-MEN-04** Total de asociados = mujeres + hombres; si ambos están vacíos, asociadosTotal o participantes.
- **RN-MEN-05** Impacto por familia: *completo* (capacitaciones, cooperativas atendidas, mujeres, hombres, jóvenes ≤35, adultos 36–59, mayores ≥60) para evento educativo, asambleas, organización, asesoría y genérica; *solo cooperativas atendidas* para libros contables, diagnóstico y seguimiento; *ninguno* para informes y material.

### 4.4 Avance del plan (RN-AVA)
- **RN-AVA-01** Solo cuentan filas de mensuales de la misma dirección y año y del departamento del plan (filas sin sección sí cuentan).
- **RN-AVA-02** Una fila es ejecución del plan si `programada = Sí`, o si no es `No` y la actividad está en el plan.
- **RN-AVA-03** ACC planificadas por actividad = TOTAL de la fila; si es 0, `cantidad` de los datos específicos (excepto familia organización).
- **RN-AVA-04** ACC ejecutadas = **1 por cada fila** de ejecución del plan.
- **RN-AVA-05** **Cumplimiento** = `min(100, round(ACC ejecutadas × 100 / ACC planificadas))`; si no hay ACC planificadas → «—»; sin mensuales → 0 %.
- **RN-AVA-06** % presupuesto = `min(100, round(Σ B/. ejecutado × 100 / Σ costo planificado))`.
- **RN-AVA-07** Asociados, cooperativas y capacitaciones se comparan (planificado vs ejecutado, mismos topes) solo como información; no alteran el cumplimiento.
- **RN-AVA-08** Actividades no programadas se muestran aparte y no afectan ningún porcentaje del plan.
- **RN-AVA-09** Con filtro de mes: planificado del mes = columna del mes; cumplimiento del mes = ACC del mes vs planificado del mes.
- **RN-AVA-10** Trimestre = suma de los 3 meses del plan vs ejecución de los mensuales de esos meses.

### 4.5 Consolidado (RN-CON)
- **RN-CON-01** Programadas del mes = actividades del plan del departamento y programa con cantidad > 0 en ese mes, separadas en técnicas/administrativas.
- **RN-CON-02** Realizadas = actividades **distintas** del plan reportadas en el mensual de ese mes.
- **RN-CON-03** No programadas = filas que no son ejecución del plan (se cuentan todas).
- **RN-CON-04** Cumplimiento = `min(100, round(realizadas técnicas × 100 / programadas técnicas))`; 0 si no hay programadas técnicas.

### 4.6 Indicadores (RN-IND)
- **RN-IND-01** Semáforo: ≥ 75 % verde · ≥ 40 % ámbar · < 40 % rojo · sin dato gris.
- **RN-IND-02** Cobertura de mensuales = programas con ≥1 mensual en el mes en curso / programas con actividades planificadas en el año.
- **RN-IND-03** Informe técnico sin respuesta en alerta si han pasado más de 15 días desde la fecha de la actividad (o de creación).

### 4.7 Números de control (RN-TEC)
- **RN-TEC-01** Informe técnico: `DEPTO-PROV-PROG-SEC-AÑO`, p. ej. `DAT-BDT-CYF-001-2026`; DEPTO = DAT/DPE; PROV = código de la provincia; PROG = código del programa; SEC = 001–500 (como la maqueta) única por (depto, provincia, programa, año); AÑO = año de la fecha de la actividad.
- **RN-TEC-02** Respuesta: `DEPTO-COOR-PROG-SEC-AÑO`, p. ej. `DAT-COOR-CYF-001-2026`, secuencia única por (depto, programa, año).
- **RN-TEC-03** La asignación es atómica en el servidor (dos usuarios simultáneos nunca obtienen el mismo número).

### 4.8 Generales
- **RN-GEN-01** Moneda B/. con 2 decimales, formato es-PA; porcentajes enteros con redondeo half-up.
- **RN-GEN-02** Comparaciones de nombres sin distinguir acentos ni mayúsculas.
- **RN-GEN-03** Los documentos impresos muestran la fecha y hora de generación.

---

## 5. Datos

### 5.1 Entidades principales
Catálogos (provincia, dirección, departamento, programa, departamento-programa por año, actividad, código, plantilla, cooperativa, cargo) · Seguridad (usuario, perfil, alcance, sesión) · Plan (plan, sección, actividad del plan, mes del plan con datos específicos, historial de estados) · Mensual (informe, aporte, fila con cifras de impacto como columnas) · Técnico (informe técnico, anexo, recomendación con estatus, respuesta, secuencias) · Consolidado (resumen analítico trimestral, cierres de mes) · Bitácora.

### 5.2 Volumen estimado
~12 direcciones × hasta 8 departamentos × ~10 programas; ~40–100 filas por plan; ~10.000–30.000 filas de mensual por año; cientos de informes técnicos por año. Retención mínima: 10 años (SUP-14).

### 5.3 Plantillas de datos específicos (precarga desde la maqueta)
| Plantilla | Campos |
|---|---|
| Evento educativo | cantidad de eventos, tema, asociados, cooperativas, duración, modalidad, técnica didáctica, lugar, período, facilitador, observaciones |
| Libros contables | cooperativas a revisar, cantidad de libros, tipos de libros, período, duración, alcance, normativa, responsable, observaciones |
| Asesoría | cantidad, materia, producto esperado, contraparte, cooperativas atendidas, asociados beneficiados, período, observaciones |
| Asambleas | cantidad, tipo de asamblea, cooperativas, asociados, documentos, lugar, período, observaciones |
| Organización | cantidad de cooperativas, etapa, tipo de cooperativa, interesados (mín. 20), sector, período, observaciones |
| Diagnóstico | cantidad, alcance, período a analizar, cooperativas, técnicas/instrumentos, período, observaciones |
| Seguimiento | cantidad de visitas, cooperativas, actividad previa, recomendaciones a verificar, área, período, observaciones |
| Informes | cantidad, periodicidad, destinatario, tema, observaciones |
| Material | cantidad, tipo de material, tema, población, período, observaciones |
| Genérica | cantidad, tema, cooperativas, asociados, período, observaciones |

Además, en todas: nombres de las cooperativas involucradas (del catálogo).

---

## 6. Requisitos no funcionales

### 6.1 Seguridad
| ID | Requisito |
|---|---|
| RNF-SEG-01 | HTTPS obligatorio (TLS 1.2+) con certificado de la CA institucional; HSTS. |
| RNF-SEG-02 | Contraseñas con argon2id; nunca se registran ni se envían en claro. |
| RNF-SEG-03 | Sesión en cookie `HttpOnly`, `Secure`, `SameSite=Strict`; protección CSRF; límite de peticiones por IP y usuario. |
| RNF-SEG-04 | Cabeceras de seguridad (CSP, X-Frame-Options, X-Content-Type-Options); validación de toda entrada en servidor; consultas parametrizadas. |
| RNF-SEG-05 | El servidor de base de datos solo acepta conexiones desde el servidor de aplicación; usuario de BD con privilegios mínimos. |
| RNF-SEG-06 | Archivos adjuntos: validación de tipo real y tamaño, almacenados fuera de la raíz web, servidos solo con autorización. |
| RNF-SEG-07 | Revisión de seguridad (OWASP Top 10 / ASVS nivel 1) antes de R4. |
| RNF-SEG-08 | Ningún dato se envía a servicios externos (incluida IA) en tiempo de ejecución. |
| RNF-SEG-09 | Cumplimiento de la Ley 81 de 2019 de protección de datos personales en el tratamiento de datos de usuarios y asociados (validar con Asesoría Legal). |

### 6.2 Rendimiento y capacidad
- RNF-REN-01 Pantallas de captura cargan en < 2 s y guardan en < 1 s en la red institucional.
- RNF-REN-02 Repositorio, consolidados e indicadores responden en < 3 s con 3 años de datos.
- RNF-REN-03 Soporta 150 usuarios con sesión y 50 simultáneos en cierre de mes sin degradación (SUP-15).

### 6.3 Disponibilidad, respaldo y recuperación
- RNF-DIS-01 Disponible en horario laboral (lun–vie 7:00–18:00) con objetivo de 99 %; mantenimientos programados fuera de horario.
- RNF-DIS-02 Respaldo completo diario + archivado continuo (WAL); retención 30 días diarios y 12 mensuales; copia fuera del servidor de BD.
- RNF-DIS-03 RPO ≤ 24 h (≤ 15 min con WAL) · RTO ≤ 4 h. Prueba de restauración antes de cada entrega a producción.

### 6.4 Usabilidad y accesibilidad
- RNF-USA-01 Interfaz en español, terminología institucional de la maqueta.
- RNF-USA-02 Funciona en Chrome, Edge y Firefox actuales; diseño adaptable a pantallas desde 1280 px (captura) y consulta en tableta.
- RNF-USA-03 Mensajes de validación claros junto al campo; confirmación antes de eliminar.
- RNF-USA-04 Contraste y navegación por teclado (WCAG 2.1 AA como referencia).
- RNF-USA-05 Tolerancia a conexión intermitente: autoguardado de borradores y reintento al reconectar.

### 6.5 Mantenibilidad y calidad
- RNF-MAN-01 Código en repositorio Git con revisión; pruebas automáticas en cada cambio.
- RNF-MAN-02 Fórmulas en un único módulo de dominio con pruebas de paridad contra la maqueta (100 % de las reglas de §4).
- RNF-MAN-03 API documentada con OpenAPI.
- RNF-MAN-04 Migraciones de base de datos versionadas y reversibles.
- RNF-MAN-05 Manuales: usuario (por perfil), administración, instalación/despliegue, respaldo/restauración.

### 6.6 Infraestructura y operación
- RNF-INF-01 Dos servidores virtuales en la granja del IPACOOP: aplicación (Nginx + Node.js) y base de datos (PostgreSQL 16); sistema operativo Linux (SUP-16).
- RNF-INF-02 Solo software libre, sin costo de licencias.
- RNF-INF-03 Entornos: producción y pruebas/aceptación.
- RNF-INF-04 Registros técnicos (logs) con rotación; monitoreo de disponibilidad y espacio en disco.
- RNF-INF-05 Despliegue reproducible por script, sin necesidad de internet en el servidor.

---

## 7. Interfaces
- **Usuario:** navegador web en la intranet.
- **Web service:** API REST/JSON autenticada, documentada (OpenAPI); es la única vía de acceso a los datos (la interfaz la consume).
- **Externas:** ninguna. Opcionales: LDAP/AD (RF-SEG-09) y SMTP institucional (RF-NOT-01).

---

## 8. Fuera de alcance
Análisis con IA de informes técnicos · importación/restauración de respaldos JSON de la maqueta · migración de datos históricos · formularios de consolidado de captura manual · aplicación móvil nativa · firma electrónica · integración con sistemas financieros o presupuestarios.

---

## 9. Trazabilidad por entrega

| Entrega | Fecha | Requisitos |
|---|---|---|
| R1 | 1-dic-2026 | SEG-01…10, CAT-01…09, PLA-01…14, REP-01 (sin avance), REP-05, AUD-01…03, RNF de seguridad, respaldo e infraestructura |
| R2 | 11-dic-2026 | MEN-01…12, REP-01…03 (con avance), RN-AVA completas |
| R3 | 22-dic-2026 | TEC-01…10, CON-01…05, EXP-01…02, REP-04 |
| R4 | 30-dic-2026 | IND-01…06, FOM-01…02, NOT-01, PLA-15, revisión de seguridad, manuales, capacitación |

---

## 10. Supuestos adoptados y preguntas abiertas

Cada supuesto se aplica para no detener el desarrollo; si la Oficina responde distinto, se ajusta.

| ID | Supuesto adoptado | Pregunta para la Oficina | Necesario para |
|---|---|---|---|
| SUP-01 | Existe el perfil Jefe/a de departamento provincial que captura la sección de su departamento | ¿Quién captura el plan en cada provincia? ¿Cuántas personas? | R1 |
| SUP-02 | DIR ve indicadores de su provincia | ¿Los directores provinciales ven indicadores? | R4 |
| SUP-03 | Usuario identificado por cédula + correo institucional | ¿Hay directorio institucional? | R1 |
| SUP-04 | Cuentas propias; AD/LDAP solo si existe | ¿Hay Active Directory? | R1 |
| SUP-05 | Número de registro opcional; identificador interno | ¿Fuente oficial de cooperativas (Registro Cooperativo)? | R1 |
| SUP-06 | Solo Activas e Intervenidas por defecto | ¿Se permiten liquidadas/inactivas? | R2 |
| SUP-07 | Notificaciones dentro del sistema; correo si hay SMTP | ¿Hay servidor de correo disponible? | R4 |
| SUP-08 | Estados Borrador / Enviado; reapertura por PLA con motivo | ¿Planificación aprueba o devuelve con observaciones? ¿Fecha límite de entrega? ¿Reformulación a mitad de año? | R1 |
| SUP-09 | Fecha límite del mensual: día 5 hábil del mes siguiente; cierre por PLA | ¿Cuál es la fecha límite real? | R2 |
| SUP-10 | Adjuntos PDF/JPG/PNG, 10 MB, 10 por informe | ¿Qué anexos se exigen? | R3 |
| SUP-11 | Vínculo informe técnico ↔ fila mensual opcional | ¿Es obligatorio? | R3 |
| SUP-12 | Consolidados solo calculados | Confirmar | R3 |
| SUP-13 | Plan y consolidados en horizontal; informes técnicos en vertical, tamaño carta | ¿Formato de papel oficial? Entregar modelos originales (.docx/.xlsx) | R1/R3 |
| SUP-14 | Retención 10 años | ¿Norma de retención documental? | R4 |
| SUP-15 | ~150 usuarios, 50 simultáneos | ¿Número real de usuarios? | R1 |
| SUP-16 | Linux (Ubuntu Server LTS o compatible) | ¿SO de las VM, salida a internet, DNS y certificado? | Semana 0 |
| SUP-17 | Umbrales del semáforo (75/40) y alerta (15 días) son vigentes y configurables | Confirmar | R4 |
| SUP-18 | Catálogo de actividades y programas 2027 = el de la maqueta | ¿Hay cambios para 2027? | R1 |
| SUP-19 | Presupuesto del plan sin tope | ¿Hay techo presupuestario por provincia/programa? | R1 |

**Insumos solicitados:** lista inicial de usuarios con perfil y alcance; modelos oficiales (INFORMES TÉCNICOS.docx, modelo de recomendaciones.docx, borrador de consolidado mensual); un plan 2026 real y uno o dos mensuales e informes técnicos de ejemplo para pruebas de aceptación.
