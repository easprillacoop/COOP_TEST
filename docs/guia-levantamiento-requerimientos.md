# Guía para el levantamiento del requerimiento final

**Sistema de Captura de Informes — IPACOOP · Oficina de Planificación**
Dirigida a: líder funcional del proyecto · Versión 1 · 10 de noviembre de 2026
**Entrega del requerimiento final: viernes 13 de noviembre de 2026** (para análisis final e inicio de la construcción)

---

## 1. Para qué sirve esta guía

El sistema ya cuenta con:
- una **maqueta funcional** (`referencia/maqueta/Sistema_Captura_Informes_IPACOOP.html`) con las pantallas, campos, catálogos y fórmulas;
- una **especificación de requisitos preliminar** (`docs/especificacion-de-requisitos.md`), escrita a partir de la maqueta y de las decisiones ya tomadas por la Oficina.

Lo que falta no es *qué se captura* (eso ya lo define la maqueta), sino **quién hace cada cosa, cuándo, con qué controles y qué pasa en las excepciones**. Esta guía le indica:
1. qué **ya está decidido** y no necesita volver a discutirse (§2);
2. con **quién** reunirse y en qué orden (§3);
3. qué **decisiones** debe dejar plasmadas, módulo por módulo, con la respuesta propuesta por defecto (§4);
4. qué **insumos** (listas, modelos, ejemplos) debe recopilar (§5);
5. cómo entregar el resultado y cuándo se considera completo (§6–§7).

> **Regla práctica:** cada pregunta de §4 trae una **respuesta propuesta** (el supuesto con el que ya se diseñó). Si el área está de acuerdo, basta marcar «Confirmado». Solo hay que redactar cuando la respuesta sea distinta.

---

## 2. Lo que ya está decidido (no reabrir)

| Tema | Decisión |
|---|---|
| Propósito | Cada dirección provincial captura **de forma autónoma** su Plan Anual; la **ejecución se centraliza** en el sistema y desde allí se calculan consolidados e indicadores |
| Fórmulas | Las fórmulas de la maqueta (avance, cumplimiento, consolidados, semáforos, indicadores) son **definitivas**; se implementan tal cual |
| Prioridad y fechas | **Plan Anual disponible el 1-dic-2026**; resto del sistema al **31-dic-2026** |
| Uso | Interno, sobre la red institucional; plataforma **administrativa, no de misión crítica** |
| Hospedaje | Granja de servidores propia: 1 servidor de aplicación + 1 servidor de base de datos |
| Tecnología | Python/Django + PostgreSQL, software libre (lo define el equipo técnico) |
| IA | **No** se envía información a servicios de IA externos |
| Datos | El sistema arranca **vacío**; no hay migración |
| Consolidados | Los calcula el sistema; no habrá formularios de consolidado de captura manual *(confirmar en D-CON-01)* |
| Fuera de alcance | Aplicación móvil, firma electrónica, integración con sistemas financieros |

---

## 3. Agenda propuesta de levantamiento

| Día | Sesión (60–90 min) | Participantes | Decisiones a cerrar |
|---|---|---|---|
| **Mié 11** AM | Plan Anual y recepción | Planificación + 1–2 directores provinciales | Bloques A, B y C — **críticos para el 1-dic** |
| **Mié 11** PM | Perfiles y usuarios | Planificación + TI | Bloque A (perfiles) y lista de usuarios |
| **Jue 12** AM | Ejecución mensual e informes técnicos | Planificación + coordinadores de P&E y AT + 1 técnico | Bloques D y E |
| **Jue 12** PM | Indicadores, reportes y operación | Planificación + Dirección de Fomento + TI + Asesoría Legal (15 min) | Bloques F, G y H |
| **Vie 13** AM | Consolidación y firma | Líder funcional + Planificación | Revisión final y entrega |

Recomendación: abrir la maqueta durante cada sesión y recorrer las pantallas con los usuarios; las preguntas de §4 siguen el mismo orden que las pantallas.

---

## 4. Decisiones que deben quedar plasmadas

Prioridad: **R1** = indispensable antes del 1-dic · **R2–R4** = se puede cerrar después, pero conviene hacerlo ahora.
Columna «Respuesta»: marcar **Confirmado** o escribir la definición del área.

### Bloque A — Organización, perfiles y usuarios (R1)

| ID | Pregunta | Respuesta propuesta (por defecto) | Respuesta |
|---|---|---|---|
| D-ORG-01 | ¿Qué direcciones capturan plan? | Las 10 direcciones provinciales. «Dirección de Fomento» y «Sede Central» solo consultan | |
| D-ORG-02 | ¿Qué departamentos elaboran plan? | Promoción y Educación, Asistencia Técnica y Dirección Provincial (con programas); los demás (Administración, Planificación Provincial, Asesoría Legal, Registro) usan solo actividades transversales con programa «N/A» | |
| D-ORG-03 | ¿Quién captura físicamente el plan en cada provincia? | El jefe de cada departamento captura su sección; el director provincial revisa y envía | |
| D-ORG-04 | ¿Cuántos usuarios tendrá el sistema y cuántos por provincia? | ~150 en total (estimado) | |
| D-PER-01 | ¿Se aprueban estos perfiles? Administrador TI · Oficina de Planificación · Director Provincial · Jefe de departamento · Técnico · Coordinador nacional de departamento · Dirección de Fomento (lectura) · Auditor | Sí (ver matriz en §2.2 de la especificación) | |
| D-PER-02 | ¿Un técnico ve solo su programa o todo su departamento en la provincia? | Captura en su programa; ve todo su departamento en su provincia | |
| D-PER-03 | ¿Los directores provinciales ven indicadores de su provincia? | Sí | |
| D-PER-04 | ¿Quién da de alta y baja a los usuarios después de la puesta en marcha? | TI, a solicitud de Planificación | |
| D-PER-05 | ¿Existe Active Directory/LDAP institucional que deba usarse para el ingreso? | No; cuentas propias del sistema | |

### Bloque B — Catálogos (R1)

| ID | Pregunta | Respuesta propuesta | Respuesta |
|---|---|---|---|
| D-CAT-01 | ¿Los programas por departamento para 2027 son los de la maqueta? (P&E: Gestión Socio Educativa, Formación Empresarial, Cooperativismo Juvenil, Coordinación y Jefatura, Coordinación del programa · AT: Contabilidad y Finanzas, Agropecuaria, Mercadeo, Organización y gestión Empresarial, Proyectos, Coordinación y Jefatura, Coordinación del programa) | Sí | |
| D-CAT-02 | ¿Las 72 actividades y sus códigos siguen vigentes para 2027? | Sí | |
| D-CAT-03 | ¿Quién es el responsable oficial de mantener los catálogos? | Oficina de Planificación | |
| D-CAT-04 | ¿Cuál es la fuente oficial de cooperativas y hay un número de registro que sirva de identificador? | Registro Cooperativo; número de registro opcional | |
| D-CAT-05 | ¿Se pueden elegir cooperativas Liquidadas, Inactivas o en Comisión Liquidadora? | Por defecto solo Activas e Intervenidas; opción «mostrar todas» | |
| D-CAT-06 | Corregir nombres del catálogo: «Panampa Sede Central», «Dirección de Fomento de Cooperativo» | Corregir a «Panamá Sede Central» y «Dirección de Fomento Cooperativo» | |
| D-CAT-07 | ¿Los códigos de provincia y programa del número de control son oficiales? (BDT, CCL, COL, CH, DAR, HER, LSA, PAN, POE, VER · CYF, AGA, AME, OGG, FEP, GSE, FEM, CJC, CPR, CYJ…) | Sí | |

### Bloque C — Plan Anual y su recepción (R1)

| ID | Pregunta | Respuesta propuesta | Respuesta |
|---|---|---|---|
| D-PLA-01 | ¿Cuál es la fecha límite para enviar el Plan Anual 2027? | Por definir por Planificación | |
| D-PLA-02 | ¿Planificación solo **recibe** el plan o también lo **revisa y aprueba / devuelve** con observaciones? | Estados Borrador → Enviado (recibido); Planificación puede reabrirlo con motivo | |
| D-PLA-03 | Una vez enviado, ¿quién puede modificarlo? | Nadie, salvo que Planificación lo reabra | |
| D-PLA-04 | ¿Se permite **reformular** el plan a mitad de año? ¿Hay que conservar la versión original? | Sí con reapertura; el historial guarda todas las versiones | |
| D-PLA-05 | ¿Qué campos son obligatorios por actividad? | Actividad y al menos una cantidad mensual; Necesidad, Objetivo y Metas recomendados | |
| D-PLA-06 | ¿Existe un **tope presupuestario** por provincia o programa? | No; el costo es informativo | |
| D-PLA-07 | Las firmas (Elaborado, Coordinador, Aprobado), ¿son solo nombres impresos o se requiere aprobación dentro del sistema? | Nombres impresos; firma manuscrita sobre el documento | |
| D-PLA-08 | Formato de impresión oficial (papel, orientación, logo, pie) | Carta horizontal con logo, como la maqueta | |
| D-PLA-09 | ¿Se desea poder copiar el plan del año anterior como base? | Sí (deseable para 2028) | |

### Bloque D — Informe Mensual (R2)

| ID | Pregunta | Respuesta propuesta | Respuesta |
|---|---|---|---|
| D-MEN-01 | ¿Hasta qué día del mes siguiente se entrega el mensual? | Día 5 hábil | |
| D-MEN-02 | ¿Cada técnico registra sus filas o el director consolida y registra todo? | Cada técnico registra; el director revisa | |
| D-MEN-03 | ¿El director debe aprobar el mensual antes de que cuente para el avance? | No; cuenta desde que se registra | |
| D-MEN-04 | ¿Existe **cierre de mes**? ¿Quién puede corregir después? | Planificación cierra; corrige solo con reapertura | |
| D-MEN-05 | ¿Se permite registrar una cooperativa que no está en el catálogo? | Sí, marcada para revisión de Planificación | |
| D-MEN-06 | ¿Quién firma el mensual? | Responsable, Coordinador, Aprobado (nombres) | |

### Bloque E — Informe Técnico y respuesta del coordinador (R3)

| ID | Pregunta | Respuesta propuesta | Respuesta |
|---|---|---|---|
| D-TEC-01 | ¿Cuándo es obligatorio hacer informe técnico? (¿toda visita, solo ciertas actividades?) | Por definir | |
| D-TEC-02 | ¿Debe vincularse a la fila del informe mensual que lo originó? | Opcional | |
| D-TEC-03 | ¿Qué anexos se adjuntan (fotos, listas de asistencia, actas)? ¿Tamaño y cantidad? | PDF/JPG/PNG, 10 MB c/u, hasta 10 | |
| D-TEC-04 | ¿Los 15 días para la respuesta del coordinador son norma oficial? | Sí, configurable | |
| D-TEC-05 | ¿Quién certifica una recomendación como «Completado» y con qué evidencia? | El coordinador, con fecha y observación | |
| D-TEC-06 | ¿El tope de 500 secuencias por departamento/provincia/programa/año es suficiente? | Sí | |

### Bloque F — Consolidados, indicadores y reportes (R3–R4)

| ID | Pregunta | Respuesta propuesta | Respuesta |
|---|---|---|---|
| D-CON-01 | Confirmar que no habrá consolidado de captura manual | Confirmado | |
| D-CON-02 | ¿El «Resumen analítico» del trimestral (principales resultados y limitantes) lo redacta cada provincia o Planificación? | Cada dirección provincial por departamento | |
| D-IND-01 | ¿Quién usa los indicadores y para qué informe (Dirección Ejecutiva, Junta Directiva, entes externos)? | Por definir | |
| D-IND-02 | ¿Los cortes del semáforo (≥75 % verde, ≥40 % ámbar) son vigentes? | Sí, configurables | |
| D-IND-03 | ¿Qué reportes deben exportarse a Excel además de Planificado/Ejecutado? | Ninguno adicional | |
| D-FOM-01 | ¿La Dirección de Fomento solo consulta? | Sí, por departamento | |

### Bloque G — Notificaciones (R4)

| ID | Pregunta | Respuesta propuesta | Respuesta |
|---|---|---|---|
| D-NOT-01 | ¿Se requieren avisos por correo (plan enviado, mensual atrasado, informe técnico pendiente)? | Avisos dentro del sistema; correo si TI provee servidor SMTP | |

### Bloque H — Operación, seguridad y datos (R1–R4)

| ID | Pregunta | Respuesta propuesta | Respuesta |
|---|---|---|---|
| D-OPE-01 | Horario de uso y de soporte | Lun–vie 7:00–18:00 | |
| D-OPE-02 | ¿Las provincias acceden por la red institucional (VPN/MPLS) o por internet? | Red institucional | |
| D-OPE-03 | ¿Se usará en campo desde tabletas? | No para captura; consulta posible | |
| D-OPE-04 | ¿Quién atiende consultas de usuarios después de la puesta en marcha? | Planificación (funcional) y TI (técnico) | |
| D-SEG-01 | ¿Qué información de las cooperativas o asociados es reservada? (Asesoría Legal, Ley 81 de 2019) | Solo datos agregados; sin datos personales de asociados | |
| D-SEG-02 | ¿Cuántos años se conservan los datos? | 10 años | |
| D-SEG-03 | ¿Hay requisitos de auditoría interna o de la Contraloría sobre la bitácora? | Bitácora completa, no modificable | |

---

## 5. Insumos que debe recopilar

| # | Insumo | Formato | Para | Prioridad |
|---|---|---|---|---|
| I-01 | **Lista inicial de usuarios**: nombre, cédula, correo institucional, cargo, dirección provincial, departamento, programa(s), perfil | Excel (una fila por persona) | Crear cuentas | **R1** |
| I-02 | **Catálogo 2027** validado: programas por departamento, actividades y códigos | Excel o marca de «sin cambios» | Cargar catálogos | **R1** |
| I-03 | Fecha límite de entrega del Plan 2027 y procedimiento de recepción | Texto | Configurar estados y avisos | **R1** |
| I-04 | **Modelos oficiales**: formato del Plan Anual impreso, «INFORMES TÉCNICOS.docx», «modelo de recomendaciones.docx», borrador de consolidado mensual/trimestral | Archivos originales | Impresiones idénticas | R1/R3 |
| I-05 | **Ejemplos reales** para pruebas: un plan 2026 de una provincia, 1–2 informes mensuales, 1 informe técnico con su respuesta | Papel escaneado o Excel | Pruebas de aceptación | R1/R2 |
| I-06 | Catálogo de cooperativas depurado (si Registro Cooperativo dispone de uno más actual) | Excel | Catálogo | R2 |
| I-07 | Logo institucional en alta resolución | PNG/SVG | Pantallas e impresiones | R1 |
| I-08 | Nombre de las personas para las **pruebas de aceptación** (1 de Planificación + 1–2 provincias piloto) | Lista | Validar R1 entre el 25 y 27-nov | **R1** |

---

## 6. Cómo entregar el resultado

1. Completar la columna «Respuesta» de §4 (en este documento o en una copia en Word/Excel).
2. Anotar en cada respuesta **quién decidió** cuando no sea la Oficina de Planificación.
3. Adjuntar los insumos de §5.
4. Registrar cualquier requisito **nuevo** que surja en las sesiones en la tabla de §8, con su prioridad propuesta. Los requisitos nuevos que no sean indispensables para el 1-dic se programan después de R1.
5. Enviar todo el **viernes 13 de noviembre**.

Con la entrega, el equipo de desarrollo actualiza la especificación de requisitos a su versión final y la somete al análisis final.

---

## 7. Criterio de «requerimiento completo»

- Bloques **A, B y C** con todas las preguntas respondidas o confirmadas.
- Insumos **I-01, I-02, I-03 e I-08** entregados.
- Bloques D–H: respondidos o con la respuesta propuesta **aceptada explícitamente** (pueden afinarse antes de su entrega).
- Firma de aprobación (§9).

---

## 8. Requisitos nuevos identificados en las sesiones

| # | Descripción | Quién lo pide | Prioridad propuesta (R1–R4 / posterior) |
|---|---|---|---|
| N-01 | | | |
| N-02 | | | |
| N-03 | | | |

---

## 9. Aprobación

| Rol | Nombre | Firma | Fecha |
|---|---|---|---|
| Líder funcional | | | |
| Jefe(a) de la Oficina de Planificación | | | |
| Responsable técnico | | | |

---

### Anexo — Recorrido de la maqueta por sesión

| Sesión | Pantallas de la maqueta a recorrer |
|---|---|
| Plan Anual | Portada → Direcciones Provinciales → **Plan Anual** (encabezado, tabla, ventana de datos específicos por mes, totales, firmas, imprimir) → Repositorio → pestaña Planes |
| Ejecución | **Informe Mensual** (fila, Programada, cooperativa, ventana de datos del resultado) → **Informe Técnico** → Repositorio → Seguimiento a Informes Técnicos → Responder |
| Indicadores | Repositorio → Consolidado Mensual y Trimestral → **Indicadores** (KPIs, matriz, alertas, gráficos) → Dirección de Fomento |
