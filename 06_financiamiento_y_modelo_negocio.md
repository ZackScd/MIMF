# Financiamiento y Modelo de Negocio — Proyecto MIMF

> **Alcance:** Modelo económico del proyecto MIMF. Cubre financiamiento, monetización y estructura de salida de los responsables del proyecto. **Corte:** julio 2026. Información de financiamiento y marco legal verificada mediante fuentes públicas vigentes a esta fecha (ver enlaces por sección).

---

## 0. Nota de alcance sobre el "Informe de Evaluación de Proyectos" (ATEP01, 16-jun-2026): Obsoleto/Rechazado.

El informe de Evaluación de Proyectos entregado para el ramo ATEP01 evaluó un el producto **VitalTag**: un servicio de identificación clínica de emergencia de adhesión voluntaria (opt-in), desacoplado de la arquitectura hospitalaria (sin Sidecar, sin RVN, sin EMPI, sin ABAC), monetizado mediante venta de etiquetas NFC, suscripción individual "premium" y convenios B2B.

Se trató de un ejercicio de evaluación financiera para el ramo, no una decisión de arquitectura del proyecto. **Este documento reemplaza esa dirección.** El modelo de negocio de MIMF se construye sobre la arquitectura ya definida en `01`–`05` (Sidecar, RVN, TPIM, EMPI/`PatientIdentityProvider`, ABAC), y sostiene el anti-objetivo ya establecido en `01_proyecto.md`:

> *"El sistema no condiciona la atención a la tenencia del chip ni del RUT."*

Ningún escenario descrito abajo contempla cobrar al paciente ni al profesional de salud por acceso a información clínica propia o de emergencia.

### 0.1. Modelo VitalTag (informe ATEP01) — desglose de las tres fuentes y traslado a MIMF

El informe de evaluación financiera del ramo proyectó **tres fuentes de ingreso** a cinco años para el producto **VitalTag** (etiqueta NFC opt-in, sin arquitectura hospitalaria). Las cifras del informe y su compatibilidad con MIMF:

| # | Fuente VitalTag (informe) | Cifras / alcance en el informe | ¿Compatible con MIMF? | Traslado en este documento |
| - | ------------------------- | ------------------------------ | --------------------- | -------------------------- |
| 1 | **Venta de etiquetas NFC** | Ingreso por unidad vendida al ciudadano; precio promedio **$5.990**, costo unitario **$1.900** (margen retail). | **No** como modelo principal. El TPIM MIMF no es una etiqueta de consumo desacoplada del RVN: se emite/actualiza en contacto clínico (CESFAM, Sidecar), con payload firmado por el operador de la malla. Cobrar al titular por el dispositivo que porta datos vitales **condiciona la participación** y replica el riesgo de “quien no paga, queda fuera” — contrario al anti-objetivo de `01_proyecto.md`. | Emisión institucional del TPIM (costo absorbido por prestador o programa público en piloto). Si hubiera costo de hardware, que lo pague la **institución** (paquete B2B), no el paciente en retail. |
| 2 | **Suscripción Premium (SaaS B2C)** | Plan anual **$19.900**: perfiles familiares, almacenamiento de documentos médicos y soporte prioritario sobre capa básica gratuita. | **No** sobre funciones clínicas. “Almacenamiento de documentos médicos” y acceso ampliado a información de salud del titular chocan con **entrega gratuita** de la ficha (Ley 20.584 art. 13) y con **acceso/portabilidad** gratuitos al menos trimestralmente (Ley 21.719 arts. 9–10). Un freemium donde lo de pago es “más salud” recrea pacientes de primera y segunda categoría. | **Excluido** — ver §2.1.1. Lo único trasladable del concepto “premium” es **soporte prioritario institucional** (SLA B2B al hospital), no al ciudadano. |
| 3 | **Convenios institucionales (B2B)** | Paquetes para clínicas, colegios, municipios y empresas que adoptan el sistema para sus comunidades. | **Sí** — es el núcleo del **Escenario B** de este documento: el pagador es la organización, no cada persona. Alineado a Sidecar por hospital, certificación de conectores, malla gestionada para Isapres/redes y convenios con Servicios de Salud. | Tabla §2.2 (Nodo Sidecar, certificación, malla privada, SLA). Esta es la vía para “sacarle plata a las empresas” sin cobrar al paciente. |

**Síntesis:** del modelo VitalTag del informe, MIMF rescata **solo el tercer pilar (B2B)** y lo ancla a la arquitectura real (`01`–`05`). Los pilares 1 (retail NFC) y 2 (premium ciudadano) **NO** se trasladan al TPIM/RVN/App Paciente de MIMF sin violar diseño y marco legal (§2.1.1). El Escenario A (Estado) sigue siendo la apuesta principal; el B institucional es el respaldo comercial compatible.

**En consideración:** Desarrollo de un plan híbrido; Piloto como empresa privada y posterior licitación para compra estatal. 

---

## 1. Escenario A — Financiamiento / Adquisición Estatal (principal)

### 1.1. Tesis

MIMF se posiciona como infraestructura nacional, no como producto — análogo a cómo Estonia trata X-Road, operado por la agencia estatal RIA y no por una empresa que cobra al ciudadano por usarlo (`00_investigacion.md` §3.B). El comprador o financista objetivo es el Estado, en cumplimiento del mandato de la Ley 21.668, no un cliente individual.

### 1.2. Estado normativo actual (jul 2026)

La Ley N.º 21.668, que modifica el art. 13 de la Ley 20.584 para establecer la interoperabilidad de las fichas clínicas, fue publicada el 28 de mayo de 2024 ([BCN — Ley Chile](https://www.bcn.cl/leychile/navegar?idNorma=1203827)). Su artículo tercero transitorio fija un plazo de 18 meses desde la entrada en vigencia para que el MINSAL actualice el reglamento del art. 13 (~noviembre 2025). A la fecha de este documento, múltiples fuentes jurídicas y sectoriales consultadas (mayo–junio 2026) siguen reportando el reglamento **sin publicar** ([EstadoDiario, 2024](https://estadodiario.com/columnas/interoperabilidad-de-las-fichas-clinicas-y-ciberseguridad/); [Reservo](https://reservo.cl/blog/profesionales/salud/conoce-lo-que-cambia-con-la-ley-sobre-interoperabilidad-de-fichas-clinicas/)). Esto confirma el atraso normativo ya documentado en `04_legal.md` §2.1 y refuerza la tesis de gobernanza inestable de `00_investigacion.md` §2.

**Implicancia práctica:** no existe aún un proceso de licitación pública derivado directamente de la Ley 21.668 al que MIMF pueda postular. La vía de entrada realista en el corto plazo es el financiamiento de innovación aplicada, no la compra pública directa.

### 1.3. Vías de financiamiento concretas, por etapa


| Etapa                     | Vía                                                                                                               | Descripción y vigencia                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Piloto / prototipo**    | **FONIS** (Fondo Nacional de Investigación y Desarrollo en Salud, ANID + MINSAL)                                  | Convocatoria anual conjunta ANID–MINSAL para I+D aplicada en salud con resultados transferibles a producto/servicio. Convocatoria 2026 corrió entre el 28-ene y el 28-abr-2026 ([ANID — Fonis 2026](https://anid.cl/concursos/proyectos-de-investigacion-y-desarrollo-en-salud-fonis-2026/)); se repite anualmente — verificar apertura 2027 en anid.cl. Es la vía más alineada temáticamente, al estar copatrocinada por el propio MINSAL. |
| **Piloto / prototipo**    | **ANID — Concurso IDeA I+D**                                                                                      | Cofinancia I+D aplicada con probabilidad razonable de convertirse en producto/servicio comercializable; incluye línea "Salud" y modalidad "de interés público" (resultados de dominio público). Convocatoria 2026 con resultados en junio 2026 ([ANID — IDeA I+D 2026](https://anid.cl/concursos/concurso-idea-id-2026/)); ciclo anual.                                                                                                     |
| **Piloto / prototipo**    | **CORFO** — subsidios de innovación (líneas rotativas: prototipos, validación, escalamiento)                      | CORFO mantiene convocatorias activas todo el año, con líneas que varían por ciclo (ej. subsidios semilla, validación de prototipo). No hay una línea fija identificada específica para salud digital a la fecha de esta investigación; se debe revisar el calendario vigente en corfo.cl al momento de postular.                                                                                                                            |
| **Piloto / prototipo**    | **FIC-R** (Fondo de Innovación para la Competitividad, asignación regional) vía Gobierno Regional de La Araucanía | Financia I+D con foco regional; vía de entrada natural dado que el piloto de MIMF está planteado como piloto regional (`01_proyecto.md`, Despliegue por Fases).                                                                                                                                                                                                                                                                             |
| **Piloto institucional**  | Convenio de colaboración académica INACAP – Servicio de Salud regional                                            | Vía de entrada típica para instalar un piloto de tesis en un hospital público real, sin pasar aún por licitación pública completa.                                                                                                                                                                                                                                                                                                          |
| **Escalamiento nacional** | **Licitación pública (Ley 19.886 / ChileCompra)**                                                                 | Aplicable una vez el reglamento del art. 13 esté publicado y el MINSAL defina el modelo de adopción (operador de malla, certificación de conectores, etc.). No existe aún convocatoria de este tipo — es una vía a futuro, condicionada al §1.2.                                                                                                                                                                                            |
| **Escalamiento nacional** | Crédito de banca multilateral (BID, Banco Mundial) gestionado por MINSAL                                          | Mecanismo histórico de financiamiento de programas de salud digital en Chile; no se gestiona directamente por el equipo del proyecto, pero es la fuente de fondos habitual detrás de licitaciones estatales de gran escala.                                                                                                                                                                                                                 |


### 1.4. Estructura de salida para el equipo: Build → Operate → Transfer (BOT)

1. **Build:** desarrollo y operación del piloto regional con fondos de innovación (fila superior de la tabla).
2. **Operate (transición):** si el Estado decide adoptar la arquitectura, se negocia un contrato de **transferencia de propiedad intelectual + soporte de transición** (típicamente 6–18 meses de acompañamiento técnico).
3. **Transfer:** el Estado, o una entidad ad-hoc (fundación u organismo técnico, análogo a NIC Chile para el dominio `.cl`, o a la RIA estonia para X-Road), pasa a operar la malla. El equipo cobra por la venta de IP y el contrato de transición.

### 1.5. Riesgos de este escenario

- Reglamento del art. 13 sin publicar a la fecha de este documento (~8 meses de atraso respecto del plazo legal).
- Gobernanza inestable: ciclos políticos de 4 años pueden reiniciar prioridades antes de que exista un proceso de licitación real (`00_investigacion.md` §2.1).
- Proveedores incumbentes (Saydex, InterSystems) tienen incentivo a resistir estándares abiertos que erosionan su vendor lock-in.

---

## 2. Escenario B — SaaS Institucional (respaldo - mayor probabilidad de éxito)

### 2.1. Principio de diseño

El sujeto de pago es la institución (hospital, clínica, Isapre, proveedor de EHR), nunca el paciente ni el profesional de salud individual. Esto no es una restricción adicional: es el mismo anti-objetivo ya definido en `01_proyecto.md` aplicado al modelo de ingresos.

**Posición del equipo (jul 2026):** el Escenario B es viable como **SaaS B2B** — cobrar a quien opera la infraestructura (Sidecar, certificación, hosting, SLA) — pero **no** como freemium o suscripción *premium* al ciudadano sobre funciones clínicas de acceso a su propia información o continuidad de urgencia (véase desglose VitalTag §0.1: retail NFC ~$5.990 y plan ~$19.900/año). Esa variante B2C choca con el marco legal chileno y con el diseño MIMF; el monetizado institucional sigue siendo el camino del Escenario B sin molestar al paciente ni al personal de salud.

### 2.1.1. Nota legal — por qué no cobrar al paciente (ni plan premium ciudadano)

> **Aviso:** síntesis de normativa pública para sustentar decisiones de diseño y negocio. No reemplaza asesoría legal. Desarrollo normativo ampliado en `04_legal.md`, fecha de corte: julio 2026.

Un modelo *freemium* o *premium* B2C — capa gratuita limitada + pago del titular por “más datos”, actualización prioritaria del TPIM, RVN ampliado o funciones de la App Paciente — no es solo una preferencia de producto: **entra en tensión directa** con obligaciones que ya pesan sobre prestadores y, bajo la vigencia de la Ley 21.719, sobre cualquier responsable de tratamiento de datos de salud.


| Fundamento                               | Qué dice la norma                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Por qué importa para MIMF (Escenario B)                                                                                                                                                                                                                                     |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Acceso gratuito a la ficha (titular)** | Ley 20.584, **art. 13** (inc. 5°, mod. Ley 21.541): el titular puede requerir la **entrega gratuita y sin dilaciones indebidas** de una copia íntegra de su ficha, en formato estructurado, de uso común y lectura legible, **susceptible de ser portado** a otro sistema o prestador.                                                                                                                                                                                                                                                             | La App Paciente / autogestión TPIM es un canal de ejercicio de ese derecho, no un producto de consumo cobrable por acceder a lo que la ley ya garantiza gratis.                                                                                                             |
| **Continuidad e interoperabilidad**      | Ley 21.668 (mod. art. 13 Ley 20.584): los prestadores deben adoptar medidas de **interoperabilidad** y garantizar **acceso oportuno** a la información necesaria para la **continuidad del cuidado** cuando la requiera un profesional que participe **directamente** en la atención.                                                                                                                                                                                                                                                              | El RVN y la malla existen para cumplir continuidad clínica — obligación del **prestador**, financiada en el modelo B por la institución, no por el paciente en urgencia. Un paywall ciudadano recrea “pacientes de primera y segunda categoría” (`00_investigacion.md` §4). |
| **Portabilidad digital**                 | Misma línea del art. 13: si la información se entrega a otro prestador, basta con habilitar acceso remoto para extraer lo necesario para continuidad. La norma técnica MINSAL de telemedicina reitera la **entrega gratuita** y la portabilidad (Norma técnica prestaciones a distancia y telemedicina, 2025, citando arts. 12 y 13 Ley 20.584).                                                                                                                                                                                                   | Cobrar por “exportar” o sincronizar el propio resumen vital equivale a condicionar económicamente un derecho que la ley fija como gratuito.                                                                                                                                 |
| **Derechos ARCOP + portabilidad (2026)** | Ley 21.719, **arts. 4, 9 y 10** (vigencia plena **1-dic-2026**): derechos de acceso, rectificación, supresión, oposición, **portabilidad** y bloqueo. Art. 10: rectificación, supresión y oposición **siempre gratuitos**; acceso y portabilidad **gratuitos al menos una vez por trimestre** (costos directos solo si el titular ejerce más de una vez en el mismo trimestre, con límites legales). Art. 35: **infracción** impedir u obstaculizar el ejercicio legítimo de esos derechos (multas hasta 20.000 UTM, agravantes por reincidencia). | Si MIMF opera como responsable/encargado del tratamiento del RVN o la App Paciente, un muro de pago sobre acceso/portabilidad a datos de salud es un vector de fiscalización APDP, no un “upgrade” de producto.                                                             |
| **Fiscalización Superintendencia**       | Ley 20.584, **art. 38**: la Superintendencia de Salud controla el cumplimiento de los derechos del paciente; ante incumplimiento instruye corrección y puede sancionar. Precedente público: negativa o entrega deficiente de ficha clínica ha derivado en multas (ej. Clínica Las Condes, ~100 UF, confirmada por Corte Suprema, 2025, por no entregar ficha en formato solicitado).                                                                                                                                                               | Un modelo que monetiza el acceso del titular a su información clínica se parece al patrón que la Superintendencia ya sanciona: poner barreras al ejercicio del derecho de acceso.                                                                                           |
| **Profesional de salud**                 | Ley 21.668 / 20.584: el **acceso oportuno** en atención directa es deber del prestador hacia el profesional habilitado, no del paciente hacia una app de terceros.                                                                                                                                                                                                                                                                                                                                                                                 | Cobrar al paramédico o médico individual por leer información vital en terreno no tiene anclaje en este marco; el Escenario B cobra al **Servicio de Salud / SAMU / hospital** (SLA, Sidecar, integración), no al profesional en turno.                                     |
| **Diseño MIMF (anti-objetivo)**          | `01_proyecto.md`: el sistema **no condiciona la atención** a la tenencia del chip ni del RUT; §0 de este documento descarta cobro al paciente por acceso clínico o de emergencia.                                                                                                                                                                                                                                                                                                                                                                  | Un premium B2C que deje al usuario gratuito con menos datos vitales en urgencia viola el mismo principio de no condicionar la atención — ahora por capacidad de pago.                                                                                                       |


**Qué sí puede monetizarse en el Escenario B (sin chocar con lo anterior):**

- Suscripción del **hospital/clínica** por Sidecar, Hub, Índice o RVN gestionado.
- Fee de **certificación** del `HospitalConnector` a proveedores EHR.
- **Hosting y SLA** para Isapres o redes privadas.
- **Analítica y reportería administrativa** para la gerencia hospitalaria (nunca RVN completo, Break-Glass, ni lectura clínica vital del TPIM tras paywall).

**Alineación:** el Escenario B **sí saca plata de las empresas** (instituciones y proveedores); lo que la ley chilena no respalda — y el diseño MIMF rechaza — es cobrarle al ciudadano por acceder a su propia información clínica o por una capa “premium” de continuidad/urgencia que la ley ya obliga a garantizar de otra forma.

**Referencias:**


| Recurso                                                    | URL                                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ley 20.584 (texto actualizado, DIPRECE/MINSAL)             | [https://diprece.minsal.cl/wp-content/uploads/2025/09/Ley-20.584.pdf](https://diprece.minsal.cl/wp-content/uploads/2025/09/Ley-20.584.pdf)                                                                                                                                                     |
| Ley 21.668 (BCN / Ley Chile)                               | [https://www.bcn.cl/leychile/navegar?idNorma=1203827](https://www.bcn.cl/leychile/navegar?idNorma=1203827)                                                                                                                                                                                     |
| Ley 21.719 (BCN; arts. 4, 9, 10, 35)                       | [https://www.bcn.cl/leychile/navegar?idNorma=1209272](https://www.bcn.cl/leychile/navegar?idNorma=1209272)                                                                                                                                                                                     |
| Ley 21.719 — D.O. 13-dic-2024                              | [https://www.diariooficial.interior.gob.cl/publicaciones/2024/12/13/44023/01/2583630.pdf](https://www.diariooficial.interior.gob.cl/publicaciones/2024/12/13/44023/01/2583630.pdf)                                                                                                             |
| MINSAL — Ley 21.668 publicada                              | [https://www.minsal.cl/ley-de-interoperabilidad-de-fichas-clinicas-fue-publicada-en-el-diario-oficial/](https://www.minsal.cl/ley-de-interoperabilidad-de-fichas-clinicas-fue-publicada-en-el-diario-oficial/)                                                                                 |
| Superintendencia — derechos sobre ficha clínica            | [https://www.superdesalud.gob.cl/preguntas-frecuentes/cuales-son-los-derechos-de-las-personas-respecto-de-la-ficha-clinica/](https://www.superdesalud.gob.cl/preguntas-frecuentes/cuales-son-los-derechos-de-las-personas-respecto-de-la-ficha-clinica/)                                       |
| Norma técnica telemedicina MINSAL (portabilidad / art. 13) | [https://portalsaluddigital.minsal.cl/wp-content/uploads/2025/01/2025.01.06_NORMA-TECNICA-PRESTACIONES-DE-SALUD-A-DISTANCIA-Y-TELEMEDICINA.pdf](https://portalsaluddigital.minsal.cl/wp-content/uploads/2025/01/2025.01.06_NORMA-TECNICA-PRESTACIONES-DE-SALUD-A-DISTANCIA-Y-TELEMEDICINA.pdf) |
| Monografía ficha clínica Superintendencia (2025)           | [https://www.superdesalud.gob.cl/app/uploads/2025/12/monografia-ficha-clinica-2025-3.pdf](https://www.superdesalud.gob.cl/app/uploads/2025/12/monografia-ficha-clinica-2025-3.pdf)                                                                                                             |


### 2.2. Estructura de ingresos (modelo *open core*, tipo Red Hat / GitLab / MongoDB)


| Capa                                | Quién paga                          | Concepto                                                                                       | Nota                                                                    |
| ----------------------------------- | ----------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| App Paciente / TPIM básico          | Nadie — sin costo                   | Acceso del titular a su propio RVN, escritura básica del chip, zona pública NDEF               | Anti-objetivo de `01_proyecto.md`; anclaje legal §2.1.1                 |
| **Plan premium ciudadano (B2C)**    | **Excluido del modelo**             | Suscripción ~$19.900/año (perfiles familiares, documentos médicos, soporte) sobre capa gratuita — modelo VitalTag informe ATEP01 | Choca con Ley 20.584 art. 13, 21.668 y 21.719 arts. 9–10 — ver §0.1 y §2.1.1 |
| **Venta retail de chip/etiqueta NFC** | **Excluido al ciudadano**           | Venta unitaria ~$5.990 al titular (modelo VitalTag)                                              | Condiciona participación; hardware vía institución (B2B) si aplica — ver §0.1 |
| Nodo Sidecar por hospital/clínica   | Hospital o clínica                  | Suscripción anual por nodo conectado, escalonada por tamaño (camas / atenciones anuales)       | Infraestructura institucional, no dato personal                         |
| Certificación de conectores         | Proveedores de EHR                  | Fee de certificación en el Sandbox del `HospitalConnector` propio                              | Ya definido en `02_conceptos_y_tecnologias.md` (Sidecar)                |
| Malla privada gestionada            | Isapres, redes de clínicas privadas | Hosting + operación de Hub/Índice/RVN para red interna, sin depender de la malla estatal       | Producto B2B puro                                                       |
| Soporte y SLA                       | Cualquier institución cliente       | Contratos de soporte, actualizaciones OTA, auditoría de seguridad                              | Análogo al modelo Red Hat: software abierto, soporte empresarial pagado |
| Funciones "premium" institucionales | Administración hospitalaria         | Dashboards analíticos, reportería avanzada, integraciones con sistemas de gestión hospitalaria | Nunca sobre funciones clínicas vitales (RVN, Break-Glass, TPIM)         |


### 2.3. Riesgos de este escenario

- Ciclo de venta B2B más largo que un modelo B2C (contratos institucionales, procesos de adquisición hospitalaria).
- Menor margen inicial que un modelo de suscripción masiva individual, compensado por mayor ticket promedio y menor *churn*.
- Requiere certificación (ISO 27001, cumplimiento Ley 21.663 si el cliente califica como Servicio Esencial u OIV — ver §5) antes de poder vender a instituciones de salud reguladas.
- **Riesgo legal y reputacional** si se reintroduce monetización B2C sobre datos clínicos del titular: fiscalización Superintendencia (Ley 20.584 art. 38) y, desde dic-2026, APDP por obstaculización de derechos ARCOP/portabilidad (Ley 21.719 art. 35) — ver §2.1.1.

---

## 3. Escenario de salida individual (uno o algunos socios, no todos)

### 3.1. Punto de partida legal: sin sociedad constituida, la responsabilidad es personal e ilimitada

Mientras el proyecto no esté constituido como persona jurídica, cualquier actividad conjunta de los integrantes puede entenderse, en términos generales del derecho societario chileno, como una **sociedad de hecho**: cada integrante responde personal, ilimitada y solidariamente por las obligaciones derivadas de la actividad, sin el beneficio de responsabilidad limitada que sí ofrecen las estructuras formales ([Edig — efectos personales del socio](https://edig.cl/2020/12/02/efectos-personales-socio-de-una-sociedad/)).

**Nota:** Obligatoria la consulta a un abogado antes de realizar cualquier acción.

### 3.2. Qué resuelve constituir una sociedad (SpA) antes de operar con datos reales

La Sociedad por Acciones (SpA, Ley 20.190) es la forma societaria más usada por startups en Chile por su flexibilidad: puede constituirse con un solo accionista, permite entrada y salida de accionistas sin reescribir estatutos completos, y **limita la responsabilidad de cada accionista al monto de su aporte** ([Simplo — SpA en Chile 2026](https://simplo.cl/que-es-spa-en-chile/); [Emprende.cl](https://www.emprende.cl/que-es-una-spa-en-chile-claves-principales/)). Constituir la SpA antes de tratar datos reales de pacientes es lo que efectivamente separa el patrimonio personal de cada integrante del patrimonio del proyecto — no el hecho de "retirarse" de él.

**Excepción relevante:** la responsabilidad limitada de una SpA no protege a un administrador que actúe con dolo, negligencia grave, o fuera del marco legal (ej. entrega de información falsa a una autoridad fiscalizadora, uso de fondos con fines personales) — en esos casos responde personalmente, sin límite ([Avla — diferencia Limitada vs. SpA](https://www.avla.com/cl/blog/diferencia-entre-empresa-limitada-y-spa)).

### 3.3. Qué NO libera una salida individual

- **Retirarse del proyecto no borra retroactivamente decisiones tomadas mientras se era parte de él.** Si una falla de seguridad o de cumplimiento (Ley 21.719, Ley 21.663 — ver §5) se origina en un diseño o decisión tomada durante la participación de una persona, esa responsabilidad no desaparece automáticamente por salir antes de que la falla se materialice, especialmente en sede civil (responsabilidad extracontractual por el hecho propio).
- Si la salida ocurre **antes** de constituir una sociedad formal y **antes** de tratar datos reales, el riesgo remanente es bajo — no hay entidad, no hay dato tratado, no hay infracción posible aún.
- Si la salida ocurre **después** de constituir una SpA, con las participaciones (acciones) correctamente cedidas y registradas, la responsabilidad de quien se retira queda limitada al aporte ya realizado, y no se extiende a obligaciones futuras de la sociedad — pero sí a lo ya ocurrido mientras era accionista/administrador, según las reglas generales de responsabilidad civil y societaria.

### 3.4. Mecanismo recomendado

1. Formalizar la sociedad (SpA) **antes** de la fase de piloto con datos reales, no después.
2. Documentar en los estatutos las reglas de cesión de acciones (libre o con derecho preferente) para que la salida de un accionista sea un trámite simple (compraventa de acciones + actualización del registro de accionistas), no una negociación desde cero.
3. Ante una salida, formalizar por escrito: cesión de acciones, renuncia a cualquier cargo de administración o representación, y (si corresponde) desvinculación como responsable de tratamiento de datos o delegado de ciberseguridad ante las autoridades pertinentes (Agencia de Protección de Datos Personales, ANCI) si esos roles ya estaban asignados formalmente.

### 3.5. Autoría y referencia en CV/portafolio tras la salida

Renunciar a la responsabilidad societaria y ceder la titularidad patrimonial del proyecto **no equivale** a renunciar a la autoría. La Ley 17.336 sobre Propiedad Intelectual distingue dos derechos separados:

- **Derecho patrimonial:** puede cederse, venderse o transferirse (es lo que se transa en los Escenarios A, B, C y D de este documento).
- **Derecho moral (art. 14):** incluye el **derecho de paternidad** — reivindicar la autoría de la obra asociando el propio nombre a ella — y es personal, vitalicio e **irrenunciable**: *"es nulo cualquier pacto en contrario"*. Este derecho no se pierde al ceder acciones, renunciar a un cargo, ni al vender la propiedad intelectual a un tercero o al Estado.

En términos prácticos: es posible seguir refiriéndose a sí mismo como arquitecto del sistema en un CV o portafolio después de desvincularse, incluso si el proyecto pasa a manos del Estado o de un comprador privado. Dos matices a considerar, distintos del derecho de autoría en sí:

1. **Tiempo verbal y alcance de la afirmación:** describir el rol en pasado y ligado al período de participación real ("diseñé la arquitectura de interoperabilidad del sistema X entre [fecha] y [fecha]"), no como afiliación o responsabilidad vigente.
2. **Confidencialidad de detalles operativos:** si al momento de la transferencia (Escenario A o D) o de un contrato B2B (Escenario B) se firma un acuerdo de confidencialidad, este puede restringir la divulgación de detalles específicos de implementación, clientes o cifras — pero no la autoría general del diseño. Se recomienda, al negociar cualquier contrato de transferencia o venta, incluir explícitamente una cláusula de atribución/portafolio que confirme por escrito el derecho a referenciar el rol desempeñado, para evitar ambigüedad futura con la contraparte.

**Nota:** esto es información general sobre el régimen de derecho de autor chileno; conviene confirmar con un abogado la redacción exacta de cualquier cláusula de confidencialidad antes de firmarla.

---

## 4. Escenario de salida colectiva (todos los socios acuerdan vender y soltar el proyecto)

### 4.1. Vía preferente — venta al Estado (BOT)

Ya descrita en el §1.4: venta de propiedad intelectual + contrato de transición al Estado o a una entidad pública/mixta que asuma la operación.

### 4.2. Vía alternativa — adquisición por un tercero privado

Venta de la sociedad (o de sus activos: código, propiedad intelectual, contratos vigentes) a un actor de salud digital establecido (ej. un proveedor de EHR, una Isapre, un fondo de inversión en salud digital). Este camino es compatible con el Escenario B (SaaS institucional) como historial de ingresos que hace la sociedad más atractiva para un comprador, incluso si el destino final es igualmente ceder el control del proyecto.

### 4.3. Disolución sin comprador

Si no hay comprador (estatal ni privado) al momento de que todos los socios deciden salir, la alternativa es la disolución formal de la sociedad conforme a sus estatutos y a las normas del Código de Comercio. Esto cierra la responsabilidad futura de la entidad, pero no exime de responsabilidad por hechos ya ocurridos durante su operación (ver §3.3).

---

## 5. Marco de responsabilidad y multas

Dos cuerpos legales concentran el riesgo económico de operar MIMF como infraestructura crítica en salud, ambos con potestad sancionatoria administrativa activa a julio de 2026:

### 5.1. Ley 21.719 — Protección de Datos Personales

Vigencia plena desde el 1 de diciembre de 2026 (24 meses desde su publicación el 13-dic-2024). La Agencia de Protección de Datos Personales (APDP) puede sancionar:


| Categoría | Multa máxima                                                 | Agravante por reincidencia (empresas no-PYME)                                      |
| --------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| Leve      | 5.000 UTM                                                    | —                                                                                  |
| Grave     | 10.000 UTM                                                   | hasta 2% de ingresos anuales, si es mayor a la multa en UTM                        |
| Gravísima | 20.000 UTM (≈ CLP 1.300–1.400 millones al valor UTM de 2026) | hasta 4% de ingresos anuales, o la multa triplicada (60.000 UTM), lo que sea mayor |


Fuente: art. 35 de la Ley 21.719 ([Anguita Osorio](https://www.anguitaosorio.cl/es/sanciones-multas/); [informe BCN](https://obtienearchivo.bcn.cl/obtienearchivo?id=repositorio%2F10221%2F37137%2F1%2FInforme_12_25_Ley_Datos_Personales_rev.pdf)). El tratamiento de datos de salud (categoría sensible, art. 16 bis) agrava el análisis de gravedad. La ley contempla además **responsabilidad civil** independiente de la multa administrativa (título 13 del régimen, según síntesis BCN).

### 5.2. Ley 21.663 — Marco de Ciberseguridad e Infraestructura Crítica de la Información

Vigente desde marzo de 2025, con la Agencia Nacional de Ciberseguridad (ANCI) ya fiscalizando. El sector salud está explícitamente listado como Servicio Esencial (SE), y una entidad puede además ser calificada como Operador de Importancia Vital (OIV) por resolución de la ANCI:


| Categoría | Multa máxima (régimen general) | Multa máxima (OIV)                                                 |
| --------- | ------------------------------ | ------------------------------------------------------------------ |
| Leve      | 5.000 UTM                      | —                                                                  |
| Grave     | 10.000 UTM                     | hasta 20.000 UTM                                                   |
| Gravísima | 20.000 UTM                     | hasta 40.000 UTM (≈ CLP 2.700–2.800 millones al valor UTM de 2026) |


Fuente: [ANCI — Ley Marco de Ciberseguridad](https://anci.gob.cl/normativa/ley-marco-ciberseguridad/); [Anguita Osorio — Ley 21.663](https://www.anguitaosorio.cl/es/ley-ciberseguridad/). Además de la multa administrativa, los OIV **responden civilmente por daños a terceros derivados de incidentes evitables** — en sectores como salud, este monto puede exceder ampliamente la sanción administrativa ([ASIMA](https://asima.cl/ciberseguridad/)).

### 5.3. Lectura para el proyecto

- Las dos leyes son acumulativas, no alternativas: un incidente de seguridad que exponga datos de salud puede activar sanción bajo ambas leyes simultáneamente, más responsabilidad civil.
- La calificación como OIV depende de la escala real de operación (número de personas afectadas, criticidad del servicio) y la determina la ANCI por resolución — no es automática por el solo hecho de operar en salud, pero la propia tesis de MIMF (infraestructura nacional crítica) apunta directamente a esa categoría si se llega a escalar.
- Esto no es una razón para no ejecutar el proyecto: es la razón concreta para no operar con datos reales de pacientes sin (a) una entidad legal formalmente constituida, (b) el Sandbox y las medidas de seguridad ya definidas en `02` y `05` efectivamente implementadas, y (c) claridad, antes de escalar, sobre quién es el responsable legal de tratamiento de datos ante la APDP y la ANCI.

---

## 6. Resumen comparativo


|                                             | A — Estatal                                | B — SaaS institucional                                 | C — Salida individual                          | D — Salida colectiva                |
| ------------------------------------------- | ------------------------------------------ | ------------------------------------------------------ | ---------------------------------------------- | ----------------------------------- |
| Comprador / contraparte                     | MINSAL / Servicio de Salud / Estado        | Hospitales, clínicas, Isapres, proveedores EHR         | N/A — reorganización interna                   | Estado (BOT) o tercero privado      |
| Quién nunca paga                            | —                                          | El paciente y el profesional de salud, individualmente | —                                              | —                                   |
| Mecanismo                                   | Fondos de innovación → piloto → licitación | Venta B2B directa + soporte                            | Cesión de acciones (si hay SpA)                | Venta de IP/sociedad o disolución   |
| Condición previa clave                      | Reglamento art. 13 publicado               | Certificación de seguridad (ISO 27001, Ley 21.663)     | SpA constituida antes de tratar datos reales   | Compromiso unánime de los socios    |
| Riesgo principal                            | Atraso regulatorio, ciclo político         | Ciclo de venta B2B largo                               | Responsabilidad por hechos previos a la salida | Ausencia de comprador               |
| Exposición si se opera sin estructura legal | —                                          | —                                                      | Responsabilidad personal ilimitada y solidaria | Igual, extendida a todos los socios |


