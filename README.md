# Auto-Grader — Calificador Automático de Cuestionarios con IA

Flujo n8n que califica cuestionarios de forma automática usando un agente de IA, búsqueda semántica (RAG) y registro automático en Google Sheets. Diseñado para docentes que necesitan calificar grandes volúmenes de respuestas sin intervención manual.

---

## ¿Qué problema resuelve?

Calificar cuestionarios manualmente es un proceso repetitivo que puede tomar horas o días dependiendo del número de alumnos. Este flujo automatiza el proceso completo: el docente carga la clave de respuestas una sola vez, y a partir de ahí el sistema califica cada entrega en segundos, con feedback por pregunta y registro automático de resultados.

---

## Arquitectura

```
POST /quiz-reference          →  Pinecone (clave de respuestas vectorizada por quiz_id)
POST /quiz-auto-grader        →  Agente IA → consulta clave → califica → Google Sheets
                                                                         → Respuesta JSON
                              En caso de error → Alerta Slack + HTTP 500
```

**Stack:**
- **n8n** — orquestación del flujo
- **OpenAI GPT-4o mini** — agente calificador (temperatura 0 para resultados consistentes)
- **Cohere embed-multilingual-v3.0** — embeddings multilingües (ideal para contenido en español)
- **Pinecone** — almacenamiento vectorial de claves de respuestas por namespace
- **Google Sheets** — registro de resultados
- **Slack** — alertas de error en tiempo real

---

## Cómo funciona

### 1. Cargar clave de respuestas
```http
POST /quiz-reference
Content-Type: application/json

{
  "quiz_id": "quiz-01",
  "content": "q1: La capital de Francia es París (1 pt)..."
}
```
La clave se vectoriza con Cohere y se almacena en Pinecone bajo el namespace `quiz_id`. Cada cuestionario tiene su propio namespace, por lo que las claves nunca se mezclan.

### 2. Calificar respuestas de un alumno
```http
POST /quiz-auto-grader
Content-Type: application/json

{
  "quiz_id": "quiz-01",
  "student_id": "A123",
  "student_name": "Ana López",
  "answers": [
    { "question_id": "q1", "question": "¿Capital de Francia?", "answer": "París" }
  ]
}
```

El agente:
1. Consulta la clave de respuestas en Pinecone por similitud semántica
2. Califica cada pregunta aceptando sinónimos y errores menores de ortografía
3. Genera feedback breve por pregunta en español
4. Un nodo de código calcula el puntaje final (no el modelo, para mayor confiabilidad)
5. Registra el resultado en Google Sheets
6. Responde con la calificación en JSON

**Respuesta de ejemplo:**
```json
{
  "quiz_id": "quiz-01",
  "student_id": "A123",
  "student_name": "Ana López",
  "score": 1,
  "max_score": 1,
  "percentage": 100.0,
  "needs_review": false,
  "summary": "Excelente desempeño.",
  "questions": [
    {
      "question_id": "q1",
      "points": 1,
      "max_points": 1,
      "correct": true,
      "feedback": "Respuesta correcta."
    }
  ]
}
```

> Si el agente no encuentra la respuesta correcta de una pregunta en la clave, asigna 0 puntos, indica en el feedback que falta en la clave y marca `needs_review: true`.

---

## Configuración

### 1. Índice de Pinecone
Crea un índice llamado `quiz-auto-grader` con:
- **Dimensiones:** 1024
- **Métrica:** cosine

### 2. Hoja de cálculo en Google Sheets
Crea una hoja llamada `Log` con los siguientes encabezados en la fila 1:

| Fecha | Quiz | ID alumno | Alumno | Puntaje | Puntaje máximo | Porcentaje | Revisar | Resumen | Detalle |
|-------|------|-----------|--------|---------|----------------|------------|---------|---------|---------|

Reemplaza `SHEET_ID` en el nodo de Google Sheets con el ID de tu hoja.

### 3. Credenciales en n8n
Configura las siguientes credenciales:
- `OpenAI` — API key de OpenAI
- `Cohere` — API key de Cohere
- `Pinecone` — API key de Pinecone
- `Google Sheets` — OAuth2 de Google
- `Slack` — API token (para alertas de error)

---

## Importar el flujo

1. Copia el contenido de `quiz_auto_grader_workflow.json`
2. En n8n, ve a **Workflows → Import from JSON**
3. Pega el JSON y guarda
4. Asigna las credenciales en cada nodo
5. Activa el flujo

---

## Prueba rápida

```bash
# 1. Cargar clave de respuestas
curl -X POST https://TU-N8N/webhook/quiz-reference \
  -H "Content-Type: application/json" \
  -d '{"quiz_id":"prueba-01","content":"q1: La capital de México es Ciudad de México (1 pt)"}'

# 2. Calificar un alumno
curl -X POST https://TU-N8N/webhook/quiz-auto-grader \
  -H "Content-Type: application/json" \
  -d '{
    "quiz_id": "prueba-01",
    "student_id": "B001",
    "student_name": "Carlos Pérez",
    "answers": [{"question_id":"q1","question":"¿Capital de México?","answer":"CDMX"}]
  }'
```

---

## Decisiones de diseño

- **Sin memoria de conversación:** Cada calificación es independiente. La memoria introduciría riesgo de mezclar contexto entre alumnos.
- **Puntaje calculado en código:** El nodo JavaScript suma los puntos en lugar de confiar en la suma del modelo, lo que garantiza precisión matemática.
- **Namespaces por quiz_id:** Cada cuestionario tiene su propio espacio en Pinecone, evitando colisiones entre distintos exámenes.

---

## Licencia

MIT
