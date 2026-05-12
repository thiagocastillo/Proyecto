# Proyecto
# Defensa Integral del Código: Proyecto EcoGrid

## 1. Estructura de Abstracción (Paquete `com.ecogrid.tda`)
El proyecto sigue el principio de "Programación orientada a Interfaces".
*   **Interfaces (TDAList, TDAQueue, TDAStack):** Definen el "qué" debe hacer cada estructura (contrato). Esto asegura que, si en el futuro queremos cambiar la `LinkedList` por una `DoubleLinkedList`, el resto del código no se rompería.
*   **Implementaciones (LinkedList, Queue, Stack):** Son el "cómo". Aquí se gestionan los punteros y nodos reales.

---

## 2. Modelos de Dominio (Entidades Reales)
Son clases puras de datos que representan los componentes físicos:
*   **EnergyNode:** Representa una microgrid (ID, tipo de fuente, capacidad máxima y carga actual).
*   **Consumer:** Representa un sector de la ciudad (ID, nombre, prioridad y demanda requerida).
*   **EnergyRequest:** El "ticket" de solicitud que entra a la cola.
*   **Transaction:** El recibo que se genera cuando una solicitud es procesada con éxito.

---

## 3. Lógica de los Gestores (Controllers)

### **NodeManager.java**
*   **`register(EnergyNode)`:** No solo guarda; realiza una **validación de unicidad**. Recorre la lista buscando si el ID ya existe antes de añadirlo.
*   **`undoLast(nodes, history)`:** Conecta dos estructuras. Saca la última transacción de la Pila (`history.pop()`) y actualiza el estado del nodo correspondiente, restándole la carga que se le había sumado.

### **ConsumerManager.java**
*   **`removeById(id)`:** Utiliza el método `find` de nuestra lista enlazada para localizar al consumidor y luego el método `remove` para extraerlo de la cadena de nodos.

### **RequestQueue.java**
*   **`processNext(NodeManager)`:** Es la función crítica. Implementa una **Búsqueda con Restricción**. Recorre los nodos hasta encontrar el primero que tenga `Capacidad >= Carga_Actual + Demanda`. Si no lo encuentra, lanza una `IllegalStateException`.

---

## 4. Utilidades y Punto de Entrada

### **CSVImporter.java (Carga Masiva)**
*   **`importNodes` / `importConsumers`:** Utiliza `BufferedReader` y `String.split(",")`. 
*   **Lógica de parseo:** Convierte las cadenas de texto del CSV a tipos `Double` e `Integer` para crear los objetos. Si una línea está mal formateada, el sistema la ignora o lanza un error controlado.

### **App.java (Orquestador)**
*   **`errorHandler`:** Es una función de orden superior (`Consumer<Runnable>`). **Explicación técnica:** Centraliza todos los `try-catch` del menú en un solo lugar, manteniendo el código limpio y evitando que el programa se cierre ante un error del usuario.
*   **`listHistory`:** Para mostrar la pila sin destruirla, utiliza una **Pila Auxiliar**. Mueve los elementos a la auxiliar (invirtiéndolos), los imprime, y luego los devuelve a la original para mantener el orden LIFO intacto.

---

## 5. Resumen de Flujo de Datos (100% de la lógica)
1.  **Entrada:** Los datos entran por consola (`register`) o por archivo (`CSVImporter`).
2.  **Almacenamiento:** Se guardan en `LinkedList` dentro de los Managers.
3.  **Procesamiento:** El usuario crea una `EnergyRequest` que entra a la `RequestQueue` (Cola).
4.  **Ejecución:** Al procesar, se busca un nodo en el `NodeManager`, se actualiza su estado y se guarda la `Transaction` en el `Stack` (Pila).
5.  **Reversión:** Si hay un error, se usa el `Stack` para volver al estado anterior.

---

## 6. Por qué este código es de "Calidad Profesional"
1.  **Modularidad:** Cada clase tiene una sola responsabilidad (Solid - SRP).
2.  **Validaciones:** El código nunca asume que el usuario ingresará datos correctos.
3.  **Independencia:** Al crear nuestros propios TDAs, el proyecto es totalmente portable y educativo.
4.  **Manejo de Memoria:** Al usar Listas Enlazadas en lugar de Arreglos, el sistema es eficiente incluso si la ciudad crece a miles de nodos.
