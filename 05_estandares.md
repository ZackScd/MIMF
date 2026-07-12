# Estándares Técnicos, ISO y Normas de Desarrollo — Proyecto MIMF

> **Alcance:** Estándares internacionales (ISO/IEC), normas chilenas (NCh), IETF/RFC, guías de industria (OWASP, IHE, HL7) e ingeniería de software aplicables a la MIMF. Por cada uno: obligatoriedad y acoplamiento arquitectónico. **Corte:** julio 2026.

---

## 1. Resumen por capa

| Capa / componente | Estándares principales | Tipo |
| ----------------- | ---------------------- | ---- |
| Seguridad de la información (ley) | ISO/IEC 27001, 27002; ISO/IEC 27799 | Obligatorio sector salud — **Ley 21.663** (servicios esenciales / OIV; calificación de nodos MIMF pendiente — ver `04_legal.md` §2.4, §8.4) |
| SGSI sectorial MINSAL | NCh-ISO 27001:2022 | Obligatorio por **política administrativa** — [PS-NC-005](https://www.minsal.cl/wp-content/uploads/2025/11/POLITICA-GENERAL-DE-SEGURIDAD-DE-LA-INFORMACION-Y-CIBERSEGURIDAD-DEL-MINSAL-PS-NC-005-V.0.5.pdf) (distinta jerarquía que la ley) |
| Privacidad | ISO/IEC 27701 | Recomendado fuerte (Ley 21.719, dic-2026) |
| Interoperabilidad clínica | HL7 FHIR R4, CL Core, IHE (PIXm, PDQm, mSVCM), SNOMED CT, LOINC | Obligatorio borde MINSAL (Ley 21.668) |
| Transporte interno | gRPC, HTTP/2, Protocol Buffers; TLS 1.3 | Implementación MIMF (camino crítico < 3s) |
| Transporte borde estatal | REST/JSON, FHIR R4 | Obligatorio hacia EMPI, HPD, Terminológicos, TEI, SNRE |
| Calidad de software / datos | ISO/IEC 25010, 25012 | Recomendado (Sandbox, SLAs) |
| Ciclo de vida | ISO/IEC 12207, 15288 | Recomendado (Sidecar OTA, conectores) |
| Software sanitario | IEC 62304, ISO 14971, ISO 81001-1 | Condicional (apps clínicas, TPIM) |
| Desarrollo seguro | OWASP Top 10, ASVS, MASVS; SBOM (SPDX/CycloneDX) | Recomendado / exigible en Sandbox |
| Identidad / acceso apps | OAuth 2.0, OpenID Connect; Clave Única (Estado) | SDK/API MIMF; apps oficiales |
| Observabilidad | OpenTelemetry | Operación distribuida (Sidecar ↔ Hub ↔ Índice) |
| Edge / NFC | NDEF (NFC Forum), NTAG216; firma X.509 / PKI | TPIM |
| Normativa salud Chile | DEIS/EIS (ex-820), Perfil CL Core; PS-NC-005, ITS-NC-007 | Borde + contratos TIC sector |

---

## 2. Seguridad de la información (ISO 27000)

### 2.1. ISO/IEC 27001:2022 — SGSI

Requisitos para un Sistema de Gestión de Seguridad de la Información, certificable por auditoría externa.

Dos fuentes convergen, con **jerarquía distinta** (no son la misma obligación):

* **Ley 21.663** (ley): sector salud como servicio esencial; gestión de riesgos, continuidad, reporte CSIRT. Los **OIV** tienen obligaciones reforzadas; si nodos MIMF (Hub, Índice, RVN) quedan en el registro OIV está **pendiente de reglamento** — ver `04_legal.md` §2.4 y §8.4.
* **PS-NC-005** (política MINSAL): adopta [NCh-ISO 27001:2022](https://www.minsal.cl/seguridad_de_la_informacion/) para el sector; aplica a operadores estatales y es requisito de contrato para integradores.

La MIMF diseña bajo el estándar más exigente **sin esperar** la calificación OIV formal de cada nodo.

| Control 27001 | Mecanismo MIMF |
| ------------- | -------------- |
| Activos | Inventario Sidecar, Hub, Índice, RVN, TPIM, apps, PKI/KMS |
| Acceso | ABAC, Clave Única, HPD, deny-by-default |
| Criptografía | TLS 1.3, firmas TPIM, KMS, HMAC opcional en locator |
| Registro | Logs WORM, OpenTelemetry, auditoría Break-Glass |
| Continuidad | Modo Degradado, snapshot + replay, réplica Índice |
| Desarrollo seguro | Sandbox, `HospitalConnector` certificable, Sidecar OTA |
| Proveedores | Certificación conectores EHR; ITS-NC-007 en contratos |

**Tipo:** obligatorio por **ley** (21.663, sector esencial) + obligatorio por **política** (PS-NC-005, NCh-ISO 27001); requisito de contrato para integradores. Calificación OIV de nodos concretos: pendiente (ver `04_legal.md`).

### 2.2. ISO/IEC 27002:2022

Catálogo de controles del SGSI. Traduce a hardening Sidecar (binario por hospital), segmentación Hub, rotación PKI, MFA, rate limiting, mínimo privilegio en APIs.

### 2.3. ISO/IEC 27701

Extensión de privacidad (PIMS). Complementa Ley 21.719: RAT, EIPD RVN/TPIM, ARCOP+Bloqueo, roles responsable/encargado.

### 2.4. ISO 27799:2024

Aplicación de 27002 en salud. Refuerza Modo Degradado (atención local no se detiene), ABAC contextual, separación RVN vs. historial profundo.

### 2.5. ISO/IEC 27017 / 27018

Controles cloud. Aplica si Hub, Índice o RVN van en nube (región, cifrado reposo, aislamiento tenant).

---

## 3. Calidad y ciclo de vida (ISO/IEC JTC 1)

### 3.1. ISO/IEC 25010 — Calidad del producto software

| Característica | Objetivo MIMF |
| -------------- | ------------- |
| Fiabilidad / disponibilidad | Modo Degradado; SLA RVN < 3s |
| Rendimiento | gRPC/Protobuf; caché Sidecar |
| Mantenibilidad | `HospitalConnector`; Core + conector |
| Portabilidad | Binario Go/Rust; coexistencia versiones FHIR |
| Usabilidad | Apps oficiales; NDEF nativo (zona pública TPIM) |

Base para SLAs y criterios de aceptación del Sandbox.

### 3.2. ISO/IEC 25012 — Calidad de datos

ETL/Staging; prioridad clínica (alergias > diagnósticos); semáforo frescura TPIM (🟢🟡🔴). Crítico para confianza del RVN.

### 3.3. ISO/IEC 12207 — Ciclo de vida software

| Proceso | MIMF |
| ------- | ---- |
| Desarrollo | Core, apps, conectores |
| Mantenimiento | OTA Sidecar; EOL Perfil Chile |
| Operación | Observabilidad centralizada; runbook (pendiente) |
| Configuración | Versionado conector por hospital |
| V&V | Sandbox; FHIR Connectathon |

### 3.4. ISO/IEC 15288 — Ciclo de vida de sistemas

Piloto regional → rollout Macrored/Servicio; aislamiento nodo no conforme; retiro adaptador temporal al converger EMPI.

---

## 4. Software en contexto sanitario

### 4.1. IEC 62304 — Software médico

Ciclo de vida por clase de seguridad (A, B, C).

| Componente | Clasificación probable |
| ---------- | ---------------------- |
| Sidecar, Hub, Índice | SIH / infraestructura |
| App Primeros Respondedores + TPIM clínico | Posible SaMD — analizar con ISP |
| TPIM (hardware NFC) | Posible dispositivo médico si se comercializa como tal |

**Marco legal:** El régimen ISP (registro sanitario, fiscalización) no está desarrollado en este documento — ver `04_legal.md` §2.6. La clasificación es **pendiente**; no asumir exención en el cronograma.

Si aplica SaMD: trazabilidad requisitos→tests + **ISO 14971** (gestión de riesgos).

### 4.2. ISO 14971 — Riesgos de dispositivos médicos

Análisis de riesgo clínico para componentes clasificados como SaMD/dispositivo. Coherente con RVN como alerta (no prescripción autónoma) y Break-Glass acotado.

### 4.3. ISO 81001-1 — Seguridad de software de salud

Gestión de riesgos y seguridad en ciclo de vida. Apps clínicas, RVN, Sandbox con regresión.

### 4.4. ISO 13606 / ISO 18308

Legado pre-FHIR. No es objetivo de implementación; Chile converge a FHIR R4.

---

## 5. Interoperabilidad clínica

### 5.1. HL7 FHIR R4 + Perfil Chile (CL Core)

Estándar sintáctico nacional ([interoperabilidad.minsal.cl](https://interoperabilidad.minsal.cl/)). Sidecars traducen legacy → FHIR. Coexistencia de versiones Perfil Chile con ventanas EOL.

Recursos clave: Patient, Encounter, AllergyIntolerance, MedicationStatement, Condition, Observation.

**DEIS / EIS:** Norma técnica chilena de información en salud, mapeada al [CL Core FHIR](https://hl7chile.cl/fhir/ig/clcore/1.9.4/).

### 5.2. IHE y NID MINSAL

| Perfil / guía | Uso MIMF |
| ------------- | -------- |
| PIXm / PDQm (IG MPI) | `PatientIdentityProvider` → EMPI |
| HPD (IG HPD) | Atributos prestador en ABAC |
| mSVCM | Servicios Terminológicos MINSAL |
| NID v0.4.9 | Contrato borde REST/FHIR |

### 5.3. Guías FHIR por dominio (MINSAL)

| IG | Alcance | Relación MIMF |
| -- | ------- | ------------- |
| [TEI v0.2.2](https://interoperabilidad.minsal.cl/fhir/ig/tei/0.2.2/index.html) | SIC especialidad No GES (APS → secundario) | Sidecar puede consumir/emitir eventos; Record Locator complementa con descubrimiento clínico |
| [SNRE v0.9.6](https://interoperabilidad.minsal.cl/fhir/ig/snre/0.9.6/index.html) | Prescripción / dispensación electrónica | Convive en borde FHIR; RVN no duplica repositorio SNRE |
| [MPI](https://interoperabilidad.minsal.cl/fhir/ig/mpi/) / [HPD](https://interoperabilidad.minsal.cl/fhir/ig/hpd/) | Identidad paciente / prestador | Destino definitivo identidad y ABAC |

### 5.4. Semántica clínica

| Estándar | Rol MIMF |
| -------- | -------- |
| SNOMED CT | Diagnósticos, alergias (RVN, TPIM zona privada) |
| LOINC | Laboratorio, observaciones |
| IPS (ISO 27269 / HL7) | Referencia de diseño del RVN (subconjunto urgencia) |

### 5.5. HL7 v2 / CDA (legado)

Presente en EHR antiguos. Sidecar traduce a FHIR; no es objetivo de la malla interna.

---

## 6. Transporte, serialización e identidad digital

### 6.1. gRPC + Protocol Buffers + HTTP/2

Camino crítico interno (Sidecar ↔ Hub ↔ Índice ↔ RVN), SLA **< 3s**. Serialización binaria; payload compacto para TPIM (~888 B en NTAG216).

**Alternativa documentada:** gRPC-Web o REST equivalente solo dentro de la malla si firewalls bloquean HTTP/2.

### 6.2. REST + JSON + FHIR R4

Borde MINSAL: EMPI, HPD, Terminológicos, TEI, SNRE. Corre en background (revalidación caché, catálogos) o en *cache-miss* de identidad — no en el camino síncrono de cada urgencia.

### 6.3. TLS 1.3 (RFC 8446)

Cifrado en tránsito entre nodos y Hub. Cifrado en reposo en cachés y staging local.

### 6.4. PKI / X.509

Autenticación mutua entre nodos; firma zona privada TPIM; timestamps en auditoría. Modelo inspirado en triángulo de confianza (autenticación, confidencialidad, integridad).

### 6.5. OAuth 2.0 / OpenID Connect

SDK/API MIMF para integración ABAC en EHR de terceros. Apps oficiales autenticadas con **Clave Única** (identidad federada del Estado).

**SMART on FHIR:** opcional para apps que consuman recursos FHIR del hospital; no reemplaza el contrato gRPC interno.

---

## 7. Edge, NFC y formatos físicos

| Estándar | Rol |
| -------- | --- |
| **NDEF** (NFC Forum) | Zona pública TPIM — texto plano, lectura nativa smartphone |
| **NTAG216** | Chip NFC referencia (~888 B útiles zona privada) |
| **Protobuf** | Zona privada TPIM — payload firmado RVN |
| **SNOMED CT** | Codificación clínica en zona privada |

Tres canales de escritura: mesón/Sidecar (prioridad) → App Paciente (Clave Única) → kioscos (complemento).

---

## 8. Observabilidad y resiliencia operativa

### 8.1. OpenTelemetry

Trazas, métricas y logs unificados. TraceID por petición clínica (Sidecar → Hub → Índice → RVN). Alineado a ISO 27001 (monitoreo) y SLAs 25010.

### 8.2. Patrones de resiliencia (industria / SRE)

No son ISO, pero están en el diseño MIMF:

| Patrón | Estándar de facto | MIMF |
| ------ | ----------------- | ---- |
| Circuit Breaker | Microservicios (Netflix/Hystrix lineage) | Evita cascada ante hospital caído |
| Backpressure / rate limiting | SRE | Protege EHR legacy |
| Edge cache + TTL | — | Caché Sidecar; flag de frescura en UI |
| Hub-and-Spoke | Topología red | Relay ante NAT/firewalls hospitalarios |
| Snapshot + replay | — | Reincorporación tras desconexión prolongada |

---

## 9. Desarrollo seguro (OWASP)

### 9.1. OWASP Top 10

| Riesgo | Mitigación MIMF |
| ------ | --------------- |
| Broken Access Control | ABAC, Break-Glass auditado, OAuth scopes |
| Cryptographic Failures | TLS 1.3, PKI, KMS |
| Injection | Validación FHIR; Sidecar sin SQL expuesto |
| Security Misconfiguration | Appliance Sidecar; hardening OTA |
| Vulnerable Components | SBOM (SPDX/CycloneDX); deps mínimas Go/Rust |
| Identification Failures | Clave Única, HPD |

### 9.2. OWASP ASVS

Apps oficiales: mínimo **L2** (datos sensibles salud). Hub/Índice: auth mutua, rate limiting.

### 9.3. OWASP MASVS

Apps SAMU y Paciente: secure storage offline (auditoría Break-Glass), certificate pinning, no loguear PHI en claro.

---

## 10. Normativa chilena técnica (NCh / MINSAL)

| Documento | Relación MIMF |
| --------- | ------------- |
| NCh-ISO 27001:2022 | SGSI operador — adoptado por PS-NC-005 (política MINSAL) |
| PS-NC-005 | Política seguridad MINSAL (jerarquía administrativa; complementa, no sustituye, Ley 21.663) |
| ITS-NC-007 | Terceros tecnológicos en salud — contratos Sidecar/proveedor |
| ITS-NC-003 | Plan ciberseguridad MINSAL |
| Perfil CL Core / EIS DEIS | Validación FHIR en Sandbox |
| [interoperabilidad.minsal.cl](https://interoperabilidad.minsal.cl/) | Arquitectura, EMPI, HPD, Terminológicos, IGs |

---

## 11. Matriz por componente

Leyenda: **A** = aplica directamente · **I** = indirecto (vía operador/contrato) · **N/A** = no es el anclaje principal

| Componente | 27001/27799 | 25010/25012 | 12207/OTA | OWASP | FHIR/IHE | Transporte |
| ------------ | ----------- | ----------- | --------- | ----- | -------- | ---------- |
| Sidecar | A perimeter, logs, TLS | A rendimiento, mantenibilidad | A OTA, conector | A API hardening | A borde FHIR, PIXm cache | A gRPC; I REST borde |
| Hub / Índice | A HA, segmentación | A latencia | A ops, DR | A auth, rate limit | I (no FHIR clínico) | A gRPC/TLS |
| RVN | A minimización | A calidad dato | A comité esquema | A API | A IPS-like, SNOMED | A gRPC; N/A REST sync urgencia |
| TPIM | A PKI, anti-clon | A frescura dato | A emisión/actualización | N/A (no web) | I subset RVN codificado | N/A red; A Protobuf |
| App Respondedores | A MASVS, ABAC | A usabilidad terreno | A releases | A ASVS L2 | N/A | I REST pre-alerta |
| App Paciente | A consentimiento, Clave Única | A usabilidad | A releases | A ASVS L2 | I portal opcional | I REST |
| Sandbox | A certificación | A criterios aceptación | A CI/CD | A pentest | A CL Core, TEI/SNRE según caso | I |

---

## 12. Roadmap de certificación

| Fase | Estándares | Objetivo |
| ---- | ---------- | -------- |
| PoC / piloto | OWASP L1, CL Core Sandbox, 27002 seleccionado | Integración sin bloquear por cert plena |
| Pre-producción | 25010 SLAs, 25012 reglas dato, OpenTelemetry | Métricas objetivas |
| Producción nacional | SGSI 27001, 27701 si APDP exige, ITS-NC-007 | Operación auditable |
| Apps / TPIM clínicos | IEC 62304 + ISO 14971 + 81001 | Decisión ISP pendiente — ver `04_legal.md` §2.6 |

---

## 13. Referencias

| Recurso | URL |
| ------- | --- |
| ISO 27001 | https://www.iso.org/standard/82875.html |
| ISO 27701 | https://www.iso.org/standard/71670.html |
| ISO 27799 | https://www.iso.org/standard/81170.html |
| ISO/IEC 25010 | https://www.iso.org/standard/78176.html |
| ISO 14971 | https://www.iso.org/standard/72704.html |
| IEC 62304 | https://webstore.iec.ch/en/publication/25323 |
| RFC 8446 (TLS 1.3) | https://www.rfc-editor.org/rfc/rfc8446 |
| gRPC | https://grpc.io/docs/what-is-grpc/introduction/ |
| OpenTelemetry | https://opentelemetry.io/docs/ |
| OWASP ASVS | https://owasp.org/www-project-application-security-verification-standard/ |
| OWASP MASVS | https://mas.owasp.org/ |
| HL7 FHIR R4 | https://hl7.org/fhir/R4/ |
| CL Core | https://hl7chile.cl/fhir/ig/clcore/1.9.4/ |
| Interoperabilidad MINSAL | https://interoperabilidad.minsal.cl/ |
| TEI | https://interoperabilidad.minsal.cl/fhir/ig/tei/0.2.2/index.html |
| SNRE | https://interoperabilidad.minsal.cl/fhir/ig/snre/0.9.6/index.html |
| NID | https://interoperabilidad.minsal.cl/fhir/ig/nid/0.4.9/ |
| MINSAL ciberseguridad | https://www.minsal.cl/seguridad_de_la_informacion/ |
