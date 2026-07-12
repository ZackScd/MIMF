
Encontré algo bueno. Van los hallazgos, de más grave a más cosmético:

**1. Contradicción real con blockchain (el más serio)**

En `00_investigacion.md` destacái el **KSI Blockchain** de Estonia como parte del "triángulo de seguridad" de X-Road — lo ponéi como un plus, una buena práctica de integridad y no repudio.

Pero en `02_conceptos_y_tecnologias.md`, sección de auditoría, descartái blockchain **"rotundamente por vender humo"** para tu propio log inmutable.

Eso es contradictorio tal cual está escrito. Si en la defensa alguien lee ambos documentos (o simplemente te pregunta "¿y por qué Estonia sí y ustedes no?"), te va a agarrar en el aire. La salida fácil es aclarar la distinción técnica que probablemente tenías en la cabeza pero no escribiste: KSI es un esquema de *hash-chaining + timestamping* centralizado de una entidad de confianza (Guardtime), no un blockchain público con consenso distribuido tipo Bitcoin/Ethereum. Lo que descartás en `02` es el blockchain con consenso descentralizado (caro, lento, innecesario cuando el Estado ya es la entidad de confianza). Agregar dos líneas que hagan esa distinción explícita cierra el hueco.

**2. "Cifrado Semántico" — mal uso del término (riesgo técnico)**

En `02`, sección "Seguridad en Terreno (TPIM/NFC)", el punto 1 dice: *"Cifrado Semántico: La Zona Privada usa SNOMED CT (números incomprensibles sin la base de datos oficial)"*.

Eso no es cifrado. SNOMED CT es una terminología clínica estandarizada y las tablas de códigos son públicas/licenciables — no son un secreto criptográfico. Llamarle "cifrado" a usar un código en vez de texto plano es **ofuscación**, no seguridad real (es literalmente el patrón "security through obscurity" que cualquier ramo de seguridad te enseña a no usar como control primario). Si un profesor de seguridad te pregunta "¿eso es cifrado de verdad?", la respuesta correcta es "no, la protección real es la firma PKI y el cifrado TLS que ya mencionan en los puntos 2 y 4 — SNOMED CT solo reduce la legibilidad casual, no es un control de confidencialidad". Te recomendaría sacar la palabra "Cifrado" de ese punto y llamarlo "Ofuscación semántica / reducción de legibilidad casual" para no regalar el error.

**3. Término huérfano: "Salt" en el glosario**

En `02` definís "Salt" en el glosario, pero en toda la arquitectura usan **HMAC + KMS** para pseudonimizar el locator, nunca salt explícitamente. No es un error grave, pero si alguien te pregunta "¿dónde usan el salt que definieron?" no vas a tener respuesta. O lo sacái, o agregái una línea que diga que el HMAC se implementa con salt/clave del KMS (que de hecho es lo técnicamente correcto — un HMAC bien hecho lleva una clave, que cumple ese rol).

**4. Cita legal a verificar antes de usarla textual en la defensa**

En `03` (§10, Break-Glass) y `04` citan **"art. 16 bis Ley 21.719, letra a) y letra e)"** como base para el tratamiento sin consentimiento en emergencias. Busqué y confirmé que el art. 16 bis efectivamente regula datos de salud con régimen reforzado y sí contempla una excepción por "salvaguardar la vida o integridad", pero no pude confirmar que esté literalmente etiquetada como "letra a)" y "letra e)" en el texto oficial — puede que esas letras correspondan al art. 16 general y no al 16 bis específicamente. Antes de citarlo textual frente a una comisión, te recomiendo verificar el texto exacto en la Ley Chile / Diario Oficial, porque una cita de artículo mal apuntada es un error fácil de pillar y barato de evitar.

En general: son errores de detalle, no de fondo — la arquitectura como tal se sostiene. El blockchain y el "cifrado semántico" son los dos que sí te recomiendo arreglar antes de la defensa, porque son el tipo de contradicción/tecnicismo que un profesor atento caza rápido y te hace quedar mal por algo que se arregla en dos líneas.