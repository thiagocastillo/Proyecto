# Proyecto
# Documentación Técnica y Guía de Defensa: Proyecto EcoGrid

## 1. Introducción al Sistema
**EcoGrid** es una solución de software diseñada para la gestión inteligente de micro-redes de energía renovable. El sistema permite administrar nodos de generación (solar, eólica, etc.) y equilibrar la demanda de diversos sectores de la ciudad (consumidores) mediante una arquitectura robusta basada en estructuras de datos personalizadas.

---

## 2. Arquitectura de Datos (TDAs Propios)
Una de las mayores fortalezas del proyecto es que no depende de las colecciones estándar de Java (`ArrayList`, `HashMap`, etc.), sino que utiliza **Tipos de Datos Abstractos (TDA)** implementados desde cero.

### **LinkedList<T>** (Lista Enlazada)
*   **Función:** Almacena de forma dinámica los nodos de energía y los consumidores.
*   **Cómo funciona:** Utiliza nodos enlazados mediante referencias. Esto permite insertar y eliminar elementos sin necesidad de redimensionar arreglos, optimizando el uso de memoria.
*   **Uso clave:** Es la base de `NodeManager` y `ConsumerManager`.

### **Queue<T>** (Cola)
*   **Función:** Gestiona el orden de atención de las solicitudes de energía.
*   **Lógica:** Sigue el principio **FIFO** (First-In, First-Out). Garantiza que las peticiones se procesen estrictamente en el orden en que fueron recibidas.

### **Stack<T>** (Pila)
*   **Función:** Almacena el historial de transacciones para permitir la reversión de cambios.
*   **Lógica:** Sigue el principio **LIFO** (Last-In, First-Out). El último proceso realizado es el primero en ser deshecho.

---

## 3. Desglose de Componentes y Funciones Clave

### A. Gestión de Recursos (`NodeManager` y `ConsumerManager`)
Estas clases actúan como controladores de las entidades del sistema.

*   **`register(Entity e)`**: 
    *   *Funcionamiento:* Valida que el objeto no sea nulo, que el ID sea único y que los valores (como capacidad máxima) sean lógicos.
    *   *Importancia:* Evita la entrada de "datos basura" al sistema, manteniendo la integridad de la red eléctrica.
*   **`findById(String id)`**: 
    *   *Funcionamiento:* Implementa una búsqueda lineal sobre la `LinkedList`. 
    *   *Importancia:* Permite localizar cualquier punto de la red rápidamente para consultar su estado actual.

### B. El Motor de Procesamiento (`RequestQueue`)
Es el cerebro que decide cómo se distribuye la energía.

*   **`processNext(NodeManager nodes)`**:
    *   *Lógica del Algoritmo:* 
        1. Extrae la solicitud al frente de la cola (`dequeue`).
        2. Itera sobre la lista de nodos disponibles.
        3. Verifica la condición: `Carga Actual + Demanda Solicitada <= Capacidad Máxima`.
        4. Si el nodo cumple, actualiza su carga y genera una `Transaction`.
    *   *Importancia:* Este algoritmo previene la sobrecarga de los nodos, evitando fallos en la infraestructura física de la microgrid.

### C. Sistema de Historial y Reversión (`App` y `NodeManager`)
*   **`undoLast(NodeManager nodes, Stack<Transaction> history)`**:
    *   *Funcionamiento:* Extrae la última transacción de la Pila, busca el nodo afectado y le resta la energía que se le había asignado.
    *   *Importancia:* Proporciona una capa de seguridad operativa, permitiendo corregir errores de asignación sin afectar la consistencia de los datos.

### D. Persistencia y Carga Masiva (`CSVImporter`)
*   **`importNodes(String path, NodeManager nm)`**:
    *   *Funcionamiento:* Lee archivos de texto separados por comas, parsea cada línea y crea objetos `EnergyNode`.
    *   *Importancia:* Permite que el sistema sea escalable. En lugar de registrar nodos manualmente, podemos cargar toda una infraestructura de ciudad en segundos.

---

## 4. Estrategia de Defensa (Preguntas y Respuestas)

| Pregunta Probable | Respuesta Recomendada |
| :--- | :--- |
| **¿Por qué usar estructuras propias?** | "Para demostrar el control total sobre la complejidad algorítmica y evitar el overhead de las librerías genéricas de Java, adaptando la memoria exactamente a nuestra necesidad." |
| **¿Qué pasa si no hay energía suficiente?** | "El método `processNext` lanza una excepción controlada, informando que no hay capacidad disponible en ningún nodo, protegiendo así la estabilidad de la red." |
| **¿Cómo garantizas que no haya IDs repetidos?** | "Mediante una validación en el método `register`, que realiza una búsqueda previa en la lista antes de permitir la inserción del nuevo elemento." |
| **¿Cuál es la ventaja de la Pila en el historial?** | "La naturaleza LIFO de la Pila es perfecta para funciones de 'Deshacer', ya que siempre queremos revertir la acción más reciente." |

---
