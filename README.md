# Asistente de WhatsApp del Complejo Del Trébol

## El caso

En el Complejo de Cabañas Del Trébol (Bariloche) los clientes consultan con frecuencia antes de reservar sobre servicios básicos: ropa blanca, cochera, parrilla, capacidad, mascotas, horarios de check-in y check-out, wifi, pileta, etc. Hoy esas consultas se responden de forma manual o con plantillas prearmadas. El volumen es alto y se repite todos los días, especialmente en temporada alta. Este proyecto automatiza ese proceso con un AI Agent conectado a WhatsApp, usando una base de respuestas controlada y dejando registro de cada interacción.

## El problema y el ROI

- **Proceso manual:** responder preguntas frecuentes de clientes antes de reservar (ropa blanca, cochera, parrilla, capacidad, mascotas, check-in/out, wifi, pileta, etc.).
- **Volumen:** 20 a 25 mensajes por día en temporada baja y más del doble en temporada alta. Cada respuesta lleva entre 4 y 5 minutos (leer el mensaje, ubicar el hilo, escribir la respuesta y enviarla). El tiempo total sube porque muchas consultas llegan mientras se están haciendo otras tareas: hay que interrumpir, volver al celular o a la PC, retomar el hilo y responder.
- **Tiempo actual:** entre 2,5 y 3 horas diarias en temporada baja (≈ 15–18 horas semanales). En temporada alta supera las 30 horas semanales.
- **Costo hoy:** tomando un costo hora administrativo de $8.000 ARS, el proceso consume entre $120.000 y $144.000 ARS por semana en temporada baja, y más de $240.000 ARS en temporada alta.
- **Costo de automatizar:**
  - n8n Cloud Starter (≈ USD 20–24/mes): ~$31.000 – $38.000 ARS/mes.
  - Gemini (modelo Flash): a este volumen el costo es muy bajo (menos de $5.000 ARS/mes).
  - WhatsApp Business API: las primeras 1.000 respuestas de servicio por mes son gratuitas; a partir de ahí ~ USD 0,026 por mensaje (~$40 ARS). Con 20–50 respuestas diarias el costo mensual se mantiene bajo.
  - **Total estimado:** entre $40.000 y $55.000 ARS/mes.
- **ROI:** el ahorro de tiempo es inmediato. En temporada baja se recuperan entre 15 y 18 horas semanales. El costo mensual de la automatización representa menos del 35 % del costo del tiempo manual. A partir del segundo mes el retorno supera el 200 %. En temporada alta el ROI es aún mayor.
- **Límite del ROI:** si el volumen cayera de forma sostenida por debajo de 8–10 consultas diarias, el costo operativo superaría el tiempo ahorrado y dejaría de ser conveniente.
  
## Cómo funciona

1. **Entrada 1:** WhatsApp Trigger (trigger) recibe el mensaje del cliente (texto, teléfono y nombre).

2. **Entrada 2:** según el tipo de consulta (detectado por un nodo Switch), el flujo consulta GraphHopper mediante un HTTP Request para obtener el tiempo de viaje al aeropuerto o al centro de Bariloche. Elegí esta segunda fuente porque los clientes suelen preguntar por la ubicación relativa del complejo (“¿qué tan lejos está del aeropuerto?” o “¿a cuánto están del centro?”) antes de reservar. Incorporar ese dato como origen independiente permite que el agente lo tenga disponible en el contexto cuando es relevante, sin que dependa de una búsqueda manual en cada consulta. Además, al ser un HTTP Request a un servicio externo, cubre el requisito de contar con un origen de datos distinto al de WhatsApp (trigger) y al de la base de respuestas (Google Sheets).

3. **Proceso:**
   - Un nodo Edit Fields (“Datos de la consulta”) normaliza el mensaje entrante: elimina emojis innecesarios, pasa a minúsculas si corresponde y extrae únicamente el nombre del cliente (sin apellido). También calcula el largo del mensaje para poder validarlo después.
   - Un nodo IF (“¿Mensaje válido?”) verifica que el mensaje limpio tenga al menos 3 caracteres. Si no cumple, el flujo se deriva a la rama de error (nodo “Error - Inválido”, respuesta fija y registro con Derivado = Sí). Si cumple, continúa hacia el nodo “OK - listo para IA”.
   - Desde “OK - listo para IA” el flujo se divide: una rama va directo al nodo Merge (“Unir datos”) y otra pasa por el Switch (“Tipo de consulta”), que decide si corresponde consultar la distancia al aeropuerto o al centro. Según el caso, se dispara el HTTP Request a GraphHopper. GraphHopper devuelve el tiempo de viaje en milisegundos dentro de un objeto anidado (`paths[0].time`). Los nodos Set “Minutos aeropuerto” y “Minutos centro” convierten ese valor a minutos enteros con la expresión `{{ Math.round($json.paths[0].time / 60000) }}`, dejando un dato limpio y legible para el agente. Esta consulta solo se activa si el mensaje del cliente menciona explícitamente el aeropuerto o el centro; en el resto de las consultas, el Switch no dispara el HTTP Request y el flujo continúa sin ese dato.
   - Los datos del mensaje y el resultado de GraphHopper (si aplica) se unen en el nodo Merge.
   - Luego se lee la base de respuestas desde Google Sheets (hoja “Base”). Esta lectura funciona como una tercera fuente de datos: contiene las respuestas autorizadas del complejo y es la única fuente que el agente puede usar para contestar. Un nodo de transformación (“Armar contexto”) convierte todas las filas en un único texto con formato “Tema: respuesta” y lo pasa al AI Agent.
   - El AI Agent (Gemini) recibe el system prompt, el mensaje del cliente, el contexto de la base y, si corresponde, el dato de distancia. Genera la respuesta aplicando estrictamente las reglas definidas. Cuando el cliente pregunta por la distancia, el agente incorpora el tiempo de viaje obtenido de GraphHopper dentro de la respuesta autorizada de la base (por ejemplo, “El complejo está a X minutos del aeropuerto”), sin inventar información ni agregar datos que no estén en las fuentes.
   - Un nodo Simple Memory, con el número de teléfono como Session ID y ventana de 5 interacciones, permite que el agente recuerde el contexto reciente de la conversación (evita repetir el saludo y mantiene coherencia).
     - En ambos caminos (válido o error) se registra la consulta en Google Sheets con fecha, nombre, teléfono, canal, pregunta, respuesta y el campo Derivado (Sí/No).

   **Sobre la API de GraphHopper**  
   En los nodos HTTP Request de distancia (aeropuerto y centro) el parámetro `key=` aparece vacío en el JSON exportado del workflow. Esto es así porque eliminé la key del archivo antes de subirlo, para cumplir con el requisito de no dejar claves, tokens ni secretos visibles en el JSON. En el flujo real la key está configurada en el nodo y funciona correctamente. De esta forma se integra la fuente externa (HTTP Request) sin dejar la clave en el archivo entregado.

4. **Salida:** se envía la respuesta por WhatsApp al mismo número del cliente y queda registrada en la hoja de Registro.

## Por qué un AI Agent y no un nodo de IA simple

Elegí un AI Agent porque el flujo requiere una decisión real basada en contexto. El agente recibe la base de respuestas desde Google Sheets, aplica las reglas del system prompt (rol, alcance, tono, límites y qué hacer ante lo ambiguo) y elige si responde con un dato concreto de la lista o deriva con la frase fija exacta. Un nodo de IA simple no tiene acceso a herramientas ni un marco de decisión: solo generaría texto libre sin poder consultar la fuente ni garantizar que respete las reglas de no inventar información. En este caso el agente funciona como un recepcionista con límites claros y trazabilidad, no como un generador de texto sin control.

## Local o nube

Elegí **Gemini en la nube** por dos razones concretas.

Primero, las consultas que recibe el agente no contienen datos sensibles: son preguntas sobre servicios públicos del complejo (ropa blanca, cochera, capacidad, etc.), no datos de pago ni información personal de los clientes. Segundo, probé Ollama local con llama3.1:8b. Las respuestas resultaron inconsistentes: en varias pruebas omitió datos que sí estaban en la base, no respetó el formato de WhatsApp de forma estable y generó variaciones no controladas en el saludo y el cierre. Con 8 GB de RAM, mantener un modelo local agrega complejidad operativa (dependencia de que la máquina esté encendida, consumo de recursos y menor predictibilidad) sin mejorar la calidad en este volumen y tipo de consulta. Gemini ofrece respuestas predecibles, baja latencia y un costo por request muy bajo a este volumen, lo que lo hace más adecuado para un uso real por WhatsApp.

## Qué pasa si algo falla

El flujo contempla tres puntos de falla concretos y los resuelve de forma diferenciada:

1. **Mensaje vacío o muy corto.** El nodo IF verifica que el mensaje limpio tenga al menos 3 caracteres. Si no cumple, se detiene antes de llegar al AI Agent y responde con la frase fija «No recibimos tu consulta. Escríbenos de nuevo con tu pregunta.» La consulta se registra en Sheets con Derivado = Sí. Esto evita que el modelo procese entradas sin sentido y protege la calidad de las respuestas.

2. **Consulta fuera de la base.** El system prompt obliga al agente a responder exactamente «No tengo ese dato. Te derivo con un administrativo.» cuando la pregunta no está en la lista o cuando no tiene certeza. De esta forma se controla el riesgo de alucinación y se evita que el cliente reciba información inventada.

3. **Fallo técnico no controlado.** Ante errores inesperados (API caída, credencial inválida, timeout, etc.), el flujo activa una notificación por mail que incluye el detalle del error y el identificador de la ejecución. Esto permite detectar y revisar incidentes sin que queden sin registro, y mantiene la operación trazable.

Estos tres mecanismos cubren tanto errores de entrada como fallos del sistema, y dejan evidencia en Sheets o por notificación.

## System Prompt

```
Sos el asistente del complejo de cabañas Del Trébol, en Bariloche. Respondés preguntas frecuentes de clientes antes de reservar. Hablás en español rioplatense, tono amable y directo, como quien atiende en recepción.
Tu única fuente es la lista de abajo. No inventes nada. Si la consulta no está en la lista, o no estás seguro, respondé exactamente: "No tengo ese dato. Te derivo con un administrativo."
Datos del complejo:
- Ropa blanca: Sí, el complejo cuenta con ropa blanca: sábanas y toallones.
- Cochera: Las cabañas tradicionales tienen cocheras compartidas. Las premium, individuales.
- Parrilla: Las premium tienen parrilla individual. Tradicionales y monoambiente: 4 parrillas compartidas en un sector del predio.
- Wifi: Sí, gratuito en todo el complejo.
- Capacidad: Tradicionales hasta 5 personas, premium hasta 6, monoambiente hasta 3.
- Mascotas: Sí, aceptamos mascotas.
- Check-in: A partir de las 14:00 hs.
- Check-out: Hasta las 10:00 hs.
- Pileta: No contamos con pileta.
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
- Respetá el singular y el plural de la base. No lo cambies.
- Usá solo los datos de la lista.
- Si falta un dato o la pregunta está fuera de alcance, usá la frase de derivación.
- Si la consulta es sobre distancia al centro o al aeropuerto y llega el campo Minutos con un número, respondé exactamente con ese número usando este formato:
Hola {{ $('OK - listo para IA').item.json.nombre }} 👋
Estás a X minutos del [centro/aeropuerto].
Cualquier otra consulta, estoy a tu disposición.
- Si Minutos está vacío o no llega, usá la frase de derivación.
Nunca:
- No inventes precios, disponibilidad, fechas ni servicios que no estén en la lista.
- No saludes de más ni agregues datos de más.
- No confirmes una reserva ni pidas datos de pago.
## Saludo y cierre:
- Solo en el primer mensaje de la conversación, empezá con «Hola» seguido del nombre y un emoji (ejemplo: Hola Juan 👋).
- Si la conversación ya empezó, no saludes ni pongas emoji.
- Terminá con «Cualquier otra consulta, estoy a tu disposición.»
Formato de salida: solo el texto de la respuesta, sin comillas, sin título, sin lista.
```

## Capturas del workflow

### Canvas del workflow principal
![Canvas del workflow principal](capturas/Captura%20de%20pantalla%202026-11-01%20180612.png)

Vista general del flujo en el editor de n8n. Se ve el recorrido completo: trigger de WhatsApp, validación del mensaje, ramas de error e IA, obtención de distancias con GraphHopper, consulta a la base de respuestas, agente de Gemini y envío final.

### Ejecución con error forzado (Code)
![Ejecución con error forzado](capturas/Captura%20de%20pantalla%202026-11-03%20190531.png)

Ejecución que falló a propósito en el nodo Code. Se observa el nodo en rojo y el resto del flujo detenido, lo que activa el Error Trigger.

### Nodo Code con el error de prueba
![Nodo Code con error de prueba](capturas/Captura%20de%20pantalla%202026-11-03%20190618.png)

Detalle del nodo Code configurado con `throw new Error("Error de prueba para el Error Trigger")` y el output que confirma el fallo.

### Mail de notificación de error
![Mail de notificación de error](capturas/Captura%20de%20pantalla%202026-11-03%20190715.png)

Correo recibido automáticamente cuando se activa el Error Workflow. Confirma que la alerta llega al administrador.

### Ejecución del Error Trigger
![Ejecución del Error Trigger](capturas/Captura%20de%20pantalla%202026-11-03%20190742.png)

Workflow de error ejecutado correctamente: Error Trigger → mensaje de error → envío por mail. Todos los nodos en verde.

### Ejecución exitosa del flujo completo
![Ejecución exitosa](capturas/Captura%20de%20pantalla%202026-11-03%20191305.png)

Ejecución correcta del workflow principal. Se ve la rama de la IA en verde, con todos los nodos procesados (datos, distancias, base, agente y envío).

### Registro en Google Sheets
![Registro en Google Sheets](capturas/Captura%20de%20pantalla%202026-11-03%20191612.png)

Hoja de Registro con dos filas: una consulta inválida (derivada) y una válida respondida por la IA. Se observan fecha, nombre, teléfono, canal, pregunta, respuesta y columna Derivado.

### Output del trigger de WhatsApp
![Output del trigger de WhatsApp](capturas/Captura%20de%20pantalla%202026-11-03%20191421.png)

JSON recibido por el nodo “Mensaje de WhatsApp” con el mensaje real del cliente (“Hola, tienen ropa blanca?”).

### Nodo de envío por WhatsApp
![Nodo de envío por WhatsApp](capturas/Captura%20de%20pantalla%202026-11-03%20191501.png)

Configuración del nodo “Enviar por Whatsapp” con las expresiones que arman el número y el cuerpo del mensaje a partir de los datos del flujo.

### Output de la respuesta de la IA
![Output de la respuesta de la IA](capturas/Captura%20de%20pantalla%202026-11-03%20191546.png)

Respuesta generada por el agente de Gemini, lista para enviar: saludo personalizado + información de ropa blanca + cierre.

### Conversación real en WhatsApp
![Conversación en WhatsApp](capturas/Captura%20de%20pantalla%202026-11-03%20191918.png)

Captura de la conversación en WhatsApp donde se ve el mensaje del cliente y la respuesta automática del flujo.

---

## Dato extra del flujo: memoria de conversación y registro para mejora continua

El uso de Simple Memory con Session ID por teléfono permite que el agente mantenga coherencia en conversaciones de varios mensajes: evita saludos repetidos y mantiene el contexto reciente sin generar respuestas descontextualizadas. Además, cada interacción queda registrada en Sheets con el campo Derivado, lo que genera un dataset real de consultas y decisiones del agente. Esto habilita, más adelante, analizar patrones de preguntas frecuentes, detectar consultas que el prompt no cubre bien y ajustar la base o el system prompt con evidencia, en lugar de hacerlo por intuición. El resultado es un asistente que no solo responde, sino que deja un registro útil para mejorarlo de forma continua y medible.
