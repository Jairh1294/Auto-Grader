# Auto-Grader
Flujo para calificar examenes

Listo. Para importarlo, copia el JSON y pégalo en el lienzo de n8n.

Cómo funciona ahora. El flujo tiene dos entradas:

POST /quiz-reference carga la clave de respuestas de cada cuestionario en Pinecone. Cada cuestionario va en su propio namespace según su quiz_id, así que las claves no se mezclan.
POST /quiz-auto-grader recibe las respuestas del alumno. El agente consulta la clave y califica pregunta por pregunta. Luego un nodo de código suma el puntaje, porque es más confiable que dejar la suma al modelo. Al final registra el resultado en Sheets y responde con la calificación en JSON. Si algo falla, avisa en Slack y responde con error 500.

Quité la memoria de conversación, porque cada calificación es independiente y solo introducía riesgo de mezclar alumnos.

Antes de activarlo:

Índice de Pinecone. Crea uno llamado quiz-auto-grader con 1024 dimensiones y métrica cosine. Es lo que usa embed-multilingual-v3.0, que elegí porque tus contenidos probablemente estén en español.
Hoja de cálculo. En la hoja "Log" pon estos encabezados: Fecha, Quiz, ID alumno, Alumno, Puntaje, Puntaje máximo, Porcentaje, Revisar, Resumen, Detalle. Después reemplaza SHEET_ID por el ID de tu hoja.
Credenciales. Asigna las de OpenAI, Cohere, Pinecone, Google Sheets y Slack.
Datos de prueba. Carga primero una clave con el webhook de referencia y después prueba la calificación.
