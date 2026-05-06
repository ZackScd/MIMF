# Plan de Implementación MIMF
## Contexto: Proyecto de Título — Ingeniería en Informática (Equipo de 3 personas)

> **Premisa de este plan:** Este documento distingue explícitamente dos horizontes. El **Horizonte 1** es el prototipo universitario ejecutable por 3 personas en aproximadamente un año académico, usando un entorno simulado con datos sintéticos. El **Horizonte 2** es la hoja de ruta hacia un despliegue real en la Región de La Araucanía, anclada a la realidad del ecosistema de salud chileno actual. Ambos horizontes comparten la misma arquitectura base; lo que cambia es la escala y el contexto.

---

## Resumen de Alcance por Horizonte

| Dimensión | Horizonte 1 (Prototipo Universitario) | Horizonte 2 (Piloto Real — Araucanía) |
|-----------|---------------------------------------|----------------------------------------|
| **Equipo** | 3 estudiantes | Equipo multidisciplinario (TI + clínico + legal) |
| **Duración** | ~12 meses (proyecto de título) | 2–3 años |
| **Hospitales** | 3 equipos simulando nodos distintos | CESFAM / Hospital de Temuco real |
| **Datos** | 100% sintéticos (generados) | Datos reales bajo Ley 19.628 y 21.668 |
| **EHR** | Sistema EHR simulado (propio) | EHR legacy real (Saydex u otro) |
| **TPIM (NFC)** | Opcional / demostración acotada | Piloto con pacientes reales voluntarios |
| **Objetivo** | Validar la arquitectura y los flujos | Validar el impacto clínico real |

---

# HORIZONTE 1: Prototipo Universitario

## Principio de Diseño del Prototipo

Antes de decidir qué construir, es fundamental acordar qué **no** se construye en esta etapa. El prototipo universitario debe ser un sistema que corra, que demuestre los flujos clínicos clave, y que justifique técnicamente cada decisión arquitectónica. No debe intentar ser el sistema final.

**Lo que el prototipo SÍ demuestra:**
- Que la arquitectura federada funciona: 3 nodos independientes que no comparten base de datos pero logran interoperar
- Que el RVN central entrega datos en < 3 segundos
- Que el Sidecar traduce datos locales a FHIR correctamente
- Que el control de acceso ABAC acepta o rechaza según el contexto
- Que el protocolo Break-Glass funciona y deja trazabilidad inmutable
- Que el sistema sigue operativo cuando se desconecta el nodo central (Modo Degradado)

**Lo que el prototipo NO necesita demostrar (pospuesto al Horizonte 2):**
- Integración con EHR legacy propietarios reales (Saydex, Intersystems)
- Alta Disponibilidad multi-región
- Escala de miles de pacientes simultáneos
- El chip NFC físico (puede demostrarse con un celular como emulador NFC)
- Certificación legal bajo Ley 21.668

---

## División de Roles del Equipo

Con 3 personas, la claridad de responsabilidades es crítica. Se propone la siguiente división, con zonas de colaboración explícitas:

| Rol | Responsabilidad Principal | Zonas de Colaboración |
|-----|--------------------------|----------------------|
| **Persona A — Arquitecto de Nodos** | Desarrollo del Sidecar, el EHR simulado y los 3 nodos | Protocolo gRPC con Persona B |
| **Persona B — Arquitecto Central** | Desarrollo del Índice Nacional y el RVN | Protocolo gRPC con Persona A; políticas ABAC con Persona C |
| **Persona C — Seguridad y UX Clínica** | Motor ABAC, Break-Glass, log inmutable, y la interfaz del médico | Políticas ABAC con Persona B; integración UI con Persona A |

> Esta división no es rígida. En un equipo de 3, todos deben entender el sistema completo. La división sirve para que cada persona tenga un área de profundidad y un área de responsabilidad en la defensa.

---

## Fase U-0: Acuerdos Técnicos de Base (Semanas 1–3)

Antes de escribir una línea de código, el equipo debe cerrar decisiones que, si se dejan abiertas, generan retrabajo costoso.

### U-0.1 Definir el Stack Tecnológico

Con 3 personas y tiempo acotado, la elección del stack debe priorizar velocidad de desarrollo y facilidad de demostración, sin sacrificar la coherencia con la arquitectura real propuesta.

**Stack sugerido para el prototipo:**

| Componente | Tecnología sugerida | Justificación |
|------------|--------------------|-|
| Sidecar | **Go** | Binario único, sin dependencias, consistente con la propuesta real |
| Índice Nacional + RVN | **Go** o **Node.js (TypeScript)** | Según preferencia del equipo; Go es más consistente con el Sidecar |
| Base de datos del RVN | **PostgreSQL** | Robusto, gratuito, fácil de levantar en Docker |
| Caché del Índice | **Redis** | Un contenedor Docker, latencia real demostrable |
| EHR Simulado | **Node.js + PostgreSQL** o **Python + FastAPI** | Rápido de construir; solo necesita exponer datos ficticios |
| Interfaz del médico | **React + TypeScript** | Permite demostrar el flujo visual de forma convincente |
| Comunicación interna | **gRPC + Protobuf** | Coherente con la arquitectura real; diferencial técnico demostrable |
| Orquestación local | **Docker Compose** | Los 3 nodos + servicios centrales corren en una sola máquina o en 3 laptops de la red local |
| Log inmutable | **PostgreSQL con restricciones** o tabla append-only en SQLite | Suficiente para el prototipo |

**Formato de serialización para el TPIM:**
- Protobuf (coherente con el resto del sistema)
- Para la demo del chip NFC: usar un celular Android con NFC como escritor/lector en lugar de fabricar chips físicos

### U-0.2 Definir el Subconjunto del RVN para el Prototipo

El esquema completo del RVN puede simplificarse para el prototipo sin perder poder demostrativo:

```
Paciente simulado tiene:
  - ID: HMAC del RUT sintético
  - Grupo sanguíneo
  - Alergias (máximo 3, codificadas en SNOMED CT)
  - Diagnósticos críticos (máximo 2)
  - Medicamentos activos críticos (máximo 2)
  - Timestamp de última actualización
```

Esto es suficiente para demostrar todos los flujos clínicos relevantes.

### U-0.3 Definir los 3 Nodos Simulados

Cada integrante del equipo opera un nodo diferente, simulando un establecimiento de salud distinto. Esta diversidad no es cosmética; demuestra que el sistema funciona entre sistemas heterogéneos.

| Nodo | Establecimiento simulado | EHR simulado | Característica diferencial |
|------|--------------------------|--------------|---------------------------|
| **Nodo A** | Hospital Regional (alta complejidad) | Base de datos PostgreSQL directa | Tiene el historial completo de los pacientes |
| **Nodo B** | Hospital rural | Base de datos SQLite (recursos limitados) | Simula conectividad intermitente; activa el Modo Degradado |
| **Nodo C** | CESFAM urbano | API REST propia (simula EHR con interfaz HTTP) | Emite y actualiza los TPIM de los pacientes |

### U-0.4 Acordar el Dataset Sintético

Generar un conjunto de 50 pacientes ficticios con datos coherentes entre sí. Esto es fundamental para que las demos sean reproducibles y convincentes.

**Herramientas sugeridas:**
- **Synthea** (generador de datos de salud sintéticos en formato FHIR, gratuito y de código abierto): produce pacientes con historiales clínicos coherentes, incluyendo alergias, diagnósticos y medicamentos codificados en SNOMED CT/LOINC
- Script Python simple para adaptar la salida de Synthea al esquema del RVN definido en U-0.2

**Escenarios de demo que el dataset debe cubrir:**
1. Paciente con alergia crítica a penicilina que llega a un hospital que no tiene sus registros
2. Paciente con epilepsia que sufre un episodio en la calle (demo del TPIM)
3. Paciente trasladado del Nodo B al Nodo A (transferencia entre hospitales)
4. Médico fuera de turno que activa Break-Glass

---

## Fase U-1: Infraestructura Base (Semanas 4–7)

Esta fase levanta el esqueleto del sistema. Al terminarla, los 3 nodos deben existir y comunicarse, aunque aún sin datos clínicos reales.

### U-1.1 Entorno de Desarrollo con Docker Compose

Crear un `docker-compose.yml` que levante todo el sistema localmente con un solo comando. Esto es crítico para que el equipo trabaje de forma sincronizada y para las demos.

**Servicios en el Compose:**
```
services:
  indice-nacional      # Índice de Descubrimiento
  rvn-central          # Repositorio Vital Nacional
  redis-cache          # Caché del Índice
  postgres-rvn         # Base de datos del RVN
  sidecar-nodo-a       # Sidecar del Hospital Regional
  ehr-simulado-a       # EHR del Hospital Regional (PostgreSQL)
  sidecar-nodo-b       # Sidecar del Hospital Rural
  ehr-simulado-b       # EHR del Hospital Rural (SQLite)
  sidecar-nodo-c       # Sidecar del CESFAM
  ehr-simulado-c       # EHR del CESFAM (API REST)
  ui-medico            # Interfaz web del médico
  log-inmutable        # Servicio de auditoría
```

En la demo final, este Compose puede correrse en una sola laptop (todos los nodos en la misma máquina) o distribuirse en 3 laptops del equipo conectadas en la misma red local, lo cual es más convincente visualmente.

### U-1.2 Definición de Contratos gRPC (los archivos `.proto`)

Antes de que cada persona desarrolle su componente, deben existir los contratos de comunicación. Estos son los archivos `.proto` que definen exactamente qué mensajes se intercambian.

**Contratos mínimos a definir:**

```protobuf
// descubrimiento.proto
service IndiceNacional {
  rpc RegistrarNodo(RegistroRequest) returns (RegistroResponse);
  rpc ConsultarNodos(ConsultaRequest) returns (ConsultaResponse);
}

// rvn.proto
service RVN {
  rpc LeerRVN(LecturaRequest) returns (RVNResponse);
  rpc EscribirRVN(EscrituraRequest) returns (EscrituraResponse);
}

// sidecar.proto
service Sidecar {
  rpc ObtenerResumenPaciente(PacienteRequest) returns (FHIRBundle);
  rpc SincronizarRVN(SincRequest) returns (SincResponse);
}
```

Una vez acordados estos contratos, cada persona puede desarrollar su componente en paralelo sin bloquearse.

### U-1.3 EHR Simulado

Cada nodo necesita un EHR simulado que exponga los datos de sus pacientes al Sidecar. No debe ser sofisticado; debe ser funcional.

**Funcionalidades mínimas del EHR simulado:**
- Base de datos con los 50 pacientes sintéticos
- Endpoint para consultar el resumen de un paciente por RUT
- Endpoint para listar los encuentros/atenciones de un paciente
- Endpoint para escribir una nueva atención (el médico registra lo que hizo)

El EHR simulado no tiene interfaz de usuario propia; la interfaz es la del médico desarrollada por Persona C.

---

## Fase U-2: Núcleo Central (Semanas 7–13)

**Responsable principal:** Persona B, con apoyo de Persona C para las políticas ABAC.

### U-2.1 Índice Nacional de Descubrimiento

**Funcionalidades a implementar:**
1. Registro de nodo: cuando el Sidecar de un hospital arranca, se registra en el Índice declarando qué RUTs tiene
2. Consulta de descubrimiento: dado un RUT (hasheado con HMAC), retorna la lista de nodos que tienen datos del paciente
3. Caché Redis: las consultas frecuentes se responden desde caché para cumplir el SLA de < 200ms
4. Expiración automática: si un nodo no renueva su registro en X minutos, sus entradas expiran (simula un hospital que se desconecta)

**Métrica demostrable en la defensa:**
- Mostrar en vivo la latencia de la consulta con y sin caché (Redis vs. PostgreSQL directo)

### U-2.2 Repositorio Vital Nacional (RVN)

**Funcionalidades a implementar:**
1. Escritura del RVN: el Sidecar envía el resumen vital de un paciente; el RVN lo valida y almacena
2. Lectura del RVN: dado un contexto (quién consulta, en qué nodo, con qué justificación), el motor ABAC decide si se entrega o no
3. Log inmutable: cada lectura y escritura queda registrada en una tabla append-only de PostgreSQL con timestamp, solicitante y resultado de la decisión ABAC
4. Firma de escrituras: cada entrada del RVN lleva la firma del nodo de origen (PKI simplificada con claves simétricas para el prototipo)

**Métrica demostrable en la defensa:**
- Mostrar en vivo que el log no puede modificarse (intentar un UPDATE y que falle por permisos de base de datos)
- Mostrar que el RVN se entrega en < 3 segundos desde la consulta del médico

---

## Fase U-3: Sidecar y Conectores (Semanas 10–16, paralelo a U-2)

**Responsable principal:** Persona A.

### U-3.1 Sidecar Core

El Sidecar es el componente más complejo del prototipo. Se construye de forma incremental:

**Iteración 1 (semanas 10–12): Sidecar básico**
- Se registra en el Índice Nacional al arrancar
- Lee datos del EHR simulado y los transforma a FHIR
- Envía el resumen del paciente al RVN cuando hay una nueva atención

**Iteración 2 (semanas 13–15): Sidecar con resiliencia**
- Implementar la Queue de sincronización: si el Índice no responde, encolar los eventos
- Implementar el Modo Degradado: si la red cae, el Sidecar sirve únicamente datos locales y muestra el indicador visual en la UI
- Implementar el Circuit Breaker: si un nodo remoto no responde en X milisegundos, fallar rápido en lugar de esperar

**Iteración 3 (semana 16): Caché local**
- El Sidecar almacena en Redis local una copia del RVN de los pacientes que atendió recientemente
- Si la red central cae, sirve desde caché con el indicador de frescura (timestamp visible en la UI)

### U-3.2 Transformador FHIR (dentro del Sidecar)

El Sidecar debe convertir los datos del EHR simulado (formato propio) al Perfil Nacional FHIR Chile simplificado.

**Recursos FHIR a implementar:**
- `Patient` (datos demográficos del paciente)
- `AllergyIntolerance` (alergias con código SNOMED CT)
- `Condition` (diagnósticos con código SNOMED CT)
- `MedicationStatement` (medicamentos activos)

Para el prototipo, no se necesita implementar el estándar FHIR completo, sino los 4 recursos anteriores con los campos mínimos exigidos por el RVN. Usar la librería `google/fhir` (Go) o `fhir.js` (JavaScript) acelera este trabajo.

---

## Fase U-4: Seguridad y Control de Acceso (Semanas 12–17)

**Responsable principal:** Persona C.

### U-4.1 Motor ABAC

El motor ABAC es el componente que diferencia técnicamente a la MIMF del estado del arte actual en Chile (que usa RBAC simple). Implementarlo correctamente es un argumento de peso en la defensa.

**Política ABAC a implementar (mínimo viable):**

```
REGLA 1 — Acceso normal:
  PERMITIR SI:
    solicitante.rol ∈ {MÉDICO, ENFERMERO}
    Y paciente.tiene_atención_activa_con(solicitante) = VERDADERO
    Y solicitante.turno.estado = EN_TURNO

REGLA 2 — Acceso de emergencia (Break-Glass):
  PERMITIR SI:
    solicitante.rol ∈ {MÉDICO, PARAMÉDICO}
    Y solicitante.justificación != NULA
    Y duración_acceso <= 30 minutos
    → REGISTRAR alerta en log inmutable
    → NOTIFICAR al responsable del nodo

REGLA 3 — Denegación por defecto:
  DENEGAR TODO LO QUE NO CUMPLA REGLA 1 O REGLA 2
```

**Implementación sugerida:** El motor ABAC puede implementarse como un servicio independiente (un microservicio más en el Compose) que expone una API sencilla: recibe el contexto de la solicitud y retorna ALLOW o DENY con la regla aplicada. Esto lo hace independiente del RVN y testeable por separado.

### U-4.2 Protocolo Break-Glass

**Flujo a implementar en la UI:**
1. El médico intenta acceder a la ficha de un paciente con quien no tiene atención activa
2. El sistema muestra: *"Acceso denegado. ¿Activar modo de emergencia?"*
3. El médico ingresa una justificación de texto libre (mínimo 20 caracteres)
4. El sistema concede acceso por 30 minutos y muestra un banner permanente: *"Modo Break-Glass activo — acceso de emergencia registrado"*
5. El log inmutable registra: timestamp, RUT del médico, RUT del paciente, justificación, duración

**Métrica demostrable en la defensa:**
- Mostrar en vivo que el log del Break-Glass existe, tiene la justificación, y que intentar borrarlo falla

### U-4.3 Interfaz del Médico

La interfaz no necesita ser hermosa, pero debe ser clínica y funcional. Evitar que parezca una aplicación de gestión genérica.

**Pantallas mínimas a implementar:**

1. **Login:** campo RUT, contraseña simulada, y selector de turno activo/fuera de turno
2. **Búsqueda de paciente:** campo RUT, botón buscar, muestra el estado de descubrimiento (en qué nodos se encontraron datos) y el indicador de red (🟢 red nacional activa / 🔴 modo degradado)
3. **Vista del RVN:** muestra las alergias, diagnósticos y medicamentos del RVN con el timestamp de frescura. Botón "Cargar historial completo" (Lazy Loading desde el nodo de origen)
4. **Vista de historial profundo:** lista de atenciones del paciente en otros hospitales, cargadas progresivamente
5. **Registro de nueva atención:** formulario simple para registrar la atención actual (actualiza el RVN)
6. **Log de auditoría:** tabla de accesos propios (el médico puede ver su propio historial de consultas)

---

## Fase U-5: Integración y Escenarios de Demo (Semanas 17–20)

Esta fase no desarrolla funcionalidades nuevas. Integra todo lo construido y prepara los escenarios de demostración que se ejecutarán en la defensa.

### U-5.1 Escenarios de Demo a Preparar

Los 4 escenarios siguientes cubren todos los componentes del sistema y permiten demostrar los argumentos centrales de la arquitectura.

**Escenario 1 — Flujo de urgencia nominal (el caso base):**
- El médico del Nodo A busca a un paciente que tiene registros en el Nodo B y el Nodo C
- El sistema descubre los nodos en < 200ms, carga el RVN en < 3 segundos
- El médico ve la alergia crítica a penicilina antes de prescribir antibiótico
- Argumento: *"Esto es lo que hoy no existe en Chile. El médico en Temuco no sabe la alergia del paciente atendido en Angol."*

**Escenario 2 — Modo Degradado (resiliencia):**
- En vivo durante la demo: apagar el Índice Nacional (detener el contenedor Docker)
- El Sidecar del Nodo A detecta la pérdida de conexión y activa el Modo Degradado
- El médico sigue atendiendo usando los datos locales; la UI muestra el banner de modo degradado
- Al volver a levantar el Índice, la Queue de sincronización vacía automáticamente los eventos pendientes
- Argumento: *"Si la MIMF cae, el hospital no cae con ella."*

**Escenario 3 — Control de acceso ABAC vs. RBAC:**
- Primero: médico fuera de turno intenta acceder → denegado (ABAC funciona)
- Segundo: el mismo médico activa Break-Glass con justificación → acceso concedido temporalmente
- Tercero: mostrar el log inmutable con el registro del Break-Glass
- Argumento: *"RBAC diría que este médico es médico y le daría acceso siempre. ABAC evalúa el contexto. La diferencia es la privacidad del paciente."*

**Escenario 4 — Demo TPIM (NFC):**
- Usar un celular Android como escritor NFC para grabar el Protobuf firmado de un paciente (la Zona Pública y la Zona Privada)
- Usar otro celular para leer el chip y mostrar la Zona Pública (accesible por cualquier persona)
- Abrir la App de Respondedores (puede ser una PWA simple) y leer la Zona Privada con autenticación
- Mostrar el semáforo de frescura (🟢/🟡/🔴) según la fecha del chip
- Argumento: *"Si este paciente sufre un accidente en Lonquimay sin internet, el paramédico tiene igualmente su alergia a la penicilina."*

### U-5.2 Preparación de la Defensa Técnica

Cada integrante debe poder defender su componente respondiendo las 4 preguntas del documento `03_guia_defensa_arquitectura.md`:
1. ¿Qué problema resuelve?
2. ¿Qué alternativas existían?
3. ¿Qué beneficios aporta la opción elegida?
4. ¿Qué se sacrifica (trade-off)?

**Distribución de temas de defensa por persona:**

| Persona A | Persona B | Persona C |
|-----------|-----------|-----------|
| Patrón Sidecar | Centralizado vs. Federado | ABAC vs. RBAC |
| Hub-and-Spoke vs. P2P | REST vs. gRPC | Break-Glass |
| Modo Degradado | FHIR vs. otros estándares | Hashing/HMAC en el Índice |
| Circuit Breaker | Cache/TTL | Log WORM |
| NFC / TPIM | Alta Disponibilidad | Privacidad y Ley 19.628 |

---

## Cronograma Resumido del Prototipo

```
SEMANA  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19 20
        [   U-0: Acuerdos   ]
                    [      U-1: Infraestructura base      ]
                             [         U-2: Núcleo central (B)        ]
                                   [       U-3: Sidecar (A)       ]
                                         [    U-4: Seguridad (C)    ]
                                                        [ U-5: Integración y Demo ]
```

> Las fases se solapan intencionalmente. Las semanas 1–3 son el cuello de botella más importante: si los contratos gRPC y el dataset sintético no están cerrados al final de la semana 3, todo lo demás se atrasa en cascada.

---

# HORIZONTE 2: Hoja de Ruta hacia el Mundo Real (Araucanía)

> Esta sección documenta cómo el prototipo universitario evolucionaría hacia un piloto real en la Región de La Araucanía. No es un plan para ejecutar durante el proyecto de título, sino el argumento de impacto real que complementa la defensa técnica y que podría ser la base de una propuesta al Servicio de Salud Araucanía o al MINSAL.

## Contexto Real de La Araucanía

La Región de La Araucanía presenta una combinación de factores que la hacen especialmente relevante para un piloto de este tipo, pero también especialmente desafiante.

**Factores que hacen urgente la interoperabilidad:**
- Alta ruralidad: comunidades en sectores con acceso limitado o nulo a internet (Lumaco, Lonquimay, Melipeuco, Curarrehue)
- Pacientes que son atendidos en CESFAM rurales pero derivan al Hospital Hernán Henríquez Aravena de Temuco (el único de alta complejidad de la región), y su información raramente los acompaña
- Altos índices de enfermedades crónicas (diabetes, hipertensión, enfermedades cardiovasculares) que exigen continuidad de tratamiento entre distintos niveles de atención
- La dispersión geográfica hace que los traslados sean frecuentes y que los tiempos de respuesta en urgencias sean más críticos que en regiones urbanas

**Factores que complican el despliegue:**
- Los CESFAM rurales pueden operar con conectividad 3G/4G inestable o con cortes frecuentes; el Modo Degradado no es una opción de diseño, es una necesidad cotidiana
- Existe una brecha de capacidad técnica TI entre el Hospital Hernán Henríquez Aravena (que tiene equipo TI) y los CESFAM más pequeños (que típicamente no tienen nadie dedicado)
- La población mapuche tiene desconfianza históricamente fundada hacia sistemas de información del Estado; el consentimiento informado y la participación comunitaria son requisitos éticos ineludibles, no trámites

## Establecimientos Candidatos para un Piloto Real

Un piloto real mínimamente significativo en La Araucanía podría involucrar los establecimientos siguientes, seleccionados para cubrir los tres niveles de atención y los dos tipos de conectividad que existen en la región:

| Establecimiento | Rol en el piloto | Complejidad de integración |
|----------------|-----------------|--------------------------|
| Hospital Hernán Henríquez Aravena (Temuco) | Nodo de alta complejidad | Alta (EHR legacy, burocracia institucional, volumen de pacientes) |
| CESFAM Amanecer (Temuco) | Nodo urbano de atención primaria | Media |
| CESFAM de Padre Las Casas | Nodo periurbano | Media |
| Posta rural de Lumaco o Lonquimay | Nodo rural (prueba de estrés del Modo Degradado y del TPIM offline) | Alta (conectividad intermitente, sin capacidad TI local) |

## Condiciones Previas para el Despliegue Real

Las siguientes condiciones deben estar cumplidas antes de conectar cualquier establecimiento real con datos de pacientes reales:

**Condiciones técnicas:**
- El Sidecar debe tener un conector funcional para el EHR específico que usa el Hospital Hernán Henríquez Aravena (el primer paso es identificar si es Saydex u otro sistema mediante un catastro técnico)
- El sistema debe operar correctamente con la latencia y pérdida de paquetes real de redes 3G/4G rurales (no solo en LAN de laboratorio)
- El log inmutable debe cumplir con los requisitos de auditoría de la Superintendencia de Salud

**Condiciones legales y éticas:**
- Convenio formal entre la universidad, el Servicio de Salud Araucanía Sur y el MINSAL
- Aprobación del Comité Ético Científico Regional (CEC) para el manejo de datos reales de pacientes, aunque sea en ambiente piloto
- Consentimiento informado individual de cada paciente cuyos datos se incorporen al RVN
- Definición del régimen de responsabilidad ante errores de dato: si un dato incorrecto en el RVN afecta una decisión clínica, el marco legal debe establecer quién responde y cómo

**Condiciones institucionales:**
- Al menos un médico o enfermero referente en cada establecimiento que actúe como "campeón" del sistema; sin este rol, la resistencia clínica al cambio es insuperable
- Capacidad TI mínima en cada establecimiento para instalar y mantener el Sidecar (para los más pequeños, esto puede requerir visitas periódicas del equipo técnico)
- Financiamiento para la infraestructura del servidor central: puede ser cloud (AWS, GCP) o el Data Center del MINSAL, que ya existe

## Hoja de Ruta Real (Post Proyecto de Título)

```
AÑO 1 (post-título)
  ├─ Búsqueda de financiamiento: CORFO Innova, ANID Fondef, o acuerdo directo
  │  con el Servicio de Salud Araucanía Sur
  ├─ Catastro técnico del Hospital Hernán Henríquez Aravena
  │  (identificar el EHR en uso y sus posibilidades de extracción de datos)
  ├─ Desarrollo del conector Sidecar para ese EHR específico
  └─ Aprobación CEC y firma de convenios institucionales

AÑO 2
  ├─ Despliegue del Núcleo Central en cloud con Alta Disponibilidad real
  ├─ Onboarding del Hospital Hernán Henríquez Aravena + 1 CESFAM urbano de Temuco
  ├─ Validación de flujos clínicos con datos reales (bajo consentimiento informado)
  └─ Piloto TPIM con una brigada SAMU de Temuco

AÑO 3
  ├─ Incorporación de 1 establecimiento rural (prueba del Modo Degradado en condiciones reales)
  ├─ Evaluación de impacto clínico: reducción de errores de medicación, reducción del tiempo
  │  de anamnesis en urgencias, casos de uso documentados del TPIM offline
  └─ Publicación de resultados y propuesta al MINSAL para escalado a otras regiones
```

## Argumento de Relevancia para la Defensa

El hecho de estar en Temuco no es una limitación del proyecto; es una ventaja argumental. La Araucanía concentra en una sola región los tres desafíos más complejos de la interoperabilidad en salud chilena: la ruralidad extrema (que exige el TPIM y el Modo Degradado), la diversidad cultural (que exige un diseño de consentimiento culturalmente pertinente), y la heterogeneidad tecnológica entre niveles de atención. Si la MIMF funciona en La Araucanía, funciona en cualquier región de Chile.

---

## Apéndice: Checklist de Entregables del Proyecto de Título

### Informe

- [ ] Introducción y problema (basado en `00_investigacion.md`)
- [ ] Propuesta de solución y arquitectura (basado en `01_proyecto.md`)
- [ ] Justificación técnica de decisiones con alternativas descartadas (basado en `02_conceptos_y_tecnologias.md`)
- [ ] Plan de implementación del prototipo (este documento, Horizonte 1)
- [ ] Hoja de ruta hacia el despliegue real en La Araucanía (este documento, Horizonte 2)
- [ ] Evaluación de riesgos
- [ ] Conclusiones

### Prototipo (artefactos entregables)

- [ ] Repositorio de código en GitHub/GitLab con README que explique cómo levantar el sistema
- [ ] `docker-compose.yml` que levanta el sistema completo con un solo comando
- [ ] Dataset de 50 pacientes sintéticos generados con Synthea
- [ ] Archivos `.proto` de todos los contratos gRPC
- [ ] Sidecar funcional (Go) conectando los 3 nodos
- [ ] Índice Nacional con caché Redis y latencia demostrable
- [ ] RVN central con log inmutable (UPDATE bloqueado por permisos)
- [ ] Motor ABAC con las 3 reglas implementadas y testeadas
- [ ] Protocolo Break-Glass funcional con trazabilidad completa
- [ ] Interfaz web del médico con los flujos de los 4 escenarios de demo
- [ ] Demo TPIM con celular Android como emulador NFC
- [ ] Video de demo de máximo 5 minutos que cubre los 4 escenarios

### Defensa

- [ ] Cada integrante puede defender su componente con las 4 preguntas del framework técnico
- [ ] Demo en vivo del Modo Degradado: apagar el Índice y mostrar que el sistema sigue operando
- [ ] Demo en vivo del Break-Glass y el log inmutable (intentar borrar y que falle)
- [ ] Capacidad de responder: *"¿Cómo se diferencia esto de SIDRA?"*
- [ ] Capacidad de responder: *"¿Cómo se diferencia de lo que está haciendo el MINSAL hoy con la Ley 21.668?"*
- [ ] Capacidad de responder: *"¿Por qué La Araucanía como caso de uso?"*

---

*Documento vivo — versión 2.0 (adaptada a contexto universitario y realidad regional). Horizonte 1 es el compromiso del proyecto de título. Horizonte 2 es la visión que da peso al impacto real del trabajo.*
