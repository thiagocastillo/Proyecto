# 🎓 Guía de Defensa Oral — Thiago Castillo
## Proyecto EcoGrid 2026

---

> [!IMPORTANT]
> Este documento está basado en tu código real. Cada explicación referencia líneas exactas de tus archivos.

---

# 📊 DIAPOSITIVA 7 — Flujo de Procesamiento (`RequestQueue`)

## ¿Qué es `RequestQueue`?

Es la clase que **gestiona la cola de solicitudes de energía** y contiene el algoritmo central de asignación. Tiene dos responsabilidades:
1. Guardar solicitudes en orden de llegada (FIFO).
2. Procesar la siguiente solicitud buscando un nodo disponible.

---

## 🔍 Código línea a línea: `processNext()`

```java
// Línea 35-67 de RequestQueue.java
public Transaction processNext(NodeManager nodes) {
```
→ Recibe el `NodeManager` (la lista de todos los nodos generadores).  
→ Devuelve una `Transaction` si tuvo éxito.

---

```java
    if (queue.isEmpty()) {
        throw new NoSuchElementException("No hay solicitudes pendientes en la cola");
    }
```
**¿Qué hace?** Antes de operar, verifica que haya algo para procesar.  
**¿Por qué?** Es una **validación defensiva**: si la cola está vacía y tratás de sacar algo, el programa se rompe. Con este chequeo, lanzás un error controlado con mensaje claro.

---

```java
    EnergyRequest req = queue.dequeue();
```
**¿Qué hace?** Saca la solicitud **más antigua** de la cola (la que llegó primero).  
**Internamente (`Queue.dequeue()`):** toma el `head` (primer nodo) de la lista enlazada, avanza `head = head.getNext()` y reduce `size--`. Es O(1).  
**Aquí se aplica FIFO:** la primera solicitud que entró es la primera que se atiende.

---

```java
    LinkedList<EnergyNode> nodeList = nodes.getNodes();
```
**¿Qué hace?** Obtiene la lista completa de todos los nodos generadores registrados en el sistema.

---

```java
    EnergyNode assignedNode = null;
    for (int i = 0; i < nodeList.size(); i++) {
        EnergyNode node = nodeList.get(i);
        if (node.getCurrentLoad() + req.getRequestedAmount() <= node.getMaxCapacity()) {
            assignedNode = node;
            break;
        }
    }
```
**¿Qué hace?** Recorre los nodos uno por uno buscando el **primero** que pueda absorber la demanda.  
**La condición clave:**
```
currentLoad + demanda <= maxCapacity
```
- `currentLoad` = energía que el nodo ya está entregando ahora mismo.
- `req.getRequestedAmount()` = energía que el consumidor está pidiendo.
- `maxCapacity` = techo físico del nodo (no puede superarlo).

**Ejemplo concreto:**
> Nodo Solar: capacidad=500 kWh, carga actual=300 kWh.  
> Consumidor pide 150 kWh.  
> 300 + 150 = 450 ≤ 500 ✅ → se asigna.

**Algoritmo "Primer Ajuste":** no busca el nodo más eficiente, busca el **primero que funcione**. Simple y rápido.

---

```java
    if (assignedNode == null) {
        throw new IllegalStateException("CAPACIDAD INSUFICIENTE para el consumidor: " + req.getConsumerId());
    }
```
**¿Qué hace?** Si después de revisar TODOS los nodos ninguno pudo cubrir la demanda, lanza un error.

**¿Por qué `IllegalStateException` y no otro error?**
- `IllegalStateException` = "el estado actual del sistema no permite esta operación".
- El sistema tiene nodos, tiene cola, pero en **este momento** no hay capacidad → el *estado* es inválido para operar.
- No es un error de programación (no es `NullPointerException`), es una situación de negocio esperada.

**¿Qué pasa con la solicitud?** Ya fue extraída de la cola con `dequeue()`. Si ningún nodo la absorbió, se descarta. El sistema no la reencola automáticamente.

---

```java
    assignedNode.setCurrentLoad(assignedNode.getCurrentLoad() + req.getRequestedAmount());
    return new Transaction(
            assignedNode.getId(),
            req.getConsumerId(),
            req.getRequestedAmount(),
            LocalDateTime.now());
```
**¿Qué hace?**
1. Actualiza la carga del nodo sumando la demanda asignada.
2. Crea y retorna una `Transaction` que registra: quién dio energía, quién la recibió, cuánto y cuándo.

**"Entrega atómica"** significa que **un solo nodo debe cubrir TODA la energía pedida**. No se divide la demanda entre varios nodos. O un nodo la cubre completa, o no se asigna. Esto simplifica la contabilidad y garantiza consistencia.

---

## 🔄 Flujo completo paso a paso (ejemplo real)

```
Estado inicial:
  Cola: [SolicitudA(200 kWh), SolicitudB(400 kWh)]
  Nodos: [Solar(500 max, 350 actual), Eólico(300 max, 0 actual)]

Paso 1: processNext() llamado
  → dequeue() saca SolicitudA (200 kWh) — La más antigua

Paso 2: Recorre nodos
  → Solar: 350 + 200 = 550 > 500 ❌ No alcanza
  → Eólico: 0 + 200 = 200 ≤ 300 ✅ Asignado!

Paso 3: Actualiza carga
  → Eólico.currentLoad = 0 + 200 = 200 kWh

Paso 4: Crea Transaction
  → {nodeId:"Eólico", consumerId:"SolicitudA", amount:200, timestamp:now}

Paso 5: En App.java (línea 77-78):
  Transaction tx = requestQueue.processNext(nodeManager);
  history.push(tx);  // ← Guarda en la pila para posible undo
```

---

## ❓ Preguntas del profesor — Diapositiva 7

### "¿Por qué FIFO y no prioridad?"
> "Elegimos FIFO porque el sistema prioriza la **equidad**: quien llega primero se atiende primero. Implementar prioridad agregaría complejidad (necesitaríamos reordenar la cola en cada inserción). Dado el alcance del proyecto, FIFO es suficiente y correcto. Si se quisiera prioridad, usaríamos una Priority Queue."

### "¿Qué pasa si el consumidor pide más de la capacidad máxima de cualquier nodo?"
> "Se lanza `IllegalStateException`. La solicitud ya salió de la cola. El sistema informa el error pero **sigue funcionando** gracias al `errorHandler` de `App.java` que captura la excepción sin detener el programa."

### "¿Por qué no regresar la solicitud a la cola si falla?"
> "Es una decisión de diseño. Si la reencolaras, podría quedarse atascada para siempre si ningún nodo nunca tiene capacidad suficiente, generando un loop infinito. El sistema prefiere informar el fallo y que el usuario genere una nueva solicitud."

### ⚠️ PREGUNTA TRAMPA: "¿Es tu cola thread-safe?"
> "No. Esta implementación no usa sincronización (`synchronized`). En un sistema concurrente con múltiples hilos, podría haber condiciones de carrera. Para este proyecto educativo con ejecución secuencial, no es necesario, pero en producción usaríamos `BlockingQueue` de Java."

### ⚠️ PREGUNTA TRAMPA: "¿Qué complejidad tiene `processNext()`?"
> "O(n) donde n es la cantidad de nodos. En el peor caso recorre todos los nodos sin encontrar capacidad. El `dequeue()` en sí es O(1) porque operamos sobre el `head` de la lista enlazada."

---

## 🎤 Mini discurso oral — Diapositiva 7

> *"En esta diapositiva presento el núcleo del procesamiento de solicitudes. El sistema usa una cola FIFO para garantizar equidad: la primera solicitud que llega es la primera en atenderse. Cuando se invoca `processNext()`, primero verificamos que la cola no esté vacía, luego extraemos la solicitud más antigua con `dequeue()`, y recorremos los nodos generadores buscando el primero que pueda cubrir la demanda completa. La condición es que la carga actual más la nueva demanda no supere la capacidad máxima del nodo. Si encontramos un nodo apto, actualizamos su carga y generamos una transacción. Si ningún nodo tiene capacidad, lanzamos una `IllegalStateException` informando al sistema. La entrega es atómica: un solo nodo cubre toda la demanda, sin dividirla."*

---
---

# 📊 DIAPOSITIVA 8 — Historial y Undo (`Stack`)

## ¿Qué es `Stack<Transaction>` en este sistema?

Es la estructura de datos que guarda el **historial de todas las transacciones exitosas**, permitiendo revertir la última operación realizada.

- Implementada en `Stack.java` → extiende `LinkedList<T>`
- Usada en `App.java` línea 29: `Stack<Transaction> history = new Stack<>();`
- Principio: **LIFO** — Last In, First Out

---

## 🔍 Código línea a línea: `Stack.java`

```java
public class Stack<T> extends LinkedList<T> implements TDAStack<T> {
```
→ `Stack` hereda de `LinkedList` (nuestra propia implementación, no la de Java).  
→ Implementa la interfaz `TDAStack` (contrato que define push/pop/peek).  
→ Es **genérica** (`<T>`): puede guardar cualquier tipo. Aquí la usamos con `Transaction`.

---

```java
@Override
public void push(T item) {
    add(0, item);  // Inserta al inicio de la lista
}
```
**¿Qué hace?** Agrega el nuevo elemento en la **posición 0** (el tope de la pila).  
**¿Por qué posición 0?** Porque en LIFO, lo último que entra debe ser lo primero en salir. Al insertar siempre al inicio, el elemento más nuevo queda siempre al frente.  
**¿Es O(1)?** En nuestra `LinkedList`, `add(0, item)` crea un nuevo nodo y lo enlaza como nuevo `head`. No recorre toda la lista → **O(1)**.

---

```java
@Override
public T pop() {
    if (isEmpty()) {
        throw new NoSuchElementException("La pila está vacía, no se puede realizar pop.");
    }
    return remove(0);  // Elimina y retorna el elemento del inicio
}
```
**¿Qué hace?** Saca el elemento del tope (posición 0) y lo elimina de la pila.  
**¿Por qué O(1)?** `remove(0)` en una lista enlazada simplemente hace `head = head.getNext()`, sin iterar.  
**Validación:** si intentás hacer `pop()` en una pila vacía → `NoSuchElementException`. Sin este chequeo, daría `NullPointerException` (más difícil de depurar).

---

```java
@Override
public T peek() {
    if (isEmpty()) {
        throw new NoSuchElementException("La pila está vacía, no hay elementos para ver.");
    }
    return get(0);  // Consulta sin eliminar
}
```
**¿Qué hace?** Muestra el elemento del tope **sin quitarlo**. Útil para "espiar" la última transacción sin revertirla.

---

## 🔍 Código línea a línea: `undoLast()` en `NodeManager.java`

```java
// Línea 60-71 de NodeManager.java
public void undoLast(NodeManager nodes, Stack<Transaction> history) {
    if (history.isEmpty()) {
        throw new NoSuchElementException("No hay transacciones registradas para deshacer");
    }
```
→ Valida que haya algo que deshacer.

---

```java
    Transaction lastTx = history.pop(); // LIFO: saca la ÚLTIMA transacción
```
**¿Qué hace?** Extrae la transacción más reciente. Si hiciste tx1, tx2, tx3 → `pop()` saca tx3.

---

```java
    EnergyNode energyNode = nodes.findById(lastTx.getNodeId());
    if (energyNode != null) {
        energyNode.setCurrentLoad(energyNode.getCurrentLoad() - lastTx.getAmount());
        System.out.println("Carga revertida con éxito en el nodo: " + lastTx.getNodeId());
    }
```
**¿Qué hace?**
1. Busca el nodo que participó en esa transacción (por su ID).
2. Le **resta** la energía que se había asignado → revierte el estado del nodo.

**Ejemplo:** Si tx3 dijo "Nodo Solar entregó 200 kWh", entonces:
```
Solar.currentLoad = 400 - 200 = 200 kWh  ← vuelve al estado anterior
```

---

## 📋 Listado del historial sin destruir la pila

```java
// Línea 197-213 de App.java
private static void listHistory(Stack<Transaction> history) {
    Stack<Transaction> aux = new Stack<>();
    while (!history.isEmpty()) {
        Transaction tx = history.pop();
        System.out.println(tx);
        aux.push(tx);      // Guarda en pila auxiliar (invierte orden)
    }
    while (!aux.isEmpty()) {
        history.push(aux.pop()); // Restaura la pila original
    }
}
```

**¿Por qué este truco con pila auxiliar?**

La `Stack` no tiene `get(i)` eficiente si quisiéramos iterar sin destruir. El truco es:
1. Vaciar la pila original a una auxiliar mientras imprimís.
2. Vaciar la auxiliar de vuelta a la original.

Al pasar por dos pilas, el orden se **invierte dos veces** → queda igual al original.

**¿Por qué no usar simplemente un `for` con `get(i)`?**
> Podríamos, pero rompería la abstracción de la pila. La pila es un TDA con operaciones definidas (push/pop/peek). Acceder por índice viola el contrato del TDA.

---

## ❓ Preguntas del profesor — Diapositiva 8

### "¿Por qué Stack y no List para el historial?"
> "Porque la operación de undo naturalmente necesita revertir la **última** operación realizada. Esto es exactamente el comportamiento LIFO. Con una List, deberíamos siempre acceder al último índice (`list.get(list.size()-1)`), lo cual es menos semántico y más propenso a errores. La Stack expresa la intención del código claramente."

### "¿Por qué push/pop son O(1)?"
> "Porque operamos siempre sobre el `head` de la lista enlazada interna. Insertar o eliminar al inicio de una lista enlazada nunca requiere recorrerla: simplemente redirigimos punteros. No importa si la pila tiene 1 o 1 millón de elementos, la operación tarda lo mismo."

### "¿Qué pasa si hago undo más veces que transacciones?"
> "La primera línea de `undoLast()` valida `history.isEmpty()` y lanza `NoSuchElementException`. El `errorHandler` de `App.java` captura esa excepción e imprime el mensaje de error sin detener el programa."

### ⚠️ PREGUNTA TRAMPA: "¿El undo revierte también el estado del consumidor?"
> "No. El undo solo revierte la `currentLoad` del nodo generador. El consumidor no tiene un campo de estado que necesite revertirse. Si el consumidor ya recibió la energía físicamente, eso es irreversible a nivel de negocio; el undo solo libera la capacidad del nodo para futuras asignaciones."

### ⚠️ PREGUNTA TRAMPA: "¿Cuál es la diferencia entre `pop()` y `peek()`?"
> "`pop()` extrae el elemento del tope y lo elimina de la pila (el historial pierde esa transacción). `peek()` solo consulta el elemento sin modificar la pila. El `undo` usa `pop()` porque efectivamente estamos borrando esa transacción del historial."

---

## 🎤 Mini discurso oral — Diapositiva 8

> *"Para el historial de transacciones utilizamos una pila, que sigue el principio LIFO: el último en entrar es el primero en salir. Cada vez que `processNext()` tiene éxito, la transacción resultante se guarda en la pila con `push()`. Cuando el usuario solicita deshacer, `undoLast()` llama a `pop()` para recuperar la última transacción, busca el nodo involucrado y le resta la energía que se había asignado, revirtiendo su estado. Elegimos Stack sobre List porque la semántica del undo es exactamente LIFO, y tanto `push` como `pop` son O(1) al operar siempre sobre el tope de la estructura. Para listar el historial sin destruirlo, usamos una pila auxiliar que nos permite recorrer los elementos y luego restaurarlos."*

---
---

# 📊 DIAPOSITIVA 9 — CSV Importer

## ¿Qué es `CSVImporter`?

Es una clase utilitaria en el paquete `util` que permite **cargar masivamente datos** (nodos o consumidores) desde archivos de texto con formato CSV, sin tener que ingresarlos manualmente uno por uno.

Tiene dos métodos estáticos:
- `importNodes(filePath, nodeManager)` — importa nodos generadores
- `importConsumers(filePath, consumerManager)` — importa consumidores

Son `static` porque no necesitan instancia de la clase: son funciones de utilidad puras.

---

## 🔍 Código línea a línea: `importConsumers()`

```java
// Línea 21-52 de CSVImporter.java
public static void importConsumers(String filePath, ConsumerManager manager) {
```
→ Recibe la ruta del archivo y el manager donde registrar los consumidores.

---

```java
    try (BufferedReader br = new BufferedReader(new FileReader(filePath))) {
```
**¿Qué hace `FileReader`?**  
Abre el archivo en disco y lo prepara para leer carácter por carácter. Es la "puerta de entrada" al archivo.

**¿Qué hace `BufferedReader`?**  
Envuelve al `FileReader` y agrega un **buffer interno de memoria**. En vez de ir al disco por cada carácter, lee un bloque grande de una vez y los sirve desde RAM. Esto hace la lectura **mucho más eficiente**.

**`try-with-resources` (el paréntesis después de `try`):**  
Garantiza que el archivo se cierra automáticamente al terminar, incluso si ocurre un error. Sin esto, el archivo quedaría abierto y ocupando recursos del sistema operativo.

---

```java
        String line;
        int lineNumber = 0;
        while ((line = br.readLine()) != null) {
            lineNumber++;
```
**¿Qué hace `readLine()`?**  
Lee una línea completa del archivo (hasta encontrar `\n`). Cuando no hay más líneas, devuelve `null` y el `while` termina.  
`lineNumber` rastrea en qué línea estamos para reportar errores con precisión.

---

```java
            if (line.trim().isEmpty()) continue;
```
**¿Qué hace?** Salta las líneas vacías (o con solo espacios). Si el CSV tiene líneas en blanco, no falla.

---

```java
            String[] values = line.split(",");
```
**¿Qué hace?** Divide la línea en partes usando la coma como separador.  
Ejemplo: `"C1,Hospital,1,500.0"` → `["C1", "Hospital", "1", "500.0"]`

---

```java
            if (values.length == 4) {
                try {
                    String id = values[0].trim();
                    String name = values[1].trim();
                    int priority = Integer.parseInt(values[2].trim());
                    double demand = Double.parseDouble(values[3].trim());
```
**¿Qué hace?**  
- Verifica que haya exactamente 4 columnas.
- Limpia espacios con `.trim()`.
- Convierte los datos numéricos: `parseInt` para entero, `parseDouble` para decimal.

---

```java
                    Consumer consumer = new Consumer(id, name, priority, demand);
                    manager.register(consumer);
                    System.out.println("Línea " + lineNumber + ": Consumidor '" + id + "' importado con éxito.");
```
**¿Qué hace?** Crea el objeto `Consumer` y lo registra usando `manager.register()`.

**Reutilización de validaciones:**  
`manager.register()` aplica las mismas validaciones que cuando el usuario registra manualmente (ID no nulo, no duplicado, etc.). Esto es **DRY: Don't Repeat Yourself**. La lógica de validación existe en un solo lugar.

---

```java
                } catch (NumberFormatException e) {
                    System.out.println("Error en línea " + lineNumber + ": Datos numéricos inválidos.");
                } catch (IllegalArgumentException e) {
                    System.out.println("Error en línea " + lineNumber + ": " + e.getMessage());
                }
```
**¿Qué hace?** Captura errores **por línea individualmente**:
- `NumberFormatException`: si "abc" estaba donde iba un número.
- `IllegalArgumentException`: si el manager rechazó el dato (ej: ID duplicado).

**Tolerancia a fallos:** si la línea 3 tiene error, el programa imprime el error y continúa con la línea 4. El archivo entero no falla por una línea mala.

---

```java
            } else {
                System.out.println("Error en línea " + lineNumber + ": Formato de columnas inválido.");
            }
```
→ Si la línea no tiene exactamente 4 columnas, se reporta el error y se continúa.

---

```java
        } catch (IOException e) {
            System.out.println("Error crítico al leer el archivo: " + e.getMessage());
        }
```
**¿Qué hace?** Captura errores de **acceso al archivo**: ruta incorrecta, permisos insuficientes, disco dañado. Este es el único error que detiene toda la importación (porque sin acceso al archivo, no hay nada que hacer).

---

## 📐 Diferencia entre los tres niveles de error

| Error | Tipo | Consecuencia |
|-------|------|-------------|
| Archivo no encontrado | `IOException` | Se detiene TODA la importación |
| Número mal formado | `NumberFormatException` | Solo falla ESA línea |
| Validación de negocio | `IllegalArgumentException` | Solo falla ESA línea |
| Columnas incorrectas | `else` del `if` | Solo falla ESA línea |

---

## ❓ Preguntas del profesor — Diapositiva 9

### "¿Por qué reutilizar las validaciones del manager?"
> "Para evitar duplicar lógica. Si las validaciones estuviesen en el CSV importer también, y luego cambiamos las reglas de negocio (por ejemplo, cambiar el rango de prioridad), tendríamos que actualizar el código en dos lugares. Al delegar a `manager.register()`, hay un único punto de verdad para las reglas de validación."

### "¿Qué pasa si el archivo CSV tiene la cabecera como primera línea?"
> "En esta implementación, la cabecera sería tratada como datos y fallaría en la conversión numérica, generando `NumberFormatException`. Una mejora sería saltar la primera línea o detectar si los valores son texto donde se espera número."

### "¿Por qué usar `BufferedReader` y no leer el archivo completo a memoria?"
> "Para archivos grandes, cargar todo el contenido en un `String` consumiría mucha memoria. `BufferedReader` lee línea por línea, lo que permite procesar archivos muy grandes sin problemas de memoria. Además, el buffer interno hace que sea más eficiente que leer carácter por carácter."

### ⚠️ PREGUNTA TRAMPA: "¿Qué pasa si el CSV tiene punto y coma en lugar de coma?"
> "El `split(',')` no funcionaría y cada línea sería tratada como una sola columna, cayendo en el `else` de columnas inválidas. Habría que parametrizar el delimitador o detectarlo automáticamente. Es una limitación conocida de esta implementación."

### ⚠️ PREGUNTA TRAMPA: "¿Es el método `importNodes` igual a `importConsumers`?"
> "La estructura es idéntica, solo cambian los campos que se parsean y el tipo de objeto que se crea. Esto podría mejorarse con programación genérica o con el patrón Strategy para evitar la duplicación entre los dos métodos. Para este proyecto, la claridad y simplicidad fueron priorizadas sobre la abstracción máxima."

---

## 🎤 Mini discurso oral — Diapositiva 9

> *"En esta parte implementamos un importador CSV que permite cargar datos masivamente desde archivos externos. El sistema abre el archivo usando `BufferedReader`, que lee línea por línea de manera eficiente usando un buffer en memoria. Para cada línea, divide los valores por coma, valida que tenga el formato correcto, convierte los tipos de datos y registra el objeto usando el mismo método `register()` del manager correspondiente. La clave del diseño es la tolerancia a fallos: si una línea tiene error, se reporta y se salta, pero las demás líneas se procesan normalmente. Al reutilizar las validaciones del manager, garantizamos que los datos importados pasen las mismas reglas que los ingresados manualmente, sin duplicar lógica."*

---
---

# 🧠 Resumen de Decisiones Técnicas para Justificar

| Decisión | Justificación |
|----------|--------------|
| Cola FIFO | Equidad: primer llegado, primer servido. Simple y correcto para este dominio. |
| `IllegalStateException` | Representa error de estado del sistema, no de programación. Semánticamente correcto. |
| Entrega atómica | Simplifica la contabilidad. Un nodo = una transacción = un posible undo. |
| Stack para historial | Undo requiere LIFO naturalmente. Push/pop O(1). |
| Pila auxiliar para listar | Respeta el contrato del TDA sin romper la abstracción. |
| BufferedReader | Eficiencia de I/O. Procesa archivos grandes sin cargar todo en memoria. |
| Reutilizar `register()` | DRY principle. Un único punto de validación. |
| `try-with-resources` | Cierra el archivo automáticamente. Previene resource leaks. |
| `errorHandler` en App.java | Centraliza el manejo de errores. El programa nunca se detiene por una operación fallida. |

---

# ⚡ Errores Comunes a Evitar en la Defensa

1. **No confundir FIFO con LIFO.** Cola = FIFO (primero en entrar, primero en salir). Pila = LIFO (último en entrar, primero en salir).
2. **No decir "elimina el consumidor"** cuando hacés undo. El undo solo revierte la `currentLoad` del nodo.
3. **No confundir `pop()` con `peek()`.** Pop destruye, peek solo mira.
4. **No decir que `BufferedReader` lee todo el archivo.** Lee línea por línea con buffer.
5. **No confundir `IOException` con `NumberFormatException`.** Son tipos de error completamente distintos.
