# Propuesta: Malla de Interoperabilidad Médica Federada (MIMF)

> **Tesis Central:** "Chile no requiere un sistema único nuevo; requiere una capa nacional de interoperabilidad que conecte sistemas heterogéneos, preserve la soberanía local y garantice que la información clínica crítica viaje con el paciente, ya sea digitalmente entre hospitales o físicamente en su bolsillo durante emergencias pre-hospitalarias."

---

## 1. El Problema

El ecosistema de salud chileno sufre de fragmentación tecnológica, ausencia de continuidad clínica y alta heterogeneidad de sistemas (sistemas legacy, desarrollos in-house y proveedores cautivos). Esto resulta en una baja interoperabilidad que impacta directamente en la experiencia clínica y el riesgo del paciente.

## 2. Principio de Diseño

El principio fundamental es la **No Fricción y No Reemplazo**. En lugar de forzar una costosa migración hacia una base de datos centralizada única, se propone integrar la infraestructura existente mediante una arquitectura híbrida: mantener la soberanía y almacenamiento profundo en las redes locales, centralizando estrictamente solo el descubrimiento de rutas y la información crítica de vida o muerte.

### Límites de Alcance (Anti-Objetivos / Lo que la MIMF NO hace)

- No reemplaza los sistemas EHR (Fichas Clínicas) existentes en los hospitales.
- No centraliza todo el historial clínico histórico en una sola base de datos gigante.
- No resuelve la calidad semántica automáticamente; requiere que el hospital origen realice el mapeo al estándar (*garbage in, garbage out*).
- No reimplementa de forma permanente el **EMPI/MPI** ni el **HPD** del MINSAL: el destino es consumirlos. El adaptador temporal de `PatientIdentityProvider` es solo para piloto y se retira al converger.
- El **Token Físico (NFC)** NO reemplaza el RVN central; es un complemento offline de último recurso. La fuente de verdad sigue siendo el sistema del hospital.
- El sistema no condiciona la atención a la tenencia del chip ni del RUT. Ante pacientes indocumentados, turistas extranjeros o ciudadanos sin TPIM, el flujo pre-hospitalario hace *fallback* inmediato al protocolo clínico de trauma estándar.



## 3. Arquitectura Propuesta (Core Técnico)

Para resolver la interoperabilidad sin sobrecargar la operación, el sistema define los siguientes componentes clave:

- **Nodos Perimetrales (Sidecars):** Agentes livianos instalados en cada hospital. Actúan como traductores entre la base de datos local y la red nacional. No existe un Sidecar "universal" con todos los conectores: cada despliegue se **compila como binario específico** (Core + conector del EHR de ese hospital), reduciendo superficie de ataque, RAM y dependencias. La MIMF define una **interfaz oficial de conectores** (`HospitalConnector`); el conector puede ser mantenido por el equipo MIMF o desarrollado y certificado por el propio proveedor del EHR.
- **Estándar Semántico y Sintáctico:** Uso de **HL7 FHIR R4** (alineado a la estrategia del [MINSAL](https://interoperabilidad.minsal.cl/)) para la estructura de datos y **SNOMED CT / LOINC** para el significado clínico, apoyándose cuando existan en los [Servicios Terminológicos](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/terminologicos.html) nacionales. Los Sidecars soportan **coexistencia de versiones** del Perfil Chile (ej. v3, v4 y v5 en paralelo) con ventanas de *End-of-Life* anunciadas. **Separación de caminos:** el SLA de urgencia (**< 3s** del RVN) corre solo por la malla interna **gRPC/Protobuf** (Sidecar ↔ Hub ↔ Índice ↔ RVN). Las consultas **REST/FHIR** hacia EMPI, HPD y Terminológicos van en un carril aparte: revalidación de caché en segundo plano, sync de catálogos, o *fallback* ante *cache-miss* (paciente nunca visto en ese Sidecar), caso excepcional donde se acepta la latencia adicional de REST.
- **Identidad del Paciente (**`PatientIdentityProvider` **→ EMPI MINSAL):** La MIMF **no inventa un maestro de identidad paralelo** como destino final. La malla habla con un contrato estable `PatientIdentityProvider` (resolver por RUN/ID local, cruzar IDs, demografía mínima, señal de fusión). El **backend definitivo** es el [EMPI/MPI](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/empi.html) del Estado (PIXm/PDQm del [NID](https://interoperabilidad.minsal.cl/fhir/ig/nid/0.4.9/) / [IG MPI](https://interoperabilidad.minsal.cl/fhir/ig/mpi/)). Mientras ese servicio nacional no esté operativo en terreno, el piloto usa un **adaptador temporal** (mismas operaciones; implementación acotada al alcance del piloto: RUN + reglas locales + merge/soporte). Cuando el EMPI real esté disponible, se **reemplaza solo el adaptador**; Record Locator, Sidecar, RVN y TPIM no se reescriben. Lo mismo aplica al [HPD](https://interoperabilidad.minsal.cl/fhir/ig/hpd/) para atributos de prestador en ABAC: opcional al inicio, enchufable después. Acoplamiento al **contrato**, no al calendario de despliegue del MINSAL. 
- **Índice Nacional de Descubrimiento (Record Locator):** Servicio distinto de la identidad. No guarda demografía ni ficha clínica: solo responde *¿en qué hospital(es)/nodos hay datos clínicos de este paciente?*. Se indexa por el **ID canónico** que entrega el `PatientIdentityProvider` (en régimen: ID del EMPI). Los Sidecars mantienen una **caché local de resoluciones** (`RUN/ID local → ID canónico`, TTL corto) para latencia y para degradar si el proveedor de identidad está temporalmente indisponible; al recuperar conectividad se revalida y, si hubo fusión, se reindexa el locator. Opcionalmente, las claves del locator en reposo pueden pseudonimizarse (HMAC + KMS); eso es privacidad del índice, **no** un sistema de identidad. El Índice cuenta además con replicación geográfica y caché de rutas en los Sidecars.
- **Resumen Vital Nacional (RVN):** Repositorio central mínimo (alergias, medicamentos, diagnósticos críticos, grupo sanguíneo). Su rol clínico no es competir con la ficha del hospital: actúa como **alerta de urgencia** ("este paciente tiene historia en Hospital X; ¿desea traerla?") más un subconjunto ultracrítico. Su esquema y evolución son regidos por un comité clínico-técnico con gobernanza estricta para evitar el *scope creep*. Ante desacuerdos, el voto dirimente recae en el Departamento de Calidad y Seguridad del Paciente (MINSAL), prevaleciendo siempre el criterio de seguridad clínica por sobre las preferencias de software.
- **Token Físico de Identidad Médica (TPIM):** Dispositivo NFC (tarjeta, pulsera o anillo) con **doble zona NDEF**: (1) **Zona pública** = texto plano NDEF, legible de forma **nativa** por cualquier smartphone con NFC (sin instalar app, sin internet); (2) **Zona privada** = subconjunto ultracrítico del RVN en Protobuf firmado, solo para personal autorizado. Canales de actualización del chip: (A) **prioridad** — contacto clínico (mesón/Sidecar); (B - En Discusión) **App Paciente / Autogestión** (Clave Única) — el dueño pide al RVN el payload firmado y lo escribe con el NFC de su celular; (C) kioscos CESFAM/farmacias (complemento). No existe “app de civiles”: el transeúnte usa el lector NFC del sistema operativo.
- **App de Primeros Respondedores:** Cliente oficial (SAMU / Bomberos) para leer la **zona privada** del TPIM. Integra ABAC (turno activo), Break-Glass en terreno y pre-alerta al hospital de destino.
- **App Paciente / Autogestión (En Discusión):** Cliente oficial autenticado con **Clave Única**. Permite (1) actualizar la zona privada de *su* TPIM con el payload firmado del RVN; (2) configurar y consentir el texto de la zona pública. No es la app de un transeúnte: solo el dueño/tutor autenticado escribe en su chip.
- **SDK de Integración (hospitales/proveedores):** Además de las dos apps oficiales, la MIMF expone **SDK / API / OAuth** para que un hospital o proveedor integre ABAC y consulta de atributos en su propio EHR, sin obligar a usar las apps de la red.
- **Modo Degradado Local:** Si la red nacional o el Índice caen, el Sidecar aísla al hospital, permitiendo que la atención médica continúe con los registros locales sin disrupciones. Si bien la interoperabilidad nacional se degrada temporalmente, la operación local no se ve afectada. Ante desconexiones prolongadas (días o meses), la reincorporación usa **snapshot + replay controlado** de la cola de sincronización, no un volcado masivo sin límites.



### Flujo Clínico 1: Atención de Urgencia en Hospital

1. **Ingreso:** Paciente llega a urgencias.
2. **Identidad:** El Sidecar resuelve al paciente vía `PatientIdentityProvider` (adaptador piloto o EMPI nacional, según fase).
3. **Descubrimiento:** Con el ID canónico, consulta el **Índice de Descubrimiento** (¿dónde hay datos clínicos?).
4. **Carga Vital:** Se despliega de inmediato el RVN como alerta de urgencia (alergias, medicamentos críticos, etc.) y se indica en qué nodos existe historial profundo.
5. **Evaluación:** El médico decide si trae el historial profundo de otros nodos (*lazy loading*).
6. **Acción y Reconciliación:** El médico ve posibles conflictos, actúa y el sistema registra la trazabilidad.



### Flujo Clínico 2: Emergencia Pre-Hospitalaria y Civiles (NFC)

**Escenario A: Asistencia por Civiles (Transeúntes)**

1. Un transeúnte encuentra a una persona sufriendo una crisis (ej. ataque de epilepsia, descompensación por autismo).
2. Acerca cualquier smartphone con NFC: el sistema operativo lee el registro **NDEF de texto plano** de la **Zona Pública** (sin instalar app MIMF, sin internet).
3. El civil ve instrucciones críticas predefinidas (ej. *"Epilepsia: No introduzca objetos en mi boca. Recuésteme de lado. Contacto de emergencia: +569..."*) sin acceder a datos clínicos sensibles, ganando minutos vitales antes de que llegue la ambulancia.

**Escenario B: Asistencia por Paramédicos (SAMU)**

1. El paramédico llega al lugar y escanea el TPIM usando la **App de Primeros Respondedores**.
2. **Control ABAC Móvil:** La app verifica en milisegundos si el paramédico está *en turno activo*. Si no lo está, la lectura privada se bloquea por defecto para evitar abusos o espionaje ("Deny by Default").
3. **Break-Glass (Fuera de turno):** Si el paramédico está fuera de turno pero se encontró con el accidente, debe presionar "Activar Break-Glass" en la app, justificando la acción, lo que libera el acceso y genera una alerta de auditoría inmutable.
4. **Tratamiento:** La App decodifica la **Zona Privada** (Protobuf), mostrando alergias y diagnósticos codificados en SNOMED CT.
5. **Pre-Alerta:** El paramédico notifica electrónicamente al hospital de destino con el RVN recuperado antes del arribo.



## 4. Seguridad y Privacidad

El modelo protege la información bajo la Ley 19.628 y la Ley 20.584 mediante:

- **Autenticación e Identidad:** Acceso de profesionales validado vía Clave Única y, cuando esté disponible, el [HPD](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/hpd.html). La identidad del **paciente** pasa por `PatientIdentityProvider` (destino: EMPI/MPI; temporal: adaptador de piloto). La MIMF no opera un segundo maestro demográfico permanente. El RUT/RUN es un identificador de entrada a resolver, no la clave primaria de la malla clínica.
- **Autorización Contextual (ABAC):** El acceso a la ficha se otorga evaluando el contexto (ej. el médico debe tener una atención activa con el paciente). Los atributos (turno, agenda, admisión, RRHH, SAMU; y credenciales de prestador vía HPD cuando aplique) se **leen** desde los sistemas existentes; el Sidecar no escribe ni crea registros en esos sistemas. Hospitales y proveedores pueden integrar el motor vía SDK/API/OAuth sin adoptar las apps oficiales.
- **Acceso de Emergencia (Break-Glass):** Protocolo estricto que permite saltar el ABAC en riesgo vital por un tiempo limitado, requiriendo justificación obligatoria y generando una alerta directa al comité de auditoría del hospital.
- **Cifrado Robusto:** Los datos clínicos viajan con cifrado en tránsito (TLS 1.3) y cifrado en reposo en las cachés locales.
- **Auditoría Inmutable:** Todo acceso queda registrado en una bitácora temporal "append-only" que garantiza la trazabilidad.
- **Seguridad Física y Antifalsificación (TPIM):** Los datos de la Zona Privada del chip están firmados digitalmente con la clave privada del RVN central (PKI). La App de Primeros Respondedores (y la App Paciente al escribir) verifica esta firma; se rechazan chips clonados o manipulados. La App Paciente **no** escribe payloads ajenos: solo el titular autenticado con Clave Única puede iniciar la escritura de *su* chip.
- **Consentimiento Informado y Emisión:** La activación/edición de la Zona Pública es voluntaria y requiere consentimiento explícito del paciente o tutor (vía App Paciente / Autogestión o en el acto de emisión institucional). La emisión física del chip se ancla a CESFAM / FONASA, vinculando el dispositivo a la identidad validada mediante Clave Única.



## 5. Operación y Resiliencia

Para evitar que el sistema se vuelva una "infraestructura de guerra" inmantenible, se definen límites claros:

- **Comunicación vía Relays/Hubs:** Reconociendo la realidad de los firewalls institucionales, la comunicación prioriza el paso seguro por un Hub gestionado en lugar de forzar conexiones P2P directas imposibles de mantener.
- **Consistencia Clínica y Resolución de Conflictos:** El hospital de origen es la fuente de verdad. Ante discrepancias, se prioriza por tipo de dato (ej. alergias > diagnósticos históricos) con versionado explícito. El sistema permite "datos reconciliados" (*soft merge* clínico) visuales, delegando la resolución final al profesional.
- **Contrato de Esquema (ETL):** El contrato de interoperabilidad obliga al hospital / proveedor a **informar previamente** cambios estructurales en su base o EHR. La actualización del conector la realiza soporte MIMF (o el proveedor certificado del conector) en paralelo; no se asume detección automática mágica de renombres de campos.
- **Observabilidad:** Trazabilidad centralizada para identificar cuellos de botella sin intervenir los servidores locales.
- **Límites del Sistema y Gestión de Carga:** Umbrales operativos de diseño (SLAs objetivo): **< 3s** para la carga vital del RVN, y un límite aceptable de **5-10s** para la carga profunda (historial) vía carga progresiva. Tamaños máximos de consulta y timeouts definidos. Si todo falla, opera autónomamente en Modo Degradado.



## 6. Gobernanza, Adopción y Marco Legal

La tecnología por sí sola no asegura la adopción. Se requiere una estrategia estatal:

- **Certificación de Proveedores:** Los proveedores de software privado deben aprobar un "Sandbox" técnico del Estado para participar en compras públicas. Pueden, además, desarrollar su propio conector bajo la interfaz `HospitalConnector` y certificarlo, en lugar de depender exclusivamente del equipo MIMF.
- **APIs Obligatorias:** Se exige por norma técnica la exposición de datos clínicos mediante el estándar definido (Perfil Chile FHIR). Cumplir la Ley N.º 21.668 no es opcional para quien quiera vender EHR al Estado.
- **Estrategia de Transición y Coexistencia:** Capacidad de integración parcial para proveedores rezagados, con **período de transición** explícito: un hospital no puede quedar sin sistema mientras cambia de proveedor o de versión de perfil. El incumplimiento de un actor no bloquea la red completa; se aísla al nodo no conforme.
- **Incentivos y Sanciones:** Despliegue anclado a la Ley N.º 21.668. Hospitales que no cumplan los plazos de integración enfrentan retención de fondos específicos.
- **Despliegue por Fases:** Piloto regional controlado antes del rollout nacional. El piloto **no espera** a que el EMPI/HPD nacionales estén en producción: usa el adaptador temporal de `PatientIdentityProvider` y, en la fase de convergencia, lo sustituye por el backend oficial sin reescribir la malla.
- **El Riesgo Principal:** Se asume explícitamente que el mayor riesgo de este proyecto no es técnico, sino institucional y político (falta de presupuesto sostenido, voluntad o coordinación; calendarios lentos de componentes nacionales). Por ello, la arquitectura minimiza la dependencia central, permite adopción asimétrica y **acopla al contrato de identidad, no al calendario del MINSAL**. La MIMF se entiende como **infraestructura nacional / ecosistema** (Sidecar, Índice, RVN, TPIM, apps, PKI, KMS, Sandbox, OTA), no como "una aplicación".

