# Model Selection Guide — Modelos en Local

> Guía de referencia para entender cuantización, leer fichas técnicas de modelos,
> y elegir el modelo adecuado para cada situación o proyecto.
>
> **Referencia complementaria**: `docs/LEARNING_ROADMAP.md` (Módulo 0.5)
> **Contexto**: Apple Silicon (M1/M2/M3/M4), Ollama, uso local y productivo.

---

## 1. ¿Qué es cuantización y por qué existe?

### El problema

Un modelo de lenguaje es, en esencia, una red neuronal con millones o miles de millones
de parámetros (pesos numéricos). En su forma original, cada peso se almacena en alta
precisión:

| Precisión | Bits por peso | Tamaño de un modelo 7B |
|---|---|---|
| FP32 (float32) | 32 | ~28 GB |
| FP16 / BF16 | 16 | ~14 GB |
| INT8 | 8 | ~7 GB |
| INT4 (Q4) | 4 | ~3.5 GB |

El problema es claro: un modelo de 7B en FP16 necesita 14 GB solo para los pesos.
Suma la memoria para el contexto (KV cache), el sistema operativo, y otras aplicaciones,
y necesitas una máquina con 32+ GB de RAM para correrlo cómodamente.

### La solución: cuantización

La cuantización reduce la precisión de cada peso. En vez de 16 bits por peso, usas
8, 4, 3, o incluso 2 bits. El modelo ocupa menos, corre más rápido, y la pérdida de
calidad varía según el método y el nivel de cuantización.

**Analogía**: Es como comprimir una imagen. JPEG con calidad 90% se ve casi idéntico
al original pero pesa mucho menos. JPEG con calidad 20% se nota la degradación. La
cuantización es lo mismo pero con pesos de red neuronal.

### Tipos de cuantización

| Tipo | Cuándo se aplica | Ejemplo |
|---|---|---|
| **PTQ (Post-Training Quantization)** | Después de entrenar, sin reentrenar | GGUF en Ollama, GPTQ, AWQ |
| **QAT (Quantization-Aware Training)** | Durante el entrenamiento, el modelo "sabe" que será cuantizado | Modelos específicos de NVIDIA |
| **GGUF (llama.cpp)** | Formato nativo de Ollama/llama.cpp | `llama3.2:3b-instruct-q4_K_M` |

Para uso local con Ollama, **GGUF** es el formato que te importa.

### Niveles de cuantización en GGUF

La nomenclatura GGUF sigue este patrón: `Q<n>_<variant>`

| Nivel | Bits/peso | Calidad | RAM para 7B | Cuándo usarlo |
|---|---|---|---|---|
| **Q2_K** | ~2.6 | Baja — se nota degradación | ~3 GB | Solo si no tienes otra opción |
| **Q3_K_S** | ~3.0 | Aceptable — pérdida perceptible | ~3.5 GB | Pruebas rápidas, hardware limitado |
| **Q4_K_M** | ~4.8 | Buena — sweet spot | ~4.5 GB | **Uso general. Empieza aquí.** |
| **Q5_K_M** | ~5.7 | Muy buena — apenas diferencia | ~5.5 GB | Si tienes RAM de sobra |
| **Q6_K** | ~6.5 | Excelente — casi idéntico a FP16 | ~6 GB | Calidad prioritaria |
| **Q8_0** | ~8.5 | Prácticamente idéntico a FP16 | ~7.5 GB | Si tienes 16+ GB de RAM |

**Regla práctica**:
- **Q4_K_M** es el sweet spot para la mayoría de usos. Buena calidad, buen tamaño.
- **Q5_K_M** si tienes RAM de sobra y quieres un poco más de calidad.
- **Q8_0** si quieres calidad casi idéntica al original.
- **Por debajo de Q4**, la degradación se nota especialmente en tareas de razonamiento y structured output.

### ¿Cuánta calidad se pierde?

Depende de la tarea:

| Tarea | Pérdida con Q4_K_M | Pérdida con Q2_K |
|---|---|---|
| Chat / texto general | Mínima (~1-2%) | Notable (~5-10%) |
| Retrieval / embeddings | Mínima | Moderada |
| Structured output (JSON) | Baja si el modelo es bueno | Alta — puede romper formato |
| Razonamiento complejo | Baja-moderada | Alta |
| Código | Baja | Moderada-alta |

---

## 2. Cómo leer la ficha técnica de un modelo

Cuando ves un modelo en Ollama Library o Hugging Face, estos son los campos que importan:

### Parámetros (3B, 7B, 8B, 70B...)

El número de parámetros del modelo. Es el indicador más directo de capacidad y coste:

| Parámetros | Categoría | Calidad típica | RAM necesaria (Q4_K_M) |
|---|---|---|---|
| 1-3B | Nano | Tareas simples, clasificación básica | 2-3 GB |
| 7-8B | Pequeño | Buen chat, RAG básico, clasificación | 4-5 GB |
| 13-14B | Mediano | Mejor razonamiento, más matices | 8-9 GB |
| 30-34B | Grande | Calidad接近 cloud, lento en local | 18-20 GB |
| 70B+ | Extra grande | Calidad máxima, requiere hardware serio | 40+ GB |

### Context window (2K, 4K, 8K, 32K, 128K)

Cuántos tokens puede procesar el modelo de una vez (prompt + respuesta incluidos).

Para RAG, esto es crítico porque necesitas que quepan:
- System prompt (~200-500 tokens)
- Contexto recuperado (5-10 chunks × ~200 tokens = 1000-2000 tokens)
- Pregunta del usuario (~50-100 tokens)
- Respuesta esperada (~200-500 tokens)

**Mínimo práctico para RAG**: 4K tokens. 8K es cómodo. 32K+ es un lujo.

### Dimensiones de embedding (solo modelos de embedding)

| Modelo | Dimensiones | Tamaño vector | Cuándo usarlo |
|---|---|---|---|
| nomic-embed-text | 768 | 3 KB | Buen balance, rápido |
| mxbai-embed-large | 1024 | 4 KB | Mayor precisión, más lento |
| bge-m3 | 1024 | 4 KB | Multilingüe, muy bueno en español |
| all-minilm | 384 | 1.5 KB | Muy rápido, menos preciso |

Más dimensiones = vectores más grandes = más espacio en el vector store = más lento
el search. Pero también puede significar mejor representación semántica.

### Formato

| Formato | Compatible con Ollama | Uso |
|---|---|---|
| **GGUF** | Sí — nativo | El que necesitas para Ollama |
| **safetensors** | No directamente | Formato estándar de Hugging Face |
| **GGML** | Legacy | Predecesor de GGUF, evitar |

### Licencia

Importante si lo usas en producto comercial:

| Licencia | Comercial | Notas |
|---|---|---|
| Apache 2.0 | Sí, sin restricciones | Mistral, algunos Qwen |
| MIT | Sí, sin restricciones | Phi-3 |
| Llama Community | Sí, con límites | >700M usuarios/mes necesitan acuerdo con Meta |
| Qwen Research | Limitado | Revisar términos específicos |
| CC-BY-NC | No comercial | Algunos modelos académicos |

### Idioma

No todos los modelos son iguales en todos los idiomas:

| Modelo | Inglés | Español | Multilingüe |
|---|---|---|---|
| Llama 3.2 | Excelente | Bueno | Sí |
| Qwen 2.5 | Excelente | Muy bueno | Sí (fuerte en CJK) |
| Mistral | Excelente | Aceptable | Limitado |
| Phi-3 | Excelente | Bueno | Limitado |
| BGE-M3 (embeddings) | Excelente | Muy bueno | Excelente |

---

## 3. Cómo elegir el modelo para tu situación

### Paso 1: ¿Qué hardware tienes?

En Apple Silicon, la RAM unificada es tu límite principal:

| RAM unificada | Modelos viables (Q4_K_M) | Embeddings | Uso práctico |
|---|---|---|---|
| 8 GB | Hasta 3B | nomic-embed-text | Tareas simples, pruebas |
| 16 GB | Hasta 7-8B | nomic-embed-text + modelo 7B | RAG básico, chat, clasificación |
| 24-32 GB | Hasta 13-14B | nomic-embed-text + modelo 13B | Mejor calidad, contexto largo |
| 64+ GB | Hasta 30-34B | Cualquier embedding | Calidad接近 cloud |

**Regla general**: El modelo + embeddings + sistema operativo no deben superar el 70-80% de tu RAM total.

### Paso 2: ¿Qué tarea necesitas?

#### Embeddings (para RAG / búsqueda semántica)

| Modelo | Dimensiones | Velocidad | Calidad | Recomendado para |
|---|---|---|---|---|
| `nomic-embed-text:v1.5` | 768 | Rápido | Buena | **Uso general. Empieza aquí.** |
| `mxbai-embed-large` | 1024 | Medio | Muy buena | Si necesitas más precisión |
| `bge-m3` | 1024 | Medio | Excelente | Español / multilingüe |
| `all-minilm` | 384 | Muy rápido | Aceptable | Prototipos rápidos |

#### Generación / Chat / RAG

| Modelo | Parámetros | Calidad | Velocidad (M2 16GB) | Recomendado para |
|---|---|---|---|---|
| `llama3.2:3b-instruct-q4_K_M` | 3B | Buena | Rápido | Clasificación, tareas simples |
| `llama3.2:8b-instruct-q4_K_M` | 8B | Muy buena | Medio | **RAG general. Sweet spot.** |
| `qwen2.5:7b-instruct-q4_K_M` | 7B | Muy buena | Medio | Mejor en español que Llama |
| `qwen2.5:14b-instruct-q4_K_M` | 14B | Excelente | Lento | Si tienes 32+ GB RAM |
| `mistral:7b-instruct-q4_K_M` | 7B | Buena | Medio | Alternativa a Llama |
| `phi3:mini-q4_K_M` | 3.8B | Buena | Rápido | Hardware limitado |

#### Structured Output (JSON / clasificación)

Para structured output, la capacidad de seguir instrucciones es más importante que
la "creatividad". Modelos pequeños instruct suelen funcionar bien:

| Modelo | Parámetros | Consistencia JSON | Recomendado para |
|---|---|---|---|
| `llama3.2:3b-instruct-q4_K_M` | 3B | Alta | Clasificación simple |
| `qwen2.5:7b-instruct-q4_K_M` | 7B | Muy alta | **Structured output complejo** |
| `llama3.2:8b-instruct-q4_K_M` | 8B | Alta | JSON mode general |

**Nota**: La cuantización importa más aquí. Un modelo Q2 puede romper el formato JSON.
Usa Q4_K_M como mínimo para structured output.

#### Código

| Modelo | Parámetros | Calidad | Recomendado para |
|---|---|---|---|
| `qwen2.5-coder:7b-instruct-q4_K_M` | 7B | Muy buena | **Mejor opción general** |
| `deepseek-coder:6.7b-instruct-q4_K_M` | 6.7B | Buena | Alternativa |
| `starcoder2:7b-q4_K_M` | 7B | Buena | Código open source |

### Paso 3: ¿Qué restricciones tiene tu proyecto?

| Restricción | Implicación |
|---|---|
| **Privacidad total** | Solo Ollama local. Sin APIs externas. Modelo en Q4-Q5 para que quepa. |
| **Latencia baja (<2s)** | Modelo pequeño (3B) + Q4. Sacrificas calidad por velocidad. |
| **Latencia aceptable (<10s)** | Modelo mediano (7-8B) + Q4_K_M. Buen balance. |
| **Calidad máxima** | Modelo grande (14B+) + Q5_K_M o Q6_K. Necesitas 32+ GB RAM. |
| **Coste cero** | Todo local con Ollama. Hardware existente. |
| **Producto comercial** | Revisa licencias. Apache 2.0 y MIT son las más permisivas. |

### Matriz de decisión rápida

```
¿Tienes 16 GB RAM o menos?
├── Sí → llama3.2:3b-instruct-q4_K_M + nomic-embed-text
│        (clasificación, chat básico)
└── No → ¿Tu texto es principalmente en español?
    ├── Sí → qwen2.5:7b-instruct-q4_K_M + bge-m3
    └── No → llama3.2:8b-instruct-q4_K_M + nomic-embed-text
```

---

## 4. Ecosistema práctico: dónde encontrar y comparar modelos

### Ollama Library

URL: https://ollama.com/library

Modelos listos para usar con `ollama pull <nombre>`. Cada modelo tiene tags que
indican tamaño y cuantización:

```bash
# Ver modelos disponibles
ollama list

# Descargar un modelo específico
ollama pull llama3.2:8b-instruct-q4_K_M

# Ver detalles de un modelo (tamaño, parámetros, etc.)
ollama show llama3.2:8b-instruct-q4_K_M
```

**Ventaja**: Cero configuración. Descargas y usas.
**Limitación**: No todos los modelos están en Ollama. Solo GGUF.

### Hugging Face

URL: https://huggingface.co/models

El repositorio universal de modelos. Aquí encuentras todo:

- Filtra por tarea: "text-generation", "sentence-similarity", "fill-mask"
- Filtra por formato: "GGUF" para modelos compatibles con Ollama
- Filtra por idioma: "es", "multilingual"
- Filtra por licencia: "apache-2.0", "mit", etc.

**Cuantizadores populares** (busca sus nombres en Hugging Face):
- **bartowski**: Cuantizaciones GGUF de alta calidad
- **MaziyarPanahi**: Variantes GGUF de modelos populares
- **TheBloke**: Uno de los primeros en cuantizar modelos (algunos legacy)

### Leaderboards y benchmarks

| Recurso | Qué mide | URL |
|---|---|---|
| **MTEB** | Modelos de embedding (retrieval, STS, clasificación) | https://huggingface.co/spaces/mteb/leaderboard |
| **LMSYS Arena** | Modelos de generación (ELO por preferencia humana) | https://chat.lmsys.org/?arena |
| **Open LLM Leaderboard** | Modelos open source en benchmarks estándar | https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard |

**Nota sobre benchmarks**: Son útiles para comparar modelos en igualdad de condiciones,
pero no sustituyen la prueba en tu dominio. Un modelo que puntúa alto en MMLU puede
funcionar mal en tu caso de uso específico. Siempre prueba con tus datos reales.

### Comparativa rápida de modelos populares (2025-2026)

| Modelo | Parámetros | Mejor en | Peor en | Licencia |
|---|---|---|---|---|
| **Llama 3.2** (Meta) | 1B, 3B, 8B, 90B | Inglés, razonamiento general | Español (mejorable) | Llama Community |
| **Qwen 2.5** (Alibaba) | 0.5B-72B | Multilingüe, código, español | — | Apache 2.0 / Qwen |
| **Mistral** (Mistral AI) | 7B, 8x7B, 8x22B | Inglés, eficiencia | Multilingüe | Apache 2.0 |
| **Phi-3** (Microsoft) | 3.8B, 14B | Hardware limitado, eficiencia | Contexto largo | MIT |
| **Gemma 2** (Google) | 2B, 9B, 27B | Inglés, velocidad | Español | Gemma Terms |

---

## 5. Errores comunes al elegir modelo

| Error | Por qué es un error | Qué hacer en su lugar |
|---|---|---|
| Elegir el modelo más grande posible | Se queda sin RAM, es lento, frustrante | Empieza con el más pequeño que funcione para tu caso |
| Usar Q2 "para que quepa" | La calidad se degrada mucho, especialmente en JSON | Mejor modelo pequeño en Q4 que modelo grande en Q2 |
| Ignorar el idioma | Un modelo optimizado para inglés puede ser mediocre en español | Prueba en tu idioma antes de decidir |
| No revisar la licencia | Puedes tener problemas si lo usas en producto comercial | Revisa la licencia antes de integrar |
| Cambiar de modelo sin evaluar | No sabes si mejora o empeora | Usa `rag eval` para medir el impacto de cada cambio |
| Usar un modelo de chat para embeddings | Son cosas distintas, modelos distintos | Usa modelos especializados para cada tarea |

---

## 6. Comandos útiles de Ollama para gestión de modelos

```bash
# Listar modelos instalados
ollama list

# Descargar modelo
ollama pull <modelo>:<tag>

# Eliminar modelo
ollama rm <modelo>:<tag>

# Ver información del modelo (tamaño, parámetros, prompt template)
ollama show <modelo>:<tag>

# Ver modelos en ejecución
ollama ps

# Ejecutar modelo interactivamente
ollama run <modelo>:<tag>

# Usar la API directamente
curl http://localhost:11434/api/tags  # listar modelos
curl http://localhost:11434/api/show -d '{"name": "llama3.2:8b-instruct-q4_K_M"}'
```

---

## 7. Resumen: checklist para elegir modelo

Antes de elegir un modelo para tu proyecto, responde:

- [ ] ¿Cuánta RAM tengo disponible? → Determina el tamaño máximo del modelo
- [ ] ¿En qué idioma trabajará principalmente? → Filtra modelos por idioma
- [ ] ¿Qué tarea principal va a hacer? → Embedding, chat, clasificación, código
- [ ] ¿Necesito structured output (JSON)? → Modelo instruct + Q4_K_M mínimo
- [ ] ¿Qué latencia necesito? → Modelo pequeño = rápido, modelo grande = lento
- [ ] ¿Es para uso comercial? → Revisa la licencia
- [ ] ¿Tengo forma de evaluarlo con mis datos? → `rag eval` o test manual

Con estas respuestas, la elección se reduce a 2-3 candidatos. Prueba cada uno con
tus datos reales y quédate con el que mejor funcione.

---

*Documento creado el 2026-06-06 como guía de referencia para selección de modelos locales.
Complementa el Módulo 0.5 del `docs/LEARNING_ROADMAP.md`.*
