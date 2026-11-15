# Asistente de WhatsApp del Complejo Del Trébol

## El caso

En el Complejo de Cabañas Del Trébol (Bariloche) los clientes consultan con frecuencia antes de reservar sobre servicios básicos: ropa blanca, cochera, parrilla, capacidad, mascotas, horarios de check-in y check-out, wifi, pileta, etc. Hoy esas consultas se responden de forma manual o con plantillas prearmadas. El volumen es alto y se repite todos los días, especialmente en temporada alta. Este proyecto automatiza ese proceso con un AI Agent conectado a WhatsApp, usando una base de conocimiento controlada mediante RAG (Retrieval-Augmented Generation) sobre un vectorstore en Supabase, y dejando registro de cada interacción.

## El problema y el ROI

- **Proceso manual:** responder preguntas frecuentes de clientes antes de reservar (ropa blanca, cochera, parrilla, capacidad, mascotas, check-in/out, wifi, pileta, etc.).
- **Volumen:** 20 a 25 mensajes por día en temporada baja y más del doble en temporada alta. Cada respuesta lleva entre 4 y 5 minutos (leer el mensaje, ubicar el hilo, escribir la respuesta y enviarla). El tiempo total sube porque muchas consultas llegan mientras se están haciendo otras tareas: hay que interrumpir, volver al celular o a la PC, retomar el hilo y responder.
- **Tiempo actual:** entre 2,5 y 3 horas diarias en temporada baja (≈ 15–18 horas semanales). En temporada alta supera las 30 horas semanales.
- **Costo hoy:** tomando un costo hora administrativo de $8.000 ARS, el proceso consume entre $120.000 y $144.000 ARS por semana en temporada baja, y más de $240.000 ARS en temporada alta.
- **Costo de automatizar:**
  - n8n Cloud Starter (≈ USD 20–24/mes): ~$31.000 – $38.000 ARS/mes.
  - Groq (modelo openai/gpt-oss-20b): a este volumen el costo es muy bajo (menos de $5.000 ARS/mes).
  - Google Gemini Embeddings + Supabase (vectorstore): el costo de generación de embeddings es muy bajo a este volumen y Supabase dispone de un plan gratuito suficiente para el tamaño de la base.
  - WhatsApp Business API: las primeras 1.000 respuestas de servicio por mes son gratuitas; a partir de ahí ~ USD 0,026 por mensaje (~$40 ARS). Con 20–50 respuestas diarias el costo mensual se mantiene bajo.
  - **Total estimado:** entre $40.000 y $55.000 ARS/mes.
- **ROI:** el ahorro de tiempo es inmediato. En temporada baja se recuperan entre 15 y 18 horas semanales. El costo mensual de la automatización representa menos del 35 % del costo del tiempo manual. A partir del segundo mes el retorno supera el 200 %. En temporada alta el ROI es aún mayor.
- **Límite del ROI:** si el volumen cayera de forma sostenida por debajo de 8–10 consultas diarias, el costo operativo superaría el tiempo ahorrado y dejaría de ser conveniente.

## Arquitectura RAG

El sistema utiliza Retrieval-Augmented Generation (RAG) para garantizar que el agente solo responda con información autorizada.

**Workflow de carga de conocimiento** (archivo JSON exportado: `Cargar embeddings Supabase`):
- Se dispara de forma manual (Manual Trigger).
- Un nodo Google Sheets lee la base de respuestas.
- Un nodo HTTP Request genera los embeddings usando la API de Google Generative Language (modelo de embeddings de Gemini).
- Un nodo Supabase Vector Store inserta los documentos junto con sus embeddings en la tabla `documentos` (columnas: id, content, metadata, embedding).

**Recuperación en tiempo real:**
- En el workflow principal, el AI Agent tiene como herramienta el nodo “Buscar en Supabase” configurado como Vector Store en modo Retrieve Documents.
- Antes de responder, el agente consulta automáticamente el vectorstore. Solo utiliza la información recuperada.
- El system prompt obliga a usar exclusivamente esa herramienta y a derivar con la frase fija si no encuentra el dato.

**Justificación técnica:**
- **Supabase**: vectorstore gratuito, fácil de integrar con n8n, control total sobre los datos y buena performance para bases pequeñas.
- **Gemini Embeddings**: buena calidad, bajo costo y excelente integración con el nodo de n8n.
- **Privacidad**: los datos de la base de respuestas no se envían a terceros en cada consulta; solo se generan embeddings una vez al cargar la base.
## Cómo funciona

1. **Entrada 1:** WhatsApp Trigger recibe el mensaje del cliente (texto, teléfono y nombre).

2. **Entrada 2:** Webhook de prueba. Se agregó un segundo disparador (Webhook) para cumplir con el requisito de al menos dos entradas al workflow. Este Webhook se utiliza únicamente para pruebas controladas desde Postman. El nodo “¿Viene de WhatsApp?” se encarga de que el envío por WhatsApp solo se ejecute cuando el mensaje proviene del trigger de WhatsApp. Cuando la entrada es por Webhook, el flujo registra la consulta en Sheets y finaliza en el nodo “Fin de prueba (Webhook)”, permitiendo visualizar la respuesta generada por el AI Agent directamente en la ejecución de n8n.

3. **Proceso:**
   - Un nodo Edit Fields (“Datos de la consulta”) normaliza el mensaje entrante: elimina emojis innecesarios, pasa a minúsculas si corresponde y extrae únicamente el nombre del cliente (sin apellido). También calcula el largo del mensaje para poder validarlo después.
   - Un nodo IF (“¿Mensaje válido?”) verifica que el mensaje limpio tenga al menos 3 caracteres. Si no cumple, el flujo se deriva a la rama de error (nodo “Error - Inválido”, respuesta fija y registro con Derivado = Sí). Si cumple, continúa hacia el nodo “OK - listo para IA”.
   - Desde “OK - listo para IA” el flujo se divide: una rama va directo al nodo Merge (“Unir datos”) y otra pasa por el Switch (“Tipo de consulta”), que decide si corresponde consultar la distancia al aeropuerto o al centro. Según el tipo de consulta (detectado por el nodo Switch), el flujo consulta GraphHopper mediante un HTTP Request para obtener el tiempo de viaje al aeropuerto o al centro de Bariloche. Elegí esta segunda fuente porque los clientes suelen preguntar por la ubicación relativa del complejo (“¿qué tan lejos está del aeropuerto?” o “¿a cuánto están del centro?”) antes de reservar. Incorporar ese dato como origen independiente permite que el agente lo tenga disponible en el contexto cuando es relevante, sin que dependa de una búsqueda manual en cada consulta. GraphHopper devuelve el tiempo de viaje en milisegundos dentro de un objeto anidado `paths[0].time`. Los nodos Set “Minutos aeropuerto” y “Minutos centro” convierten ese valor a minutos enteros con la expresión `{{ Math.round($json.paths[0].time / 60000) }}`, dejando un dato limpio y legible para el agente. Esta consulta solo se activa si el mensaje del cliente menciona explícitamente el aeropuerto o el centro; en el resto de las consultas, el Switch no dispara el HTTP Request y el flujo continúa sin ese dato.
   - Los datos del mensaje y el resultado de GraphHopper (si aplica) se unen en el nodo Merge (“Unir datos”).
   - Un nodo “Un solo item” limita la salida a un único ítem para evitar que el AI Agent reciba múltiples elementos y genere respuestas duplicadas o inconsistentes.
   - El AI Agent (Groq) recibe el system prompt, el mensaje del cliente y, si corresponde, el dato de distancia. Su única fuente de información es la herramienta “Buscar en Supabase” (nodo Vector Store en modo Retrieve Documents). El system prompt obliga explícitamente al agente a consultar esta herramienta antes de responder y a utilizar exclusivamente el contenido recuperado del vectorstore. El agente consulta automáticamente el vectorstore de Supabase, recupera los documentos relevantes mediante embeddings de Gemini y genera la respuesta aplicando estrictamente las reglas del system prompt. Si no encuentra el dato, debe derivar con la frase fija. De esta forma el agente trabaja con información dinámica recuperada mediante RAG. Cuando el cliente pregunta por la distancia, el agente incorpora el tiempo de viaje obtenido de GraphHopper dentro de la respuesta, sin inventar información.
   - Un nodo Simple Memory, con el número de teléfono como Session ID y ventana de 5 interacciones, permite que el agente recuerde el contexto reciente de la conversación (evita repetir el saludo y mantiene coherencia).
   - En ambos caminos (válido o error) se registra la consulta en Google Sheets con fecha, nombre, teléfono, canal, pregunta, respuesta y el campo Derivado (Sí/No).
   - Un nodo IF (“¿Viene de WhatsApp?”) verifica el origen de la entrada. Si proviene del trigger de WhatsApp, continúa hacia el nodo de envío por WhatsApp. Si proviene del Webhook de prueba, finaliza en el nodo “Fin de prueba (Webhook)”.

   **Sobre la API de GraphHopper**  
   En los nodos HTTP Request de distancia (aeropuerto y centro) el parámetro `key=` aparece vacío en el JSON exportado del workflow. Esto es así porque eliminé la key del archivo antes de subirlo, para cumplir con el requisito de no dejar claves, tokens ni secretos visibles en el JSON. En el flujo real la key está configurada en el nodo y funciona correctamente. De esta forma se integra la fuente externa (HTTP Request) sin dejar la clave en el archivo entregado.

4. **Salida:** se envía la respuesta por WhatsApp al mismo número del cliente (solo si la entrada vino por WhatsApp) y queda registrada en la hoja de Registro.

## Por qué un AI Agent y no un nodo de IA simple

Elegí un AI Agent porque el flujo requiere una decisión real basada en contexto recuperado dinámicamente. El agente tiene acceso a la herramienta “Buscar en Supabase”, que consulta el vectorstore en tiempo real mediante RAG. Aplica las reglas del system prompt (rol, alcance, tono, límites y qué hacer ante lo ambiguo) y elige si responde con un dato concreto recuperado de la base o deriva con la frase fija exacta. Un nodo de IA simple no tiene acceso a herramientas ni un marco de decisión: solo generaría texto libre sin poder consultar la fuente de verdad ni garantizar que respete las reglas de no inventar información. En este caso el agente funciona como un recepcionista con límites claros, trazabilidad y base de conocimiento controlada mediante embeddings.

## Local o nube

Elegí **Groq en la nube** por dos razones concretas.

Primero, las consultas que recibe el agente no contienen datos sensibles: son preguntas sobre servicios públicos del complejo (ropa blanca, cochera, capacidad, etc.), no datos de pago ni información personal de los clientes. Segundo, probé Ollama local con llama3.1:8b. Las respuestas resultaron inconsistentes: en varias pruebas omitió datos que sí estaban en la base, no respetó el formato de WhatsApp de forma estable y generó variaciones no controladas en el saludo y el cierre. Con 8 GB de RAM, mantener un modelo local agrega complejidad operativa (dependencia de que la máquina esté encendida, consumo de recursos y menor predictibilidad) sin mejorar la calidad en este volumen y tipo de consulta. Groq ofrece respuestas predecibles, baja latencia y un costo por request muy bajo a este volumen, lo que lo hace más adecuado para un uso real por WhatsApp.

## Qué pasa si algo falla

El flujo contempla tres puntos de falla concretos y los resuelve de forma diferenciada:

1. **Mensaje vacío o muy corto.** El nodo IF verifica que el mensaje limpio tenga al menos 3 caracteres. Si no cumple, se detiene antes de llegar al AI Agent y responde con la frase fija «No recibimos tu consulta. Escríbenos de nuevo con tu pregunta.» La consulta se registra en Sheets con Derivado = Sí. Esto evita que el modelo procese entradas sin sentido y protege la calidad de las respuestas.

2. **Consulta fuera de la base.** El system prompt obliga al agente a responder exactamente «No tengo ese dato. Te derivo con un administrativo.» cuando la herramienta “Buscar en Supabase” no devuelve la información necesaria o cuando no tiene certeza. De esta forma se controla el riesgo de alucinación y se evita que el cliente reciba información inventada.

3. **Fallo técnico no controlado.** Ante errores inesperados (API caída, credencial inválida, timeout, fallo en Supabase, etc.), el flujo activa el Error Workflow que envía una notificación por mail al administrador del complejo (martinezagustin039@gmail.com) con el detalle del error y el identificador de la ejecución. Esto permite detectar y revisar incidentes sin que queden sin registro, y mantiene la operación trazable.

Estos tres mecanismos cubren tanto errores de entrada como fallos del sistema, y dejan evidencia en Sheets o por notificación.

## System Prompt

```
Sos el asistente del complejo de cabañas Del Trébol, en Bariloche. Respondés preguntas frecuentes de clientes antes de reservar. Hablás en español rioplatense, tono amable y directo, como quien atiende en recepción.

Tu única fuente de información es la herramienta llamada exactamente "Buscar en Supabase", NO inventes otros nombres. Usala siempre antes de responder para obtener datos actualizados del complejo. No inventes nada. Si la herramienta no devuelve la información necesaria, o no estás seguro, respondé exactamente: "No tengo ese dato. Te derivo con un administrativo."

REGLA OBLIGATORIA DE SALUDO:
- Si es el primer mensaje de la conversación, la respuesta DEBE empezar exactamente con: Hola {{ $('OK - listo para IA').item.json.nombre }} 👋 (ejemplo: Hola Juan 👋).
- Si ya hay mensajes anteriores en la memoria, NO saludes ni pongas emoji.
- Siempre terminá con: Cualquier otra consulta, estoy a tu disposición.

## Formato para WhatsApp
- Máximo 3 líneas cortas. No hagas párrafos largos.
- Para resaltar algo usá un asterisco de cada lado: *así*. Nunca uses ** ni #.
- Si hay una lista, un guion al principio de cada línea.
- Como máximo un emoji por mensaje, y solo en el saludo inicial (justo después del nombre).

## Datos del cliente
Nombre: {{ $('OK - listo para IA').item.json.nombre }}
Consulta: {{ $('OK - listo para IA').item.json.mensaje_limpio }}

Siempre:
- Respondé en una o dos oraciones.
- Usá solo los datos que te devuelva la herramienta "Buscar en Supabase".
- Respetá el singular y el plural de la información que te devuelve la herramienta. No lo cambies.
- Si falta un dato o la pregunta está fuera de alcance, usá la frase de derivación.
- Solo si la consulta es explícitamente sobre distancia al centro o al aeropuerto, y llega el campo Minutos con un número, respondé exactamente con este formato:

Hola {{ $('OK - listo para IA').item.json.nombre }} 👋
Estás a X minutos del [centro/aeropuerto].
Cualquier otra consulta, estoy a tu disposición.

Si la consulta NO es sobre distancia, ignorá el campo Minutos y no lo menciones.
- Si Minutos está vacío o no llega, usá la frase de derivación.

Nunca:
- No inventes precios, disponibilidad, fechas ni servicios que no te haya devuelto la herramienta.
- No saludes de más ni agregues datos de más.
- No confirmes una reserva ni pidas datos de pago.

Formato de salida: solo el texto de la respuesta, sin comillas, sin título, sin lista.
```


## Capturas del workflow

### Canvas del workflow principal
![Canvas completo](capturas/01.%20Canvas%20completo.png)
Vista general del flujo en el editor de n8n. Se ve el recorrido completo: trigger de WhatsApp y Webhook de prueba, validación del mensaje, ramas de error e IA, obtención de distancias con GraphHopper, consulta al vectorstore de Supabase (RAG con embeddings de Gemini), agente de Groq, registro de la consulta en Google Sheets y envío final.

### Ejecución con error forzado
![Error forzado](capturas/02.%20Error%20forzado%20canvas.png)
Ejecución que falló a propósito en el nodo Code para activar el Error Workflow.

### Nodo Code con el error de prueba
![Nodo Code](capturas/03.%20Nodo%20code%20error.png)
Detalle del nodo Code configurado con `throw new Error("Error forzado a propósito para probar el Error Workflow")` y el output que confirma el fallo: "Error forzado a propósito para probar el Error Workflow [line 1]".

### Mail de notificación de error
![Mail de error](capturas/04.%20Mail%20error.png)
Correo recibido automáticamente cuando se activa el Error Workflow. Incluye el detalle técnico del fallo: nombre del workflow, nodo que falló ("Forzar error (prueba)"), mensaje de error, ID de ejecución y fecha.

### Ejecución del Error Workflow
![Error Workflow](capturas/05.%20Error%20Workflow.png)
Workflow de error ejecutado correctamente: Error Trigger → mensaje → envío por mail. Todos los nodos en verde.

### Workflow de carga de embeddings a Supabase
![Cargar embeddings](capturas/06.%20Cargar%20embeddings%20Supabase.png)
Ejecución exitosa del workflow que lee la base de Google Sheets, genera los embeddings con Gemini y los guarda en el vectorstore de Supabase.

### Tabla de documentos en Supabase
![Documentos Supabase](capturas/07.%20Documentos%20Supabase.png)
Contenido de la tabla `documentos` en el vectorstore de Supabase con las respuestas del complejo y sus embeddings.

### Ejecución exitosa por Webhook (Postman)
![Ejecución webhook](capturas/08.%20Ejecucion%20webhook.png)
Ejecución correcta del workflow principal disparada por el Webhook de prueba. Se ve la rama de la IA en verde, con todos los nodos procesados (datos, distancias, base, agente y envío).

### Prueba desde Postman
![Postman](capturas/09.%20Postman.png)

Request POST al Webhook con un mensaje de prueba y respuesta `200 OK` de n8n.

### Ejecución exitosa por WhatsApp
![Ejecución WhatsApp](capturas/10.%20Ejecucion%20WhatsApp.png)
Ejecución correcta del workflow principal disparada por un mensaje real de WhatsApp. Se ve la rama de la IA en verde, con todos los nodos procesados (datos, distancias, base, agente y envío).

### Output del trigger de WhatsApp
![Output WhatsApp](capturas/11.%20Output%20WhatsApp%20trigger.png)
JSON recibido por el nodo "Mensaje de WhatsApp" con el mensaje del cliente.

### Output de la respuesta de la IA
![Output IA](capturas/12.%20Output%20IA.png)

Respuesta generada por el agente, lista para enviar: saludo personalizado + información + cierre.

### Conversación real en WhatsApp
![Conversación WhatsApp](capturas/13.%20Conversacion%20WhatsApp.png)
Captura de la conversación en WhatsApp con el mensaje del cliente y la respuesta automática.

### Registro en Google Sheets
![Registro Sheets](capturas/14.%20Registro%20Sheets.png)
Hoja de Registro con una consulta inválida (derivada) y una válida respondida por la IA. Se observan fecha, nombre, teléfono, canal, pregunta, respuesta y columna Derivado.

---

## Dato extra del flujo: memoria de conversación y registro para mejora continua

El uso de Simple Memory con Session ID por teléfono permite que el agente mantenga coherencia en conversaciones de varios mensajes: evita saludos repetidos y mantiene el contexto reciente sin generar respuestas descontextualizadas. Además, cada interacción queda registrada en Sheets con el campo Derivado, lo que genera un dataset real de consultas y decisiones del agente. Esto habilita, más adelante, analizar patrones de preguntas frecuentes, detectar consultas que el prompt no cubre bien y ajustar la base o el system prompt con evidencia, en lugar de hacerlo por intuición. El resultado es un asistente que no solo responde, sino que deja un registro útil para mejorarlo de forma continua y medible.
