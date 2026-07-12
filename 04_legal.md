# Marco Legal, Normativo y de Cumplimiento — Proyecto MIMF

> **Alcance:** Leyes, normas y proyectos estatales relevantes para la MIMF. Por cada norma: qué exige, quién la implementa en Chile y cómo la arquitectura cumple o se acopla. **Corte:** julio 2026.

---

## 1. Resumen ejecutivo

Chile pasó de décadas de digitalización fragmentada (SIDRA, Hospital Digital) a un **mandato legal explícito de interoperabilidad** con la [Ley N.º 21.668](https://www.minsal.cl/ley-de-interoperabilidad-de-fichas-clinicas-fue-publicada-en-el-diario-oficial/) (mayo 2024). Esa ley modifica la [Ley N.º 20.584](https://diprece.minsal.cl/wp-content/uploads/2025/09/Ley-20.584.pdf) (derechos y deberes del paciente) y obliga a prestadores públicos y privados a adoptar medidas de interoperabilidad, sujetas a un **reglamento del MINSAL** (plazo original: 18 meses desde la publicación).

En paralelo, el MINSAL define la **Arquitectura Nacional de Interoperabilidad** ([documentación pública](https://interoperabilidad.minsal.cl/)): modelo híbrido, **FHIR R4**, **EMPI/MPI**, **HPD**, **NID** y Servicios Terminológicos. La MIMF **no compite** con ese ecosistema: lo **consume** en identidad y terminología, y **complementa** con Sidecars, Record Locator clínico, RVN de urgencia y TPIM offline — piezas que la documentación pública aún no materializa como malla operativa nacional.

**Posicionamiento legal de la MIMF:**

| Pregunta | Respuesta |
| -------- | --------- |
| ¿Reemplaza la ley o el MINSAL? | **No.** Es infraestructura de cumplimiento y continuidad clínica alineada al estándar nacional. |
| ¿Centraliza toda la ficha clínica? | **No.** Respeta soberanía del dato (hospital origen = fuente de verdad del historial profundo). |
| ¿Inventa identidad paralela? | **No** como destino final. Usa `PatientIdentityProvider` → **EMPI/MPI** oficial; adaptador temporal solo en piloto. |
| ¿Cumple protección de datos? | **Sí por diseño:** ABAC, auditoría, cifrado, consentimiento TPIM, Break-Glass acotado, minimización en RVN. |
| ¿Es compatible con ciberseguridad sector salud? | **Sí:** federación reduce mega-repositorio; PKI, logs inmutables, Modo Degradado; alineable a ISO 27001 / NCh-ISO 27001:2022 |

---

## 2. Leyes y normas nacionales aplicables

### 2.1. Ley N.º 21.668 — Interoperabilidad de fichas clínicas (2024)

**Qué es:** Modifica la Ley 20.584 para establecer la **interoperabilidad obligatoria** de fichas clínicas entre prestadores de salud públicos y privados, con el fin de garantizar la **continuidad del cuidado del paciente** con independencia del prestador.

**Obligaciones relevantes (síntesis):**

* Los prestadores deben adoptar medidas que permitan la **interoperabilidad** con otros prestadores.
* Deben garantizar el **acceso oportuno** a la información de la ficha necesaria para la continuidad del cuidado, cuando la requiera un profesional que participe **directamente** en la atención del titular.
* Las fichas clínicas electrónicas y los sistemas que las soporten deben estar **diseñados para interoperar** con otros sistemas necesarios para el otorgamiento de acciones y prestaciones de salud.
* El MINSAL debe **actualizar el reglamento** del artículo 13 del Código Sanitario (vía Ley 20.584) para fijar estándares técnicos, administrativos, plazos y condiciones de almacenamiento, protección, eliminación e interoperabilidad.

**Estado (jul 2026):** Ley **vigente** desde mayo 2024. El estándar técnico de referencia del MINSAL es **HL7 FHIR R4** y el **Perfil Chile (CL Core)**; la exigibilidad detallada y plazos por tipo de prestador dependen del **reglamento** aún en elaboración/publicación.

**Atraso del reglamento (art. 13, Código Sanitario):** La Ley 21.668 encargó al MINSAL actualizar ese reglamento en un plazo de **18 meses** desde mayo 2024 (~**noviembre 2025**). Al corte de **julio 2026** el reglamento **sigue sin publicarse** — un atraso de ~8 meses sobre el plazo legal. Eso no invalida el mandato de interoperabilidad, pero confirma el patrón de gobernanza inestable descrito en `00_investigacion.md`: la ley avanza antes que la norma técnica operativa. Por eso la MIMF acopla al **contrato estable** (FHIR R4, `PatientIdentityProvider`, `HospitalConnector`) y usa **adaptador temporal** en identidad, en lugar de depender del calendario del reglamento para arrancar el piloto.

**Cómo cumple / se acopla la MIMF:**

| Obligación legal | Mecanismo MIMF |
| ---------------- | -------------- |
| Interoperabilidad entre prestadores | Sidecars perimetrales + `HospitalConnector` certificable; traducción a **FHIR R4 / Perfil Chile** en el borde estatal |
| Acceso oportuno en atención directa | Flujo de urgencia: identidad → Record Locator → **RVN < 3s** + *lazy loading* del historial profundo |
| Diseño para interoperar | Sidecar no toca código legacy; expone/consuma APIs estándar; Sandbox estatal para certificación |
| Estándares técnicos MINSAL | Coexistencia de versiones del Perfil Chile + EOL anunciado; REST/FHIR hacia EMPI, HPD y Terminológicos |
| Continuidad sin bloquear al rezagado | Período de transición; nodo no conforme **aislado**, red operativa para el resto |

**Referencia:** [MINSAL — Ley publicada en Diario Oficial](https://www.minsal.cl/ley-de-interoperabilidad-de-fichas-clinicas-fue-publicada-en-el-diario-oficial/), [texto Ley 21.668 (BCN)](https://nuevo.leychile.cl/servicios/Consulta/Exportar?exportar_formato=pdf&hddResultadoExportar=1203827.2024-05-28.0.0%23&nombrearchivo=Ley-21668_28-MAY-2024&radioExportar=Normas).

---

### 2.2. Ley N.º 20.584 — Derechos y deberes de los pacientes (2012, modificada 2024)

**Qué es:** Regula los derechos de las personas en relación con su atención en salud. Tras la Ley 21.668, refuerza el deber de **interoperabilidad**, **conservación** (mínimo **15 años**) y **acceso controlado** a la ficha clínica.

**Artículos clave para MIMF:**

* **Art. 12:** La ficha clínica es instrumento obligatorio; puede ser electrónica u otro soporte si se asegura acceso, conservación, confidencialidad y autenticidad.
* **Art. 13:** Prestadores responsables de cumplir Ley 19.628, adoptar medidas de interoperabilidad y dar acceso oportuno a profesionales **directamente vinculados** a la atención. **Terceros no vinculados** no acceden a la ficha.
* **Acceso del titular:** El paciente (o representante legal, o quien tenga poder simple ante notario) puede solicitar copia o acceso a su información.

**Cómo cumple / se acopla la MIMF:**

| Principio legal | Mecanismo MIMF |
| --------------- | -------------- |
| Acceso solo a quien atiende directamente | **ABAC** contextual (turno, admisión, relación activa con el paciente); deny-by-default |
| Emergencia con paciente inconsciente | **Break-Glass** con justificación, ventana temporal y **auditoría inmutable** |
| Confidencialidad / dato reservado | Cifrado TLS 1.3 en tránsito; cifrado en reposo en cachés; pseudonimización opcional del locator (HMAC + KMS) |
| Derecho del paciente sobre su información | **App Paciente / Autogestión** (Clave Única): actualización TPIM, configuración zona pública con **consentimiento explícito** |
| Conservación en hospital origen | Historial profundo permanece en el EHR del prestador; MIMF no centraliza petabytes |
| Profesional sin relación de atención | Sidecar **lee** atributos de agenda/RRHH; **no escribe** en sistemas ajenos |

**Referencia:** [Superintendencia de Salud — Ley de derechos y deberes](https://www.superdesalud.gob.cl/tax-materias-prestadores/ley-de-derechos-y-deberes-4185/), [Ley 20.584 (MINSAL/DIPRECE)](https://diprece.minsal.cl/wp-content/uploads/2025/09/Ley-20.584.pdf).

---

### 2.3. Protección de datos personales — Ley N.º 19.628 y Ley N.º 21.719

Chile tiene **dos capas** en el mismo cuerpo legal (19.628, reescrita por 21.719):

| Ley | Estado (corte jul 2026) | Rol |
| --- | ----------------------- | --- |
| **19.628** (1999) | **Vigente hoy** hasta el 30-nov-2026 | Régimen actual de tratamiento de datos personales |
| **21.719** (publicada 13-dic-2024) | **Vigencia plena el 1-dic-2026** | Nuevo marco integral; modifica sustancialmente la 19.628 |

**Referencia vigencia:** [Balance legislativo BCN — Ley 21.719](https://www.bcn.cl/balance-legislativo/detalle/ficha_LEY_21719_2024-12-13) (art. primero transitorio: mes vigésimo cuarto desde publicación → **1° diciembre de 2026**). Hasta esa fecha rige la 19.628 en lo que no choque con normas ya aplicables; desde el 1-dic-2026 entra el articulado completo, incluido régimen sancionatorio.

---

#### 2.3.1. Ley N.º 19.628 — Régimen vigente (transitorio)

**Qué es:** Regula el **tratamiento de datos personales**. Los **datos sensibles** (incluye estados de salud) tienen **prohibición general de tratamiento**, salvo autorización legal, consentimiento o necesidad para beneficios de salud (art. 10).

**Cómo cumple / se acopla la MIMF hoy:**

| Requisito | Mecanismo MIMF |
| --------- | -------------- |
| Minimización | **RVN** ultracrítico; Record Locator solo rutas |
| Seguridad | TLS 1.3, KMS, PKI, logs WORM, ABAC |
| Consentimiento (TPIM zona pública) | Voluntario; App Paciente / emisión institucional |
| Urgencia sin consentimiento | Break-Glass + Ley 20.584 (profesional en atención directa) |

**Referencia:** [Ley 19.628 (BCN)](https://www.bcn.cl/leychile/navegar?idNorma=141599).

---

#### 2.3.2. Ley N.º 21.719 — Protección de datos personales (vigencia plena 1-dic-2026)

**Qué es:** Publicada el [13 de diciembre de 2024](https://www.diariooficial.interior.gob.cl/publicaciones/2024/12/13/44023/01/2583630.pdf). **No deroga** la 19.628: la **reescribe** y crea la **Agencia de Protección de Datos Personales (APDP)** como autoridad fiscalizadora. Alinea el marco chileno a estándares internacionales (RGPD, LGPD).

**Elementos centrales relevantes para MIMF:**

| Tema | Qué establece | Implicancia MIMF |
| ---- | ------------- | ---------------- |
| **Vigencia** | Plena desde **1-dic-2026** | Piloto y diseño deben converger antes de esa fecha |
| **APDP** | Fiscalización, instrucciones, política de datos | Operador de la malla (Estado/concesión) sujeto a fiscalización |
| **Derechos del titular** | **ARCOP** ampliados + **Bloqueo** (art. 7° y ss.) | App Paciente / portal deben soportar acceso, rectificación, cancelación, oposición, portabilidad y bloqueo |
| **Datos sensibles de salud** | Art. **16 bis**: tratamiento solo para fines de leyes sanitarias; excepciones sin consentimiento acotadas | Break-Glass, urgencia, RVN y TPIM deben documentar base legal y finalidad |
| **Sin consentimiento (salud)** | Salvaguarda de vida/integridad; emergencia sanitaria; medicina preventiva/diagnóstico/prestación asistencial; etc. | Paramédico + Break-Glass encaja en letra a); deber de **informar al titular** post-impedimento |
| **Medidas de seguridad** | Obligaciones reforzadas al responsable del tratamiento | Cifrado, ABAC, auditoría, KMS, Modo Degradado sin filtrar datos |
| **Registro de operaciones** | Trazabilidad del tratamiento | Logs WORM, OpenTelemetry, auditoría Break-Glass |
| **Sanciones** | Multas hasta **20.000 UTM** | Refuerza costo de acceso indebido o filtración masiva |
| **Encargado / responsable** | Roles definidos (responsable vs. encargado del tratamiento) | Sidecar/hospital = responsable local del EHR; operador Hub/Índice/RVN = responsable o encargado según contrato estatal |

**Art. 16 bis — Datos de salud (texto útil para comisiones):**

Tratamiento de datos sensibles de salud **sin consentimiento** permitido, entre otros, cuando:

* a) Es indispensable para salvaguardar la **vida o integridad** del titular u otra persona, o el titular está impedido de consentir → **informar** al cesar el impedimento.
* b) Emergencia sanitaria legalmente decretada.
* e) Prestación de asistencia o tratamiento sanitario, gestión de sistemas de asistencia sanitaria.

**Cita verificada (D.O. 13-dic-2024):** Las letras **a)** y **e)** corresponden al **art. 16 bis** (no al art. 16 general). Fuente: [Ley 21.719 — publicación D.O.](https://www.diariooficial.interior.gob.cl/publicaciones/2024/12/13/44023/01/2583630.pdf).

→ La MIMF en urgencia (RVN, Break-Glass, App Primeros Respondedores) se apoya en **20.584** (acceso del profesional en atención) + **16 bis 21.719** (integridad vital) + obligación de **auditoría e información posterior**.

**Cómo cumple / se acopla la MIMF (diseño hacia dic-2026):**

| Obligación 21.719 | Mecanismo MIMF |
| ----------------- | -------------- |
| Minimización y finalidad | RVN acotado; locator sin contenido clínico; anti-objetivos explícitos |
| Consentimiento expreso (TPIM, zona pública) | App Paciente / Autogestión; registro de consentimiento |
| Derechos ARCOP + Bloqueo | Canal App Paciente + hospital origen; no bloquear derechos en Sidecar |
| Seguridad técnica y organizacional | TLS 1.3, PKI, ABAC, Sidecar por hospital, pseudonimización opcional del índice |
| Urgencia / Break-Glass | Protocolo documentado; logs inmutables; informe al titular cuando corresponda |
| Portabilidad / interoperabilidad | Alineación FHIR + Ley 21.668 (complementarias) |
| Notificación de incidentes | Alineable a Ley 21.663 (CSIRT) + futuras instrucciones APDP |
| Evitar tratamiento innecesario | ABAC deny-by-default; lazy loading del historial profundo |

**Lo que la MIMF debe cerrar antes del 1-dic-2026 (operativo, no solo arquitectura):**

1. Política de privacidad y registro de actividades de tratamiento (RAT) por componente (Sidecar, Índice, RVN, TPIM, apps).
2. Procedimiento de ejercicio de derechos ARCOP+Bloqueo vía App Paciente y mesón hospitalario.
3. Procedimiento post-Break-Glass: notificación al titular y registro en auditoría.
4. Designación de responsable/encargado del tratamiento en contratos de despliegue con Servicios de Salud y privados.
5. Evaluación de impacto en protección de datos (EIPD) para RVN central y TPIM, si la APDP lo exige por volumen/riesgo.

**Referencia:** [Ley 21.719 — BCN balance legislativo](https://www.bcn.cl/balance-legislativo/detalle/ficha_LEY_21719_2024-12-13), [publicación D.O. 13-dic-2024](https://www.diariooficial.interior.gob.cl/publicaciones/2024/12/13/44023/01/2583630.pdf).

---

### 2.4. Ley N.º 21.663 — Marco de ciberseguridad (2024)

**Qué es:** Establece obligaciones de **ciberseguridad** para el Estado y califica sectores como **servicios esenciales**. El **sector salud** está explícitamente en el ámbito de aplicación. Los operadores de importancia vital deben gestionar riesgos, continuidad operacional y **reportar incidentes significativos** al CSIRT Nacional.

**Nota OIV — dos niveles de certeza:** La Ley 21.663 obliga al **sector salud como servicio esencial** en general (gestión de riesgos, continuidad, reporte de incidentes). La **calificación específica** de qué nodos MIMF (Hub, Índice, RVN) entran al **registro de Operadores de Importancia Vital (OIV)** está **pendiente de reglamento** (ver §8, punto 4). La MIMF **no espera** esa calificación formal para diseñar: ya opera bajo el estándar más exigente (NCh-ISO 27001 / ISO 27799, ver `05_estandares.md` §2). Si luego un nodo queda calificado OIV, el diseño no requiere reescritura arquitectónica.

**Jerarquía normativa (no confundir con PS-NC-005):** La Ley 21.663 es **ley**; la [PS-NC-005](https://www.minsal.cl/wp-content/uploads/2025/11/POLITICA-GENERAL-DE-SEGURIDAD-DE-LA-INFORMACION-Y-CIBERSEGURIDAD-DEL-MINSAL-PS-NC-005-V.0.5.pdf) es **política administrativa del MINSAL** que adopta NCh-ISO 27001:2022 para el sector. Ambas convergen en SGSI, pero tienen fuentes distintas — en `05_estandares.md` se separan explícitamente.

**Incidente con efecto significativo:** Interrupción de servicio esencial, afectación a integridad física/salud de personas, o compromiso de sistemas con datos personales.

**Cómo cumple / se acopla la MIMF:**

| Obligación de ciberseguridad | Mecanismo MIMF |
| ---------------------------- | -------------- |
| Protección de activos críticos | Sidecar como perímetro; Hub gestionado; PKI mutua entre nodos |
| Disponibilidad del servicio clínico | **Modo Degradado** local: el hospital sigue operando aunque caiga la malla nacional |
| Integridad y no repudio | Firmas digitales en mensajes; logs append-only; timestamps |
| Gestión de incidentes | OpenTelemetry/tracing; procedimientos de aislamiento de nodo; alineable a instructivos MINSAL (ITS-NC-007, PS-NC-005) |
| Reducir superficie de ataque | Sidecar **compilado por hospital** (no monolito universal); sin mega-repositorio central de historiales |
| Continuidad operacional | Snapshot + replay controlado tras desconexiones prolongadas |

**Referencia:** [MINSAL — Seguridad de la información y ciberseguridad](https://www.minsal.cl/seguridad_de_la_informacion/), [Política PS-NC-005](https://www.minsal.cl/wp-content/uploads/2025/11/POLITICA-GENERAL-DE-SEGURIDAD-DE-LA-INFORMACION-Y-CIBERSEGURIDAD-DEL-MINSAL-PS-NC-005-V.0.5.pdf).

---

### 2.5. Otras normas y referencias transversales

| Norma / instrumento | Relevancia para MIMF |
| ------------------- | -------------------- |
| **Código Sanitario (art. 13, reglamento)** | Marco de almacenamiento, protección e interoperabilidad de fichas; reglamento pendiente de actualización post-21.668 |
| **Decreto / Norma técnica DEIS (EIS, ex-820)** | Estándares de información en salud; base del [Perfil CL Core / EIS FHIR](https://hl7chile.cl/fhir/ig/clcore/1.9.4/) |
| **Ley N.º 19.799** | Firma electrónica avanzada; aplicable a trazas, consentimientos y documentos clínicos electrónicos |
| **Ley N.º 20.285** | Transparencia y acceso a información pública; **no** abre fichas clínicas (siguen siendo reservadas) |
| **Constitución (art. 19 N°4)** | Respeto a la vida privada; informa diseño de minimización y ABAC |
| **Clave Única del Estado** | Autenticación de profesionales y pacientes en apps oficiales MIMF (App Paciente, integración prestador) |
| **Superintendencia de Salud** | Fiscalización de derechos del paciente; reclamos por acceso indebido o negativa de interoperabilidad |
| **ISP — dispositivos médicos / SaMD** | Registro y fiscalización de software y hardware con fines médicos; ver §2.6 |

---

### 2.6. ISP — Software como dispositivo médico (SaMD) y dispositivos médicos

**Qué es:** El **Instituto de Salud Pública (ISP)** fiscaliza en Chile el registro, comercialización y vigilancia de **dispositivos médicos**, incluido el **software como dispositivo médico (SaMD)** cuando el software cumple la definición regulatoria (destinado a fines de diagnóstico, monitoreo, tratamiento o prevención en salud, según normativa sanitaria vigente).

**Componentes MIMF con posible alcance regulatorio:**

| Componente | Riesgo regulatorio | Postura MIMF |
| ---------- | ------------------ | ------------ |
| **App de Primeros Respondedores** | Posible **SaMD** si se interpreta como ayuda a diagnóstico o tratamiento en urgencia pre-hospitalaria | Arquitectura condiciona IEC 62304 + ISO 14971 — ver `05_estandares.md` §4 |
| **TPIM** (zona privada con datos clínicos codificados) | Posible **dispositivo médico** si se comercializa o distribuye como producto sanitario | Chip físico + payload firmado; decisión ISP **pendiente** |
| **Sidecar, Hub, Índice, RVN** | Clasificación probable como **infraestructura / SIH**, no como dispositivo médico autónomo | Foco en ISO 27001/27799; no es el eje SaMD |

**Estado (jul 2026):** No existe **decisión ISP publicada** para los componentes MIMF. El cronograma de despliegue **no debe asumir** que la App de Primeros Respondedores o el TPIM quedan fuera del régimen sanitario: se diseña con trazabilidad requisitos→tests y gestión de riesgos clínicos para no bloquear una eventual clasificación.

**Mitigaciones de diseño (reducen presión regulatoria, no sustituyen consulta ISP):**

* RVN como **alerta de urgencia**, no prescripción autónoma ni decisión clínica automatizada.
* Break-Glass acotado, justificado y con auditoría inmutable.
* Zona pública del TPIM = texto consentido por el titular, no ficha clínica reservada completa.

**Pendiente:** Consulta o clasificación formal con ISP **antes** de comercialización o despliegue masivo de TPIM / App Respondedores como producto sanitario. Ver §8, punto 6.

**Referencia cruzada:** `05_estandares.md` §4.1 (IEC 62304, ISO 14971).

---

## 3. Ecosistema MINSAL y proyectos asociados

Fuente canónica: [interoperabilidad.minsal.cl](https://interoperabilidad.minsal.cl/). El portal es el esquema oficial de decisiones arquitecturales del MINSAL (en evolución permanente). Define la estrategia de interoperabilidad con el **paciente como eje central**, estándares comunes y una referencia para implementadores.

### 3.1. Arquitectura Nacional de Interoperabilidad

**Qué es:** Según la [especificación de arquitectura](https://interoperabilidad.minsal.cl/docs/especificacion-de-la-arquitectura/arquitectura.html), la estrategia se basa en:

1. Un **modelo de gobernanza transversal**
2. **Estándares de interoperabilidad** definidos
3. Un **modelo híbrido (centralizado-distribuido)** para el manejo de activos de información, permitiendo movilidad de datos al interior de la red asistencial

**Jerarquía de la red asistencial (modelo híbrido):**

```text
Ministerio de Salud
        ↕
   Macroredes (1 … n)
        ↕
  Servicios de Salud
        ↕
   Establecimientos
```

Cada nivel mantiene activos de información (bases/sistemas). El flujo es **bidireccional**: reporte y gobernanza hacia arriba; operación clínica y consumo de estándares hacia abajo. Esto es coherente con la tesis MIMF de soberanía local + capa nacional de descubrimiento/urgencia (no un mega-EHR único).

**Gobernanza oficial (quién decide):**

| Nivel | Quién | Rol |
| ----- | ----- | --- |
| **Ejecución** | Departamento de TICs del MINSAL + Departamentos TIC de los Servicios de Salud | Ejecutar lineamientos de interoperabilidad |
| **Estratégico** | Comisión directiva: Subsecretaría de Salud Pública, Subsecretaría de Redes Asistenciales, DEIS, DTIC, Salud Digital; presidida por el/la Ministro(a) o delegado | Decisiones de largo plazo, inversión, intersector |
| **Implementación clínica** | Comité: DTIC (técnico/sintáctico) + DEIS (semántica) + áreas técnicas del proceso clínico (organizacional) | Normas técnicas, IGs FHIR, control de adopción |

**Componentes oficiales y relación con MIMF:**

| Componente MINSAL | Función oficial | Acoplamiento MIMF |
| ----------------- | --------------- | ----------------- |
| [Modelo híbrido](https://interoperabilidad.minsal.cl/docs/especificacion-de-la-arquitectura/arquitectura.html) | Centralizado-distribuido por Macrored / Servicio / Establecimiento | Compatible: historial federado en establecimientos + RVN mínimo central |
| [EMPI / MPI](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/empi.html) | Identidad demográfica unívoca (RUN, IDs cruzados, merge) | **Destino** de `PatientIdentityProvider`; adaptador temporal en piloto |
| [HPD](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/hpd.html) | Directorio de prestadores | Atributos ABAC cuando esté operativo; no bloquea piloto |
| [Servicios Terminológicos](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/terminologicos.html) | CodeSystem, ValueSet, ConceptMap (IHE mSVCM) | Sidecars consumen catálogos oficiales (SNOMED CT / LOINC vía perfil) |
| [NID](https://interoperabilidad.minsal.cl/fhir/ig/nid/0.4.9/) | Guía FHIR transversal (MPI + HPD) | Contrato de integración en borde REST/FHIR R4 |
| **FHIR R4 + Perfil Chile** | Estándar sintáctico nacional | Obligatorio hacia el Estado; gRPC solo en malla interna MIMF |

**Principio rector compartido:** El **paciente como eje central** del intercambio — coherente con la tesis MIMF (continuidad clínica y urgencia).

**Cómo se acopla la MIMF a la jerarquía:**

| Nivel MINSAL | Rol en MIMF |
| ------------ | ----------- |
| **Establecimiento** | Sidecar compilado por hospital/EHR; fuente de verdad del historial profundo |
| **Servicio de Salud** | Contexto de despliegue piloto / gobernanza regional; Departamentos TIC ejecutan lineamientos |
| **Macrored** | Agrupación lógica para rollout y replicación del Índice/Hub |
| **Ministerio** | Estándares, EMPI/HPD/Terminología, Sandbox, PKI; RVN como activo central mínimo |

---

### 3.2. Guías de implementación FHIR oficiales (proyectos vivos)

Además de los componentes transversales (NID/MPI/HPD), el MINSAL publica **IGs FHIR** por dominio clínico. La MIMF **no reemplaza** estos proyectos: convive con ellos en el borde FHIR y puede enriquecer continuidad clínica cuando el paciente cruza establecimientos.

#### A) TEI — Tiempos de Espera Interoperable

* **IG:** [hl7.fhir.cl.minsal.tei v0.2.2](https://interoperabilidad.minsal.cl/fhir/ig/tei/0.2.2/index.html) (draft, 2026-04-30)
* **Alcance:** Solicitud de interconsulta (SIC) de **consulta nueva de especialidad No GES** desde APS (nivel primario) a nivel secundario.
* **Eventos del ciclo:** Iniciar → Referenciar → Revisar → Priorizar → Agendar → Atender → Terminar.
* **Impulsores:** DEIS, DIGERA, DIVAP, Unidad de Interoperabilidad (DTIC), con apoyo de CENS.
* **Dependencias:** FHIR R4 + [CL Core](https://hl7chile.cl/fhir/ig/clcore/).

**Relación con MIMF:**

| Aspecto TEI | Acoplamiento MIMF |
| ----------- | ----------------- |
| Intercambio SIC entre establecimientos | Sidecar / `HospitalConnector` puede emitir o consumir eventos FHIR TEI sin reemplazar la plataforma de Tiempos de Espera |
| Trazabilidad del paciente en la red | Record Locator responde *dónde hay datos clínicos*; TEI responde *estado del flujo de espera* — dominios distintos y complementarios |
| Identidad del paciente / prestador | TEI depende de perfiles Patient/Practitioner; MIMF resuelve identidad vía `PatientIdentityProvider` → EMPI |
| Continuidad clínica en la cita | Al llegar a “Atender”, el médico en especialidad puede usar RVN + historial federado (valor clínico que TEI no cubre por sí solo) |

#### B) SNRE — Sistema Nacional de Receta Electrónica

* **IG:** [hl7.fhir.cl.minsal.snre v0.9.6](https://interoperabilidad.minsal.cl/fhir/ig/snre/0.9.6/index.html) (draft / en evolución hacia v1.0)
* **Alcance:** Interoperabilidad de sistemas de **prescripción** y **dispensación** con el repositorio nacional de receta electrónica (público y privado).
* **Procesos:** Prescripción → Dispensación → Cambio de estado.
* **Flujo típico:** Validación de paciente/prescriptor → registro en repositorio SNRE → lectura por folio (farmacia / app paciente) → notificación de dispensación.
* **Colaboración:** MINSAL + HL7 Chile. Perfiles clave: Prescripción, Dispensación, Receta (agrupa prescripciones).

**Relación con MIMF:**

| Aspecto SNRE | Acoplamiento MIMF |
| ------------ | ----------------- |
| Medicamentos activos en urgencia | El **RVN** incluye medicamentos críticos; SNRE es la fuente de verdad de la receta formal — no se duplica el repositorio |
| Validación de paciente/prestador | Mismos maestros de identidad (EMPI/HPD) que consume MIMF |
| Dispensación en farmacia / app ciudadana | Canal paralelo a App Paciente MIMF (TPIM); no hay conflicto de roles si se delimita: SNRE = fármacos; App Paciente MIMF = chip + consentimiento zona pública |
| Sidecar | Puede coexistir: un hospital emite a SNRE por FHIR y, en paralelo, alimenta el RVN/locator vía Sidecar |

#### C) Otras IGs / núcleos del mismo portal

| Guía / componente | URL | Relación MIMF |
| ----------------- | --- | ------------- |
| **NID** (MPI + HPD) | [nid/0.4.9](https://interoperabilidad.minsal.cl/fhir/ig/nid/0.4.9/) | Contrato de identidad / prestadores |
| **MPI** | [fhir/ig/mpi](https://interoperabilidad.minsal.cl/fhir/ig/mpi/) | Backend definitivo de `PatientIdentityProvider` |
| **HPD** | [fhir/ig/hpd](https://interoperabilidad.minsal.cl/fhir/ig/hpd/) | Atributos ABAC de prestador |
| **CL Core** (HL7 Chile) | [clcore](https://hl7chile.cl/fhir/ig/clcore/1.9.4/) | Perfil base nacional que heredan TEI, SNRE y Sidecars |

**Regla de alcance:** TEI y SNRE son **procesos clínicos verticales** (espera / farmacia). La MIMF es **capa horizontal de malla** (identidad, descubrimiento, urgencia, offline). No se pisan; se conectan por FHIR R4 e identidad compartida.

---

### 3.3. Proyectos e iniciativas históricas

Estos proyectos **no son ley**, pero explican el contexto institucional y las deudas que la Ley 21.668 y la MIMF intentan cerrar.

| Proyecto / actor | Qué fue / es | Lección legal-operativa para MIMF |
| ---------------- | ------------ | --------------------------------- |
| **SIDRA (2008–2017)** | Digitalización masiva con EHR heterogéneos por Servicio de Salud | Sin estándar obligatorio + gobernanza inestable = islas digitales; la 21.668 corrige el vacío normativo |
| **Hospital Digital (2018–2022)** | Telemedicina y conectividad; no resolvió interoperabilidad de fichas presenciales | Un proyecto sectorial no sustituye **mandato legal** ni arquitectura de identidad + estándar |
| **RCE MINSAL** | Registro Clínico Electrónico en red pública | Base instalada de EHR donde el Sidecar puede acoplarse sin reemplazar |
| **TEI** (en curso) | Plataforma interoperable de tiempos de espera APS → especialidad No GES | Primer gran caso de uso vertical FHIR; la malla MIMF aporta continuidad clínica en el evento “Atender” |
| **SNRE** (en curso) | Receta electrónica nacional (prescripción / dispensación) | Precedente de repositorio FHIR nacional; Sidecars deben convivir, no competir |
| **HL7 Chile / CL Core** | Perfil FHIR nacional | Referencia técnica para certificación Sandbox y `HospitalConnector` |
| **CENS** | Centro Nacional en Sistemas de Información en Salud | Validación de guías, Connectathons; alineación de implementación |
| **FHIR Connectathon 2026** | Prueba de conformidad | Mecanismo de maduración previo a exigencia reglamentaria plena |

---

### 3.4. Estándares técnicos con respaldo normativo indirecto

| Estándar | Rol | Cumplimiento MIMF |
| -------- | --- | ----------------- |
| **HL7 FHIR R4** | Sintaxis de intercambio exigida / referenciada por MINSAL y Ley 21.668 | Sidecars publican/consumen recursos según Perfil Chile (incl. perfiles TEI/SNRE cuando aplique) |
| **SNOMED CT / LOINC** | Semántica clínica | Mapeo en ETL local; catálogos vía Servicios Terminológicos |
| **IHE PIXm / PDQm** | Identidad paciente (vía IG MPI) | Contrato `PatientIdentityProvider` alineado a operaciones EMPI |
| **IHE mSVCM** | Servicios terminológicos | Consumo de ValueSet/CodeSystem nacionales |
| **IPS (International Patient Summary)** | Referencia clínica internacional de resumen vital | RVN diseñado como alerta ultracrítica compatible en espíritu, acotado en alcance |

---

## 4. Matriz de cumplimiento por componente MIMF

| Componente MIMF | Ley 21.668 / 20.584 | Ley 19.628 / 21.719 | Ley 21.663 | Arquitectura MINSAL |
| ----------------- | ------------------- | ------------------- | ---------- | ------------------- |
| **Sidecar + `HospitalConnector`** | Traduce a FHIR; habilita interoperabilidad sin reemplazar EHR | Minimiza extracción; acceso solo vía políticas ABAC | Perímetro endurecido; binario por hospital | Borde FHIR R4; no reimplementa NID |
| **`PatientIdentityProvider` → EMPI** | Identificación unívoca para continuidad | Tratamiento con base legal de prestación de salud | Protección de identificadores | Consume MPI; adaptador temporal en piloto |
| **Record Locator** | Descubrimiento oportuno de fichas en otros nodos | Solo metadatos de enrutamiento; pseudonimización opcional | Activo crítico con replicación y auditoría | Complemento (EMPI no responde “dónde está el historial”) |
| **RVN** | Acceso inmediato a datos críticos en urgencia | Minimización; comité clínico-técnico (MINSAL) fija alcance | Repositorio acotado, no petabytes | Compatible con modelo híbrido estatal |
| **TPIM (NFC)** | Continuidad **pre-hospitalaria** (gap offline) | Zona pública con consentimiento; zona privada firmada y acotada | Dispositivo físico; verificación PKI anti-clon | No sustituye EMPI ni RVN; último recurso offline |
| **App Primeros Respondedores** | Acceso profesional en emergencia | ABAC + Break-Glass; art. 16 bis 21.719 (integridad vital) | App hardening; sync de auditoría offline | Integración SAMU / prestadores vía HPD (futuro) |
| **App Paciente / Autogestión** | Derechos del titular sobre su información | Consentimiento explícito zona pública; Clave Única | Autenticación federada estatal | Canal ciudadano complementario al portal MINSAL |
| **Sandbox + certificación** | Habilita cumplimiento verificable de Perfil Chile | — | Evaluación de seguridad previa a producción | Mecanismo de adopción para proveedores EHR |
| **Modo Degradado + snapshot/replay** | Continuidad asistencial aunque falle la red | — | Continuidad operacional / resiliencia | No contradice arquitectura híbrida |

---

## 5. Gobernanza, adopción e incentivos (marco legal-operativo)

La tecnología sola no cumple la ley; hace falta **gobernanza** explícita. La MIMF se alinea a la gobernanza oficial del portal MINSAL y propone mecanismos operativos anclados a la Ley 21.668.

### 5.0. Gobernanza oficial MINSAL (referencia)

Según la [arquitectura nacional](https://interoperabilidad.minsal.cl/docs/especificacion-de-la-arquitectura/arquitectura.html):

* **Ejecución:** DTIC MINSAL + TIC de Servicios de Salud.
* **Estratégico:** comisión directiva (Subsecretarías, DEIS, DTIC, Salud Digital; presidida por Ministro/a o delegado).
* **Implementación de procesos:** comité DTIC (sintaxis) + DEIS (semántica) + área clínica del proceso (organización) → produce normas técnicas e IGs (TEI, SNRE, NID, etc.).

La MIMF no crea una gobernanza paralela: se inserta como **capa de malla** bajo esos lineamientos y propone un comité clínico-técnico del RVN con voto dirimente del Departamento de Calidad y Seguridad del Paciente.

### 5.1. Certificación y mercado público

* **Sandbox estatal obligatorio** para proveedores que venden EHR al Estado.
* Cumplir **Perfil Chile FHIR** (y perfiles de dominio TEI/SNRE cuando el proceso lo exija) como condición de contratación pública (anclaje a Ley 21.668).
* El proveedor puede implementar su propio **`HospitalConnector`** certificado — distribuye responsabilidad legal-técnica del mantenimiento cuando cambia el esquema interno.

### 5.2. Transición y no bloqueo

* **Período de transición** explícito: ningún hospital queda sin sistema durante migración de EHR o versión de perfil.
* **Aislamiento del nodo no conforme** en lugar de apagar la red nacional.
* **Coexistencia de versiones FHIR** con ventanas de End-of-Life anunciadas (ej. 6 meses).

### 5.3. Incentivos y sanciones (referencia legal)

* La Ley 21.668 habilita al reglamento a fijar plazos y condiciones; la estrategia MIMF contempla **retención de fondos** a prestadores que no integren en plazo (mecanismo de presión presupuestaria ya usado en políticas MINSAL).
* Reclamos de pacientes por acceso indebido o negativa de entrega de información: vía prestador y **Superintendencia de Salud**.

### 5.4. Comité clínico del RVN

* Evolución del esquema del RVN bajo comité clínico-técnico; voto dirimente **Departamento de Calidad y Seguridad del Paciente (MINSAL)**.
* Defensa legal contra *scope creep*: el RVN es **alerta de urgencia**, no historia clínica nacional duplicada.

---

## 6. Flujos clínicos vs. obligaciones legales

### 6.1. Urgencia hospitalaria

```text
Ingreso → Identidad (PatientIdentityProvider / EMPI)
       → Descubrimiento (Record Locator)
       → RVN inmediato (continuidad + acceso oportuno, art. 13 Ley 20.584)
       → ABAC valida relación profesional-paciente
       → Historial profundo bajo demanda (minimización)
```

**Base legal del acceso:** Profesional que participa **directamente** en la atención (20.584) + necesidad de continuidad del cuidado (21.668).

### 6.2. Emergencia pre-hospitalaria (TPIM)

| Actor | Qué ve | Base legal / ética |
| ----- | ------ | ------------------ |
| Transeúnte | Zona pública NDEF (texto consentido) | Consentimiento informado del titular; no es ficha clínica reservada completa |
| Paramédico en turno | Zona privada (RVN ultracrítico) | Prestación de emergencia; ABAC + registro de acceso |
| Paramédico fuera de turno | Zona privada solo vía Break-Glass | Art. 16 bis Ley 21.719 (salvaguarda integridad); auditoría obligatoria |
| Sin TPIM / sin identidad | Protocolo trauma estándar | MIMF **no condiciona** atención a tenencia de chip o RUT |

---

## 7. Riesgos legales y mitigaciones explícitas

| Riesgo | Mitigación MIMF |
| ------ | --------------- |
| Tratar la MIMF como “base de datos nacional” de fichas | Anti-objetivo explícito; historial en hospital origen |
| UUID o maestro de identidad paralelo | Descartado; EMPI como destino; adaptador temporal acotado al piloto |
| Acceso indiscriminado por rol (“Médico ve todo”) | ABAC contextual, no RBAC estático |
| Abuso de Break-Glass | WORM logs, alertas, revisión comité auditoría |
| Chip desactualizado engaña en urgencia | Semáforo de frescura; RVN en red como fuente primaria; TPIM = offline último recurso |
| Incumplimiento de proveedor bloquea país | Aislamiento de nodo; transición; Sandbox |
| Reglamento 21.668 publicado con perfiles distintos | Coexistencia de versiones; Sidecar OTA; contrato de esquema |
| Reglamento 21.668 atrasado respecto al plazo legal | Adaptador temporal de identidad; acoplamiento a contrato FHIR/EMPI, no al calendario del MINSAL (ver §2.1) |
| ISP clasifica TPIM o App Respondedores como dispositivo médico / SaMD | Diseño condicional IEC 62304/14971 ya documentado; consulta ISP antes de rollout masivo (ver §2.6) |
| Brecha digital (prestadores pequeños) | Sidecar liviano; piloto regional; subsidio / acompañamiento (política pública, no solo técnica) |

---

## 8. Pendientes normativos (jul 2026)

Estos puntos **no están resueltos en ley publicada al corte de este documento** y deben monitorearse:

1. **Reglamento del art. 13 (Código Sanitario / Ley 20.584)** actualizado post-21.668: estándares definitivos, plazos por tipo de prestador, certificación. Plazo legal original: 18 meses desde mayo 2024 (~nov 2025); al corte jul 2026 **sigue sin publicarse** (~8 meses de atraso).
2. **Madurez operativa EMPI/HPD** en terreno (IGs NID en evolución): la MIMF mitiga con adaptador temporal, no elimina la necesidad de converger.
3. **Ley 21.719 (vigencia plena 1-dic-2026):** reglamentos e instrucciones APDP; procedimientos operativos MIMF (RAT, ARCOP+Bloqueo, post-Break-Glass, EIPD RVN/TPIM). Hasta el 30-nov-2026 sigue aplicándose la 19.628 en lo vigente.
4. **Calificación de operadores de importancia vital** en salud bajo Ley 21.663: definir si nodos MIMF (Hub, Índice, RVN) quedan en registro OIV y obligaciones CSIRT exactas. La ley ya obliga al sector como servicio esencial; la MIMF diseña bajo NCh-ISO 27001/27799 **sin esperar** la calificación formal de cada nodo (ver §2.4).
5. **Portal / App ciudadana MINSAL** vs. App Paciente MIMF: delimitar funciones para no duplicar canales oficiales (MIMF como complemento de autogestión TPIM + consulta RVN, sujeto a acuerdo institucional).
6. **Clasificación ISP (SaMD / dispositivo médico):** determinar si la App de Primeros Respondedores y/o el TPIM requieren registro sanitario antes de despliegue masivo (ver §2.6).

---

## 9. Conclusión

La MIMF se inserta en un momento en que Chile **ya tiene mandato legal** de interoperabilidad (Ley 21.668), **marco de derechos del paciente** exigente (Ley 20.584), **protección reforzada de datos de salud** (Ley 19.628 / 21.719) y **ciberseguridad sectorial** (Ley 21.663), mientras el MINSAL materializa la **Arquitectura Nacional** (FHIR R4, EMPI, HPD, NID).

La propuesta **no sustituye** ese marco: lo **operacionaliza** con Sidecars no intrusivos, identidad acoplada al EMPI, descubrimiento clínico federado, resumen vital de urgencia y token físico offline. El diseño prioriza:

* **Cumplimiento** (interoperabilidad, acceso oportuno, privacidad, seguridad).
* **Soberanía del dato** (legalmente defendible ante centralización excesiva).
* **Realismo de despliegue** (adaptador temporal, transición, nodos aislados).
* **Complementariedad** con el Estado, no duplicación de EMPI/HPD/terminología.

---

## 10. Referencias y enlaces oficiales

| Recurso | URL |
| ------- | --- |
| Portal Interoperabilidad MINSAL | https://interoperabilidad.minsal.cl/ |
| Arquitectura nacional | https://interoperabilidad.minsal.cl/docs/especificacion-de-la-arquitectura/arquitectura.html |
| EMPI / MPI | https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/empi.html |
| HPD | https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/hpd.html |
| Servicios Terminológicos | https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/terminologicos.html |
| NID (IG FHIR) | https://interoperabilidad.minsal.cl/fhir/ig/nid/0.4.9/ |
| TEI — Tiempos de Espera Interoperable | https://interoperabilidad.minsal.cl/fhir/ig/tei/0.2.2/index.html |
| SNRE — Receta Electrónica | https://interoperabilidad.minsal.cl/fhir/ig/snre/0.9.6/index.html |
| Perfil CL Core (HL7 Chile) | https://hl7chile.cl/fhir/ig/clcore/1.9.4/ |
| Ley 21.668 — MINSAL | https://www.minsal.cl/ley-de-interoperabilidad-de-fichas-clinicas-fue-publicada-en-el-diario-oficial/ |
| Ley 20.584 — DIPRECE/MINSAL | https://diprece.minsal.cl/wp-content/uploads/2025/09/Ley-20.584.pdf |
| Ley 19.628 — BCN | https://www.bcn.cl/leychile/navegar?idNorma=141599 |
| Ley 21.719 — BCN (vigencia 1-dic-2026) | https://www.bcn.cl/balance-legislativo/detalle/ficha_LEY_21719_2024-12-13 |
| Ley 21.719 — D.O. 13-dic-2024 | https://www.diariooficial.interior.gob.cl/publicaciones/2024/12/13/44023/01/2583630.pdf |
| Ciberseguridad MINSAL | https://www.minsal.cl/seguridad_de_la_informacion/ |
| Superintendencia — Derechos paciente | https://www.superdesalud.gob.cl/tax-materias-prestadores/ley-de-derechos-y-deberes-4185/ |
| Estrategia digitalización MINSAL (CEPAL) | https://www.cepal.org/sites/default/files/news/files/estrategia-de-digitalizacion-e-interoperabilidad-en-salud-en-chile-minsal.pdf |
