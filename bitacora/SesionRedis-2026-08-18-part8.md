# Sesión Redis — 2026-08-18 (part 8)

**Bloque 4 (opcional del arco Redis) — Parte 4: Cache-aside extendido al endpoint `POST /analyze/pdf` en `document-analyzer-ai`.**

Duración efectiva: ~3h, martes tarde. Café cargado al arrancar (Tole descansó del roadmap el lunes tras 3h intensivas del domingo). Energía media-alta hasta la fase de debugging del segundo PDF, después descenso natural. Sesión larga: teoría de diseño + implementación autónoma con pausa acordada en la parte de bytes + validación empírica exhaustiva + tres bugs encadenados + dos commits limpios.

Contexto de sesión: continuación directa del bloque 4 parte 3 (cache-aside sobre `POST /analyze` de texto, cerrado el domingo 16 con commit `9896228`). Alcance acordado desde el arranque: solo `/analyze/pdf` con cache-aside. Tole lo intenta solo, con acuerdo explícito de pausar Sócrates cuando llegue a la parte técnica novedosa (hashing de bytes del PDF + manejo del `MultipartFile` stream one-shot).

## Repo state al cerrar

- Rama `main`, HEAD en `8e03afb`, dos commits por delante del `9896228` del domingo. Sincronizado con `origin/main`.
- Working tree clean.
- Contenedor `document-analyzer-redis` levantado al cerrar. Recomendación: `docker compose down` al terminar el día.
- Redis flusheado con `FLUSHALL` antes del commit para dejar estado neutral.
- App Spring Boot parada.

## Commits generados en la sesión

1. **`152327e feat(analyze-pdf): add cache-aside for /analyze/pdf reusing CvAnalysisCache`** — extiende `CvAnalysisCache` con sobrecarga `get(byte[])` / `put(byte[], CvSummary)`, refactoriza el corazón a métodos `getInternal`/`putInternal` compartidos, y modifica `AnalyzePdfController` para orquestar el patrón. Cirugía mínima: no toca `AnalyzeController`. Mensaje bilingüe con validaciones empíricas (miss/hit/miss con hash distintos por PDF, aceleración ~700x en hit).

2. **`8e03afb fix(analyze): harden prompts against markdown fences and raise max-tokens`** — refuerza ambos prompts (`analyze-cv.st` y `analyze-cv-pdf.st`) con bloque de instrucciones estrictas en ASCII plano (sin backticks para no romper el parser StringTemplate), y sube `spring.ai.anthropic.chat.max-tokens` de 1024 a 4096. Fix de bug latente heredado (punto residual `1024.` que Spring toleraba). Aplica a ambos endpoints, no solo al PDF.

## Conceptos cubiertos

### Sobrecarga de métodos vs componente separado (decisión arquitectónica)

Discusión explícita de tres opciones antes de decidir:

- **Opción A: `CvAnalysisPdfCache` como componente separado**. Ventaja: separación clara, un componente por endpoint. Desventaja: duplicación (dos clases casi idénticas cambiando solo el input del hash y el prefijo).
- **Opción B: extender la clase con métodos con nombre distinto (`getForPdf`, `putForPdf`)**. Reutiliza la maquinaria pero el nombre `CvAnalysisCache` pierde propósito único.
- **Opción C: sobrecarga de métodos con extracción de corazón común**. Métodos privados `getInternal`/`putInternal` sobre la clave ya construida; públicos que difieren solo por firma (`get(String)` vs `get(byte[])`) y construyen la clave con `buildKeyFromText` / `buildKeyFromPdf`.

Decisión: **C**. Razones documentadas: cero duplicación de lógica Redis + JSON + fail-safe; Java resuelve sobrecarga por firma sin ambigüedad; extensible sin explosión de clases; nombre de la clase sigue siendo apropiado.

Argumento contrario considerado y descartado: Tole propuso inicialmente A alegando "PDF es más caro, TTL superior". Argumento contra-desarrollado en sesión: el análisis del LLM cuesta lo mismo en tokens sea texto o PDF; lo único más caro es re-uploadear el fichero, que se cachea en frontend, no en Redis. TTL divergente sería una divergencia sin justificación técnica sólida.

### Sobrecarga de métodos en Java aplicada a caché

- `get(String cvText)` y `get(byte[] pdfBytes)` coexisten porque Java distingue firmas por tipo de parámetro, no por nombre.
- Los cuerpos públicos son one-liners orquestadores: `return getInternal(buildKeyFromText(cvText));`.
- Toda la maquinaria pesada (try/catch, opsForValue, JSON, fail-safe) vive en `getInternal(String key)` UNA sola vez.
- **Regla lockeada**: cuando dos operaciones difieren solo en un paso pero comparten el resto, extraer el resto a método privado y diferenciar los públicos por sobrecarga o por delegación explícita.

### `sha256Hex` unificado sobre `byte[]`

- El método privado ahora opera sobre `byte[]` directamente porque todo termina siendo bytes al final.
- Texto llama `cvText.trim().getBytes(StandardCharsets.UTF_8)` explícitamente antes de delegar al hash.
- PDF pasa `pdfBytes` directamente.
- Elimina la duplicación anterior donde había dos `MessageDigest` en dos overloads.

### Bug crítico evitado: `.digest()` sin argumento

Durante la implementación Tole escribió por error:

```java
byte[] digest = MessageDigest.getInstance("SHA-256").digest();  // BUG
```

**Consecuencia**: `.digest()` sin argumento devuelve el hash del contenido acumulado (nulo tras `update`), es decir, **el hash del string vacío** (`e3b0c44298fc1c...`), siempre el mismo, ignorando `input` por completo. Todos los CVs colisionarían en la misma clave. Caché roto silenciosamente (compila, no lanza excepción, "funciona" en el sentido de que devuelve un valor).

Fix aplicado: `.digest(input)` — el método con argumento hace `update(bytes) + digest()` en un solo paso.

**Regla interview-ready**: `MessageDigest.digest()` sin argumento vs `.digest(byte[])` es una trampa clásica. Los métodos "no-arg" en APIs de hashing típicamente asumen que has hecho `update()` previamente, y devuelven algo "sensato" (hash de vacío) cuando no. Bug silencioso peligrosísimo: pasa validación superficial, rompe en producción cuando alguien mira las claves.

### `MultipartFile` — stream one-shot y "materializar una vez"

Concepto clave visto por primera vez en el proyecto:

- `MultipartFile` es un envoltorio sobre datos que Spring puede haber recibido de distintos sitios (memoria, disco temporal según tamaño).
- Métodos disponibles: `getBytes()`, `getInputStream()`, `getResource()`. **Cada acceso puede provocar una nueva lectura** del contenido subyacente.
- En Spring moderno con ficheros pequeños suele ser consistente por buffering en memoria. **No es garantía del contrato**, es detalle de implementación.
- **Principio general aplicable a cualquier recurso consumible**: materializar UNA vez en una estructura reutilizable (`byte[]`, `List`, etc.) y trabajar siempre con la copia. **Nunca volver al recurso original**.

Aplicado a la solución:

```java
byte[] pdfBytes = file.getBytes();  // ÚNICO acceso al file
Optional<CvSummary> cached = cache.get(pdfBytes);
// ...
Media pdfMedia = new Media(mimeType, new ByteArrayResource(pdfBytes));  // usa la copia
```

**`ByteArrayResource`** es la contrapartida: un `Resource` de Spring construido sobre `byte[]` en memoria. Sirve al LLM el PDF leyendo del array local, no del `MultipartFile` original.

**Analogía**: es como un vaso de agua (`MultipartFile`) del que necesitas beber dos veces. Opción mala: bebes del vaso, esperas que aún haya agua para la segunda. Opción buena: primero llenas una botella (`byte[]`), y ambas veces bebes de la botella.

**Aplicable a**: `InputStream.mark/reset` (no todos soportan), `Iterator` (una sola pasada), `Publisher` de Reactor sin `cache()`, `JMS Messages`, `Kafka records` fuera de rebalance, etc. Patrón transversal.

### `IOException` de `MultipartFile.getBytes()` — checked, `throws` vs try/catch

Discusión de estilo con decisión razonada:

- `MultipartFile.getBytes()` puede lanzar `IOException` (checked).
- **Opción idiomática**: declarar `throws IOException` en la firma del método del controller. Spring MVC captura excepciones no manejadas del controller y las mapea a respuesta 500 vía el `GlobalExceptionHandler` central del proyecto (heredado del arco Spring Security 6 con ProblemDetail RFC 7807).
- **Opción anti-idiomática (usada en primera iteración)**: try/catch local con `throw new RuntimeException(e)`. Envuelve checked en unchecked sin aportar información, pierde semántica de la excepción original, añade indentación innecesaria. Único caso legítimo de este patrón: APIs funcionales que no permiten checked, o operaciones "verdaderamente imposibles" (como `NoSuchAlgorithmException` de SHA-256).

Regla lockeada: si la excepción rompe la petición legítimamente (no hay recuperación local útil), **propagar declarándola**, dejar que el manejador global aplique la política del proyecto.

### El bug de las dos conversiones (código muerto sutil)

En primera iteración Tole escribió:

```java
CvSummary summary = converter.convert(rawResponse);
cache.put(pdfBytes, summary);
return converter.convert(rawResponse);  // ← BUG: segunda conversión
```

**Consecuencias**: ineficiente (parsea JSON dos veces) + abre puerta a bug sutil si el converter tuviera efectos secundarios o estado interno. **Regla**: una operación, un resultado, una variable, una vez.

Fix: `return summary;` — el objeto ya lo tienes en variable, devuélvelo.

### StringTemplate 4 en Spring AI — caracteres reservados

**Descubrimiento empírico crítico**: los ficheros `.st` de Spring AI se procesan con **StringTemplate 4** (biblioteca de Terence Parr, mismo autor que ANTLR), que tiene sintaxis propia con caracteres reservados:

- `{ }` para bloques.
- `<` `>` para expresiones (modo `<expr>`).
- **Backticks** (`` ` ``) tienen significado sintáctico.

Cualquier prompt que contenga estos caracteres literalmente rompe el parseo **antes** de que la plantilla llegue al LLM. Error observado:

```
8:34: invalid character '`'
8:38: 'terminar' came as a complete surprise to me
```

El propio parser reporta el offset del carácter ofensivo y el token que no supo interpretar.

Consecuencia: los prompts con instrucciones sobre formato JSON deben **describir en prosa** los caracteres especiales, no mostrarlos literalmente. En vez de "empieza con `` ` `` `{` `` ` ``", usar "empieza con la llave abierta".

**Regla lockeada**: prompts en `resources/prompts/` en ASCII plano cuando incluyan instrucciones sobre formato. Considerar migrar a `NoOpTemplateRenderer` en el futuro si el sistema deja de necesitar interpolación de variables (Spring AI permite configurar el renderer).

Frase interview-ready:

> "Los ficheros `.st` de Spring AI se procesan con StringTemplate 4, que tiene caracteres reservados (llaves, ángulos, backticks). Prompts con estos caracteres rompen el parser antes de llegar al LLM. Mitigación: prompts en ASCII plano describiendo caracteres especiales en prosa; alternativamente, cambiar el `TemplateRenderer` a uno no-op si no se necesita interpolación."

### `max-tokens` por defecto y respuestas truncadas

Bug encadenado descubierto con CV largo (`CV_MT.pdf`):

```
UnexpectedEndOfInputException: Unexpected end-of-input: was expecting closing quote for a string value
(through reference chain: CvSummary["languages"] -> ArrayList[0] -> Language["level"])
```

**Traducción**: Jackson estaba parseando el campo `languages[0].level` cuando el JSON se cortó. **No es problema de sintaxis** — es respuesta LLM **truncada**. El modelo llegó al límite de `max-tokens=1024` a mitad de escribir la cadena y paró.

Confirmación en métricas: latencia 24s (vs 9-10s de CVs cortos), y `tokens_out` habitual ~599 pero la respuesta necesitaba más.

Fix: `spring.ai.anthropic.chat.max-tokens=4096`. Cuadruplica el margen sin coste desmedido (recuerda que solo se factura por tokens realmente generados, no por el techo). Beneficio adicional: cubre también `/analyze` de texto para el caso de CVs muy densos.

**Bug latente extra corregido de paso**: el valor original tenía un punto final (`1024.`). Spring lo parseaba con tolerancia (probablemente como `1024.0` casteado a int) pero es malformado. Ahora `4096` limpio.

**Regla interview-ready**: cuando parseas JSON estructurado de respuestas LLM, `max-tokens` demasiado bajo se manifiesta como `UnexpectedEndOfInputException` a mitad de un campo. No es bug de código, es truncado. Dimensionar `max-tokens` según el peor caso esperado del schema, con margen 3-4x.

### Cache-aside "cero-basura" ante errores LLM

Observación arquitectónica **importante y positiva**:

Cuando el LLM devuelve una respuesta rota (bugs 1 y 2 de esta sesión), el flujo del método muere en `converter.convert()`. El `cache.put()` está DESPUÉS. **Consecuencia**: la caché **no se pobla con basura**. Verificado en MONITOR: hay `GET` sin `SET` correspondiente para los curls fallidos.

Ventaja: cuando arregles el problema (mejor prompt, mejor `max-tokens`, retry, etc.) y reintentes el mismo PDF, se cachea el resultado **correcto**. Si hubieras cacheado la respuesta rota, tendrías que hacer `FLUSHALL` manual antes de reintentar.

**Regla lockeada**: en cache-aside, el `put` va **después** de toda validación y transformación del resultado. Nunca cachear resultados intermedios ni excepciones. El caché solo ve valores "listos para servir".

Frase interview-ready:

> "En cache-aside, la posición del `put` es diseño intencional: va después de que el resultado esté completamente validado y transformado. Si el flujo revienta antes, la caché no se pobla con basura, y un retry posterior con el bug arreglado cachea el resultado correcto sin intervención manual."

### GlobalExceptionHandler y ProblemDetail RFC 7807 en acción

Verificación empírica indirecta de infraestructura heredada del arco Spring Security 6:

- Bug del truncado LLM disparó `UnexpectedEndOfInputException` en `converter.convert()`.
- El `GlobalExceptionHandler` capturó, mapeó a HTTP 502, y devolvió respuesta JSON estructurada:
  ```json
  {"detail":"El modelo devolvió una respuesta con formato inesperado. Se recomienda reintentar la petición.","instance":"/analyze/pdf","status":502,"title":"Respuesta de LLM no procesable"}
  ```
- Toda esa maquinaria (ProblemDetail RFC 7807, mensaje traducido, status code apropiado) funcionando sin tocarla. Arquitectura previa pagando dividendos.

## Laboratorio empírico (ejecutado)

**Fase 1 — repaso de notas del domingo antes de arrancar**: Tole optó por releer sus propias notas de la sesión anterior (bloque 4 parte 3) antes de tocar código. Decisión buena y proactiva. Cargó contexto sobre decisiones lockeadas (SHA-256, atomicidad SET+TTL, fail-safe con WARN) y frases interview-ready.

**Fase 2 — diseño de la extensión de `CvAnalysisCache`**: discusión socrática de las tres opciones (componente separado / método con nombre distinto / sobrecarga con corazón común). Tole eligió inicialmente A por criterio "PDF más caro, TTL superior". Contra-argumento presentado y aceptado: TTL divergente sin razón sólida. Decisión final: C.

**Fase 3 — implementación de la extensión**: Tole escribió la refactorización solo. Bugs detectados en revisión:
- `.digest()` sin argumento (crítico, hash siempre vacío).
- Bloque de código comentado sobrante de la versión anterior.
- Espaciado y estilo (formateo automático de IDE).
- Import `com.fasterxml.jackson` vs `tools.jackson` verificado correcto (Jackson 3).

Fix aplicados, compila `BUILD SUCCESS`.

**Fase 4 — modificación de `AnalyzePdfController`**: Tole escribió la orquestación. **Pausa acordada** cuando llegó a la parte de bytes. Discusión Sócrates sobre:
1. `IOException` checked → `throws` vs try/catch (decisión: `throws`, propaga al `GlobalExceptionHandler`).
2. Stream one-shot del `MultipartFile` → materializar en `byte[] pdfBytes` una vez, reutilizar en hash y en `ByteArrayResource`.
3. `ByteArrayResource(pdfBytes)` como alternativa idiomática a `file.getResource()`, elimina segunda lectura del `MultipartFile`.
4. Consideraciones de producción sobre uploads en memoria: `spring.servlet.multipart.max-file-size`, streaming a disco temporal, S3 direct upload con presigned URL, rate limiting por IP/usuario.

Frase interview-ready generada:

> "El manejo de uploads en Spring MVC por defecto materializa el fichero en memoria vía `MultipartFile.getBytes()`. Para producción: límite de tamaño en `spring.servlet.multipart.max-file-size` como primera línea, streaming a disco temporal o subida directa a S3 con URL prefirmada según tamaño y volumen esperados. Sin límite duro, cualquier upload sin control es vector de ataque trivial."

Iteración de código: primera versión con `try/catch` + `RuntimeException` (anti-patrón), corregida a `throws IOException`. Bug de doble `converter.convert()` corregido. Nombre `pdfFiles` renombrado a `pdfBytes` por precisión semántica. Constructor con trailing comma corregido y formateado uno por línea.

Compila `BUILD SUCCESS`.

**Fase 5 — validación empírica con MONITOR + curls a PDFs reales**:

Setup: tres terminales (app / MONITOR / curl con multipart `-F`). `FLUSHALL` previo.

- **Curl 1** (`CV_MT_ES.pdf`, primer upload): 
  - MONITOR: `GET analyze:pdf:84917d15...` (null) → 9.2s después `SET analyze:pdf:84917d15... {json completo} EX 86400`.
  - Curl: 9.8s total, respuesta CvSummary completa.
  - Métricas LLM: `latency_ms=9239 tokens_in=5794 tokens_out=599 cost_usd=0.026367`.
  - **7x más tokens de input** que el equivalente en texto (~827 tokens), **5x más caro** (~0.005 USD). Confirmado: PDF envía base64 completo al payload.

- **Curl 2** (`CV_MT_ES.pdf`, segundo upload idéntico):
  - MONITOR: solo `GET analyze:pdf:84917d15...` (devuelve JSON).
  - Curl: **0.013s total**. Aceleración ~700x. Overhead multipart despreciable.
  - Ahorro: 9s de latencia + 0.026 USD por hit.

- **Curl 3** (`CV_MT.pdf`, PDF distinto):
  - MONITOR: `GET analyze:pdf:3298e848...` (null, hash distinto por contenido distinto — confirmación de aislamiento por contenido).
  - **Falló con HTTP 502**: `BeanOutputConverter` explotó con `StreamReadException: Unexpected character (backtick)`. LLM devolvió respuesta envuelta en fences markdown (`` ```json ... ``` ``) ignorando la instrucción del prompt.
  - MONITOR **no muestra SET correspondiente** — confirmación de "cache-aside cero-basura" ante errores.

**Fase 6 — debugging encadenado de tres bugs**:

Tres bugs distintos afloraron en cascada, cada arreglo revelando el siguiente ("onion debugging" de Fred Brooks):

- **Bug 1**: LLM envuelve JSON en fences markdown. Diagnóstico correcto: prompt genérico ("responde con formato solicitado") ignorado ocasionalmente por el modelo. **Fix intentado**: reforzar prompt con instrucciones estrictas.

- **Bug 2** (introducido por el fix del 1): prompt reforzado incluía backticks literales (`` ` ``) para mostrar ejemplos de "no uses esto". **StringTemplate 4 los interpreta como sintaxis** y explota antes de renderizar. Descubrimiento del motor de plantillas de Spring AI. **Fix**: reescribir el prompt en ASCII plano, describir caracteres especiales en prosa.

- **Bug 3** (encadenado, distinto): con prompt corregido, el CV largo (`CV_MT.pdf`) devolvió respuesta **truncada** por `max-tokens=1024`. Jackson explotó a mitad de `languages[0].level`. **Fix**: `max-tokens=4096` en `application.properties`. Bug latente adicional: `1024.` con punto final, parseado por Spring con tolerancia pero malformado.

Reflexión honesta en sesión: los tres bugs son inherentes a trabajar con LLMs no deterministas + stack Spring AI todavía inmaduro, no bugs del código propio. Decisión Tole: continuar hasta cerrar los tres (opción B ofrecida por Claude). Buena decisión: cierre limpio, no queda deuda encadenada.

**Fase 7 — cleanup + dos commits separados**:

- `test-data/` verificado en `.gitignore` (línea 41). Sin riesgo de subir CVs personales al repo público.
- Estrategia elegida: **B, dos commits separados** por naturaleza distinta del cambio (feat de cache-aside vs fix de prompts+config, este último aplicable también al endpoint de texto).
- Commit 1: cache-aside PDF (`CvAnalysisCache.java` + `AnalyzePdfController.java`).
- Commit 2: fix prompts + max-tokens (`analyze-cv.st` + `analyze-cv-pdf.st` + `application.properties`).
- Ambos con mensaje bilingüe extenso, parte española sin tildes/ñ.

Push exitoso a `origin/main`. Historia limpia.

## Decisiones lockeadas

1. **Sobrecarga de métodos con corazón común extraído** como patrón preferido cuando dos operaciones difieren solo en un paso pero comparten el resto. Alternativa (clases separadas) reservada a casos con divergencia real de comportamiento o política.

2. **Materializar recursos consumibles UNA vez** (`byte[] pdfBytes = file.getBytes()`) y trabajar con la copia local. **Nunca volver al recurso original** aunque tu implementación actual sea consistente entre lecturas — el contrato de la interfaz no lo garantiza y crea dependencia oculta.

3. **`ByteArrayResource` como wrapper idiomático** cuando necesitas pasar bytes en memoria a APIs que esperan `Resource` de Spring (Spring AI multimodal, algunos exporters, etc.).

4. **Checked exceptions se declaran con `throws`** cuando no hay recuperación local útil. **Anti-patrón**: envolver checked en `RuntimeException` sin aportar información contextual.

5. **En cache-aside, el `put` va después de toda validación y transformación** del resultado. Nunca cachear resultados intermedios ni respuestas rotas. Garantiza que un retry posterior con el bug arreglado repuebla la caché con valor correcto sin intervención manual.

6. **`MessageDigest.digest()` sin argumento devuelve hash del vacío** (`e3b0c44...`). Trampa clásica que compila y "funciona" pero rompe la lógica silenciosamente. Siempre usar `.digest(byte[])` con el input explícito.

7. **Prompts en `resources/prompts/` en ASCII plano** cuando incluyan instrucciones sobre formato JSON o mencionen caracteres especiales. StringTemplate 4 (el motor de Spring AI) reserva `{ } < > \`` y explota en tiempo de compilación. Describir caracteres especiales en prosa (`la llave abierta`, `el acento grave`), no mostrarlos literalmente.

8. **`max-tokens` dimensionado con margen 3-4x** sobre el peor caso esperado del schema de salida estructurada. Respuestas truncadas se manifiestan como `UnexpectedEndOfInputException` a mitad de un campo. Solo se factura por tokens generados, no por el techo — subir el margen no aumenta el coste normal.

9. **Estrategia de commits: separar `feat` de `fix` incluso cuando surgen en la misma sesión**. Refleja mejor la historia real, facilita `git bisect`, y permite revert individual. En este caso: cache-aside PDF (feat, solo afecta al endpoint nuevo) separado de prompts+max-tokens (fix, aplica también al endpoint de texto ya existente).

10. **`test-data/` ignorado en `.gitignore` desde el inicio**. Los ficheros de test con CVs personales, datos reales, o cualquier input sensible para validación local NO viajan al repo público. Verificar con `git check-ignore -v` antes de cualquier `git add`.

## Deuda / Gotchas al cerrar

1. **Bug 1 (fences markdown en JSON) mitigado solo con prompt reforzado**. Es primera línea de defensa. **Deuda pendiente**: si el bug reaparece estadísticamente en producción, añadir **post-procesado defensivo** en los controllers antes de `converter.convert()` — strip de `` ```json `` inicial y `` ``` `` final. Es defensa en profundidad (patrón interview-ready): prompt fuerte + strip defensivo + validación estricta del parseado. Los tres cubren capas distintas.

2. **Contenedores Redis levantados al cerrar sesión**. Recomendación: `docker compose down` al terminar el día.

3. **`analyze-cv-pdf.st` y `analyze-cv.st` sin tildes en las instrucciones nuevas**. Legible pero visualmente inferior. Deuda futura: verificar si StringTemplate 4 en configuración por defecto realmente rompe con UTF-8 no ASCII en zonas de texto (probablemente no, solo en zonas de sintaxis) y restaurar tildes si es seguro. Prioridad baja.

4. **Deuda técnica heredada del arco Redis (sin cambios respecto a sesiones anteriores)**:
   - Investigar bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` (duplicación en Redis cuando ambos activos). Probar `getOrder()` con `MONITOR`.
   - `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository` vía `SessionCallback<>`.
   - Mover TTL de chat memory Y de `CvAnalysisCache` a `application.properties` como propiedades tipo `document-analyzer.chat-memory.ttl-hours=24` y `document-analyzer.analyze-cache.ttl-hours=24`.

5. **`spring.servlet.multipart.max-file-size` no configurado explícitamente**. Usa el default de Spring Boot (típicamente 1 MB). Deuda si se despliega producción: fijar límite explícito. Para lab local con CVs de 200-500 KB no es problema.

6. **`catch (Exception e)` demasiado genérico en `CvAnalysisCache`** — heredado de la sesión anterior, aún vigente. Refactor recomendado a excepciones específicas (`DataAccessException`, `JsonProcessingException`) en producción.

## Frases ⭐⭐⭐ interview-ready

**Sobrecarga con corazón común**:
> Cuando dos operaciones difieren solo en un paso pero comparten toda la lógica de infraestructura (Redis, JSON, fail-safe, TTL), la solución idiomática Java es sobrecarga de métodos con extracción del corazón común a métodos privados. Los públicos quedan como one-liners orquestadores que construyen la clave y delegan; toda la maquinaria pesada existe una sola vez. Alternativa (clases separadas) queda reservada a casos con divergencia real de comportamiento o política, no solo de input.

**Recursos consumibles y "materializar una vez"**:
> Cuando trabajas con un recurso consumible (stream, upload, iterator, publisher) y necesitas usarlo N veces, materialízalo una vez en memoria o disco y trabaja con la copia. Nunca vuelvas al recurso original, aunque tu implementación actual sea consistente entre lecturas — el contrato de la interfaz no lo garantiza y creas dependencia oculta que puede romperse en producción con implementaciones streamed o rebalances.

**`MessageDigest` trampa clásica**:
> `MessageDigest.digest()` sin argumento devuelve el hash del contenido acumulado con `update()` — si no has hecho `update()` previo, devuelve el hash del string vacío (`e3b0c44...`) siempre. Es una trampa clásica: compila, "funciona" en el sentido de que devuelve algo, pero rompe la lógica silenciosamente. Siempre `.digest(byte[])` con el input explícito.

**StringTemplate en Spring AI**:
> Los ficheros `.st` de Spring AI se procesan con StringTemplate 4, que tiene caracteres reservados (llaves, ángulos, backticks). Prompts con estos caracteres literales rompen el parser antes de llegar al LLM, con error tipo "invalid character" en tiempo de compilación de la plantilla. Mitigación: prompts en ASCII plano, describir caracteres especiales en prosa. Alternativamente, configurar `NoOpTemplateRenderer` si no se necesita interpolación de variables.

**Respuestas LLM truncadas y `max-tokens`**:
> Cuando parseas JSON estructurado de respuestas LLM y aparece `UnexpectedEndOfInputException` a mitad de un campo, no es bug de código: es la respuesta LLM truncada por `max-tokens` demasiado bajo. Dimensionar con margen 3-4x sobre el peor caso esperado del schema. Solo se factura por tokens realmente generados, no por el techo — subir el margen no aumenta el coste normal, solo evita truncados en outliers.

**Cache-aside cero-basura**:
> En cache-aside, la posición del `put` es diseño intencional: va después de que el resultado esté completamente validado y transformado. Si el flujo revienta antes (LLM devuelve respuesta rota, parseo falla, transformación explota), la caché no se pobla con basura. Consecuencia práctica: un retry posterior con el bug arreglado cachea el resultado correcto sin intervención manual — no hace falta `FLUSHALL` para "limpiar respuestas rotas cacheadas".

**Manejo de uploads en producción**:
> El manejo de uploads en Spring MVC por defecto materializa el fichero en memoria vía `MultipartFile.getBytes()`. Para producción: límite explícito en `spring.servlet.multipart.max-file-size` como primera línea de defensa, streaming a disco temporal o subida directa a S3 con URL prefirmada según el tamaño y volumen esperados, y rate limiting por IP/usuario con Bucket4j. Sin límite duro, cualquier upload sin control es vector de ataque trivial: 100 uploads concurrentes de 50 MB tumban la JVM con OOM.

**Estrategia de commits feat vs fix**:
> Separar `feat` de `fix` en commits distintos incluso cuando surgen en la misma sesión refleja mejor la historia del cambio. Facilita `git bisect` al debuggear regresiones y permite revert individual: si mañana el fix rompe algo, lo revierto sin perder el feat. Especialmente útil cuando el fix aplica a más superficie que el feat: en este caso, cache-aside PDF (feat, un solo endpoint) vs prompts+max-tokens (fix, ambos endpoints).

## Estado de la app y roadmap

- `document-analyzer-ai` con **cache-aside en ambos endpoints** (`/analyze` y `/analyze/pdf`), prompts robustecidos, `max-tokens` dimensionado.
- **Proyecto 1 del roadmap AI Engineer** (`document-analyzer-ai`) sustancialmente completo. Falta según roadmap oficial: tres proveedores LLM intercambiables (solo Anthropic hoy), tests con Testcontainers, defensa activa contra prompt injection.
- **Bloque 4 Redis cerrado 100%** (RDB/AOF + replicación + cache-aside texto + cache-aside PDF). El arco Redis sobre `document-analyzer-ai` queda terminado.
- Próximo hito natural del roadmap AI: **Fase 2.2, ReAct agent en `erp-purchasing-agent`** (Proyecto 2 oficial). Consume el `erp-mcp-server` ya construido. Confirmado que este es el orden correcto del roadmap `roadmap_java_ai_engineer_v2_2.md`.
- Roadmap post-Git/Bash oficial: **AWS Sesión 11** (Terraform remote state con S3+DynamoDB) sigue siendo el siguiente hito del eje AWS.

## Correcciones y aprendizajes de proceso

- **Feedback aplicado durante la sesión sobre "código regalado"**: en sesión anterior Tole pidió explícitamente Sócrates puro sin cuerpos de métodos pegados. Aplicado consistentemente en esta sesión. Excepción única y justificada: pegué el esqueleto de la opción C (con nombres de métodos, no cuerpos) para que Tole viera la forma antes de decidir opción A/B/C. Aceptado por el patrón "ver la forma antes de comprometerse a la opción".

- **Distracción de Tole al ignorar recomendación sobre `throws IOException`**: implementó try/catch + RuntimeException a pesar de recomendación explícita. Al preguntar por qué, respondió honestamente "distracción". Aceptado, fix aplicado. Aprendizaje mutuo: cuando doy recomendación explícita y el interlocutor la ignora, pedir justificación breve antes de aceptar el código como final.

- **Error mío importante con backticks en prompt**: propuse un prompt "reforzado" con backticks literales sin verificar compatibilidad con StringTemplate. Cambié un bug (fences markdown) por otro peor (parser explota antes de llegar al LLM). Reconocido explícitamente en sesión, disculpa clara, fix aplicado. Aprendizaje interno: siempre verificar que las propuestas de código/config no introduzcan nuevos bugs, especialmente cuando toco componentes del stack que conozco menos (StringTemplate 4 específicamente).

- **Onion debugging (Fred Brooks) reconocido en sesión**: tres bugs encadenados donde cada arreglo revela el siguiente. No fue mala planificación, es la naturaleza inherente de LLMs no deterministas + stack Spring AI aún inmaduro. Tole eligió continuar (opción B) en vez de commitear y aplazar (opción A). Buena decisión: cierre completo sin dejar deuda encadenada.

- **Tole leyó sus propias notas antes de arrancar**: iniciativa proactiva. Cargó decisiones lockeadas y frases interview-ready del domingo antes de la implementación de hoy. Refuerza el valor de las notas de sesión como memoria externa consolidada. Sistema funcionando como estaba diseñado.

- **Fatiga natural en fase de debugging**: energía descendió tras el bug 2 (StringTemplate). Reconocido y ofrecida opción A (cerrar limpio, deuda técnica) o B (continuar 20 min). Tole eligió B con criterio, no por inercia. Cierre limpio validado con éxito real.

- **Ansiedad sobre job search reconocida al arrancar y NO forzada durante la sesión**: Tole mencionó "Candidaturas... mejor no hablar" en la sesión del domingo. En esta sesión no volvió a mencionar el tema. Claude respetó la señal, no insistió, y la sesión técnica avanzó sin friction emocional. Buena calibración.

- **Comentario "// imports que averigüas tú" arrastrado desde fase de andamiaje**: bug de higiene detectado en review, corregido antes del commit. Recordatorio: código comentado y placeholders de fase temprana NO deben viajar a commits.

## Próxima sesión

**Bloque 4 Redis cerrado (4/4 subtemas)**. Arco Redis sobre `document-analyzer-ai` completo.

Opciones ordenadas por continuidad y valor:

1. **AWS Sesión 11**: Terraform remote state con S3+DynamoDB. Continuación del roadmap post-Git/Bash oficial. Cabe en 2h. Eje que más pesa para ATS de banca/consultoría. Toca por rotación día sí/no.

2. **AI Fase 2.2 — ReAct agent en `erp-purchasing-agent`** (S16 según planificación previa). Continuación natural del roadmap AI (Proyecto 2 oficial). Consume el `erp-mcp-server` ya construido. Sesión propia extensa (probablemente 2 sesiones).

3. **Bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor`** en `/chat` — con `MONITOR` y `getOrder()`. 45-60 min. Cierra deuda técnica antigua del arco Redis en el proyecto.

4. **Sesión de re-lectura guiada** de algún tema oxidado. Formato acordado (no socrático) documentado en `PromptArranque-ReLecturaGuiada.md`. Tema propuesto para primera vez: **MapStruct** (más contenido, menos amenazante). Cuando la energía esté baja y Tole se sienta con margen para intentarlo.

5. **Defensa en profundidad para bug de fences markdown**: añadir post-procesado en ambos controllers que strip `` ```json `` inicial y `` ``` `` final antes de `converter.convert()`. 20-30 min. Cierra la deuda #1 documentada.

Ver también `PromptContinuacion-*.md` cuando se genere el de la próxima sesión (no creado aún — pendiente decisión de qué priorizar).
