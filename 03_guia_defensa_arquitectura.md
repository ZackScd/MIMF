# Guía de Defensa de Arquitectura: Decisiones, Alternativas y Justificación

> **Regla Fundamental:** Para justificar cualquier decisión en la defensa, se debe responder estructuradamente a estas cuatro premisas para cada concepto:
> 1.  ¿Qué problema resuelve?
> 2.  ¿Qué alternativas existían?
> 3.  ¿Qué beneficios aporta la opción elegida?
> 4.  ¿Qué se sacrifica (trade-off) al optar por esta solución?

---

## 1. Centralizado vs. Federado

#### Conceptos a Estudiar
Arquitectura centralizada vs federada, Sistemas distribuidos, Data sovereignty (Soberanía de datos), Master data vs Source systems.

#### Problema a Resolver
Definir el modelo de almacenamiento y acceso a los datos a nivel nacional, balanceando la necesidad de una vista unificada del paciente con la realidad de sistemas heterogéneos y la soberanía de datos de cada institución.

#### Alternativas
*   **Centralización Total:** Consolidar todo el historial clínico en una base de datos nacional única.
*   **Federación Total:** Cada hospital expone sus datos de manera independiente (P2P puro).
*   **Híbrido Federado (Opción elegida):** Datos profundos mantenidos localmente, con una capa nacional para descubrimiento y un subconjunto de datos críticos (RVN).
*   **Data Mesh:** Arquitectura distribuida orientada a dominios autónomos.
*   **Replicación completa:** Copia asíncrona de todo el volumen de datos a múltiples nodos.

#### Decisión y Beneficios (Modelo Híbrido)
La centralización total tiene un costo prohibitivo, plazos inviables y alta fricción política. La federación pura tiende al caos sin un control central. El modelo híbrido reduce la fricción operativa al no forzar migraciones masivas, pero unifica la información crítica vital, logrando un equilibrio pragmático.

#### Sacrificio (Trade-off)
Incrementa la complejidad del diseño arquitectónico, exige una gobernanza técnica central muy fuerte y no garantiza consistencia absoluta ni latencia mínima para la consulta de historiales profundos antiguos.

---

## 2. Patrón Sidecar vs. Otras Formas de Integración

#### Conceptos a Estudiar
Patrones (Sidecar, Adapter, API Gateway, Plugin architecture), Enterprise Service Bus (ESB) y Middleware.

#### Problema a Resolver
Integrar sistemas *legacy* cerrados y heterogéneos sin modificar su código fuente, evitando el "vendor lock-in" y permitiendo una evolución tecnológica centralizada.

#### Alternativas
*   **Reescribir el sistema legacy** de cada hospital.
*   Desarrollar **adaptadores ad-hoc** por institución.
*   **API Gateway** central.
*   **Enterprise Service Bus (ESB)** clásico.
*   **Patrón Sidecar / Appliance gestionado (Opción elegida).**

#### Decisión y Beneficios (Sidecar)
Permite la integración sin intervenir el código fuente del sistema legacy. Desacopla la lógica de negocio (hospital) de la lógica de plataforma (red nacional). Es modular y menos invasivo que una refactorización completa o un ESB monolítico.

#### Sacrificio (Trade-off)
Aumenta la superficie de componentes desplegados en la red. Incorpora complejidad a la operación de TI de los hospitales y requiere mecanismos robustos para monitoreo y actualizaciones seguras (Over-The-Air).

---

## 3. Hub-and-Spoke vs. P2P vs. Bus de Eventos

#### Conceptos a Estudiar
Topologías de red (Peer-to-Peer, Hub-and-Spoke, Mesh pura), NAT traversal y arquitectura Relay, Message Brokers.

#### Problema a Resolver
Establecer una topología de red que funcione en el mundo real, superando las barreras de firewalls institucionales y NATs simétricos que impiden la comunicación directa.

#### Alternativas
*   **Peer-to-Peer (P2P) puro.**
*   **Hub central tradicional.**
*   **Hub-and-Spoke con Relay (Opción elegida).**
*   **Bus de eventos asíncrono** (ej. Kafka).

#### Decisión y Beneficios (Hub-and-Spoke con Relay)
Un modelo P2P puro es operativamente inviable en redes corporativas de salud. La arquitectura basada en Relay facilita una conectividad bidireccional estable mediante conexiones TCP persistentes de salida, sin exigir a los equipos de TI locales la apertura de puertos de entrada.

#### Sacrificio (Trade-off)
Genera una dependencia hacia una infraestructura de red intermedia. El Relay puede convertirse en un cuello de botella si no está diseñado para Alta Disponibilidad (HA), exigiendo SLAs muy estrictos.

---

## 4. REST vs. gRPC

#### Conceptos a Estudiar
Protocolos y arquitecturas (REST, gRPC, GraphQL, SOAP), Comparativas de rendimiento (HTTP/1.1 vs HTTP/2 y RPC).

#### Problema a Resolver
Elegir un protocolo de comunicación para la red interna de microservicios que sea de alto rendimiento, bajo consumo de ancho de banda y baja latencia.

#### Alternativas
*   **REST** con JSON.
*   **gRPC** con Protocol Buffers (Opción elegida).
*   **GraphQL**.
*   **SOAP** basado en XML.

#### Decisión y Beneficios (gRPC)
Ofrece un menor *overhead* de red gracias a HTTP/2 y serialización binaria. Maximiza el rendimiento y disminuye la latencia, factor crítico en zonas con ancho de banda restringido. REST se mantiene como opción viable solo para APIs públicas o integraciones simples.

#### Sacrificio (Trade-off)
Se sacrifica la legibilidad humana directa de los *payloads* (a diferencia de JSON). Requiere mayor disciplina para el manejo de los contratos (`.proto`) y representa una curva de aprendizaje más pronunciada.

---

## 5. JSON vs. XML vs. Protobuf

#### Conceptos a Estudiar
Formatos de serialización de datos (JSON, XML, Protobuf, Avro, MessagePack, CBOR).

#### Problema a Resolver
Seleccionar un formato de serialización de datos que sea eficiente, compacto y rápido para la comunicación entre los componentes de la MIMF.

#### Alternativas
*   **JSON.**
*   **XML.**
*   **Protocol Buffers / Protobuf (Opción elegida).**
*   **Avro** o **MessagePack.**

#### Decisión y Beneficios (Protobuf)
Es significativamente más compacto y rápido de procesar que JSON o XML, mejorando el rendimiento de la red. Es la opción ideal cuando se controla ambos extremos de la comunicación (Sidecar-A a Sidecar-B).

#### Sacrificio (Trade-off)
No permite inspección visual nativa (requiere herramientas para decodificar). Demanda la definición y compilación estricta de esquemas antes de la comunicación.

---

## 6. HL7 FHIR vs. Otros Estándares

#### Conceptos a Estudiar
Evolución de los estándares médicos (HL7 v2, HL7 v3, CDA, FHIR), openEHR.

#### Problema a Resolver
Establecer un "idioma" común (sintaxis) para que todos los sistemas intercambien datos de salud de forma estructurada y comprensible.

#### Alternativas
*   **HL7 v2** (Estándar histórico legacy).
*   **HL7 v3 / CDA** (Basado en documentos XML rígidos).
*   **HL7 FHIR (Opción elegida).**
*   **openEHR.**

#### Decisión y Beneficios (FHIR)
Es el estándar mundial moderno, basado en recursos modulares (RESTful) que se adaptan perfectamente a arquitecturas web. Es más flexible y fácil de implementar que sus predecesores.

#### Sacrificio (Trade-off)
FHIR establece la sintaxis, pero no resuelve la semántica por sí solo. Requiere la construcción de "Perfiles" nacionales y una fuerte gobernanza para evitar implementaciones divergentes.

---

## 7. SNOMED CT / LOINC vs. CIE-10

#### Conceptos a Estudiar
Interoperabilidad semántica, SNOMED CT, LOINC, CIE-10 / CIE-11.

#### Problema a Resolver
Lograr la interoperabilidad semántica: que un concepto clínico (ej. "infarto") signifique lo mismo en todos los sistemas, permitiendo análisis computacionales.

#### Alternativas
*   **Texto libre** sin codificación.
*   **CIE-10 / CIE-11** de manera exclusiva.
*   Uso combinado de **SNOMED CT y LOINC (Opción elegida).**
*   **Terminologías y diccionarios propietarios locales.**

#### Decisión y Beneficios (SNOMED CT y LOINC)
SNOMED CT ofrece la granularidad para describir diagnósticos y hallazgos clínicos. LOINC estandariza exámenes de laboratorio. Juntos, eliminan la ambigüedad. CIE-10 se descarta como vocabulario clínico primario porque está diseñado para estadística y facturación, no para el detalle de la atención.

#### Sacrificio (Trade-off)
Impone un alto costo inicial de mapeo semántico ("cross-mapping") en cada hospital para traducir sus diccionarios locales a la norma internacional.

---

## 8. ETL/Staging vs. Consulta Directa

#### Conceptos a Estudiar
Patrones de integración de datos (ETL, ELT, Staging area), Change Data Capture (CDC), Materialized views.

#### Problema a Resolver
Acceder a los datos de los sistemas *legacy* sin afectar su rendimiento ni estabilidad, garantizando lecturas rápidas para la red nacional.

#### Alternativas
*   **Consultas síncronas y directas** a la base de datos legacy.
*   **Sincronización por eventos (CDC).**
*   **ETL con Staging intermedio híbrido (Opción elegida).**
*   **Arquitecturas completas de Data Lake o Data Warehouse.**

#### Decisión y Beneficios (Staging Híbrido)
Protege la base de datos antigua al desacoplar la carga de lectura. Acelera los tiempos de respuesta al tener los datos pre-transformados a formato FHIR. Se descarta la consulta directa por el riesgo crítico de saturar y botar los sistemas que operan el hospital.

#### Sacrificio (Trade-off)
Los datos en el *staging* presentan un desfase temporal. Exige categorizar qué datos (como el RVN) justifican una sincronización en "near real-time" y cuáles (historial antiguo) pueden usar procesos batch nocturnos.

---

## 9. RBAC vs. ABAC (Control de Acceso)

#### Conceptos a Estudiar
Modelos de control de accesos (RBAC, ABAC, DAC, MAC, ReBAC).

#### Problema a Resolver
Garantizar que el acceso a la información del paciente cumpla con la ley de protección de datos, yendo más allá de la simple validación del rol del profesional.

#### Alternativas
*   **RBAC (Role-Based Access Control):** Control por rol (ej. "Médico").
*   **ABAC (Attribute-Based Access Control) (Opción elegida):** Control por contexto.

#### Decisión y Beneficios (ABAC)
En salud, el rol "Médico" es insuficiente. ABAC evalúa el contexto: ¿el profesional está de turno?, ¿tiene una cita con el paciente? Esto bloquea accesos no autorizados y cumple estrictamente con la ley de privacidad.

#### Sacrificio (Trade-off)
Incrementa la complejidad. Depende de la disponibilidad de sistemas periféricos (agendas, admisión) para obtener los atributos en tiempo real.

---

## 10. Protocolo Break-Glass vs. Bloqueo Total

#### Conceptos a Estudiar
Sistemas de acceso en emergencias de salud, Protocolo Break-glass, emergency override, audit logging forense.

#### Problema a Resolver
Manejar situaciones de emergencia (ej. paciente inconsciente) donde los controles de acceso normales (ABAC) denegarían el acceso a información vital.

#### Alternativas
*   **Denegación estricta** y bloqueo total.
*   **Desbloqueo mediante aprobación manual de un supervisor.**
*   **Break-Glass automático con auditoría inmutable (Opción elegida).**

#### Decisión y Beneficios (Break-Glass)
La seguridad informática nunca debe comprometer la vida de un paciente. Este protocolo permite un acceso de emergencia controlado, temporal y justificado, garantizando la continuidad de la atención.

#### Sacrificio (Trade-off)
Introduce una excepción controlada a la seguridad. Requiere obligatoriamente mecanismos de auditoría no alterables (WORM logs) y monitoreo para disuadir y sancionar el abuso.

---

## 11. Hashing/Salt vs. Cifrado en el Índice

#### Conceptos a Estudiar
Criptografía aplicada a identificadores (Hashing, Salt, HMAC), Tokenización vs cifrado reversible, Pseudonimización.

#### Problema a Resolver
Proteger la identidad de los pacientes en el Índice Nacional de Descubrimiento, que solo debe enrutar peticiones, no almacenar datos personales legibles.

#### Alternativas
*   Almacenar el RUT en **texto plano**.
*   **Cifrado bidireccional** del RUT.
*   Aplicación de **Hash con Salt criptográfico (Opción elegida).**
*   **Plataformas de Tokenización centralizada.**

#### Decisión y Beneficios (Hash + Salt)
Minimiza la superficie de ataque. Ante una vulneración del Índice, los identificadores no son directamente legibles, limitando la reidentificación masiva. El Índice puede enrutar sin necesidad de conocer el RUT original.

#### Sacrificio (Trade-off)
La irreversibilidad del hash elimina la capacidad del sistema central para recuperar el dato. Implica una gestión robusta y segura del 'salt' (vía un KMS).

---

## 12. Circuit Breaker vs. Retries Infinitos

#### Conceptos a Estudiar
Patrones de resiliencia en microservicios y sistemas distribuidos (Circuit Breaker, Retries, Bulkhead pattern, Fallback, Timeout handling).

#### Problema a Resolver
Evitar fallas en cascada y tiempos de espera frustrantes para el usuario cuando un nodo de la red (un hospital) está lento o caído.

#### Alternativas
*   **Reintentos (Retries)** de conexión infinitos.
*   **Reintentos con exponencial backoff.**
*   **Circuit Breaker / Cortocircuito (Opción elegida).**
*   **Patrones de aislamiento Bulkhead.**

#### Decisión y Beneficios (Circuit Breaker)
Mitiga las fallas en cascada. Si un nodo remoto falla, el circuito se "abre" y las peticiones fallan rápidamente (*fail fast*), evitando que la aplicación del médico se congele y que la red se sature con reintentos inútiles.

#### Sacrificio (Trade-off)
Requiere una calibración precisa de los umbrales (tiempos de espera, porcentajes de error). Un mal ajuste podría bloquear temporalmente el acceso a un servicio que solo experimenta una fluctuación breve.

---

## 13. Cache/TTL vs. Lectura Siempre al Origen

#### Conceptos a Estudiar
Estrategias de rendimiento y latencia (Caching, TTL), Edge caching, invalidación de caché.

#### Problema a Resolver
Reducir la latencia y la dependencia de la red, especialmente en zonas rurales con mala conectividad, para que el sistema se sienta rápido y responsivo.

#### Alternativas
*   Obtener siempre los datos **desde la fuente de origen.**
*   **Edge Caching / Caché Local (Opción elegida).**
*   **Motores de Caché distribuidos a nivel nacional.**

#### Decisión y Beneficios (Edge Caching)
Es el mecanismo más efectivo para reducir la latencia en accesos repetitivos. Acerca los datos al usuario, mejorando drásticamente la experiencia y la resiliencia ante fallas de red.

#### Sacrificio (Trade-off)
Introduce el riesgo crítico de servir datos médicos obsoletos (ej. una alergia no actualizada). Exige políticas rigurosas de invalidación, TTLs cortos para datos críticos y mostrar siempre en la UI la "frescura" del dato cacheado.

---

## 14. Alta Disponibilidad (HA) vs. SPOF

#### Conceptos a Estudiar
Conceptos de HA, Redundancia, Multi-region architecture, Single Point of Failure (SPOF), Replicación Activa-Activa.

#### Problema a Resolver
Garantizar que los componentes centrales del sistema (Índice y RVN) no sean un Punto Único de Fallo (SPOF) y puedan sobrevivir a caídas de infraestructura.

#### Alternativas
*   Despliegue en un **nodo central único (SPOF).**
*   Replicación de contingencia **Activa-Pasiva.**
*   Arquitectura de **Replicación Activa-Activa / Multi-Región (Opción elegida).**

#### Decisión y Beneficios (Replicación Multi-Región)
Para una infraestructura crítica nacional, un SPOF es inaceptable. La replicación geográfica garantiza máxima tolerancia a fallos catastróficos, manteniendo el servicio operativo.

#### Sacrificio (Trade-off)
Eleva exponencialmente los costos de infraestructura (CapEx y OpEx). Introduce una alta complejidad técnica para mantener la consistencia de datos a través de múltiples zonas geográficas.

---

## 15. Chip NFC (TPIM) vs. Acceso Puramente en la Nube

#### Conceptos a Estudiar
Computación Offline-first, Estructuras de datos binarias (Protobuf memory footprint), Firmas digitales asimétricas (PKI), Capacidades NFC (NTAG216).

#### Problema a Resolver
Permitir que paramédicos accedan a información vital offline en emergencias, guiando además a los civiles presentes sobre cómo actuar (ej. ataque de epilepsia), garantizando al mismo tiempo que el personal médico no pueda abusar del sistema leyendo chips arbitrariamente fuera de turno.

#### Alternativas
*   **Modelo Israelí 100% Online:** Depender exclusivamente del RUT dictado por un familiar conectado a la nube central.
*   **Código QR con enlace web:** Impreso en carnet (Inútil sin señal de internet).
*   **Biometría móvil:** Lectura de huella por paramédicos conectada a base central (Alto costo de hardware y requiere internet).
*   **Token Físico NFC (TPIM) con RVN comprimido en Protobuf (Opción elegida).**

#### Decisión y Beneficios (TPIM NFC)
Implementa una arquitectura de memoria de doble zona (NDEF). La zona pública instruye al civil sobre el padecimiento específico (ganando minutos vitales). La zona privada usa Protobuf para encajar todo el RVN clínico en <500 bytes. Además, la aplicación de lectura extiende el ABAC: restringe la lectura privada solo a paramédicos en turno activo, forzando un Break-Glass auditable si intervienen estando fuera de servicio.

#### Sacrificio (Trade-off)
La zona pública expone intencionalmente la condición del paciente, requiriendo su consentimiento explícito previo. La información en el chip puede quedar desactualizada (mitigado por el "semáforo de frescura" 🟢🟡🔴 en la App). Además, no asume cobertura universal: ante pacientes turistas o no enrolados, el sistema hace un *fallback* directo al protocolo pre-hospitalario a ciegas tradicional, asegurando que el chip sea un acelerador de la atención, pero nunca un bloqueador.

---

## Orden de Estudio y Preparación de Defensa

1.  **Fundamentos:** Arquitectura distribuida e híbrida federada.
2.  **Integración local:** Patrones Sidecar y topologías Hub-and-Spoke.
3.  **Capa de red:** Diferencias técnicas entre REST y gRPC, JSON vs Protobuf.
4.  **Dominio médico:** Estructura FHIR y semántica SNOMED CT / LOINC.
5.  **Optimización legacy:** Operaciones ETL y almacenamiento Staging.
6.  **Ciberseguridad clínica:** Modelos ABAC, protocolo Break-glass y Hashing.
7.  **Resiliencia operativa:** Implementación de Cache, Circuit Breaker y mitigación de SPOF.
8.  **Edge / Pre-Hospitalario:** Protocolos NFC, compresión binaria extrema y seguridad de TPIM offline.