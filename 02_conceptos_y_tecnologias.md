# Guía de Conceptos, Tecnologías y Justificación Técnica - Proyecto MIMF

Este documento justifica las decisiones arquitectónicas de la MIMF, separando los componentes esenciales de los accesorios, y definiendo la estrategia operativa real.

### Límites de Alcance (Anti-Objetivos de la MIMF)

*   No reemplaza los sistemas (EHR) existentes en los hospitales.
*   No unifica todo el historial clínico profundo en una base de datos central.
*   No resuelve automáticamente la calidad semántica del dato; si el nodo de origen no realiza un mapeo local correcto, el sistema no lo adivina.

---

## 1. Arquitectura Base y Patrones de Diseño

### A) Arquitectura Híbrida (Red Federada + RVN)

*   **Rama / Concepto:** Arquitectura de Sistemas Distribuidos.
*   **¿Qué es?:** El 99% de la información (historia profunda) vive federada en los nodos locales. Sin embargo, existe un "Resumen Vital Nacional" (RVN - Alergias y Diagnósticos Críticos) que se aloja en un repositorio central.
*   **Profundización:** Se distribuye la carga pesada (imágenes, evoluciones diarias) manteniendo al hospital como "fuente de verdad". Se centraliza estrictamente el enrutamiento (Índice) y la urgencia (RVN). El contenido del RVN es regido por un comité nacional clínico-técnico (MINSAL y Colegios Profesionales) que aprueba y publica nuevas versiones del esquema anualmente bajo estrictos criterios de evidencia médica. Si el centro cae, los hospitales operan en "Modo Degradado".
*   **Justificación:** Evita el costo inasumible de centralizar petabytes de datos legados, pero asegura disponibilidad inmediata de información crítica en urgencias.
*   **Alternativas descartadas:**
    *   *Nube Centralizada Nacional (Migración Total):* Descartada por el alto costo de migración, resistencia institucional, y complejidad de mantener un modelo de datos único frente a sistemas heterogéneos.

### B) Patrón Sidecar (Como Appliance / Caja Negra)

*   **Rama / Concepto:** Patrones de Arquitectura Cloud-Native / Microservicios.
*   **¿Qué es?:** Un software auxiliar gestionado de forma centralizada (Appliance) que se despliega junto a la base legacy. El hospital local no tiene permisos para modificarlo.
*   **Profundización:** Desacopla la lógica de negocio (el sistema del hospital) de la lógica de plataforma (cómo comunicarse con la red nacional). Si el estándar FHIR se actualiza, el Estado empuja la actualización OTA (Over-The-Air) al Sidecar.
*   **Justificación:** Los sistemas de los hospitales ("EHRs") son frágiles y cerrados (vendor lock-in). Tocar su código es inviable legal y técnicamente. El Sidecar actúa como un traductor externo no intrusivo.
*   **Alternativas descartadas:**
    *   *Refactorizar / Modificar el código de cada hospital:* Descartado por ser legalmente imposible (viola garantías de proveedores comerciales) y requerir esfuerzo humano inabarcable.

### C) Modo Degradado (Resiliencia Local)

*   **Rama / Concepto:** Ingeniería de Confiabilidad (SRE) / Tolerancia a Fallos.
*   **¿Qué es?:** Capacidad del sistema de seguir funcionando con funcionalidades limitadas (localmente) cuando la conexión a la red externa falla.
*   **Profundización:** Implementa una filosofía "Offline-First". Si la conexión a internet del hospital se corta, el Sidecar encola los eventos locales y los sincroniza automáticamente (Sync Queue) una vez que vuelve la conexión.
*   **Justificación:** En salud, la tolerancia a fallos es cero. Si la MIMF se cae, el médico debe poder seguir atendiendo usando la información de la base de datos local del hospital sin experimentar bloqueos ("freezes").

### D) Consistencia Clínica (Fuentes de Verdad)

*   **Rama / Concepto:** Gestión de Datos Maestros (MDM).
*   **¿Qué es?:** Definición de reglas para resolver conflictos cuando dos hospitales tienen datos divergentes sobre un mismo paciente.
*   **Justificación:** En un modelo federado, el hospital que generó el registro es el dueño legal y mantiene la autoría inmutable del dato (fuente de verdad). La política explícita del sistema es: ciertos tipos de datos tienen prioridad fija (ej. Alergias Críticas > Diagnósticos Históricos), el sistema expone visualmente el conflicto y su versión (timestamp) sin sobreescribir ni borrar nada, y delega la resolución final al profesional en su contexto clínico actual.

---

## 2. Stack de Red y Descubrimiento

### A) Índice Híbrido Centralizado (Modelo X-Road)

*   **Rama / Concepto:** Sistemas Distribuidos / Motores de Búsqueda y Enrutamiento.
*   **¿Qué es?:** Un directorio nacional. No guarda la ficha, solo metadata: `hash(RUT+salt) -> ¿En qué hospital(es) hay datos?`
*   **Profundización:** A diferencia de una red P2P pura (como un torrent) donde la búsqueda es lenta, el índice central ofrece enrutamiento determinístico pero es inherentemente "stateless-ish" (su estado es reconstruible consultando a los nodos). No guarda datos clínicos. Para evitar ser un SPOF, usa replicación multi-región y caché distribuido en Sidecars. Una falla de red centralizada no significa una caída clínica local.
*   **Justificación:** La consistencia eventual es inaceptable en salud. El índice central garantiza respuestas predecibles.
*   **Alternativas descartadas:**
    *   *DHT Pura (Kademlia):* Descartada. La latencia de búsqueda impredecible y las "rutas rotas" por nodos inestables lo hacen inviable para escenarios críticos.

### B) gRPC + Protocol Buffers

*   **Rama / Concepto:** Redes / Protocolos de Comunicación RPC.
*   **¿Qué es?:** gRPC es un framework RPC (Remote Procedure Call) de Google. Usa HTTP/2 y Protocol Buffers (binarios) en lugar de JSON para enviar datos.
*   **Profundización:** Al usar binarios en lugar de texto plano, el payload (peso del mensaje) se reduce drásticamente. Además, HTTP/2 permite multiplexar múltiples peticiones en una sola conexión TCP, reduciendo la latencia de "handshake".
*   **Justificación:** Ofrece menor overhead de red, hace un mejor uso de conexiones persistentes y garantiza alta eficiencia en entornos de bajo ancho de banda, haciéndolo infinitamente más defendible que una API REST estándar.
*   **Alternativas descartadas:**
    *   *API REST (con JSON):* Descartada para la comunicación interna de red por el alto peso de los payloads (texto plano) y la lentitud al serializar/deserializar grandes volúmenes de datos clínicos en comparación con binarios.

### C) Topología Hub-and-Spoke / Relays Controlados

*   **Rama / Concepto:** Topología de Red.
*   **¿Qué es?:** Comunicación enrutada a través de un servidor central de mensajería (Relay) o conexiones TCP persistentes salientes desde los hospitales.
*   **Justificación:** En la salud pública real, los hospitales están detrás de NATs simétricos y firewalls rígidos. Un modelo P2P puro es operativamente irrealizable. Usar un Relay/Hub manejado facilita el paso a través de redes seguras sin pedir a los hospitales que abran puertos de entrada.

---

## 3. Estándares Médicos e Interoperabilidad

### A) HL7 FHIR (Fast Healthcare Interoperability Resources)

*   **Rama / Concepto:** Informática Médica / Arquitectura de Datos y Estándares.
*   **¿Qué es?:** El estándar mundial actual para intercambio de datos de salud. Usa "Recursos" (JSON o XML) con estructuras predefinidas (ej. Patient, Observation).
*   **Perfil IPS (International Patient Summary):** Un subconjunto mínimo de FHIR (Alergias, Medicamentos, Problemas Activos) diseñado para urgencias.
*   **Profundización:** Es altamente modular. En lugar de enviar un documento gigante (paradigma antiguo HL7 v3), permite consultar solo el recurso específico necesario (RESTful).
*   **Justificación:** Evita el "efecto Torre de Babel". Todos los Sidecars traducen la base de datos local a este JSON estándar antes de enviarlo por la red.
*   **Alternativas descartadas:**
    *   *HL7 v2 / HL7 v3 (CDA):* Descartados por ser estándares antiguos, basados en documentos estáticos pesados (XML rígidos), difíciles de parsear y poco amigables con arquitecturas web modernas.

### B) SNOMED CT y LOINC (Interoperabilidad Semántica)

*   **Rama / Concepto:** Informática Médica / Ontologías y Semántica de Datos.
*   **¿Qué es?:** Diccionarios médicos estandarizados. SNOMED codifica diagnósticos (ej. Infarto agudo = 22298006), LOINC codifica exámenes de laboratorio.
*   **Profundización:** Las bases de datos guardan texto libre ("dolor de guata") que un computador no entiende. Estos diccionarios establecen relaciones jerárquicas conceptuales para que las máquinas puedan cruzar datos clínicamente válidos.
*   **Justificación:** FHIR asegura que el JSON esté bien formado, pero SNOMED asegura que "Ataque al corazón" en el Hospital A signifique exactamente lo mismo que "Infarto Agudo de Miocardio" en el Hospital B.
*   **Alternativas descartadas:**
    *   *CIE-10 (CIE-11):* Descartado para el core clínico porque CIE es un estándar diseñado para facturación y estadística de mortalidad, no para capturar el detalle clínico granular que necesita un médico en la consulta.

### C) ETL Asíncrono y Área de Staging

*   **Rama / Concepto:** Ingeniería de Datos / Procesamiento Batch.
*   **¿Qué es?:** Extraer (E), Transformar (T) y Cargar (L) datos desde la base legacy hacia una base temporal ultrarrápida (Staging) en formato FHIR.
*   **Profundización:** Utiliza un enfoque híbrido: sincronización "near real-time" (vía eventos) para el RVN, y procesos Batch nocturnos para el historial antiguo. Para evitar la sobreingeniería y el fracaso temprano, el diseño prioriza la simplicidad operativa en fases iniciales (comenzando exclusivamente con Batch) e introduce la complejidad del "near real-time" solo en fases maduras del proyecto.
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
*   **Profundización:** Supera al clásico RBAC (Roles) evaluando dinámicamente políticas basadas en Sujeto, Recurso, Acción y Entorno. En la práctica, requiere integración con los sistemas locales de agenda/admisión para obtener los atributos en tiempo real (ej. ¿está de turno?, ¿el paciente ingresó hoy?). Ante la falta de atributos, aplica un fallback conservador "Deny by Default" (denegar por defecto), obligando a usar el protocolo Break-Glass en caso de urgencia real.
*   **Justificación:** Cumplimiento estricto de la Ley 19.628 (Protección de Datos). Bloquea accesos de personal médico curioso a fichas de famosos o familiares.
*   **Alternativas descartadas:**
    *   *RBAC puro (Control por Roles):* Descartado. RBAC dice "todos los médicos pueden ver todo". En salud, eso es una violación de privacidad; el médico solo debe ver la ficha de *sus pacientes actuales*.

### B) Protocolo Break-Glass ("Romper el cristal")

*   **Rama / Concepto:** Ciberseguridad / Gestión de Emergencias Críticas.
*   **¿Qué es?:** Mecanismo para saltar el control ABAC exclusivamente en riesgo vital.
*   **Controles de Uso:** Solo personal médico autorizado puede activarlo, tiene un límite de tiempo activo (ej. 4 horas), exige ingresar un motivo justificado en la interfaz, y genera automáticamente un ticket de auditoría para revisión del comité de ética.
*   **Justificación:** Un sistema rígido es peligroso. El Break-Glass es necesario, pero sin estos controles estrictos se convierte en una brecha de seguridad.

### C) Log de Auditoría Append-Only Inmutable

*   **Rama / Concepto:** Ciberseguridad / Auditoría, Compliance y Análisis Forense.
*   **¿Qué es?:** Una base de datos (o archivo) donde solo se pueden "agregar" (append) eventos. No permite UPDATE ni DELETE. Además, lleva un sello de tiempo y firma criptográfica.
*   **Profundización:** Concepto WORM (Write Once, Read Many). Usa hashing criptográfico (como el árbol de Merkle) para asegurar que si un administrador de bases de datos borra un registro, la cadena completa se invalida, detectando el fraude de inmediato.
*   **Justificación:** Trazabilidad legal. Si hay una demanda por negligencia o filtración (Ley 21.668), el Ministerio de Salud necesita pruebas irrefutables de quién vio qué.
*   **Alternativas descartadas:**
    *   *Blockchain / Smart Contracts:* Descartado rotundamente por vender humo. Introduce latencia, altos costos computacionales de consenso y complejidad innecesaria para un entorno donde el Estado ya actúa como entidad central de confianza.

### D) Cifrado y Protección en Tránsito

*   **Rama / Concepto:** Ciberseguridad / Criptografía.
*   **¿Qué es?:** En lugar de prometer un E2EE (Cifrado Extremo a Extremo) irrealizable en este ecosistema, se implementa cifrado en tránsito estricto (TLS 1.3) entre los nodos y el hub central, y cifrado en reposo para cachés y áreas de staging locales.

---

## 6. Gobernanza, Estrategia Política y de Adopción

*   **Interoperabilidad Forzada:** Basado en la Ley 21.668. Los hospitales y proveedores deben cumplir normativas vinculadas a incentivos y restricciones presupuestarias.
*   **APIs Obligatorias:** Corta el monopolio ("vendor lock-in") exigiendo a los privados pasar por un Sandbox estatal.
*   **Estrategia de Transición:** Contempla coexistencia e integración parcial. Si un proveedor grande se niega temporalmente, el sistema no se bloquea; aísla al nodo no conforme, manteniendo el valor de la red para los demás hospitales.

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
*   **Base de Datos del Índice:** Redis (por su velocidad en memoria de lectura de Hashes).
*   **Cache Local:** Redis o SQLite en memoria.
*   **Log Inmutable:** PostgreSQL (con restricciones de permisos a nivel base de datos) o una base optimizada para Time-Series.

---

## 9. Glosario / Diccionario de Términos

*   **ABAC (Attribute-Based Access Control):** Control de acceso dinámico basado en atributos y contexto (ej. turno del médico, relación con el paciente), superior al clásico rol estático.
*   **API REST:** Interfaz de programación que usa HTTP y texto plano (JSON/XML). En la MIMF se descarta para el core de red a favor de gRPC por temas de peso y latencia.
*   **Appliance:** Dispositivo o software preconfigurado (como el Sidecar) que se despliega "llave en mano" y es gestionado centralizadamente sin que el cliente local lo modifique.
*   **Backpressure / Rate Limiting:** Mecanismo de control de flujo que limita la cantidad de peticiones concurrentes para evitar que un sistema legacy colapse por sobrecarga.
*   **Break-Glass ("Romper el cristal"):** Protocolo de seguridad que permite saltar controles de acceso (ABAC) en casos de riesgo vital, dejando una estricta traza de auditoría.
*   **CapEx / OpEx:** Gasto en inversión de capital (comprar servidores físicos) vs. Gasto operativo (pagar suscripciones de nube o red mensual).
*   **Circuit Breaker (Cortocircuito):** Patrón de microservicios que bloquea las conexiones a un nodo fallido para evitar fallas en cascada y tiempos de espera de red infinitos.
*   **DDoS (Distributed Denial of Service):** Ataque o falla arquitectónica masiva que satura un servidor con peticiones simultáneas, botando el servicio.
*   **DHT (Distributed Hash Table):** Patrón P2P (como Kademlia) descartado para la MIMF por sus tiempos de respuesta y latencia impredecibles al buscar información crítica.
*   **Edge Caching:** Almacenamiento temporal de datos de red lo más cerca posible del consumidor final (caché local del Sidecar) para mitigar la latencia de internet.
*   **EHR (Electronic Health Record):** Ficha Clínica Electrónica. El software principal que utiliza el hospital para gestionar a los pacientes.
*   **ETL / Staging:** Proceso de Extraer, Transformar y Cargar datos desde una base de datos antigua a una temporal (Staging Area) en formato FHIR para lecturas ultrarrápidas.
*   **FHIR (Fast Healthcare Interoperability Resources):** Estándar médico internacional moderno de HL7 basado en "recursos" web (JSON) para interoperabilidad en salud.
*   **gRPC:** Framework RPC de Google que emplea HTTP/2 y Protocol Buffers (binarios) logrando un intercambio de red inmensamente más rápido y liviano que las APIs REST.
*   **HA (High Availability) / SLA:** Alta Disponibilidad de un sistema, garantizada por un Acuerdo de Nivel de Servicio (ej. 99.9% de uptime o tiempo encendido).
*   **HL7 (Health Level Seven):** Organización internacional y conjunto de estándares históricos (v2, v3, CDA) para el intercambio de información clínica.
*   **Hub-and-Spoke / Relay:** Topología de red donde los nodos (hospitales) se comunican mediante un intermediario gestionado, solucionando bloqueos de firewalls institucionales.
*   **Índice Nacional de Descubrimiento (X-Road):** Directorio central "stateless" que mapea (vía hashes) en qué hospitales existen datos de un RUT, sin guardar la ficha clínica.
*   **IPS (International Patient Summary):** Subconjunto estandarizado de FHIR enfocado en información vital mínima (alergias, medicamentos, diagnósticos) para atención no programada.
*   **KMS (Key Management Service):** Sistema central que administra de forma segura las claves y "salts" criptográficos usados en el Índice Nacional.
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
*   **RVN (Resumen Vital Nacional):** Componente central de la MIMF. Repositorio de respuesta inmediata (< 3s) que contiene alergias, medicamentos y diagnósticos críticos.
*   **Salt:** Texto aleatorio añadido a un dato antes de aplicar una función hash criptográfica, evitando ataques de reidentificación de RUTs o fuerza bruta.
*   **Sandbox Estatal:** Entorno de pruebas técnico obligatorio donde los proveedores privados de software demuestran que cumplen el estándar antes de venderle al Estado.
*   **Sidecar (Patrón):** Arquitectura donde un agente auxiliar (Sidecar) se despliega junto a un sistema base para interceptar, traducir y enrutar datos sin modificar el código legacy.
*   **SIDRA:** Sistemas de Información de la Red Asistencial. Estrategia histórica en Chile que logró digitalizar hospitales, pero generó "islas digitales" inconexas.
*   **SNOMED CT:** Nomenclatura médica mundial de terminología clínica (diagnósticos, hallazgos). Permite que la interoperabilidad sea semánticamente idéntica.
*   **Soft Merge (Datos Reconciliados):** Enfoque que muestra versiones conflictivas de un diagnóstico (con fechas y orígenes) para que el médico decida, sin borrar registros ajenos.
*   **SPOF (Single Point of Failure):** Punto Único de Fallo. Componente que, de caer, apaga toda la red. La MIMF lo evita replicando el Índice y usando cachés de emergencia.
*   **TLS 1.3:** Protocolo criptográfico de transporte que cifra de manera robusta los datos médicos mientras viajan por internet ("cifrado en tránsito").
*   **TTL (Time To Live):** Tiempo de vida útil de un dato almacenado en el caché antes de forzar al sistema a ir a buscar la versión más nueva al hospital de origen.
*   **Vendor Lock-in (Cautividad):** Cuando una empresa o Estado queda atrapado con un solo proveedor de software porque el costo y formato de extracción de datos hace inviable migrar.
*   **WORM (Write Once, Read Many) / Log Inmutable:** Base de datos de auditoría temporal donde los accesos se registran como bloques añadidos al final, impidiendo alteraciones forenses.