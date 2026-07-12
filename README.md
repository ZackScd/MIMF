# MIMF — Malla de Interoperabilidad Médica Federada

Repo del proyecto de arquitectura para interoperabilidad clínica en Chile.

La idea de fondo es simple (y un poco obvia cuando la miras de cerca): **Chile no necesita un EHR nacional único**. Necesita una capa que conecte lo que ya existe, sin romper hospitales, sin centralizar petabytes de historia clínica, y sin dejar a un paramédico a ciegas en la carretera cuando no hay señal.

MIMF propone una arquitectura **híbrida federada**, alineada a la [Arquitectura Nacional de Interoperabilidad del MINSAL](https://interoperabilidad.minsal.cl/):

- el historial profundo se queda en el hospital de origen (soberanía del dato)
- la **identidad del paciente** pasa por `PatientIdentityProvider` (destino: **EMPI/MPI** oficial; adaptador temporal en piloto)
- un **Índice / Record Locator** solo descubre *dónde* hay información clínica
- un **RVN** (Resumen Vital Nacional) entrega lo crítico en urgencias
- **Sidecars** perimetrales traducen sistemas legacy a FHIR sin tocar el código del proveedor
- un **TPIM** (chip NFC) cubre el escenario offline pre-hospitalario
- **2 apps oficiales:** Primeros Respondedores + Paciente/Autogestión; la zona pública del chip la lee cualquier celular NFC (NDEF), sin app de civiles

Para leer esta wea, empieza por `01_proyecto.md` y después baja a `02` / `03` cuando te pidan justificar decisiones.

---

## Estructura del repo

| Archivo / carpeta | Qué es |
| ----------------- | ------ |
| [`00_investigacion.md`](00_investigacion.md) | Contexto: historia de interoperabilidad en Chile (SIDRA, Hospital Digital), causas del fracaso, casos internacionales (Israel, Estonia/X-Road), Ley 21.668, alineación con MINSAL y cómo se posiciona MIMF frente a la tendencia estatal. |
| [`01_proyecto.md`](01_proyecto.md) | La propuesta propiamente tal. Tesis, anti-objetivos, componentes, flujos clínicos (urgencia + NFC), seguridad, operación y gobernanza. Es el documento "ejecutivo" del proyecto. |
| [`02_conceptos_y_tecnologias.md`](02_conceptos_y_tecnologias.md) | Justificación técnica profunda. Qué es cada pieza, por qué se eligió, qué se descartó, stack sugerido para PoC y glosario. |
| [`03_guia_defensa_arquitectura.md`](03_guia_defensa_arquitectura.md) | Manual de defensa. Cada decisión con el formato: problema → alternativas → beneficios → trade-off. Útil para comisiones, ramos y cuando alguien diga "¿y por qué no REST?". |
| [`04_legal.md`](04_legal.md) | Marco legal y normativo: leyes (21.668, 20.584, 19.628/21.719, 21.663), ecosistema MINSAL (EMPI, HPD, NID, FHIR, TEI, SNRE), proyectos asociados y matriz de cumplimiento MIMF. |
| [`05_estandares.md`](05_estandares.md) | Estándares técnicos: ISO/IEC (27001, 27701, 25010, 12207…), NCh, OWASP, IEC 62304, calidad de software y matriz por componente MIMF. Complementa `04` (ley) y `02` (FHIR/gRPC). |
| [`Informes/`](Informes/) | Informes académicos presentados en la universidad (entregas, evaluaciones, retroalimentaciones). No son la arquitectura "viva"; son evidencia de proceso del ramo. |
| `README.md` | Este archivo. |

---

## Cómo leer esto (orden sugerido)

1. `00_investigacion.md` — para entender *por qué* el problema existe
2. `01_proyecto.md` — para entender *qué* propone MIMF
3. `02_conceptos_y_tecnologias.md` — para el *cómo* y el *por qué técnico*
4. `03_guia_defensa_arquitectura.md` — para sobrevivir preguntas difíciles
5. `04_legal.md` — para el marco legal, MINSAL y cómo MIMF cumple o se acopla
6. `05_estandares.md` — para ISO, OWASP, calidad SW y certificación técnica

---

## Actualizaciones (jul 2026) — entrega única

La arquitectura base no se reescribió: se **cerraron huecos operativos** y se alineó la identidad con el MINSAL. Lo que una comisión técnica suele preguntar cuando el diseño ya está maduro.

### 1. Sidecar compilado por hospital (no universal)

**Qué cambió:** El Sidecar deja de describirse como un appliance genérico con "todos los conectores". Cada despliegue es un binario **Core + conector del EHR de ese hospital**.

**Por qué:** Un ejecutable con Oracle + SAP + InterSystems + SQL Server + Visual Basic arrastra módulos fantasma, más RAM, más superficie de ataque y testing imposible. Compilar por hospital es más realista en servidores viejos y reduce bugs irrelevantes.

**Dónde:** `01_proyecto.md`, `02_conceptos_y_tecnologias.md` (sección Sidecar), `03_guia_defensa_arquitectura.md` (§2), `00_investigacion.md` (tabla comparativa).

---

### 2. Interfaz oficial `HospitalConnector` (conectores certificables)

**Qué cambió:** Se define un contrato de interfaz (estilo CSI/CNI de Kubernetes). El conector lo puede mantener el equipo MIMF **o** el proveedor del EHR, certificado en Sandbox.

**Por qué:** Si un solo equipo nacional tiene que conocer el esquema interno de todos los EHR del país durante décadas, el proyecto muere por OpEx. Quien cambia el esquema del EHR debería ser responsable de mantener el conector compatible.

**Dónde:** `01`, `02`, `03` (nueva §16), glosario en `02`, gobernanza en `00`/`01`.

---

### 3. Contrato de esquema para el ETL (humano, no magia)

**Qué cambió:** Se explicitó que los cambios estructurales del EHR/base local deben **notificarse previamente**. La actualización del conector la hace soporte (MIMF o proveedor certificado). No se asume auto-detección de renombres de campos.

**Por qué:** Si `Paciente.Nombre` pasa a `Paciente.NombreCompleto`, un algoritmo no sabe si fue rename, split o cambio semántico. Automatizar eso entrega datos incorrectos en silencio — peor que un error ruidoso en salud.

**Dónde:** `01` (Operación), `02` (ETL/Staging), `03` (§8).

---

### 4. Coexistencia de versiones FHIR + End-of-Life

**Qué cambió:** Los Sidecars soportan varias versiones del Perfil Chile en paralelo. Las versiones viejas entran en EOL tras una ventana anunciada (ej. 6 meses). Se descartó la "actualización global forzada mañana".

**Por qué:** Forzar v6 de un día para otro rompe hospitales rezagados. La meta de gobernanza sigue siendo convergencia; el mecanismo de despliegue tiene que ser gradual.

**Dónde:** `01`, `02` (FHIR), `03` (§6).

---

### 5. Identidad: `PatientIdentityProvider` → EMPI MINSAL (sin UUID propio)

**Qué cambió:** Se descartó inventar un UUID MIMF. La malla usa el contrato estable **`PatientIdentityProvider`**. Destino definitivo = EMPI/MPI oficial. Mientras no esté operativo en terreno: **adaptador temporal de piloto** (mismas operaciones). Al madurar el servicio nacional, se reemplaza solo el adaptador. El Record Locator indexa por el ID canónico que entrega ese contrato. Resiliencia adicional: caché de resoluciones en el Sidecar.

**Por qué:** Acoplarse al calendario del MINSAL (IGs draft) bloquearía el piloto. Acoplarse al contrato permite avanzar y converger después sin reescribir Sidecar/RVN/TPIM/locator.

**Dónde:** `01`, `02` (Índice/identidad), `03` (§11), glosario, tabla en `00`. Ver también §11 y §12.

---

### 6. RVN como alerta de urgencia (no ficha nacional)

**Qué cambió:** Se reforzó el rol clínico del RVN: subconjunto ultracrítico + señal de que hay historial profundo en otros nodos ("¿desea traerlo?"). No compite con el EHR.

**Por qué:** Sin ese marco, cardiología / oncología / pediatría van a querer meter campos "imprescindibles" y el RVN se convierte en ficha central por *scope creep* político. El comité sigue existiendo; ahora tiene un criterio de diseño más fácil de defender.

**Dónde:** `01` (arquitectura + flujo de urgencia), `02` (Arquitectura Híbrida), `03` (nueva §17).

---

### 7. ABAC: lectura de atributos + SDK/API/OAuth

**Qué cambió:** El Sidecar/clientes **leen** turnos, agenda, admisión, RRHH, SAMU (y HPD cuando aplique); no escriben en esos sistemas. Se agregan apps oficiales y, además, un camino de integración vía SDK/API/OAuth para proveedores que no quieran usar las apps de la MIMF.

**Por qué:** Escribir en sistemas ajenos es la forma más rápida de romper un hospital. Y forzar una única app clínica nacional frena adopción; el SDK permite que el control de acceso viva dentro del EHR del proveedor.

**Dónde:** `01` (seguridad + clientes), `02` (ABAC), `03` (§9).

---

### 8. TPIM: actualización prioritaria en cada contacto clínico

**Qué cambió:** La actualización activa en mesón/Sidecar queda como camino principal. Los kioscos siguen, pero no se depende de que el paciente "recuerde" ir.

**Por qué:** La gente no va a actualizar el chip por voluntad propia. Si el valor del TPIM depende de disciplina ciudadana, el componente falla en producción aunque la criptografía esté perfecta.

**Dónde:** `01`, `02` (NFC), `03` (§15).

---

### 9. Reincorporación tras desconexión prolongada

**Qué cambió:** Además del Modo Degradado de horas/días, se define **snapshot + replay controlado** para hospitales que vuelven tras semanas/meses offline.

**Por qué:** Una cola de millones de eventos sin límites satura el Hub/Índice. Hay que priorizar datos del RVN, limitar tasa y comprimir.

**Dónde:** `01` (Modo Degradado), `02` (Modo Degradado), glosario, orden de estudio en `03`.

---

### 10. Gobernanza: ley + transición + ecosistema

**Qué cambió:** Se explicitó que cumplir Perfil Chile FHIR es condición para vender EHR al Estado (Ley 21.668), con **período de transición** para no dejar hospitales sin sistema. Se nombra a MIMF como **ecosistema / infraestructura nacional**, no como "una app".

**Por qué:** Los privados van a resistir (vendor lock-in es su negocio). La respuesta no es negociar el estándar a la baja; es anclarlo a norma y dar tiempo operativo realista. Y mentalmente ayuda: Sidecar + Índice + RVN + TPIM + PKI + KMS + Sandbox + OTA no es un monorepo de frontend.

**Dónde:** `01` (§6), `02` (§6), conclusión de `00`.

---

### 11. Alineación con Interoperabilidad MINSAL (EMPI ≠ Record Locator)

**Qué cambió:** Tras recorrer la documentación pública de [interoperabilidad.minsal.cl](https://interoperabilidad.minsal.cl/), se separó formalmente:

1. **EMPI/MPI** = identidad demográfica — *destino oficial vía `PatientIdentityProvider`*
2. **Índice MIMF / Record Locator** = descubrimiento clínico — *clave = ID canónico del proveedor de identidad*
3. **HPD** = prestadores — opcional al inicio para ABAC
4. **Servicios Terminológicos / NID** = catálogos y contratos FHIR R4 en el borde
5. **gRPC** = malla *interna*; FHIR/REST hacia MINSAL

Se descartó UUID/HMAC(RUT) como maestro de identidad. HMAC, si se usa, solo pseudonimiza el locator.

**Por qué:** El EMPI es la dirección correcta de identidad, pero no responde dónde está el historial clínico.

**Dónde:** `00`, `01`, `02`, `03` (§11).

---

### 12. Adaptador temporal de identidad (no bloquear el piloto)

**Qué cambió:** Se documentó el patrón: contrato `PatientIdentityProvider` → backend **temporal (piloto)** mientras MPI/HPD nacionales no estén operativos → backend **EMPI real** cuando existan. Mismo enchufe opcional para HPD en ABAC. El piloto no espera al calendario del Estado; al converger se reemplaza solo el adaptador (no se reescribe Sidecar/RVN/TPIM/locator).

**Por qué:** La Ley 21.668 no garantiza que los IGs draft estén en producción. Dependencia dura del servicio nacional = bloqueante. Dependencia del contrato = reemplazo limpio.

**Dónde:** `01` (identidad, flujo, despliegue por fases, anti-objetivos), `02` (sección Índice/identidad + glosario), `03` (§11), `00` (tabla de alineación).

---

### 13. gRPC en camino crítico vs REST/FHIR en el borde (no es contradicción)

**Qué cambió:** Se explicitó que el SLA **< 3s** del RVN aplica solo al camino interno **gRPC** (Sidecar ↔ Hub ↔ Índice ↔ RVN). REST/FHIR hacia EMPI/HPD/Terminológicos corre en **background** (revalidación de caché, catálogos) o como *fallback* ante **cache-miss** — no “en el medio” de cada urgencia.

**Por qué:** Evita la pregunta de comisión “¿para qué gRPC si igual hay REST?”. La ganancia de latencia está en el camino feliz; el costo REST se acepta solo en el caso excepcional.

**Dónde:** `01` (estándar/transporte), `02` (gRPC), `03` (§4), tabla en `00`.

---

### 14. Apps: dos oficiales + NDEF nativo (sin app de civiles)

**Qué cambió:**
- **No hay app de civiles.** La Zona Pública es **NDEF texto plano**: cualquier smartphone con NFC la lee nativamente (sin instalar nada, sin internet).
- **App de Primeros Respondedores** (SAMU/Bomberos): zona privada, ABAC, Break-Glass, pre-alerta.
- **App Paciente / Autogestión** (Clave Única): actualiza *su* TPIM (payload firmado del RVN vía NFC del celular) y configura/consiente la zona pública.
- Canales de escritura del chip: mesón/Sidecar (prioridad) → App Paciente → kioscos (complemento).
- SDK/API/OAuth sigue para hospitales/proveedores (no es una tercera app de calle).

**Por qué:** Una app solo para mostrar texto NDEF es redundante con el SO. Mezclar lectura de transeúntes con escritura del chip rompería el modelo de confianza. La App Paciente cubre al que no pisa un hospital conectado pero sí tiene celular/internet; el resto hace *fallback* a trauma estándar.

**Dónde:** `01` (componentes, flujos NFC, consentimiento), `02` (TPIM/NFC + glosario), `03` (§15 + orden de estudio).

---

## Nota para el equipo

Si van a editar docs:

- mantengan el tono técnico y el formato de listas / secciones que ya usan `00`–`05`
- en `03`, cualquier decisión nueva debería caber en el molde **problema → alternativas → beneficios → trade-off**

Pendiente razonable a futuro: un **Runbook Nacional** con escenarios operativos a 10 años (ransomware, rotación PKI, hospital offline 72h, terremoto regional, etc.).
