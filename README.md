#VERIFICANUTRICIONLORENZOGOMEZCOUNAHAN
# PROYECTO NUTRISCORE 3.2 

## Historial de Cambios e Historial de Versiones (v3.1 / v3.2)
Esta versión consolida la estabilidad y los problemas de seguimiento de flujo de los nodos de error:

1. **Se reintroduce el Nodo Webhook:** Se transicionó nuevamente a un disparador por nodo "Webhook" eliminando la versión anterior del nodo "On form submission" de n8n. (método HTTP POST a la ruta `barcode_number`), permitiendo que el flujo funcione como una API síncrona en tiempo real.
2. **Respuestas de Error Nativas:** Se implementaron nodos especializados `Respond to Webhook` para entregar respuestas JSON inmediatas y estructuradas directamente al cliente cuando ocurren fallos críticos en validación sintáctica y en comprobación de existencia en base de datos de la API.  Esto simplifica la integración de errores en el nodo LLM.
3. **Integración de Inteligencia Artificial (LangChain):** Se ha delegado la lógica de consolidación y el veredicto final a una cadena LLM (`IA RECOLECCIÓN Y VEREDICTO`) conectada a un modelo externo mediante Groq (`Groq Chat Model`), la cual unifica los veredictos cualitativos y cuantitativos generando un reporte nutricional enriquecido con emojis y texto persuasivo.
4. **Multiplexación Avanzada:** Se configuraron los nodos `MERGE` en modalidad multi-entrada (3 canales independientes de entrada por cada tipo de evaluación) para colectar ordenadamente los estados lógicos (*Saludable*, *Moderado*, *No Saludable*) previo a su inyección en el prompt de la IA.

---

## Arquitectura de Nodos

### Bloque Inicial y Conectividad Externa

#### NODO: Webhook
* **Acción:** Escuchar peticiones HTTP entrantes.
* **Propósito:** Actuar como el disparador (*Trigger*) inicial del flujo mediante un método `POST` expuesto en la ruta `barcode_number`.
* **Input:** Objeto JSON con el cuerpo de la petición.
* **Output:** JSON con el campo `body.barcode_number`.

#### NODO: Validación Código de Barras
* **Acción:** Evaluar condiciones lógicas concurrentes (`AND`).
* **Propósito:** Validar estrictamente por Regex (`^\d{8}$|^\d{13}$`) que la propiedad inyectada no esté vacía y que cuente exclusivamente con una longitud exacta de 8 o 13 caracteres numéricos.
* **Input:** Payload crudo del Webhook.
* **Output:**
  * **ÉXITO (True):** Enruta hacia la API de Open Food Facts.
  * **FAILURE (False):** Desvía hacia la respuesta de Bad Request en nodo Webhook Response.

#### NODO: Respond to Webhook (Error de Sintaxis)
* **Acción:** Responder a la llamada HTTP entrante de forma inmediata.
* **Propósito:** Entregar una estructura JSON de contingencia simulando un **Error 400 (Bad Request)** debido a un código de barras vacío o no válido, ofreciendo un ejemplo de solución (`80176800`).
* **Output:** 
  ```json
  {
    "Code": "Bad Request",
    "Code": 400,
    "Reason": "el código de barras está vacío o no es válido",
    "Solution": "Intenta con un código de 8 o 13 números como el siguiente:80176800"
  }
  ```

#### NODO: Consulta API Open Food Facts
* **Acción:** Ejecución de petición HTTP GET.
* **Propósito:** Consultar la base de datos pública mediante la URL estructurada `https://openfoodfacts.org{{ $json.body.barcode_number }}` inyectando un `User-Agent` personalizado e impidiendo rupturas del flujo global con la instrucción `neverError: true`.
* **Output:** Objeto de respuesta HTTP incluyendo el código de estado (`statusCode`).

#### NODO: Validación ¿Existe en Data Base de OFF?
* **Acción:** Evaluar igualdad numérica.
* **Propósito:** Validar que el código de estado (`statusCode`) de la consulta sea diferente de `0` (o respuestas nulas).
* **Output:**
  * **ÉXITO (True):** Continúa al formateador de datos.
  * **FAILURE (False):** Desvía hacia la respuesta de error 404 en nodo Webhook Response.

#### NODO: Respond to Webhook1 (Error de Existencia)
* **Acción:** Enviar respuesta HTTP inmediata.
* **Propósito:** Retornar una estructura JSON limpia que notifica un **Error 404 (Not Found)** indicando que el producto no se encuentra catalogado.
* **Output:** 
  ```json
  {
    "Status": "Not Found",
    "Code": 404,
    "Reason": "el código de barras no existe en nuestra base de datos",
    "Solution": "¿Quieres intentar con otro código"
  }
  ```

---

### Tratamiento de Datos

#### NODO: RECOLECCIÓN INFORMACION NUTRICIONAL
* **Acción:** Seteo y normalización de variables (`Data Cleansing`).
* **Propósito:** Mapear y aislar del payload extenso original las métricas indispensables para el negocio bajo llaves normalizadas por cada 100 gramos de producto.
* **Variables Extraídas:**
  * `Producto` (Cadena)
  * `Nutriscore` (Extraído de la propiedad del año `"2021".grade`)
  * `Azúcares por 100g` (Numérico)
  * `Grasas por 100g` (Numérico)
  * `Sal por 100g` (Numérico)

---

### Subflujo A: Evaluación Cualitativa (Nutriscore)

#### NODO: CLASIFICACIÓN: NUTRISCORE ¿ES SALUDABLE?
* **Acción:** Evaluación condicional con compuerta OR.
* **Propósito:** Comprobar si la variable `Nutriscore` es exactamente igual a la cadena `"a"` o `"b"`.
* **Output:**
  * **True:** Enruta a `NSCORE "SALUDABLE"`.
  * **False:** Enruta al evaluador restrictivo `NSCORE CLASIF NO SALUDABLE`.

#### NODO: NSCORE CLASIF NO SALUDABLE
* **Acción:** Evaluación condicional simple.
* **Propósito:** Validar si el residuo del Nutriscore equivale a la letra `"c"` para catalogarlo como Intermedio/Moderado; en caso contrario, se asume por descarte como nocivo (`"d"` o `"e"`).
* **Output:**
  * **True:** Enruta a `NUTRISCORE "MODERADO"`.
  * **False:** Enruta a `NUTRISCORE "NO SALUDABLE"`.

#### NODOS DE ASIGNACIÓN: NSCORE "SALUDABLE" / "MODERADO" / "NO SALUDABLE"
* **Acción:** Setear variable homogénea.
* **Propósito:** Inyectar una cadena descriptiva normalizada bajo la propiedad común `NUTRISCORE` (Ej: `tu nutriscore es {{ $json.Nutriscore }}, SALUDABLE`).
* **Output:** Flujo redirigido hacia el concentrador de datos `MERGE NSCORE` según el índice de entrada correspondiente (0, 1 o 2).

---

### Subflujo B: Evaluación Cuantitativa (Nutrientes Críticos)

#### NODO: CLASIFICACIÓN NUTRIENTES ¿SON SALUDABLES?
* **Acción:** Evaluación numérica restrictiva con compuerta AND.
* **Propósito:** Comprobar si el producto posee niveles óptimos simultáneos basados en tres umbrales estrictos por cada 100g:
  * `Azúcares por 100g` < 5
  * `Grasas por 100g` < 17.5
  * `Sal por 100g` < 1.5
* **Output:**
  * **True:** Enruta a `NUTRIENTES "SALUDABLE"`.
  * **False:** Enruta al evaluador secundario de excesos.

#### NODO: CLASIF NUTRIENTES NO SALUDABLE
* **Acción:** Evaluación condicional con compuerta AND.
* **Propósito:** Confirmar si el producto supera de manera unánime y simultánea todos los límites máximos permitidos (`Azúcares` >= 5, `Grasas` >= 17.5, `Sal` >= 1.5).
* **Output:**
  * **True:** Enruta a `NUTRIENTES "NO SALUDABLE"`.
  * **False (Criterio Mixto):** Deriva a `NUTRIENTES "MODERADO"` al detectar que solo algunos nutrientes exceden la marca.

#### NODOS DE ASIGNACIÓN: NUTRIENTES "SALUDABLE" / "MODERADO" / "NO SALUDABLE"
* **Acción:** Seteo de textos informativos paramétricos.
* **Propósito:** Redactar un string dinámico que detalla de forma explícita las cantidades de azúcar, grasas y sodio encontradas, adaptando el tono de la recomendación al nivel de saludabilidad actual del ítem.
* **Output:** Flujo redirigido hacia el concentrador de datos `MERGE NUTRIENTES` según el índice de entrada correspondiente (0, 1 o 2).

---

### Inteligencia Artificial y Cierre

#### NODOS: MERGE NSCORE / MERGE NUTRIENTES
* **Acción:** Multiplexación y recolección de flujos paralelos por número de entradas (3 canales).
* **Propósito:** Actuar como barreras síncronas que recogen las salidas de los respectivos subflujos lógicos independientemente de qué rama lógica condicional haya resultado activada, evitando la fragmentación y unificando el canal de transmisión de datos.

#### NODO: IA RECOLECCIÓN Y VEREDICTO
* **Acción:** Ejecución de cadena LLM basada en prompts estructurados (`ChainLLM`).
* **Propósito:** Procesar las variables consolidadas del producto actuando bajo el rol de un **Asistente Nutricional**. Aplica una matriz de decisión interna cruzada para calcular un veredicto final de saludabilidad combinando Nutriscore y nutrientes, generando de forma autónoma una respuesta final formateada con semáforos visuales (`🟢`, `🟡`, `🔴`).
* **Input de Datos Cruzados:** Vincula dinámicamente las variables `Producto`, `Nutriscore`, `Azúcares por 100g`, `Grasas por 100g` y `Sal por 100g` provenientes del nodo centralizador de recolección nutricional.

#### NODO: Groq Chat Model
* **Acción:** Proveedor del Modelo de Lenguaje (*Language Model*).
* **Propósito:** Suministrar la potencia cognitiva a la cadena de IA utilizando las credenciales integradas de Groq y exponiendo el modelo de ejecución `openai/gpt-oss-120b`.

#### NODO: Edit Fields
* **Acción:** Formateo final del payload de salida.
* **Propósito:** Limpiar la respuesta entregada por la Inteligencia Artificial aislando exclusivamente el texto enriquecido generado (`{{ $json.text }}`) bajo el atributo simplificado `text` para el consumo final del cliente o aplicación consumidora.
