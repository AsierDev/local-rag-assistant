# Paths después del Learning Roadmap

> Documento de referencia para decidir qué hacer después de completar
> las 19 sesiones del `LEARNING_ROADMAP.md`.
>
> **Contexto**: FE developer (React/TS) → quiere ser más fullstack con foco en AI.
> Trabaja en un producto existente (React/TS FE + SQL/PHP BE) que ya integra AI.
> Quiere entender el flujo completo, no solo su pieza.

---

## Situación actual

| Dimensión | Estado |
|---|---|
| **Stack actual** | React + TypeScript (FE), SQL + PHP (BE) |
| **Producto** | Existe, ya tiene integración de AI en curso |
| **Tu rol** | FE, pero quieres entender y poder hacer el flujo completo |
| **Objetivo** | Salir del rol FE puro → ser capaz de diseñar e implementar flujos AI end-to-end |
| **Infra** | No necesitas profundizar en AWS/CDK, pero sí base de conocimiento |
| **Modelos** | Abierto a local (Ollama) y externos (OpenAI, Claude) |
| **Timeline** | Aprendizaje personal, sin deadline de producto |

---

## Lo que tendrás después del Roadmap

Al completar las 19 sesiones (~3-4 semanas, 45-60 min/día) tendrás:

1. **Un RAG funcional en Python** con CLI: ingest, ask, search, eval, classify
2. **Comprensión profunda** de embeddings, vector stores, chunking, retrieval, generación
3. **Adapter pattern** implementado: sabes cómo desacoplar modelos y stores
4. **Structured output**: sabes hacer que un LLM devuelva JSON válido y consistente
5. **Evaluación**: sabes medir si un sistema RAG funciona bien o no
6. **Mentalidad de diagnóstico**: inspect → search → ask → eval, no adivinar

Esto es **independiente del lenguaje**. Los conceptos transferen 1:1 a TypeScript.

---

## Paths posibles después del Roadmap

### Path A: Integrar en tu producto actual (el más directo)

**Objetivo**: Llevar lo aprendido al producto de tu empresa.

**Qué harías**:
- Implementar un endpoint en PHP (o un microservicio Node) que envuelva el pipeline RAG
- O crear un servicio separado en Python/FastAPI que tu frontend React llame
- Integrar structured output para el caso de clasificación automática de reportes
- Añadir evaluación continua para medir la calidad del sistema en producción

**Stack sugerido**:
- **Opción 1 (más simple)**: FastAPI en Python + tu frontend React llama a la API
- **Opción 2 (más alineado con tu stack)**: Reimplementar en TypeScript con `@langchain/core` + pgvector (si ya tienes Postgres)
- **Opción 3 (híbrida)**: Python para el pipeline de AI, PHP para la integración con el BE existente

**Tiempo estimado**: 2-4 semanas después del roadmap

**Cuándo elegir este path**:
- ✅ Quieres impacto inmediato en tu trabajo
- ✅ Tu empresa ya está integrando AI y puedes aportar
- ✅ Prefieres aprender haciendo en un contexto real

---

### Path B: Profundizar en AI Engineering (el más completo)

**Objetivo**: Dominar el ecosistema AI más allá de RAG básico.

**Qué harías**:
- **LangGraph / LangChain agents**: Flujos multi-paso con estado y memoria
- **Fine-tuning / LoRA**: Entrenar modelos pequeños para tareas específicas
- **Evaluación avanzada**: Ragas, DeepEval, A/B testing de prompts
- **Multi-modal**: Imágenes, audio, documentos escaneados
- **Agentes autónomos**: Tool use, planning, self-correction

**Stack sugerido**:
- Python (el ecosistema es mucho más rico aquí)
- LangGraph, LlamaIndex, Haystack
- Ollama + modelos open source + APIs externas

**Tiempo estimado**: 4-8 semanas después del roadmap

**Cuándo elegir este path**:
- ✅ Te engancha AI y quieres ir más allá de RAG básico
- ✅ Quieres entender fine-tuning, agentes, evaluación avanzada
- ✅ No tienes prisa por integrar en producto

---

### Path C: Fullstack con AI (el más equilibrado)

**Objetivo**: Ser capaz de diseñar e implementar un producto AI completo, de la DB al frontend.

**Qué harías**:
- **Backend**: FastAPI o Node/Express con endpoints de AI
- **Base de datos**: Postgres + pgvector para producción
- **Frontend**: React con Vercel AI SDK para streaming y chat UI
- **Infra básica**: Docker, deploy en un VPS o plataforma simple (Railway, Render)
- **Monitoreo**: Logs, métricas de calidad, alertas

**Stack sugerido**:
- TypeScript (tu zona de confort) + Python (para lo que no tenga equivalente TS)
- Postgres + pgvector
- Vercel AI SDK para el frontend
- Docker + Railway/Render para deploy

**Tiempo estimado**: 4-6 semanas después del roadmap

**Cuándo elegir este path**:
- ✅ Quieres ser fullstack con AI, no solo FE o solo BE
- ✅ Te interesa construir productos completos
- ✅ Prefieres TypeScript pero aceptas Python donde tenga sentido

---

### Path D: Simplificar y enfocarte en tu caso concreto

**Objetivo**: Resolver los dos casos de uso de tu producto y nada más.

**Qué harías**:
1. **Chatbot con citas**: Implementar RAG para tu base de datos legal/patentes
2. **Clasificación automática**: Pipeline de structured output para reportes
3. **Evaluación**: Test set con casos reales de tu dominio
4. **Integración**: Conectar con tu frontend React

**Stack sugerido**:
- El que tenga menos fricción para ti (probablemente TypeScript)
- Ollama local para desarrollo, API externa para producción si hace falta

**Tiempo estimado**: 2-3 semanas después del roadmap

**Cuándo elegir este path**:
- ✅ Quieres resultados concretos rápido
- ✅ No te interesa el ecosistema AI más amplio
- ✅ Tu foco es tu producto, no aprender por aprender

---

## Comparación de paths

| | A: Integrar en producto | B: AI Engineering | C: Fullstack AI | D: Caso concreto |
|---|---|---|---|---|
| **Tiempo** | 2-4 sem | 4-8 sem | 4-6 sem | 2-3 sem |
| **Complejidad** | Media | Alta | Media-Alta | Baja |
| **Impacto laboral** | Alto | Medio | Alto | Alto |
| **Aprendizaje** | Práctico | Profundo | Equilibrado | Enfocado |
| **Stack** | Flexible | Python | TS + Python | TS |
| **Ideal si...** | Quieres impacto ya | Te engancha AI | Quieres ser fullstack | Quieres resultados rápidos |

---

## Mi recomendación para tu caso

Dado que:
- Tu objetivo es **salir del rol FE** y entender flujos completos
- Tu empresa **ya está integrando AI**
- No necesitas infra compleja
- Quieres **principios, no tecnología específica**

**Recomiendo Path A + Path D combinados**:

1. **Primero Path D** (2-3 sem): Resuelve tus dos casos concretos (chatbot + clasificación) con el stack que tenga menos fricción. Esto te da confianza y resultados tangibles.
2. **Luego Path A** (2-4 sem): Integra lo aprendido en tu producto real. Aquí es donde realmente te conviertes en "fullstack con AI".

El Path B (AI Engineering profundo) y Path C (fullstack completo) los puedes explorar después, cuando ya tengas experiencia práctica.

---

## Decisiones que no necesitas tomar ahora

| Decisión | Por qué esperar |
|---|---|
| ¿Python o TypeScript para producción? | Los conceptos transferen. Decide cuando tengas algo funcionando. |
| ¿Ollama local o API externa? | Depende de privacidad, coste, y latencia de tu producto. |
| ¿Chroma o pgvector? | Chroma para aprender, pgvector para producción. No necesitas pgvector hasta que escales. |
| ¿AWS CDK o infra simple? | No lo necesitas para un MVP de AI. Empieza con algo simple. |
| ¿LangChain o código propio? | El adapter pattern que aprendes en el roadmap te da ambas opciones. |

---

## Checklist de preparación para después del Roadmap

Cuando termines las 19 sesiones, antes de elegir un path, responde:

- [ ] ¿Qué me ha gustado más? (embeddings, retrieval, generación, clasificación, evaluación...)
- [ ] ¿Qué me ha costado más? (Python, Chroma, Ollama, chunking, prompts...)
- [ ] ¿Tengo un caso concreto de mi producto que pueda atacar?
- [ ] ¿Mi equipo necesita ayuda con algo de AI que yo pueda resolver?
- [ ] ¿Prefiero seguir aprendiendo o ya quiero construir algo real?

Con estas respuestas, el path se elige solo.

---

*Documento creado el 2026-05-17 como guía de paths posteriores al Learning Roadmap.
El roadmap canónico está en `docs/LEARNING_ROADMAP.md`.
El plan de implementación está en `docs/plans/local-rag-assistant/IMPLEMENTATION_PLAN.md`.*
