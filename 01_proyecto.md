# Propuesta: Malla de Interoperabilidad Médica Federada (MIMF)

> **Tesis Central:** "Chile no requiere un sistema único nuevo; requiere una capa nacional de interoperabilidad que conecte sistemas heterogéneos, preserve la soberanía local y garantice que la información clínica crítica viaje con el paciente, ya sea digitalmente entre hospitales o físicamente en su bolsillo durante emergencias pre-hospitalarias."

---

## 1. El Problema

El ecosistema de salud chileno sufre de fragmentación tecnológica, ausencia de continuidad clínica y alta heterogeneidad de sistemas (sistemas legacy, desarrollos in-house y proveedores cautivos). Esto resulta en una baja interoperabilidad que impacta directamente en la experiencia clínica y el riesgo del paciente.

## 2. Principio de Diseño

El principio fundamental es la **No Fricción y No Reemplazo**. En lugar de forzar una costosa migración hacia una base de datos centralizada única, se propone integrar la infraestructura existente mediante una arquitectura híbrida: mantener la soberanía y almacenamiento profundo en las redes locales, centralizando estrictamente solo el descubrimiento de rutas y la información crítica de vida o muerte.

### Límites de Alcance (Anti-Objetivos / Lo que la MIMF NO hace)

*   No reemplaza los sistemas EHR (Fichas Clínicas) existentes en los hospitales.
*   No centraliza todo el historial clínico histórico en una sola base de datos gigante.
*   No resuelve la calidad semántica automáticamente; requiere que el hospital origen realice el mapeo al estándar (*garbage in, garbage out*).
*   El **Token Físico (NFC)** NO reemplaza el RVN central; es un complemento offline de último recurso. La fuente de verdad sigue siendo el sistema del hospital.
*   El sistema no condiciona la atención a la tenencia del chip ni del RUT. Ante pacientes indocumentados, turistas extranjeros o ciudadanos sin TPIM, el flujo pre-hospitalario hace *fallback* inmediato al protocolo clínico de trauma estándar.

## 3. Arquitectura Propuesta (Core Técnico)

Para resolver la interoperabilidad sin sobrecargar la operación, el sistema define los siguientes componentes clave:

*   **Nodos Perimetrales (Sidecars):** Agentes livianos instalados en cada hospital. Actúan como traductores entre la base de datos local y la red nacional.
*   **Estándar Semántico y Sintáctico:** Uso de **HL7 FHIR** para la estructura de datos y **SNOMED CT / LOINC** para asegurar que el significado clínico sea idéntico entre instituciones.
*   **Índice Nacional de Descubrimiento:** Servicio centralizado de enrutamiento. No es una fuente de datos clínicos, sino un mapa "stateless" (reconstruible desde los nodos). Cuenta con replicación geográfica y caché local en los Sidecars, garantizando que una falla del Índice no implique una caída de la atención clínica local.
*   **Resumen Vital Nacional (RVN):** Repositorio central mínimo (alergias, medicamentos, diagnósticos críticos). Su esquema y evolución son regidos por un comité clínico-técnico con una gobernanza estricta para evitar el crecimiento descontrolado del alcance ("scope creep") y asegurar que se mantenga como un resumen de urgencia y no una ficha clínica centralizada. Ante desacuerdos, el voto dirimente recae en el Departamento de Calidad y Seguridad del Paciente (MINSAL),  prevaleciendo siempre el criterio de seguridad clínica por sobre las preferencias de software.
*   **Token Físico de Identidad Médica (TPIM):** Dispositivo NFC (tarjeta, pulsera o anillo) que posee una **arquitectura de doble zona**: una zona pública (instrucciones de primeros auxilios legibles por cualquier celular) y una zona privada (copia offline de un **subconjunto ultracrítico del RVN** en Protobuf firmado) exclusiva para personal médico.
*   **App de Primeros Respondedores:** Cliente especializado (SAMU / Bomberos) para leer la zona privada del TPIM. Integra el motor ABAC para verificar si el personal está de turno, gestiona el Break-Glass en terreno y permite pre-alertar al hospital de destino.
*   **Modo Degradado Local:** Si la red nacional o el Índice caen, el Sidecar aísla al hospital, permitiendo que la atención médica continúe con los registros locales sin disrupciones. Si bien la interoperabilidad nacional se degrada temporalmente, la operación local no se ve afectada.

### Flujo Clínico 1: Atención de Urgencia en Hospital

1.  **Ingreso:** Paciente llega a urgencias.
2.  **Descubrimiento:** El Sidecar consulta el Índice automáticamente.
3.  **Carga Vital:** Se despliega de inmediato el RVN.
4.  **Evaluación:** El médico decide si trae el historial profundo de otros nodos (*lazy loading*).
5.  **Acción y Reconciliación:** El médico ve posibles conflictos, actúa y el sistema registra la trazabilidad.

### Flujo Clínico 2: Emergencia Pre-Hospitalaria y Civiles (NFC)

**Escenario A: Asistencia por Civiles (Transeúntes)**
1. Un transeúnte encuentra a una persona sufriendo una crisis (ej. ataque de epilepsia, descompensación por autismo).
2. Al escanear la pulsera NFC con un smartphone común, se abre instantáneamente la **Zona Pública**.
3. El civil lee instrucciones críticas predefinidas (ej. *"Epilepsia: No introduzca objetos en mi boca. Recuésteme de lado. Contacto de emergencia: +569..."*) sin acceder a datos clínicos sensibles, ganando minutos vitales antes de que llegue la ambulancia.

**Escenario B: Asistencia por Paramédicos (SAMU)**
1. El paramédico llega al lugar y escanea el TPIM usando la App Oficial de la MIMF.
2. **Control ABAC Móvil:** La app verifica en milisegundos si el paramédico está *en turno activo*. Si no lo está, la lectura privada se bloquea por defecto para evitar abusos o espionaje ("Deny by Default").
3. **Break-Glass (Fuera de turno):** Si el paramédico está fuera de turno pero se encontró con el accidente, debe presionar "Activar Break-Glass" en la app, justificando la acción, lo que libera el acceso y genera una alerta de auditoría inmutable.
4. **Tratamiento:** La App decodifica la **Zona Privada** (Protobuf), mostrando alergias y diagnósticos codificados en SNOMED CT.
5. **Pre-Alerta:** El paramédico notifica electrónicamente al hospital de destino con el RVN recuperado antes del arribo.

## 4. Seguridad y Privacidad

El modelo protege la información bajo la Ley 19.628 y la Ley 20.584 mediante:

*   **Autenticación e Identidad:** Acceso validado vía Clave Única y sistemas de identidad profesional.
*   **Autorización Contextual (ABAC):** El acceso a la ficha se otorga evaluando el contexto (ej. el médico debe tener una atención activa con el paciente).
*   **Acceso de Emergencia (Break-Glass):** Protocolo estricto que permite saltar el ABAC en riesgo vital por un tiempo limitado, requiriendo justificación obligatoria y generando una alerta directa al comité de auditoría del hospital.
*   **Cifrado Robusto:** Los datos clínicos viajan con cifrado en tránsito (TLS 1.3) y cifrado en reposo en las cachés locales.
*   **Auditoría Inmutable:** Todo acceso queda registrado en una bitácora temporal "append-only" que garantiza la trazabilidad.
*   **Seguridad Física y Antifalsificación (TPIM):** Los datos de la Zona Privada del chip están firmados digitalmente con la clave privada del RVN central (PKI). La App verifica esta firma antes de decodificar, rechazando automáticamente chips clonados o manipulados.
*   **Consentimiento Informado y Emisión:** La activación de la Zona Pública del TPIM es estrictamente voluntaria y requiere consentimiento explícito del paciente o tutor legal. La emisión del chip físico se ancla a la institucionalidad (CESFAM / FONASA), vinculando el dispositivo a la identidad validada mediante Clave Única.

## 5. Operación y Resiliencia

Para evitar que el sistema se vuelva una "infraestructura de guerra" inmantenible, se definen límites claros:

*   **Comunicación vía Relays/Hubs:** Reconociendo la realidad de los firewalls institucionales, la comunicación prioriza el paso seguro por un Hub gestionado en lugar de forzar conexiones P2P directas imposibles de mantener.
*   **Consistencia Clínica y Resolución de Conflictos:** El hospital de origen es la fuente de verdad. Ante discrepancias, se prioriza por tipo de dato (ej. alergias > diagnósticos históricos) con versionado explícito. El sistema permite "datos reconciliados" (*soft merge* clínico) visuales, delegando la resolución final al profesional.
*   **Observabilidad:** Trazabilidad centralizada para identificar cuellos de botella sin intervenir los servidores locales.
*   **Límites del Sistema y Gestión de Carga:** Umbrales operativos de diseño (SLAs objetivo): **< 3s** para la carga vital del RVN, y un límite aceptable de **5-10s** para la carga profunda (historial) vía carga progresiva. Tamaños máximos de consulta y timeouts definidos. Si todo falla, opera autónomamente en Modo Degradado.

## 6. Gobernanza, Adopción y Marco Legal

La tecnología por sí sola no asegura la adopción. Se requiere una estrategia estatal:

*   **Certificación de Proveedores:** Los proveedores de software privado deben aprobar un "Sandbox" técnico del Estado para participar en compras públicas.
*   **APIs Obligatorias:** Se exige por norma técnica la exposición de datos clínicos mediante el estándar definido.
*   **Estrategia de Transición y Coexistencia:** Capacidad de integración parcial para proveedores rezagados. El incumplimiento de un actor no bloquea la red completa, operando con los nodos disponibles y aislando a los no conformes.
*   **Incentivos y Sanciones:** Despliegue anclado a la Ley N.º 21.668. Hospitales que no cumplan los plazos de integración enfrentan retención de fondos específicos.
*   **Despliegue por Fases:** Comenzando por un piloto regional controlado para validar flujos clínicos antes del rollout nacional.
*   **El Riesgo Principal:** Se asume explícitamente que el mayor riesgo de este proyecto no es técnico, sino institucional y político (falta de presupuesto sostenido, voluntad o coordinación). Por ello, la arquitectura minimiza la dependencia central y permite adopción asimétrica.