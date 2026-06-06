# Quizzes y ejercicios interactivos

> **Para el agente copiloto**: Este archivo contiene quizzes, break it exercises y
> decision points para el `LEARNING_ROADMAP.md`. Presenta cada pregunta como una tarjeta
> interactiva. Espera la respuesta del usuario antes de mostrar la solución.
>
> **Formato de presentación**:
> - Para quizzes: muestra la pregunta y las opciones (a, b, c, d). Pide al usuario que elija.
> - Para break it: muestra la instrucción y pregunta "¿Qué observas? ¿Por qué crees que pasa esto?"
> - Para decision points: muestra el escenario y las opciones. Pide al usuario que justifique su elección.
>
> Después de cada respuesta, muestra la solución y la explicación.

---

## 📝 Mini-quizzes por módulo

---

### Quiz M0: Fundamentos

> **Cuándo presentar**: Al completar la Sesión 0.1 (antes de empezar M0.5)

#### Pregunta M0.1: Embeddings

Un embedding convierte texto en un vector de números. Si dos textos tienen vectores muy cercanos en el espacio vectorial, ¿qué significa?

a) Que los textos son idénticos palabra por palabra
b) Que los textos tienen significado o contexto similar
c) Que los textos tienen la misma longitud
d) Que los textos fueron escritos por el mismo autor

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

Los embeddings capturan **similitud semántica**, no similitud léxica. "El perro corre" y "El can galopa" tendrán vectores cercanos aunque no compartan palabras, porque significan algo similar.

**Por qué las otras están mal**:
- a) Los embeddings no requieren palabras idénticas, solo significado similar
- c) La longitud del texto no determina la cercanía de los vectores
- d) Los embeddings no detectan autoría
</details>

---

#### Pregunta M0.2: Vector store

¿Qué operación realiza un vector store cuando buscas documentos relevantes para una pregunta?

a) Busca palabras clave exactas en los documentos
b) Calcula la similitud entre el vector de la pregunta y los vectores de los documentos
c) Cuenta cuántas veces aparece la pregunta en cada documento
d) Ordena los documentos por fecha de creación

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El vector store calcula **similitud coseno** (o distancia euclidiana) entre el embedding de la query y los embeddings almacenados. Los documentos con vectores más cercanos son más relevantes semánticamente.

**Por qué las otras están mal**:
- a) Eso es búsqueda por palabras clave (BM25), no búsqueda semántica
- c) Contar ocurrencias es un enfoque ingenuo que no captura significado
- d) La fecha no tiene relación con la relevancia semántica
</details>

---

#### Pregunta M0.3: RAG

¿Cuál es el orden correcto del flujo RAG?

a) Usuario pregunta → LLM genera respuesta → se buscan documentos → se devuelve respuesta con fuentes
b) Usuario pregunta → se buscan documentos relevantes → se da contexto al LLM → LLM genera respuesta con fuentes
c) Usuario pregunta → LLM busca en internet → se filtran resultados → se genera respuesta
d) Usuario pregunta → se clasifica la pregunta → se elige modelo → se genera respuesta

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El flujo RAG es: **Recuperar** (buscar documentos relevantes) → **Aumentar** (añadir contexto al prompt) → **Generar** (LLM responde con el contexto). El orden es crítico: primero recuperas, luego generas.

**Por qué las otras están mal**:
- a) Generar antes de buscar es alucinación, no RAG
- c) RAG no busca en internet, busca en tu base de documentos
- d) Clasificar y elegir modelo son pasos opcionales, no el flujo RAG estándar
</details>

---

### Quiz M2: Documentos y Loaders

> **Cuándo presentar**: Al completar la Sesión 2.3 (antes de empezar M3)

#### Pregunta M2.1: Loader-first architecture

¿Por qué el pipeline usa un modelo `Document` común en vez de procesar cada formato directamente?

a) Porque es más rápido procesar un solo tipo de objeto
b) Porque el resto del pipeline (chunking, embedding, query) no necesita saber el formato original
c) Porque Python no puede manejar múltiples formatos a la vez
d) Porque los embeddings solo funcionan con el modelo Document

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El patrón **loader-first** desacopla la carga de documentos del procesamiento. Los loaders normalizan todo a `Document`, y el resto del pipeline (chunking, embedding, retrieval) trabaja con ese modelo común. Si mañana añades PDF, solo creas un loader nuevo sin tocar nada más.

**Por qué las otras están mal**:
- a) La velocidad no es el motivo principal, es la mantenibilidad
- c) Python puede manejar múltiples formatos, pero sería más complejo
- d) Los embeddings funcionan con texto, no con el modelo Document
</details>

---

#### Pregunta M2.2: Chunking

Tienes un documento Markdown de 5.000 palabras con 10 secciones (H2). ¿Cuál es la estrategia de chunking más apropiada?

a) Un solo chunk con todo el documento
b) Dividir por párrafos (doble salto de línea)
c) Dividir por headings H2, respetando la estructura del documento
d) Dividir en chunks de 500 tokens fijos

<details>
<summary><strong>Respuesta correcta: c</strong></summary>

Para Markdown, **dividir por headings H2** preserva la estructura semántica del documento. Cada sección es un chunk coherente. Esto mejora el retrieval porque los chunks tienen contexto completo.

**Por qué las otras están mal**:
- a) Un chunk de 5.000 palabras es demasiado grande para el context window y diluye la relevancia
- b) Dividir por párrafos puede cortar en medio de una idea
- d) Chunks fijos pueden cortar en medio de una sección o frase
</details>

---

#### Pregunta M2.3: Structured loader

¿Por qué el loader JSON/YAML "aplana" los datos a texto legible en vez de guardar el JSON original?

a) Porque los embeddings no funcionan con JSON
b) Porque el texto aplanado se busca mejor semánticamente que un JSON estructurado
c) Porque ocupa menos espacio en el vector store
d) Porque el modelo de embedding solo acepta texto plano

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

Un JSON como `{"ingredients": ["huevos", "patatas"]}` no se busca bien semánticamente. Al aplanarlo a "Ingredients: huevos, patatas", el embedding captura mejor el significado. El retrieval funciona con texto natural, no con estructura JSON.

**Por qué las otras están mal**:
- a) Los embeddings funcionan con texto, y el JSON se puede convertir a texto
- c) El espacio ocupado es similar
- d) Los modelos de embedding aceptan texto, y el JSON aplanado es texto
</details>

---

### Quiz M3: Embeddings + Vector Store

> **Cuándo presentar**: Al completar la Sesión 3.3 (antes de empezar M4)

#### Pregunta M3.1: Task prefixes

¿Por qué el adapter de Ollama usa prefijos distintos para documentos (`search_document:`) y queries (`search_query:`)?

a) Porque es un requisito de la API de Ollama
b) Porque ayuda al modelo de embedding a distinguir entre indexar y buscar, mejorando la calidad del retrieval
c) Porque los prefijos hacen que los vectores sean más pequeños
d) Porque sin prefijos, Chroma rechaza los embeddings

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

Los **task prefixes** (también llamados instruction prefixes) indican al modelo de embedding qué tarea está realizando. Esto mejora la calidad porque el modelo ajusta la representación según el contexto: indexar un documento es diferente a buscar una pregunta.

**Por qué las otras están mal**:
- a) Ollama no requiere prefijos, son una mejora de calidad
- c) Los prefijos no afectan al tamaño de los vectores
- d) Chroma acepta embeddings con o sin prefijos
</details>

---

#### Pregunta M3.2: Adapter pattern

¿Qué ventaja principal ofrece el adapter pattern en este proyecto?

a) Hace que el código sea más rápido
b) Permite cambiar de Ollama a OpenAI (o de Chroma a Qdrant) sin tocar el dominio
c) Reduce el consumo de memoria
d) Simplifica la instalación de dependencias

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El **adapter pattern** desacopla las interfaces (`Embedder`, `VectorStore`) de las implementaciones concretas (`OllamaEmbedder`, `ChromaStore`). Si mañana quieres usar OpenAI en vez de Ollama, solo creas un `OpenAIEmbedder` que implemente la interfaz `Embedder`. El pipeline de ingestión y query no se toca.

**Por qué las otras están mal**:
- a) El adapter pattern no afecta al rendimiento
- c) No reduce memoria, solo mejora mantenibilidad
- d) No simplifica la instalación, es un patrón de diseño
</details>

---

#### Pregunta M3.3: Incremental indexing

¿Cómo sabe el pipeline de ingestión qué archivos han cambiado desde la última indexación?

a) Compara la fecha de modificación del archivo
b) Compara el SHA256 del archivo con el manifest almacenado
c) Compara el tamaño del archivo
d) Vuelve a indexar todo cada vez

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El pipeline calcula el **SHA256** de cada archivo y lo compara con el manifest (`index_manifest.json`). Si el hash cambió, el archivo se re-indexa. Si no cambió, se omite. Esto es más fiable que la fecha de modificación (que puede cambiar sin que el contenido cambie).

**Por qué las otras están mal**:
- a) La fecha de modificación puede cambiar sin que el contenido cambie (ej. al copiar un archivo)
- c) El tamaño puede ser el mismo aunque el contenido sea diferente
- d) Re-indexar todo es ineficiente para colecciones grandes
</details>

---

### Quiz M4: Generación + RAG completo

> **Cuándo presentar**: Al completar la Sesión 4.3 (antes de empezar M5)

#### Pregunta M4.1: System prompt

¿Cuál es la función principal del system prompt en el pipeline RAG?

a) Hacer que el modelo responda más rápido
b) Enmarcar el comportamiento del modelo y evitar alucinaciones
c) Reducir el consumo de tokens
d) Mejorar la calidad de los embeddings

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El **system prompt** instruye al modelo sobre cómo comportarse: "Responde basándote exclusivamente en los documentos proporcionados. Si no hay información, di que no encuentras." Esto reduce alucinaciones porque el modelo sabe que debe limitarse al contexto.

**Por qué las otras están mal**:
- a) El system prompt no afecta a la velocidad
- c) El system prompt añade tokens, no los reduce
- d) Los embeddings se generan antes de la generación, el system prompt no los afecta
</details>

---

#### Pregunta M4.2: Streaming

¿Por qué el streaming es importante en una aplicación de chat con LLM?

a) Porque reduce el coste de la API
b) Porque el usuario ve la respuesta progresivamente, mejorando la UX
c) Porque hace que el modelo genere respuestas más cortas
d) Porque permite usar modelos más pequeños

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El **streaming** muestra los tokens conforme se generan, en vez de esperar a que el modelo termine toda la respuesta. Esto mejora la UX porque el usuario no espera 10 segundos sin feedback. Es especialmente importante en modelos locales que son más lentos.

**Por qué las otras están mal**:
- a) El streaming no reduce el coste, solo cambia cómo se recibe la respuesta
- c) El streaming no afecta a la longitud de la respuesta
- d) El streaming funciona con cualquier tamaño de modelo
</details>

---

#### Pregunta M4.3: Diagnóstico first

Antes de cambiar el system prompt porque "las respuestas son malas", ¿qué deberías hacer primero?

a) Cambiar el modelo a uno más grande
b) Usar `rag search` para verificar que el retrieval está funcionando bien
c) Aumentar la temperatura del modelo
d) Añadir más documentos al índice

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El principio **diagnóstico first** dice: antes de cambiar generación, verifica retrieval. Usa `rag search "pregunta"` para ver si los chunks correctos aparecen en los top-5. Si el retrieval falla, cambiar el prompt no ayudará. Si el retrieval funciona, entonces el problema es la generación.

**Por qué las otras están mal**:
- a) Cambiar de modelo es una solución cara que puede no resolver el problema real
- c) Aumentar la temperatura hace las respuestas más aleatorias, no mejores
- d) Añadir más documentos no ayuda si los documentos correctos ya están indexados pero no se recuperan
</details>

---

### Quiz M5: Evaluación + Structured Output

> **Cuándo presentar**: Al completar la Sesión 5.3 (antes de empezar M5.5)

#### Pregunta M5.1: Evaluation set

¿Por qué es importante tener un evaluation set antes de cambiar parámetros del sistema?

a) Para demostrar que el sistema funciona a tu jefe
b) Para medir objetivamente si un cambio mejora o empeora el sistema
c) Para cumplir con requisitos de calidad de software
d) Para generar datos de entrenamiento para el modelo

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El **evaluation set** te da una métrica objetiva. Cambias chunk size de 500 a 1000 tokens, re-ejecutas `rag eval`, y ves si el hit rate subió o bajó. Sin evaluación, estás adivinando si los cambios mejoran o empeoran el sistema.

**Por qué las otras están mal**:
- a) La evaluación es para ti, no para demostrar nada a terceros
- c) No es un requisito burocrático, es una herramienta técnica
- d) El evaluation set no se usa para entrenar, solo para medir
</details>

---

#### Pregunta M5.2: JSON mode

¿Por qué es importante usar `format: "json"` en Ollama cuando necesitas structured output?

a) Porque hace que el modelo responda más rápido
b) Porque fuerza al modelo a devolver JSON válido en vez de texto libre
c) Porque reduce el número de tokens generados
d) Porque mejora la calidad de las respuestas

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El **JSON mode** (`format: "json"`) fuerza al modelo a generar solo JSON válido. Sin esto, el modelo puede añadir texto antes o después del JSON, o usar comillas simples en vez de dobles, rompiendo el parsing. Con JSON mode, sabes que la salida es parseable.

**Por qué las otras están mal**:
- a) El JSON mode no afecta a la velocidad
- c) El JSON mode no reduce tokens, puede que los aumente ligeramente
- d) El JSON mode no mejora la calidad, solo garantiza el formato
</details>

---

#### Pregunta M5.3: Consistency testing

¿Por qué ejecutas el mismo documento 3 veces en el pipeline de clasificación?

a) Para triplicar el número de documentos procesados
b) Para verificar que el modelo es consistente y no varía aleatoriamente
c) Para promediar los tiempos de ejecución
d) Para entrenar al modelo con más datos

<details>
<summary><strong>Respuesta correcta: b</strong></summary>

El **consistency testing** verifica que el modelo da la misma clasificación (o muy similar) para la misma entrada. Si las 3 ejecuciones dan categorías diferentes, el modelo no es fiable para esa tarea y necesitas ajustar el prompt o cambiar de modelo.

**Por qué las otras están mal**:
- a) Ejecutar 3 veces no multiplica los documentos, solo verifica consistencia
- c) No se promedian tiempos, se verifica consistencia de categorías
- d) No se entrena al modelo, solo se evalúa
</details>

---

## 🔨 Break it exercises

---

### Break it 3.2: Ollama adapter

> **Cuándo presentar**: Al completar la Sesión 3.2

#### Experimento 1: Sin prefijos

**Instrucción**: En `src/rag_assistant/adapters/ollama.py`, cambia `self.doc_prefix` y `self.query_prefix` a `""` (string vacío). Re-indexa los documentos con `rag ingest --force` (o borra `data/chroma/` y re-indexa). Luego ejecuta `rag search "tortilla"` y compara los resultados con los originales.

**Pregunta para el usuario**: ¿Qué observas? ¿Los resultados son diferentes? ¿Por qué crees que pasa esto?

<details>
<summary><strong>Qué deberías observar</strong></summary>

Los resultados pueden ser ligeramente peores o diferentes. Los prefijos `search_document:` y `search_query:` ayudan al modelo de embedding (especialmente nomic-embed-text) a distinguir entre indexar un documento y buscar una pregunta. Sin prefijos, el modelo trata ambos casos igual, lo que puede reducir la precisión del retrieval.

**Si no notas diferencia**: Es posible que con documentos muy cortos o queries muy específicas, el efecto sea mínimo. Prueba con documentos más largos o queries más ambiguas.
</details>

---

#### Experimento 2: Documento sin chunking

**Instrucción**: Crea un documento `docs/largo.md` con 10.000 palabras (puedes copiar un artículo largo de Wikipedia). Modifica temporalmente el chunking en `src/rag_assistant/ingest.py` para que no divida el documento (un solo chunk). Re-indexa y busca algo específico de ese documento.

**Pregunta para el usuario**: ¿El retrieval encuentra el chunk correcto? ¿Por qué sí o no? ¿Qué problemas puede causar un chunk tan grande?

<details>
<summary><strong>Qué deberías observar</strong></summary>

El retrieval puede encontrar el chunk, pero:
1. El chunk es tan grande que satura el context window del LLM
2. La respuesta del LLM puede ser imprecisa porque hay demasiada información
3. El embedding del chunk es un "promedio" de todo el documento, lo que diluye la relevancia para queries específicas

**Lección**: Los chunks deben ser lo suficientemente pequeños para ser específicos, pero lo suficientemente grandes para tener contexto completo.
</details>

---

### Break it 3.3: Chroma + ingest

> **Cuándo presentar**: Al completar la Sesión 3.3

#### Experimento 1: Cambiar métrica de similitud

**Instrucción**: En `src/rag_assistant/adapters/chroma.py`, cambia `{"hnsw:space": "cosine"}` a `{"hnsw:space": "l2"}` (distancia euclidiana). Borra `data/chroma/` y re-indexa. Ejecuta `rag search "receta con huevos"` y compara los resultados.

**Pregunta para el usuario**: ¿Los resultados son diferentes? ¿Cuál métrica parece funcionar mejor para tu caso? ¿Por qué?

<details>
<summary><strong>Qué deberías observar</strong></summary>

Los resultados pueden ser ligeramente diferentes. **Coseno** mide el ángulo entre vectores (dirección), mientras que **L2** mide la distancia euclidiana (magnitud). Para embeddings normalizados (como nomic-embed-text), coseno suele funcionar mejor porque los vectores ya están normalizados.

**Lección**: La métrica de similitud importa, pero para la mayoría de casos, coseno es la opción segura.
</details>

---

#### Experimento 2: Sin incremental indexing

**Instrucción**: En `src/rag_assistant/ingest.py`, comenta temporalmente la lógica del manifest (las líneas que calculan SHA256 y comparan con el manifest). Haz que `run_ingest` siempre procese todos los archivos. Ejecuta `rag ingest` dos veces y mide el tiempo.

**Pregunta para el usuario**: ¿Cuánto tarda la segunda ejecución comparada con la original? ¿Por qué es importante el incremental indexing?

<details>
<summary><strong>Qué deberías observar</strong></summary>

La segunda ejecución tarda lo mismo que la primera, porque re-procesa todos los archivos. Con incremental indexing, la segunda ejecución es casi instantánea (solo verifica hashes).

**Lección**: El incremental indexing es crítico para colecciones grandes. Sin él, cada ingest es O(n) donde n es el número total de archivos. Con incremental, es O(k) donde k es el número de archivos cambiados.
</details>

---

### Break it 4.1: Query chain

> **Cuándo presentar**: Al completar la Sesión 4.1

#### Experimento 1: Sin system prompt

**Instrucción**: En `src/rag_assistant/query.py`, cambia `system=SYSTEM_PROMPT` a `system=""` (o elimina el parámetro). Ejecuta `rag ask "¿Qué dice el documento de prueba?"` y compara la respuesta con la original.

**Pregunta para el usuario**: ¿La respuesta es diferente? ¿El modelo alucina o se inventa información? ¿Por qué el system prompt es importante?

<details>
<summary><strong>Qué deberías observar</strong></summary>

Sin system prompt, el modelo puede:
1. Inventar información que no está en los documentos (alucinación)
2. Responder con conocimiento general en vez de limitarse al contexto
3. No citar las fuentes

**Lección**: El system prompt es tu principal herramienta de control. Sin él, el modelo no sabe que debe limitarse al contexto proporcionado.
</details>

---

#### Experimento 2: Temperatura alta

**Instrucción**: En `src/rag_assistant/query.py`, cambia `temperature=0.1` a `temperature=1.0`. Ejecuta `rag ask "¿Qué ingredientes necesita la tortilla?"` tres veces.

**Pregunta para el usuario**: ¿Las tres respuestas son iguales? ¿Son precisas? ¿Por qué la temperatura afecta a la consistencia?

<details>
<summary><strong>Qué deberías observar</strong></summary>

Con temperatura alta, las respuestas varían significativamente entre ejecuciones. Algunas pueden ser incorrectas o inventar ingredientes. La temperatura controla la "creatividad" del modelo: baja (0.1) = determinista, alta (1.0) = aleatorio.

**Lección**: Para RAG y tareas que requieren precisión, usa temperatura baja (0.1-0.3). Para tareas creativas, puedes subir a 0.7-0.9.
</details>

---

## 🎯 Decision points

---

### Decision point 3.3: Estrategia de chunking

> **Cuándo presentar**: Al completar la Sesión 3.3, antes de pasar a M4

#### Escenario

Estás indexando un documento legal de 50 páginas con secciones muy desiguales: algunas secciones tienen 200 palabras, otras tienen 5.000 palabras. El documento tiene una estructura clara con headings H2 y H3.

#### Opciones

**Opción A**: Chunking por headings (como en el roadmap). Cada sección H2 es un chunk.

**Opción B**: Chunking por tamaño fijo. Dividir en chunks de 500 tokens con overlap de 50 tokens.

**Opción C**: Chunking híbrido. Dividir por headings, pero si una sección supera 1.000 tokens, subdividirla por párrafos.

#### Pregunta para el usuario

¿Cuál opción eliges y por qué? Piensa en:
- ¿Qué pasa con las secciones de 5.000 palabras?
- ¿Qué pasa con las secciones de 200 palabras?
- ¿Cómo afecta al retrieval?

<details>
<summary><strong>Análisis de cada opción</strong></summary>

**Opción A (por headings)**:
- ✅ Preserva la estructura semántica del documento
- ❌ Las secciones de 5.000 palabras son chunks demasiado grandes (saturan context window, diluyen relevancia)
- ❌ Las secciones de 200 palabras pueden ser chunks demasiado pequeños (pierden contexto)

**Opción B (tamaño fijo)**:
- ✅ Todos los chunks tienen tamaño uniforme
- ❌ Corta en medio de frases o secciones
- ❌ Pierde la estructura del documento (un chunk puede contener partes de dos secciones diferentes)

**Opción C (híbrido) — RECOMENDADA**:
- ✅ Preserva la estructura semántica (headings)
- ✅ Maneja secciones largas subdividiéndolas por párrafos (coherencia)
- ✅ Las secciones cortas se mantienen como chunks únicos
- ❌ Más compleja de implementar

**Conclusión**: La opción C es la más robusta para documentos legales. Combina lo mejor de A y B: estructura semántica + manejo de secciones largas.
</details>

---

### Decision point 4.1: System prompt para chatbot legal

> **Cuándo presentar**: Al completar la Sesión 4.1, antes de pasar a 4.2

#### Escenario

Estás construyendo un chatbot para abogados que buscan información en una base de datos de sentencias judiciales. Los abogados necesitan respuestas precisas con citas exactas a las sentencias.

#### Opciones

**Opción A (estricto)**:
```
Eres un asistente legal experto. Responde ÚNICAMENTE basándote en las sentencias proporcionadas.
Si la información no está en las sentencias, responde: "No encuentro información en las sentencias proporcionadas."
Cita siempre el número de sentencia y el párrafo exacto.
No interpretes ni opines, solo reporta lo que dicen las sentencias.
```

**Opción B (permissivo)**:
```
Eres un asistente legal. Ayuda al usuario a encontrar información en las sentencias.
Si no encuentras información exacta, puedes inferir o sugerir basándote en tu conocimiento legal.
Sé útil y proporciona contexto adicional cuando sea relevante.
```

**Opción C (balanceado)**:
```
Eres un asistente legal experto. Responde basándote principalmente en las sentencias proporcionadas.
Si la información está en las sentencias, cítalas con número de sentencia y párrafo.
Si la información no está completamente clara, puedes complementar con conocimiento legal general, pero indícalo claramente.
Prioriza la precisión sobre la exhaustividad.
```

#### Pregunta para el usuario

¿Cuál opción eliges para un chatbot legal? ¿Por qué? Piensa en:
- ¿Qué pasa si el modelo alucina en un contexto legal?
- ¿Los abogados necesitan respuestas estrictas o pueden aceptar inferencias?
- ¿Qué pasa si el modelo dice "no sé" demasiado a menudo?

<details>
<summary><strong>Análisis de cada opción</strong></summary>

**Opción A (estricto)**:
- ✅ Minimiza alucinaciones (crítico en legal)
- ✅ Citas exactas (los abogados necesitan referencias verificables)
- ❌ Puede decir "no sé" demasiado a menudo si las sentencias no son exhaustivas
- ❌ No aporta valor añadido más allá de lo que está en las sentencias

**Opción B (permissivo)**:
- ✅ Más útil y conversacional
- ❌ Alto riesgo de alucinaciones (peligroso en legal)
- ❌ Los abogados no pueden confiar en respuestas no verificadas
- ❌ "Inferir" en legal es peligroso

**Opción C (balanceado) — RECOMENDADA para la mayoría de casos**:
- ✅ Prioriza precisión (crítico en legal)
- ✅ Permite complementar cuando es necesario (flexibilidad)
- ✅ Indica claramente cuándo se sale de las sentencias (transparencia)
- ⚠️ Requiere que el modelo sea bueno distinguiendo entre "está en las sentencias" y "no está"

**Conclusión**: Para legal, la opción A o C son las adecuadas. La A es más segura, la C es más flexible. La B es peligrosa en contextos donde la precisión es crítica.
</details>

---

### Decision point 5.2: Clasificación con muchas categorías

> **Cuándo presentar**: Al completar la Sesión 5.2, antes de pasar a 5.3

#### Escenario

Necesitas clasificar documentos en 15 categorías diferentes (tipos de contratos: compraventa, arrendamiento, servicio, trabajo, etc.). El modelo debe devolver la categoría y un nivel de confianza.

#### Opciones

**Opción A (single prompt)**:
Un solo prompt con las 15 categorías listadas. El modelo elige una.

```
Clasifica el documento en UNA de estas categorías:
1. Compraventa
2. Arrendamiento
3. Servicio
... (15 categorías)

Responde con JSON: {"category": "...", "confidence": 0.0-1.0}
```

**Opción B (two-step)**:
Primer prompt: clasifica en 3 grupos grandes (contratos comerciales, laborales, otros).
Segundo prompt: dentro del grupo, clasifica en la categoría específica.

**Opción C (few-shot)**:
Prompt con 2-3 ejemplos de cada categoría antes de pedir la clasificación.

#### Pregunta para el usuario

¿Cuál opción eliges para 15 categorías? ¿Por qué? Piensa en:
- ¿El modelo puede manejar 15 categorías en un solo prompt?
- ¿Qué pasa si las categorías son similares (ej. "compraventa" vs "arrendamiento")?
- ¿Cuántos tokens consume cada opción?

<details>
<summary><strong>Análisis de cada opción</strong></summary>

**Opción A (single prompt)**:
- ✅ Simple, una sola llamada al modelo
- ✅ Rápido y barato (pocos tokens)
- ❌ Con 15 categorías, el modelo puede confundirse entre categorías similares
- ❌ Difícil de depurar si falla

**Opción B (two-step)**:
- ✅ Reduce el espacio de búsqueda (primero 3 grupos, luego 5 categorías por grupo)
- ✅ Más preciso para categorías similares
- ❌ Dos llamadas al modelo (más lento, más caro)
- ❌ Más complejo de implementar

**Opción C (few-shot) — RECOMENDADA**:
- ✅ Los ejemplos ayudan al modelo a entender qué espera cada categoría
- ✅ Mejora significativamente la precisión para categorías ambiguas
- ⚠️ Consume más tokens (los ejemplos ocupan espacio)
- ⚠️ Requiere preparar ejemplos representativos

**Conclusión**: Para 15 categorías, la opción C (few-shot) es la más robusta. Los ejemplos clarifican las categorías ambiguas. Si el coste de tokens es un problema, combina B + C: two-step con few-shot en el segundo paso.
</details>

---

## 📋 Instrucciones para el agente

Cuando el usuario llegue al final de un módulo o sesión indicada, presenta el quiz/ejercicio correspondiente:

1. **Lee el archivo `docs/QUIZZES.md`**
2. **Identifica el quiz/ejercicio correspondiente** al módulo o sesión actual
3. **Presenta la pregunta** como una tarjeta interactiva:
   - Para quizzes: muestra la pregunta y las opciones (a, b, c, d)
   - Para break it: muestra la instrucción y pregunta "¿Qué observas?"
   - Para decision points: muestra el escenario y las opciones
4. **Espera la respuesta del usuario**
5. **Compara con la solución** en el `<details>` correspondiente
6. **Da feedback**: explica si la respuesta es correcta, y por qué las otras opciones están mal
7. **Continúa** con el siguiente quiz o avanza al siguiente módulo

**Ejemplo de presentación**:

```
📝 Quiz M0.1: Embeddings

Un embedding convierte texto en un vector de números. Si dos textos tienen
vectores muy cercanos en el espacio vectorial, ¿qué significa?

a) Que los textos son idénticos palabra por palabra
b) Que los textos tienen significado o contexto similar
c) Que los textos tienen la misma longitud
d) Que los textos fueron escritos por el mismo autor

¿Cuál es tu respuesta?
```

---

*Documento creado el 2026-06-06 como complemento interactivo del LEARNING_ROADMAP.md.*
