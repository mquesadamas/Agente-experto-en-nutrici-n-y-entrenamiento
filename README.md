# Asistente Experto en Hipertrofia y Ciencias del Deporte (RAG + LangGraph)

Este proyecto implementa un asistente conversacional especializado en ganancia de masa muscular, estructurado sobre tres pilares: **entrenamiento, nutrición deportiva y descanso**. 

En lugar de depender del conocimiento general de un modelo de lenguaje —propenso a mezclar mitos del fitness con evidencia científica—, el sistema utiliza una arquitectura **RAG (Retrieval-Augmented Generation)** conectada a una base de documentación técnica propia. El flujo está orquestado con **LangGraph** para mantener el estado de la conversación y reformular preguntas ambiguas antes de consultar la base vectorial.

---

## Arquitectura y funcionamiento

El pipeline del notebook se divide en cuatro bloques principales:

1. **Ingesta y vectorización:** Se cargan los documentos PDF del dominio (`Nutrición.pdf`, `Hipertrofia.pdf`, `Descanso Nocturno.pdf`, `Descanso entre Series.pdf`, `RIR.pdf`), se fragmentan en *chunks* de 1000 caracteres con un solapamiento (*overlap*) de 200 y se indexan en una base vectorial local **ChromaDB** utilizando el modelo `gemini-embedding-2`.
2. **Reformulación contextual (Query Rewriting):** Antes de buscar en ChromaDB, una cadena intermedia analiza el historial del chat. Si el usuario hace una pregunta de seguimiento (por ejemplo, *"¿qué pasa si no descanso?"*), el sistema la reescribe desambiguando términos técnicos (distinguiendo entre *privación de sueño* y *descanso entre series*) para mejorar la precisión del *retrieval*.
3. **Generación controlada (LCEL):** Se recuperan los `k=5` fragmentos más relevantes y se inyectan en el prompt junto con el historial para que **Gemini (`gemini-3.5-flash-lite`)** redacte la respuesta.
4. **Memoria con LangGraph:** El grafo utiliza `StateGraph` junto con `MemorySaver` y un `thread_id` por sesión para persistir el historial de mensajes entre turnos de forma nativa.

---

## Decisiones de Ingeniería de Prompts

El diseño del `system_prompt` responde a la necesidad de evitar alucinaciones en un ámbito donde los datos fisiológicos y nutricionales deben ser exactos[cite: 4]:

* **Patrón Persona/Rol:** Se define al modelo como un consultor experto en ciencias del deporte e hipertrofia para que adopte un registro técnico, claro y estructurado[cite: 1].
* **Uso de delimitadores XML (`<contexto>`):** Separa de forma inequívoca las instrucciones del sistema de los fragmentos recuperados de los PDFs, evitando que el modelo confunda datos externos con directrices[cite: 1, 9].
* **Verificación de condiciones y filtros semánticos:** Se impone la regla estricta de responder *únicamente* con la información presente en el contexto[cite: 1, 9]. Si la consulta solo se cubre parcialmente, debe indicar qué falta; si el tema es ajeno al dominio (como preguntas de cultura general), está obligado a responder textualmente que no dispone de información suficiente en sus documentos[cite: 1].
* **Desambiguación previa en el *retriever*:** En entrenamiento de fuerza, palabras como "descanso" tienen significados distintos según el contexto. Incluir una instrucción específica de desambiguación en el prompt de reformulación evita que el buscador vectorial cruce documentos de sueño nocturno con tiempos de recuperación entre series[cite: 1].
