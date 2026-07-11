# Guía de Conceptos, Tecnologías y Justificación Técnica - Proyecto MIMF

Este documento justifica las decisiones arquitectónicas de la MIMF, separando los componentes esenciales de los accesorios, y definiendo la estrategia operativa real.

### Límites de Alcance (Anti-Objetivos de la MIMF)

*   No reemplaza los sistemas (EHR) existentes en los hospitales.
*   No unifica todo el historial clínico profundo en una base de datos central.
*   No resuelve automáticamente la calidad semántica del dato; si el nodo de origen no realiza un mapeo local correcto, el sistema no lo adivina.
*   No reimplementa de forma permanente el EMPI/MPI ni el HPD del MINSAL; el destino es consumirlos. El adaptador temporal de identidad es solo para piloto.
*   El Token Físico (NFC) NO reemplaza el sistema de red, es estrictamente una caché física offline de último recurso.

---

## 1. Arquitectura Base y Patrones de Diseño

### A) Arquitectura Híbrida (Red Federada + RVN)

*   **Rama / Concepto:** Arquitectura de Sistemas Distribuidos.
*   **¿Qué es?:** El 99% de la información (historia profunda) vive federada en los nodos locales. Sin embargo, existe un "Resumen Vital Nacional" (RVN - Alergias y Diagnósticos Críticos) que se aloja en un repositorio central.
*   **Profundización:** Se distribuye la carga pesada (imágenes, evoluciones diarias) manteniendo al hospital como "fuente de verdad". Se centraliza estrictamente el enrutamiento (Índice) y la urgencia (RVN). El RVN no pretende ser una ficha nacional: es un **recordatorio clínico de urgencia** (alergias, medicamentos críticos, grupo sanguíneo, diagnósticos vitales) más una señal de que existe historial profundo en otros nodos. El contenido del RVN es regido por un comité nacional clínico-técnico (MINSAL y Colegios Profesionales) que aprueba y publica nuevas versiones del esquema anualmente bajo estrictos criterios de evidencia médica. Si el centro cae, los hospitales operan en "Modo Degradado".
*   **Justificación:** Evita el costo inasumible de centralizar petabytes de datos legados, pero asegura disponibilidad inmediata de información crítica en urgencias. Enmarcar el RVN como alerta (y no como competencia del EHR) es la principal defensa contra el *scope creep* político de especialidades que quieran "meterse" al resumen central.
*   **Alternativas descartadas:**
    *   *Nube Centralizada Nacional (Migración Total):* Descartada por el alto costo de migración, resistencia institucional, y complejidad de mantener un modelo de datos único frente a sistemas heterogéneos.

### B) Patrón Sidecar (Como Appliance / Caja Negra)

*   **Rama / Concepto:** Patrones de Arquitectura Cloud-Native / Microservicios.
*   **¿Qué es?:** Un software auxiliar gestionado de forma centralizada (Appliance) que se despliega junto a la base legacy. El hospital local no tiene permisos para modificarlo.
*   **Profundización:** Desacopla la lógica de negocio (el sistema del hospital) de la lógica de plataforma (cómo comunicarse con la red nacional). El Sidecar **no es un binario universal** con todos los conectores del país: cada hospital recibe un ejecutable compilado con **Core + el conector de su familia de EHR** (Oracle, InterSystems, SAP, SQL Server, etc.). Si el estándar FHIR se actualiza, el Estado empuja la actualización OTA al Sidecar correspondiente.
*   **Interfaz oficial de conectores (`HospitalConnector`):** La MIMF publica un contrato de interfaz (estilo CSI/CNI de Kubernetes) con operaciones tipadas (`GetPatient`, `GetEncounter`, `GetMedication`, `GetAllergies`, `SearchPatient`, `MapFHIR`, etc.). Dos caminos de mantenimiento:
    1. El equipo MIMF desarrolla y mantiene el conector.
    2. El proveedor del EHR desarrolla su propio conector y lo certifica en el Sandbox estatal.
    Así el Core permanece estable y la carga de conocer el esquema interno de cada EHR no cae eternamente sobre un único equipo nacional.
*   **Justificación:** Los sistemas de los hospitales ("EHRs") son frágiles y cerrados (vendor lock-in). Tocar su código es inviable legal y técnicamente. El Sidecar actúa como un traductor externo no intrusivo. Compilar por hospital reduce RAM, dependencias, superficie de ataque y simplifica el testing.
*   **Alternativas descartadas:**
    *   *Refactorizar / Modificar el código de cada hospital:* Descartado por ser legalmente imposible (viola garantías de proveedores comerciales) y requerir esfuerzo humano inabarcable.
    *   *Sidecar monolítico con todos los plugins:* Descartado. Arrastra módulos fantasma, aumenta el riesgo de seguridad y complica el despliegue en servidores con poca RAM.

### C) Modo Degradado (Resiliencia Local)

*   **Rama / Concepto:** Ingeniería de Confiabilidad (SRE) / Tolerancia a Fallos.
*   **¿Qué es?:** Capacidad del sistema de seguir funcionando con funcionalidades limitadas (localmente) cuando la conexión a la red externa falla.
*   **Profundización:** Implementa una filosofía "Offline-First". Si la conexión a internet del hospital se corta, el Sidecar encola los eventos locales y los sincroniza automáticamente (Sync Queue) una vez que vuelve la conexión. Para cortes cortos (horas/días) basta el replay de la cola. Para desconexiones **prolongadas** (semanas/meses), la reincorporación usa **snapshot del estado relevante + replay controlado** (límites de tasa, priorización de datos del RVN, compresión), evitando saturar el Hub o el Índice con millones de eventos de golpe.
*   **Justificación:** En salud, la tolerancia a fallos es cero. Si la MIMF se cae, el médico debe poder seguir atendiendo usando la información de la base de datos local del hospital sin experimentar bloqueos ("freezes").

### D) Consistencia Clínica (Fuentes de Verdad)

*   **Rama / Concepto:** Gestión de Datos Maestros (MDM).
*   **¿Qué es?:** Definición de reglas para resolver conflictos cuando dos hospitales tienen datos divergentes sobre un mismo paciente.
*   **Justificación:** En un modelo federado, el hospital que generó el registro es el dueño legal y mantiene la autoría inmutable del dato (fuente de verdad). La política explícita del sistema es: ciertos tipos de datos tienen prioridad fija (ej. Alergias Críticas > Diagnósticos Históricos), el sistema expone visualmente el conflicto y su versión (timestamp) sin sobreescribir ni borrar nada, y delega la resolución final al profesional en su contexto clínico actual.

### E) Identidad Médica Físicamente Distribuida (NFC)

*   **Rama / Concepto:** Computación Ubicua / Dispositivos Edge.
*   **¿Qué es?:** El Token Físico de Identidad Médica (TPIM). Un chip NFC (NTAG216) portado por el paciente que actúa como el "nodo" más pequeño y externo de la red.
*   **Profundización:** Utiliza un sistema de **Doble Registro NDEF**. El Registro 1 es público (texto plano con instrucciones de primeros auxilios como "Epilepsia: poner de lado") y el Registro 2 es privado (Protobuf con un **subconjunto ultracrítico del RVN** comprimido, ej. alergias severas y grupo sanguíneo). Su emisión es gestionada centralmente (vía CESFAM/Clave Única). El chip se reescribe en dos instancias: (A) **Actualización activa (prioridad)**, cuando el paciente sale de una consulta en un establecimiento conectado a la MIMF, el Sidecar ofrece sincronizar el chip acercándolo a un lector del mesón — este es el camino principal, porque no se puede depender de que el paciente "recuerde" actualizar; (B) **Actualización pasiva** mediante kioscos de autoatención en CESFAM y farmacias de la red pública. Para mitigar el riesgo de datos obsoletos, la App paramédica lee la fecha del chip y aplica un semáforo de frescura visual (🟢 < 30 días, 🟡 30-90 días, 🔴 > 90 días).
*   **Justificación:** Protege al paciente en las dos fases de la emergencia (civil y profesional). El modelo de emisión institucional resuelve la viabilidad de distribución; la actualización en cada contacto clínico reduce el riesgo de chips "muertos"; y el semáforo visual transfiere la decisión de confianza en el dato offline al criterio clínico del paramédico.

---

## 2. Stack de Red y Descubrimiento

### A) Índice de Descubrimiento Clínico (Record Locator) + Identidad vía `PatientIdentityProvider`

*   **Rama / Concepto:** Sistemas Distribuidos / Record Locator Service + Master Patient Index + patrón Adapter / Ports & Adapters.
*   **Separación crítica:** La **identidad** (quién es el paciente) y el **descubrimiento** (dónde hay datos clínicos) son problemas distintos. El destino de identidad es el [EMPI/MPI del MINSAL](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/empi.html) (PIXm/PDQm). El **Índice de Descubrimiento de la MIMF** responde `ID canónico -> nodos/hospitales`.
*   **Contrato `PatientIdentityProvider`:** Interfaz estable que la malla usa para resolver RUN/ID local, cruzar identificadores, demografía mínima y fusiones. Dos backends:
    1. **Adaptador temporal (piloto):** misma interfaz; implementación acotada al alcance del piloto (RUN + reglas locales + merge/soporte). Permite avanzar sin bloquearse por el calendario de los IGs draft del MINSAL ([MPI](https://interoperabilidad.minsal.cl/fhir/ig/mpi/), [HPD](https://interoperabilidad.minsal.cl/fhir/ig/hpd/)).
    2. **EMPI nacional (régimen):** mismo contrato; backend = servicio oficial. Se **reemplaza solo el adaptador**; Record Locator, Sidecar, RVN y TPIM no se reescriben.
    Regla: acoplamiento al **contrato**, no al despliegue nacional. El adaptador temporal no es un segundo maestro permanente ni un UUID paralelo.
*   **¿Qué es el Índice MIMF?:** Directorio de enrutamiento clínico. Flujo: `RUT/ID local -> PatientIdentityProvider -> ID canónico -> Record Locator -> ¿en qué hospital(es) hay datos?`.
*   **Profundización:** Enrutamiento determinístico; estado reconstruible desde nodos; replicación multi-región y caché en Sidecars. Ante latencia/caída del proveedor de identidad: **caché de resoluciones** (`RUN/ID local → ID canónico`, TTL corto) + revalidación al volver. HMAC opcional solo para pseudonimizar claves del locator en reposo.
*   **Política de cambio de identificadores:** En régimen, fusiones/cambios de RUN las resuelve el EMPI; el locator se reindexa. En piloto, el merge lo gobierna soporte bajo el mismo contrato, con trazabilidad, y se migra al EMPI cuando esté disponible.
*   **Justificación:** La Ley 21.668 no garantiza que el EMPI esté operativo mañana. Bloquear la malla al calendario nacional hereda su congelamiento. Consumir el EMPI como destino y usar un adaptador temporal permite demostrar valor en piloto sin pelear el estándar futuro. El Record Locator sigue siendo necesario: el EMPI no responde dónde está el historial clínico.
*   **Alternativas descartadas:**
    *   *DHT Pura (Kademlia):* Descartada. Latencia impredecible.
    *   *RUT como identidad canónica de la red:* Descartado.
    *   *Usar solo el EMPI como índice de fichas:* Descartado.
    *   *HMAC(RUT) o UUID MIMF como maestro de identidad:* Descartados.
    *   *Esperar a que el EMPI/HPD nacionales estén 100% listos antes de cualquier piloto:* Descartado. Es dependencia de calendario, no de arquitectura.

### B) gRPC + Protocol Buffers

*   **Rama / Concepto:** Redes / Protocolos de Comunicación RPC.
*   **¿Qué es?:** gRPC es un framework RPC (Remote Procedure Call) de Google. Usa HTTP/2 y Protocol Buffers (binarios) en lugar de JSON para enviar datos.
*   **Profundización:** Al usar binarios en lugar de texto plano, el payload (peso del mensaje) se reduce drásticamente. Además, HTTP/2 permite multiplexar múltiples peticiones en una sola conexión TCP, reduciendo la latencia de "handshake". Se debe considerar que algunas redes hospitalarias antiguas con proxies o firewalls restrictivos pueden presentar problemas con HTTP/2; para estos casos, la arquitectura debe contemplar un mecanismo de fallback como gRPC-Web (que encapsula sobre HTTP/1.1) o una API REST equivalente para garantizar la conectividad.
*   **Justificación:** Ofrece menor overhead de red en la malla interna. **Habilitador Crítico NFC:** la compresión de Protobuf permite empaquetar el subconjunto ultracrítico del RVN dentro de los ~888 bytes de un chip NFC básico.
*   **Alternativas descartadas:**
    *   *API REST (con JSON) para el core interno Sidecar↔Hub:* Descartada por peso y latencia frente a binarios.
    *   *Reemplazar FHIR/REST en el borde MINSAL:* Descartado. La estrategia nacional ([estándares MINSAL](https://interoperabilidad.minsal.cl/docs/especificacion-de-la-arquitectura/estandares-perfiles.html)) usa FHIR R4 sobre REST/JSON hacia EMPI, HPD y terminología. gRPC es complemento interno, no un desafío al estándar oficial.

### C) Topología Hub-and-Spoke / Relays Controlados

*   **Rama / Concepto:** Topología de Red.
*   **¿Qué es?:** Comunicación enrutada a través de un servidor central de mensajería (Relay) o conexiones TCP persistentes salientes desde los hospitales.
*   **Profundización:** Para evitar que el Relay se convierta en un cuello de botella de rendimiento o un SPOF, su diseño debe ser inherentemente distribuido y escalable horizontalmente, utilizando balanceadores de carga y despliegues multi-región para distribuir el tráfico y garantizar alta disponibilidad.
*   **Justificación:** En la salud pública real, los hospitales están detrás de NATs simétricos y firewalls rígidos. Un modelo P2P puro es operativamente irrealizable. Usar un Relay/Hub manejado facilita el paso a través de redes seguras sin pedir a los hospitales que abran puertos de entrada.

---

## 3. Estándares Médicos e Interoperabilidad

### A) HL7 FHIR (Fast Healthcare Interoperability Resources)

*   **Rama / Concepto:** Informática Médica / Arquitectura de Datos y Estándares.
*   **¿Qué es?:** El estándar mundial actual para intercambio de datos de salud. Usa "Recursos" (JSON o XML) con estructuras predefinidas (ej. Patient, Observation).
*   **Perfil IPS (International Patient Summary):** Un subconjunto mínimo de FHIR (Alergias, Medicamentos, Problemas Activos) diseñado para urgencias.
*   **Profundización:** Es altamente modular. En lugar de enviar un documento gigante (paradigma antiguo HL7 v3), permite consultar solo el recurso específico necesario (RESTful). El **Perfil Chile** evoluciona por versiones. Cada Sidecar declara qué versiones soporta y puede operar en **coexistencia** (ej. v3 + v4 + v5). Las versiones antiguas entran en *End-of-Life* tras una ventana anunciada (ej. 6 meses), no con un corte forzado nacional de un día para otro. La negociación de versión ocurre en el handshake entre Sidecars / Hub.
*   **Justificación:** Evita el "efecto Torre de Babel". Todos los Sidecars traducen la base de datos local a este JSON estándar antes de enviarlo por la red. Forzar "todos a vN mañana" rompería hospitales rezagados; la coexistencia con EOL es el único camino operable a escala país.
*   **Alternativas descartadas:**
    *   *HL7 v2 / HL7 v3 (CDA):* Descartados por ser estándares antiguos, basados en documentos estáticos pesados (XML rígidos), difíciles de parsear y poco amigables con arquitecturas web modernas.
    *   *Actualización global forzada inmediata:* Descartada como política operativa. Es correcta como meta de gobernanza, pero peligrosa como mecanismo de despliegue sin ventana de coexistencia.

### B) SNOMED CT y LOINC (Interoperabilidad Semántica)

*   **Rama / Concepto:** Informática Médica / Ontologías y Semántica de Datos.
*   **¿Qué es?:** Diccionarios médicos estandarizados. SNOMED codifica diagnósticos (ej. Infarto agudo = 22298006), LOINC codifica exámenes de laboratorio.
*   **Profundización:** Las bases de datos guardan texto libre ("dolor de guata") que un computador no entiende. Estos diccionarios establecen relaciones jerárquicas conceptuales para que las máquinas puedan cruzar datos clínicamente válidos.
*   **Justificación:** FHIR asegura que el JSON esté bien formado, pero SNOMED asegura que "Ataque al corazón" en el Hospital A signifique exactamente lo mismo que "Infarto Agudo de Miocardio" en el Hospital B.
*   **Alternativas descartadas:**
    *   *CIE-10 (CIE-11):* Descartado para el core clínico porque CIE es un estándar diseñado para facturación y estadística de mortalidad, no para capturar el detalle clínico granular que necesita un médico en la consulta.

### C) ETL Asíncrono y Área de Staging

*   **Rama / Concepto:** Ingeniería de Datos / Procesamiento Batch.
*   **¿Qué es?:** Extraer (E), Transformar (T) y Cargar (L) datos desde la base legacy hacia una base temporal ultrarrápida (Staging) en formato FHIR. El riesgo de inconsistencia temporal (datos obsoletos) se mitiga con una estrategia de sincronización diferenciada.
*   **Profundización:** Utiliza un enfoque híbrido: sincronización "near real-time" (vía eventos o micro-batches de minutos) para datos críticos del RVN (ej. nuevas alergias), y procesos Batch nocturnos para el historial profundo. Esta separación es clave para balancear consistencia y rendimiento, asegurando que la información vital esté actualizada sin sobrecargar los sistemas legacy con consultas constantes.
*   **Contrato de esquema (no automatizar lo imposible):** Si un hospital renombra `Paciente.Nombre` a `Paciente.NombreCompleto`, ningún algoritmo sabe si fue un rename, un split o un cambio semántico. El contrato de interoperabilidad **obliga a informar previamente** cambios estructurales. La actualización del conector la hace soporte MIMF o el proveedor certificado del conector, en paralelo al cambio del hospital. Automatizar la "auto-reparación" del mapeo se descarta: rompería el Sidecar con datos incorrectos silenciosos.
*   **Justificación (Hardware-Awareness):** Si consultas directamente una base de datos Oracle del año 2008 en medio de una urgencia, la botas. El staging protege la base antigua y garantiza lecturas instantáneas.

---

## 4. Resiliencia y Rendimiento Distribuido

### A) Control de Carga (Backpressure)

*   **Rama / Concepto:** Arquitectura de Sistemas Resilientes.
*   **¿Qué es?:** Mecanismo para evitar que peticiones masivas ahoguen a un servidor legacy.
*   **Justificación:** En lugar de acoplar un broker pesado (como Kafka) en cada nodo local, se maneja limitación de peticiones directas (Rate Limiting) en el Sidecar.

### B) Circuit Breaker (Cortocircuito)

*   **Rama / Concepto:** Patrones de Resiliencia de Microservicios.
*   **¿Qué es?:** Patrón de diseño que, si detecta que un servicio remoto (Hospital B) está caído o lento, deja de intentar conectar y devuelve un error inmediato ("circuito abierto").
*   **Profundización:** Tiene tres estados: Cerrado (normal), Abierto (bloqueando tráfico tras fallos). Establece umbrales operativos de diseño (SLAs objetivo): < 3s para la carga vital (RVN) y un umbral aceptable de 5-10s para la carga profunda (historial) mediante carga progresiva (lazy loading visual).
*   **Justificación:** Evita la "falla en cascada" y los "tiempos de espera infinitos" (loading screens interminables) que frustran a los médicos.

### C) Edge Caching + Flag de Frescura + TTL agresivo

*   **Rama / Concepto:** Optimización de Rendimiento / Sistemas de Memoria Caché.
*   **¿Qué es?:** Guardar respuestas recientes en memoria temporal (Caché). TTL (Time-To-Live) es el tiempo de vida de ese dato.
*   **Profundización:** Acerca los datos a quien los consume ("Data Locality"). El problema es la invalidación de la caché. Al forzar un TTL corto y un flag visual, se delega el juicio clínico final al médico, previniendo errores por latencia de sincronización.
*   **Justificación:** Mejora la velocidad. Sin embargo, en salud, mostrar alergias cacheadas hace 3 días puede matar a un paciente. Por eso el TTL debe ser de minutos, y se debe mostrar en la pantalla del médico a qué hora exacta se extrajo ese dato ("Flag").

---

## 5. Seguridad y Cumplimiento Legal (Gov-Tech)

### A) ABAC (Attribute-Based Access Control)

*   **Rama / Concepto:** Ciberseguridad / Gestión de Identidad y Accesos (IAM).
*   **¿Qué es?:** Control de acceso que no solo evalúa "quién eres" (Rol: Médico), sino el "contexto" (Atributo: ¿Estás de turno? ¿Este paciente tiene cita hoy contigo?).
*   **Profundización:** Supera al clásico RBAC (Roles) evaluando dinámicamente políticas basadas en Sujeto, Recurso, Acción y Entorno. En la práctica, requiere integración con los sistemas locales de agenda/admisión/RRHH (y, en terreno, turnos SAMU) para obtener los atributos en tiempo real. El Sidecar y los clientes **solo leen** esos atributos; no escriben ni crean registros en los sistemas del hospital. Ante la falta de atributos, aplica un fallback conservador "Deny by Default", obligando a usar el protocolo Break-Glass. Para no imponer las apps oficiales a todos, la MIMF expone **SDK / API / OAuth** para que un proveedor integre ABAC dentro de su propio EHR. El principal riesgo operacional es la dependencia de la calidad y disponibilidad de estas fuentes de atributos; si fallan, se corre el riesgo de un aumento en denegaciones incorrectas y un abuso del protocolo Break-Glass como solución informal, convirtiendo una medida de seguridad en una puerta trasera.
*   **Justificación:** Cumplimiento estricto de la Ley 19.628 (Protección de Datos). Bloquea accesos de personal médico curioso a fichas de famosos o familiares. Leer sin escribir reduce la probabilidad de romper sistemas ajenos.
*   **Alternativas descartadas:**
    *   *RBAC puro (Control por Roles):* Descartado. RBAC dice "todos los médicos pueden ver todo". En salud, eso es una violación de privacidad; el médico solo debe ver la ficha de *sus pacientes actuales*.
    *   *Forzar una única app clínica nacional:* Descartado. Muchos hospitales querrán integrar el control en su propio software; el SDK/API es el camino de adopción.

### B) Protocolo Break-Glass ("Romper el cristal")

*   **Rama / Concepto:** Ciberseguridad / Gestión de Emergencias Críticas.
*   **¿Qué es?:** Mecanismo para saltar el control ABAC exclusivamente en riesgo vital.
*   **Controles de Uso:** Solo personal médico autorizado puede activarlo, tiene un límite de tiempo activo (ej. 4 horas), exige ingresar un motivo justificado en la interfaz, y genera automáticamente un ticket de auditoría para revisión del comité de ética.
*   **Justificación:** Un sistema rígido es peligroso. El Break-Glass es necesario, pero sin estos controles estrictos se convierte en una brecha de seguridad.

### C) Log de Auditoría Append-Only Inmutable

*   **Rama / Concepto:** Ciberseguridad / Auditoría, Compliance y Análisis Forense.
*   **¿Qué es?:** Una base de datos (o archivo) donde solo se pueden "agregar" (append) eventos. No permite UPDATE ni DELETE. Además, lleva un sello de tiempo y firma criptográfica.
*   **Profundización:** Concepto WORM (Write Once, Read Many). Usa hashing criptográfico (como el árbol de Merkle) para asegurar que si un administrador de bases de datos borra un registro, la cadena completa se invalida. Dado el volumen masivo de eventos, se debe implementar una estrategia de ciclo de vida de datos que contemple el archivado en almacenamiento de bajo costo (cold storage) y sistemas de indexación eficientes para permitir búsquedas forenses sin degradar el rendimiento.
*   **Justificación:** Trazabilidad legal. Si hay una demanda por negligencia o filtración (Ley 21.668), el Ministerio de Salud necesita pruebas irrefutables de quién vio qué.
*   **Alternativas descartadas:**
    *   *Blockchain / Smart Contracts:* Descartado rotundamente por vender humo. Introduce latencia, altos costos computacionales de consenso y complejidad innecesaria para un entorno donde el Estado ya actúa como entidad central de confianza.

### D) Cifrado y Protección en Tránsito

*   **Rama / Concepto:** Ciberseguridad / Criptografía.
*   **¿Qué es?:** En lugar de prometer un E2EE (Cifrado Extremo a Extremo) irrealizable en este ecosistema, se implementa cifrado en tránsito estricto (TLS 1.3) entre los nodos y el hub central, y cifrado en reposo para cachés y áreas de staging locales.

### E) Seguridad en Terreno (TPIM / NFC)

*   **Rama / Concepto:** Seguridad Física e Integridad de Dispositivos Periféricos.
*   **¿Qué es?:** Mecanismos para evitar clonación, alteración o lectura maliciosa de los chips de pacientes.
*   **Profundización:** 
    1. **Cifrado Semántico:** La Zona Privada usa SNOMED CT (números incomprensibles sin la base de datos oficial).
    2. **Firma Criptográfica:** El Protobuf está firmado (PKI) por el RVN Central para invalidar chips adulterados.
    3. **ABAC en App Móvil:** La lectura de la Zona Privada exige que el paramédico autenticado esté **de turno activo** en el sistema. Si está fuera de turno, la lectura se bloquea para evitar espionaje.
    4. **Break-Glass Móvil:** Si un médico fuera de turno interviene en un accidente real, puede forzar la lectura activando el Break-Glass en la App. La auditoría de este evento (GPS, usuario, motivo) se almacena de forma segura en el dispositivo móvil y se sincroniza con el servidor central en cuanto recupera la conectividad, garantizando la trazabilidad incluso en escenarios offline.

---

## 6. Gobernanza, Estrategia Política y de Adopción

*   **Interoperabilidad Forzada:** Basado en la Ley 21.668. Los hospitales y proveedores deben cumplir normativas vinculadas a incentivos y restricciones presupuestarias. Quien quiera vender EHR al Estado implementa el Perfil Chile FHIR; no es opcional.
*   **APIs Obligatorias y Conectores Certificables:** Corta el monopolio ("vendor lock-in") exigiendo a los privados pasar por un Sandbox estatal. El proveedor puede certificar su propio conector bajo `HospitalConnector`, asumiendo la responsabilidad de mantenerlo cuando cambie su esquema interno.
*   **Contrato de Esquema:** Obligación contractual de notificar cambios estructurales del EHR/base local antes de aplicarlos en producción, para actualizar el conector en paralelo.
*   **Estrategia de Transición:** Contempla coexistencia e integración parcial, con período de transición explícito (un hospital no puede quedar sin sistema mientras migra). Si un proveedor grande se niega temporalmente, el sistema no se bloquea; aísla al nodo no conforme, manteniendo el valor de la red para los demás hospitales.
*   **Naturaleza del proyecto:** La MIMF es un **ecosistema / infraestructura nacional** (Sidecar, Record Locator, RVN, TPIM, apps, PKI, KMS, OTA, Sandbox, observabilidad), no "una aplicación". Se alinea y **consume** componentes oficiales del MINSAL ([Arquitectura Nacional](https://interoperabilidad.minsal.cl/docs/especificacion-de-la-arquitectura/arquitectura.html)): EMPI/MPI, HPD y Servicios Terminológicos vía el NID; no los reemplaza. Hasta que esos servicios estén operativos en terreno, opera con adaptadores temporales detrás de contratos estables (`PatientIdentityProvider`, y HPD cuando aplique).

---

## 7. Observabilidad

### A) OpenTelemetry (Tracing Distribuido)

*   **Rama / Concepto:** DevOps / SRE (Site Reliability Engineering).
*   **¿Qué es?:** Un estándar para recolectar métricas (uso de CPU), logs (errores) y traces (el camino que recorre una petición a través de múltiples hospitales).
*   **Profundización:** Unifica los "Tres Pilares de la Observabilidad". Asigna un TraceID único a la petición del médico, permitiendo ver exactamente cuántos milisegundos pasó en el Índice, cuántos en la red y cuántos en el Sidecar B.
*   **Justificación:** En un sistema descentralizado, si la petición es lenta, no sabes si es culpa de la red local, del Índice, del Relay o de la base de datos de destino. El Tracing te dibuja el mapa exacto de dónde está el cuello de botella en tiempo real.

---

## 8. Tecnologías Sugeridas para el Desarrollo (PoC / MVP)

*   **Lenguaje del Sidecar:** Go (Golang) o Rust.
    *   **¿Por qué?:** Se compilan en un único archivo binario (.exe o ejecutable de Linux). No requieren instalar entornos complejos (como Node.js, Python o Java) en servidores viejos de hospitales. Consumen ínfimas cantidades de RAM.
*   **Alternativas descartadas para el Sidecar:** Java, Python, Node.js.
    *   Requieren instalar runtimes pesados (JVM, V8). Muchos servidores de hospitales rurales corren Windows Server 2008 con 2GB de RAM; no soportan una máquina virtual Java.
*   **Lenguaje del Índice Central / Frontend API:** Node.js (TypeScript) o Go.
*   **Base de Datos del Índice:** Redis (como capa de caché de alto rendimiento) respaldado por una base de datos persistente (ej. PostgreSQL) como fuente de verdad para la recuperación ante desastres.
*   **Cache Local:** Redis o SQLite en memoria.
*   **Log Inmutable:** PostgreSQL (con restricciones de permisos a nivel base de datos) o una base optimizada para Time-Series.

---

## 9. Glosario / Diccionario de Términos

*   **ABAC (Attribute-Based Access Control):** Control de acceso dinámico basado en atributos y contexto (ej. turno del médico, relación con el paciente), superior al clásico rol estático.
*   **API REST:** Interfaz HTTP + JSON/XML. En la MIMF se descarta para el *core interno* Sidecar↔Hub a favor de gRPC; se mantiene obligatoria en el borde hacia componentes oficiales MINSAL (EMPI, HPD, terminología) bajo FHIR R4.
*   **Appliance:** Dispositivo o software preconfigurado (como el Sidecar) que se despliega "llave en mano" y es gestionado centralizadamente sin que el cliente local lo modifique.
*   **Backpressure / Rate Limiting:** Mecanismo de control de flujo que limita la cantidad de peticiones concurrentes para evitar que un sistema legacy colapse por sobrecarga.
*   **Break-Glass ("Romper el cristal"):** Protocolo de seguridad que permite saltar controles de acceso (ABAC) en casos de riesgo vital, dejando una estricta traza de auditoría.
*   **CapEx / OpEx:** Gasto en inversión de capital (comprar servidores físicos) vs. Gasto operativo (pagar suscripciones de nube o red mensual).
*   **Circuit Breaker (Cortocircuito):** Patrón de microservicios que bloquea las conexiones a un nodo fallido para evitar fallas en cascada y tiempos de espera de red infinitos.
*   **DDoS (Distributed Denial of Service):** Ataque o falla arquitectónica masiva que satura un servidor con peticiones simultáneas, botando el servicio.
*   **DHT (Distributed Hash Table):** Patrón P2P (como Kademlia) descartado para la MIMF por sus tiempos de respuesta y latencia impredecibles al buscar información crítica.
*   **Edge Caching:** Almacenamiento temporal de datos de red lo más cerca posible del consumidor final (caché local del Sidecar) para mitigar la latencia de internet.
*   **EHR (Electronic Health Record):** Ficha Clínica Electrónica. El software principal que utiliza el hospital para gestionar a los pacientes.
*   **EMPI / MPI (Enterprise Master Patient Index):** Componente oficial del MINSAL para identidad demográfica unívoca del paciente. Destino definitivo del `PatientIdentityProvider`. Ver [EMPI](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/empi.html) e [IG MPI](https://interoperabilidad.minsal.cl/fhir/ig/mpi/).
*   **ETL / Staging:** Proceso de Extraer, Transformar y Cargar datos desde una base de datos antigua a una temporal (Staging Area) en formato FHIR para lecturas ultrarrápidas.
*   **FHIR (Fast Healthcare Interoperability Resources):** Estándar médico internacional moderno de HL7. El MINSAL adopta **FHIR R4** como estándar sintáctico nacional.
*   **gRPC:** Framework RPC de Google (HTTP/2 + Protocol Buffers). En la MIMF se usa en la malla interna; no reemplaza FHIR/REST hacia el Estado.
*   **HA (High Availability) / SLA:** Alta Disponibilidad de un sistema, garantizada por un Acuerdo de Nivel de Servicio (ej. 99.9% de uptime).
*   **HL7 (Health Level Seven):** Organización internacional y conjunto de estándares (v2, v3, CDA, FHIR) para el intercambio de información clínica.
*   **HPD (Healthcare Provider Directory):** Directorio nacional de prestadores. Enchufable a ABAC cuando esté operativo; no bloquea el piloto. Ver [IG HPD](https://interoperabilidad.minsal.cl/fhir/ig/hpd/).
*   **HospitalConnector:** Interfaz oficial de conectores de la MIMF hacia un EHR.
*   **Hub-and-Spoke / Relay:** Topología de red con intermediario gestionado.
*   **Índice Nacional de Descubrimiento / Record Locator:** Directorio MIMF `ID canónico -> hospitales con datos clínicos`. No es el EMPI.
*   **IPS (International Patient Summary):** Subconjunto FHIR de información vital mínima para atención no programada.
*   **KMS (Key Management Service):** Gestión de claves criptográficas (incl. pseudonimización opcional del locator).
*   **NID (Núcleo de Interoperabilidad de Datos):** Guía/núcleo MINSAL (MPI + HPD) y casos transversales.
*   **PatientIdentityProvider:** Contrato estable de identidad de paciente en la MIMF. Backend temporal (piloto) o EMPI nacional (régimen); se intercambia el adaptador sin reescribir la malla.
*   **Lazy Loading (Carga Progresiva):** Estrategia de red y UI donde la carga pesada (historial profundo) solo se descarga desde los hospitales remotos si el médico lo pide explícitamente.
*   **Legacy System (Sistema Legado):** Sistemas antiguos, frágiles pero críticos, que actualmente operan en los hospitales y que la MIMF integra sin intentar reemplazar de golpe.
*   **LOINC:** Estándar internacional enfocado en la codificación de pruebas, mediciones y observaciones de laboratorio clínico.
*   **Middleware:** Software que actúa como puente o "traductor" entre dos aplicaciones diferentes para que puedan intercambiar datos (rol que cumple el Sidecar).
*   **MIMF:** Malla de Interoperabilidad Médica Federada.
*   **MINSAL:** Ministerio de Salud de Chile. Entidad encargada de normar la gobernanza clínica.
*   **Modo Degradado (Offline-First):** Estado del Sidecar cuando el hospital pierde internet nacional. Permite continuar atenciones locales y encola los datos hasta recuperar conexión.
*   **NAT (Network Address Translation):** Tecnología de redes corporativas que oculta las IP internas, dificultando las conexiones directas P2P entre hospitales.
*   **OpenTelemetry / Tracing Distribuido:** Estándar de observabilidad que añade un ID a cada petición clínica para rastrear y medir en qué punto exacto de la red hay lentitud o errores.
*   **OTA (Over-The-Air):** Actualizaciones enviadas de forma remota. El Estado actualiza los Sidecars OTA sin requerir personal TI en cada hospital rural.
*   **Protocol Buffers (Protobuf):** Mecanismo de Google para serializar datos estructurados en formato binario, siendo mucho más pequeño y veloz de transmitir que el XML o JSON.
*   **RBAC (Role-Based Access Control):** Control de acceso básico según "cargo" (ej. "Rol Médico"). Insuficiente en salud por violar la privacidad de atenciones no asignadas.
*   **RVN (Resumen Vital Nacional):** Componente central de la MIMF. Repositorio de respuesta inmediata (< 3s) con alergias, medicamentos y diagnósticos críticos. Actúa como alerta de urgencia, no como ficha clínica nacional.
*   **Salt:** Texto aleatorio añadido a un dato antes de aplicar una función hash criptográfica, evitando ataques de reidentificación de RUTs o fuerza bruta.
*   **Sandbox Estatal:** Entorno de pruebas técnico obligatorio donde los proveedores privados de software demuestran que cumplen el estándar (y, si aplica, certifican su conector) antes de venderle al Estado.
*   **Sidecar (Patrón):** Arquitectura donde un agente auxiliar se despliega junto a un sistema base para interceptar, traducir y enrutar datos sin modificar el código legacy. En la MIMF se compila por hospital (Core + conector específico).
*   **Snapshot + Replay:** Estrategia de reincorporación tras desconexiones prolongadas: se sincroniza un estado base (snapshot) y luego se reproducen eventos pendientes a tasa controlada.
*   **SIDRA:** Sistemas de Información de la Red Asistencial. Estrategia histórica en Chile que logró digitalizar hospitales, pero generó "islas digitales" inconexas.
*   **SNOMED CT:** Nomenclatura médica mundial de terminología clínica (diagnósticos, hallazgos). Permite que la interoperabilidad sea semánticamente idéntica.
*   **Soft Merge (Datos Reconciliados):** Enfoque que muestra versiones conflictivas de un diagnóstico (con fechas y orígenes) para que el médico decida, sin borrar registros ajenos.
*   **SPOF (Single Point of Failure):** Punto Único de Fallo. Componente que, de caer, apaga toda la red. La MIMF lo evita replicando el Índice y usando cachés de emergencia.
*   **TLS 1.3:** Protocolo criptográfico de transporte que cifra de manera robusta los datos médicos mientras viajan por internet ("cifrado en tránsito").
*   **TPIM (Token Físico de Identidad Médica):** Componente NFC de la MIMF portado por el paciente. Almacena offline un resumen vital codificado en Protobuf para emergencias de primeros respondedores.
*   **TTL (Time To Live):** Tiempo de vida útil de un dato almacenado en el caché antes de forzar al sistema a ir a buscar la versión más nueva al hospital de origen (o a revalidar una resolución EMPI cacheada).
*   **Vendor Lock-in (Cautividad):** Cuando una empresa o Estado queda atrapado con un solo proveedor de software porque el costo y formato de extracción de datos hace inviable migrar.
*   **WORM (Write Once, Read Many) / Log Inmutable:** Base de datos de auditoría temporal donde los accesos se registran como bloques añadidos al final, impidiendo alteraciones forenses.