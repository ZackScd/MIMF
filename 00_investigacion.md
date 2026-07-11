# Investigación: Estado de la Interoperabilidad en Salud (Chile y el Mundo)

**Problema Central:** En Chile no existe un sistema unificado para los recintos hospitalarios. Los datos clínicos no viajan con el paciente al trasladarse de un hospital o región a otra, generando "islas digitales". Esta carencia de interoperabilidad afecta la continuidad de la atención y la seguridad del paciente.

---

## 1. Historia de los Intentos en Chile (Más de 15 años de proyectos)
El fracaso histórico en Chile no se debe a la falta de software, sino a la falta de interoperabilidad y a los constantes cambios en las prioridades políticas.

### SIDRA (Sistemas de Información de la Red Asistencial) - 2008 en adelante
Fue el intento más ambicioso para digitalizar hospitales y consultorios.
* **El enfoque:** En lugar de crear un único software nacional, el MINSAL permitió que cada Servicio de Salud licitara y comprara su propio sistema a distintas empresas privadas (Saydex, Intersystems, etc.).
* **El resultado (SIDRA 1):** El país logró digitalizar decenas de hospitales, pero los sistemas no se comunicaban entre sí.
* **SIDRA 2.0:** Intentó corregir la fragmentación exigiendo estándares de comunicación, pero su implementación fue lenta, derivó en disputas legales y generó rechazo médico por problemas de usabilidad.

### Hospital Digital (2018)
Presentado como la gran solución en la nube para conectar al país.
* **El problema:** Su enfoque viró hacia la telemedicina (atención remota) y la reducción de listas de espera de especialistas. No abordó el problema de fondo de la unificación e interoperabilidad de fichas clínicas presenciales.

### El "Vicio" de los Proyectos In-House
Ante los altos costos o la rigidez del software comercial, muchos hospitales desarrollan sistemas "artesanales". Esto agrava la fragmentación, ya que rara vez cumplen con estándares internacionales (HL7 o FHIR).

---

## 2. Causas Estructurales del Fracaso (El Nudo Crítico)
El problema trasciende lo técnico y se asienta en fallas estructurales:
1. **Gobernanza Inestable:** Los ciclos políticos de 4 años reinician las estrategias de salud digital, interrumpiendo proyectos que requieren al menos una década para madurar.
2. **Falta de Estandarización de Identidad y Clínica:** Aunque existe el RUT, la estructura semántica de los datos (cómo se guarda una alergia o diagnóstico) varía de un hospital a otro.
3. **Resistencia al Cambio:** Históricamente, el software médico fue diseñado con fines estadísticos o de facturación, no como una herramienta de apoyo clínico, generando fricción con los médicos.

---

## 3. Casos de Éxito Internacional
Las naciones líderes abandonaron la idea de forzar un "software único" y se enfocaron en construir una **capa de intercambio de datos**.

### A. El Modelo de Israel (Interoperabilidad Forzada Centralizada)
* **La Solución:** Centralización de un repositorio con "datos clave" (resumen clínico, alergias, últimas hospitalizaciones).
* **Resultado:** Ante cualquier atención de urgencia a nivel nacional, el sistema rescata automáticamente este "resumen vital" del paciente.
* **La debilidad (El Gap Offline):** Este modelo depende 100% de la conexión a internet. Si un paciente sufre un accidente en una zona rural o catastrófica sin señal, el paramédico no tiene cómo acceder a su información vital.

### B. El Modelo de Estonia: X-Road (Referente Mundial Distribuido)
Estonia no utiliza un software único, sino una arquitectura distribuida (*Distributed by Design*) llamada **X-Road**, una Capa de Intercambio de Datos (Data Exchange Layer).

**Principios y Arquitectura de X-Road:**
* **Once-Only:** El Estado no pide datos que ya posee. Los sistemas se consultan entre sí.
* **Sin Repositorio Central:** Los datos de salud residen inmutables en el servidor del hospital de origen.
* **Security Servers (Servidores de Seguridad):** Son los puntos de entrada para cada institución. El intercambio de datos es "Peer-to-Peer" (P2P) cifrado entre servidores de seguridad, eludiendo cuellos de botella centrales.
* **Central Server:** Solo administra el ecosistema, mantiene el registro de miembros y las políticas de confianza, pero no almacena ni ve historias clínicas.

**El Triángulo de Seguridad (PKI):**
1. **Autenticación Mutua:** Los servidores verifican sus identidades mediante certificados digitales antes de comunicarse.
2. **Confidencialidad:** Cifrado extremo a extremo. El administrador central no puede leer los datos.
3. **Integridad y No Repudio:** Uso de *KSI Blockchain* y *Timestamps*. Todo mensaje es firmado digitalmente, haciendo los logs inalterables y válidos como prueba judicial.

**Sinergia X-Road y HL7 FHIR (El Idioma vs. La Carretera):**
Un error común es confundir el transporte con el contenido. X-Road provee la "carretera" (transporte seguro e identidad), pero se requiere un "idioma" común para que los sistemas se entiendan automáticamente. Ese idioma es **HL7 FHIR**. El Servidor de Seguridad toma un recurso FHIR, lo envuelve en la seguridad de X-Road y lo entrega al destino.

*Nota sobre adopción:* X-Road es Open Source y ya se implementa en Finlandia y en varios gobiernos latinoamericanos (Argentina, Brasil, Colombia).

---

## 4. Estado Actual en Chile (Perspectiva 2024 - 2026)
En mayo de 2024, se publicó la **Ley N.º 21.668**, que obliga legalmente a la interoperabilidad de las fichas clínicas entre prestadores públicos y privados.

* **Situación 2025-2026:** El MINSAL se encuentra definiendo el reglamento técnico. El debate ya no es *si* se debe compartir, sino *cómo* hacerlo sin comprometer la ciberseguridad (activos críticos nacionales) ni afectar el rendimiento clínico.
* **Esfuerzos del Ecosistema:** Instituciones como el CENS (Centro Nacional en Sistemas de Información en Salud) y HL7 Chile están validando guías de implementación (ej. *FHIR Connectathon 2026*).
* **El Riesgo de la Brecha Digital:** Existe incertidumbre sobre si los prestadores más pequeños (consultorios rurales) tendrán el subsidio y la capacidad técnica para implementar estos nodos, arriesgando crear pacientes de "primera y segunda categoría".

---

## 5. Análisis de Viabilidad: Proyecto MIMF vs. Estado Actual
La propuesta arquitectónica de la **Malla de Interoperabilidad Médica Federada (MIMF)** responde directamente a los vacíos actuales del ecosistema chileno. Mientras muchos proyectos gubernamentales esperan que los proveedores de software *legacy* se abran voluntariamente, el enfoque de MIMF utiliza **Sidecars** perimetrales, una estrategia disruptiva que extrae y traduce de forma externa sin intervenir el código fuente propietario.

**Comparativa de Enfoques (MIMF vs. Tendencia Estatal actual):**

| Componente        | Tu Propuesta (MIMF)                   | Tendencia Estatal 2026                        |
| ------------------| --------------------------------------| ----------------------------------------------|
| **Arquitectura**  | **Federada Híbrida** (Soberanía local)| Repositorio central / Migración masiva        |
| **Transporte**    | **gRPC + Protobuf** (Optimizado para baja latencia) | Mayoritariamente **API REST + JSON** (Alto peso) |
| **Seguridad**     | **ABAC** (Control por Contexto clínico) | **RBAC** (Control por Roles estáticos)        |
| **Integración**   | **Sidecar compilado por hospital** + interfaz `HospitalConnector` certificable | Adaptadores *ad-hoc* por cada proveedor       |
| **Identidad**     | **Consume EMPI/MPI MINSAL** + Record Locator propio para descubrimiento clínico | EMPI/MPI + HPD + Terminología (NID); sin locator clínico explícito en la doc pública |

**Ventajas Críticas de la Arquitectura MIMF:**
1. **Interoperabilidad Semántica (SNOMED CT / LOINC):** Resuelve el problema de que una simple conexión de red no garantiza entendimiento médico. Asegura que el significado clínico de un "Infarto" sea universal en todo Chile, complementando la sintaxis de FHIR (con coexistencia de versiones del Perfil Chile y ventanas de EOL), y alineándose a los [Servicios Terminológicos](https://interoperabilidad.minsal.cl/docs/componentes-de-la-arquitectura/terminologicos.html) del MINSAL cuando estén disponibles.
2. **Soberanía y Protección de Datos:** Al ser un diseño federado puro para el historial profundo, preserva la autoría inmutable en el hospital de origen. Esto es inherentemente más robusto y defendible ante la **Ley de Protección de Datos (19.628)** en comparación a un "mega repositorio" centralizado susceptible a vulneraciones masivas.
3. **Continuidad Pre-Hospitalaria (Offline-First en terreno):** A diferencia de las visiones estatales puramente de "nube", MIMF incorpora el uso de **Chips NFC (Tokens Físicos)** para paramédicos, permitiendo que la información vital esté disponible de forma instantánea en accidentes vehiculares o zonas rurales sin internet, pre-alertando a los hospitales antes de que el paciente ingrese por urgencias.
4. **Integración sostenible:** El Sidecar no pretende ser un ejecutable mágico universal. Se despliega como binario específico (Core + conector) y permite que los proveedores certifiquen sus propios conectores, alineando la carga de mantenimiento con quien conoce el esquema interno del EHR.
5. **Complemento (no competencia) con el NID:** La MIMF **consume** el EMPI para identidad y puede apoyarse en el HPD para prestadores. Aporta lo que la documentación pública del MINSAL aún no materializa como producto operativo de malla: Sidecars, Record Locator clínico, RVN de urgencia y TPIM offline.

### Alineación con la Arquitectura Nacional de Interoperabilidad (MINSAL)

Fuente: [interoperabilidad.minsal.cl](https://interoperabilidad.minsal.cl/).

| Componente MINSAL | Rol oficial | Relación con MIMF |
| ----------------- | ----------- | ----------------- |
| **Modelo híbrido** centralizado-distribuido | Estrategia base de la [arquitectura](https://interoperabilidad.minsal.cl/docs/especificacion-de-la-arquitectura/arquitectura.html) | Compatible con la tesis federada + RVN mínimo |
| **EMPI / MPI** | Identidad demográfica unívoca | **Destino** de `PatientIdentityProvider`; adaptador temporal en piloto hasta que esté operativo |
| **HPD** | Directorio de prestadores | Atributos ABAC cuando esté disponible; no bloquea el piloto |
| **Servicios Terminológicos** | CodeSystem / ValueSet / ConceptMap (IHE mSVCM) | Apoyo semántico; Sidecars consumen catálogos oficiales |
| **NID** | Núcleo FHIR (MPI + HPD) | Contrato de integración en el borde (FHIR R4 / REST) |
| **FHIR R4** | Estándar sintáctico nacional | Obligatorio hacia el Estado; gRPC queda para malla interna |

**Conclusión:**
El Estado chileno está fijando las reglas normativas y los componentes de identidad/prestadores/terminología, pero los IGs públicos (MPI/HPD) aún lucen inmaduros como producto en terreno. La MIMF se posiciona como **capa de malla clínica** que se acopla al **contrato** del NID/EMPI —no a su calendario— mediante `PatientIdentityProvider` (adaptador temporal → EMPI real). Aporta lo que la doc pública aún no materializa operativamente: Sidecars, Record Locator, RVN y TPIM.