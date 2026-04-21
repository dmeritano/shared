# Issues — ExportActivityFactory: Fuga de puertos TCP

**Fecha de análisis:** 2026-04-21  
**Síntoma reportado:** Al ejecutarse el servicio, crece exponencialmente la cantidad de puertos TCP abiertos en el stack de Windows. Los puertos aparecen como IDLE con PID 0.

---

## Contexto técnico: por qué aparece PID 0 / IDLE en Windows

Cuando Java libera el file descriptor de un socket, el kernel de Windows mantiene la conexión en estado `TIME_WAIT` durante aproximadamente **4 minutos** (valor por defecto del parámetro de registro `TcpTimedWaitDelay` = 240 s). Este mecanismo existe para evitar que paquetes tardíos de una conexión anterior sean interpretados por una nueva conexión en el mismo par de puertos.

Durante ese período de TIME_WAIT, la conexión ya no pertenece a ningún proceso → aparece como **PID 0 / IDLE** en herramientas como `netstat` o TCPView.

El servicio corre un cron Quartz cada **5 segundos**. Si los clientes HTTP no se destruyen correctamente, cada ejecución deja conexiones en TIME_WAIT que se acumulan durante 240 s antes de liberarse:

```
240 s de TIME_WAIT / 5 s de intervalo = 48 ciclos conviviendo en paralelo
× 3 clientes Jersey por ciclo
= 144 instancias de clientes vivas simultáneamente (como mínimo)
```

Esto produce el crecimiento exponencial de puertos que observa sistemas.

---

## Issue #1 — CRÍTICO: Los clientes Jersey nunca se destruyen

### Archivos afectados
- `src/main/java/com/addalia/ExportActivity/util/ClientsRest.java` — líneas 40–91

### Descripción

Cada ejecución del job crea tres clientes REST nuevos en `clientsLogIn()`:

```java
dmsRestClient    = new DMSRestClient(configurationModel.getFactoryRestUrlDms());
wmsRestClient    = new WMSRestClient(configurationModel.getFactoryRestUrlAtril());
dmsRestClient_V6 = new DMSRestClient(configurationModel.getV6ConnectTo());
```

Cada uno de estos objetos contiene internamente una instancia de `javax.ws.rs.client.Client` (Jersey), que gestiona su propio **connection pool TCP**.

En `clientsLogOut()` se llama al endpoint de logout HTTP (cierra la sesión en el servidor) y se cierra la respuesta, pero **en ningún momento se llama a `close()` sobre el objeto Jersey `Client` interno**. El pool de conexiones de cada cliente queda vivo hasta que el garbage collector lo recoja, lo cual en la JVM no está garantizado que ocurra de forma inmediata ni predecible.

Agravante: el método `closeInternalClient()` (línea 110) fue escrito específicamente para resolver este problema mediante reflexión, pero **no se llama en ningún punto del código**. Está completamente muerto.

```java
// ClientsRest.java — método closeInternalClient existe pero nunca se invoca
private void closeInternalClient(Object restClient) {
    // ... código correcto que llama a close() por reflexión ...
}

// clientsLogOut() — termina sin llamar a closeInternalClient()
LOGGER.info("REST: Cleanup process completed.");  // línea 90 — aquí debería cerrarse
```

### Acción recomendada

Llamar a `closeInternalClient()` al final de `clientsLogOut()`, después de haber ejecutado los logouts HTTP:

```java
public void clientsLogOut() {
    // ... bloque existente de logouts HTTP (DMS V5, WMS, DMS V6) ...

    // AÑADIR: destruir los clientes Jersey y liberar sus connection pools
    closeInternalClient(dmsRestClient);
    closeInternalClient(wmsRestClient);
    closeInternalClient(dmsRestClient_V6);

    LOGGER.info("REST: Cleanup process completed.");
}
```

Si `DMSRestClient` y `WMSRestClient` implementan `java.io.Closeable` o exponen un método `close()` público (verificar en `atril-core`), es preferible un cast explícito en lugar de reflexión para evitar el silenciado de errores:

```java
// Alternativa más robusta si los clientes son Closeable
private void closeInternalClient(Object restClient) {
    if (restClient == null) return;
    if (restClient instanceof AutoCloseable) {
        try {
            ((AutoCloseable) restClient).close();
        } catch (Exception e) {
            LOGGER.warn("Error cerrando cliente REST: " + e.getMessage());
        }
    }
}
```

---

## Issue #2 — ALTO: Las respuestas de login nunca se cierran

### Archivos afectados
- `src/main/java/com/addalia/ExportActivity/util/ClientsRest.java` — líneas 42–52

### Descripción

Las tres respuestas obtenidas en `clientsLogIn()` se verifican con `checkStatusResponseWithException()` pero **nunca se cierran**:

```java
Response resp1 = dmsRestClient.logIn(...);
checkStatusResponseWithException(resp1, "Login fail in DMS");
// resp1 queda abierta — comentario incorrecto: "dejamos que viva durante el proceso"

Response resp2 = wmsRestClient.logIn(...);
checkStatusResponseWithException(resp2, "Login fail in WMS");
// resp2 queda abierta

Response resp3 = dmsRestClient_V6.logIn(...);
checkStatusResponseWithException(resp3, "Login fail in DMS V6");
// resp3 queda abierta
```

El comentario `"No cerramos resp1 aquí, dejamos que viva durante el proceso"` es incorrecto. `checkStatusResponseWithException()` únicamente llama a `response.getStatus()` — no lee el entity body ni consume el stream. La conexión TCP subyacente queda en estado "prestada" del pool y no se devuelve hasta que la respuesta se cierre explícitamente.

Esto produce **3 fugas de conexión por ejecución del job**, es decir, 36 conexiones por minuto que no regresan al pool.

### Acción recomendada

Cerrar cada respuesta inmediatamente después de verificar el status:

```java
public void clientsLogIn() throws InvalidCfgException {
    dmsRestClient = new DMSRestClient(configurationModel.getFactoryRestUrlDms());
    Response resp1 = dmsRestClient.logIn(configurationModel.getFactoryUser(), configurationModel.getFactoryPassword());
    try {
        checkStatusResponseWithException(resp1, "Login fail in DMS");
    } finally {
        if (resp1 != null) resp1.close();
    }

    wmsRestClient = new WMSRestClient(configurationModel.getFactoryRestUrlAtril());
    Response resp2 = wmsRestClient.logIn(configurationModel.getFactoryUser(), configurationModel.getFactoryPassword(), configurationModel.getFactoryConnectTo());
    try {
        checkStatusResponseWithException(resp2, "Login fail in WMS");
    } finally {
        if (resp2 != null) resp2.close();
    }

    dmsRestClient_V6 = new DMSRestClient(configurationModel.getV6ConnectTo());
    Response resp3 = dmsRestClient_V6.logIn(configurationModel.getV6User(), configurationModel.getV6Password());
    try {
        checkStatusResponseWithException(resp3, "Login fail in DMS V6");
    } finally {
        if (resp3 != null) resp3.close();
    }
}
```

---

## Issue #3 — MEDIO: `getDocTree` se llama de forma redundante dentro de `doExport`

### Archivos afectados
- `src/main/java/com/addalia/ExportActivity/job/ExportJob.java` — línea 152
- `src/main/java/com/addalia/ExportActivity/job/ExportUtils.java` — líneas 104–127

### Descripción

`getDocTree()` en `ExportUtils` es **recursiva**: cuando se llama con el documento raíz, recorre toda la jerarquía y devuelve la lista aplanada de todos los descendientes.

Sin embargo, `doExport()` en `ExportJob` llama a `getDocTree()` para obtener los hijos directos, y luego por cada elemento de la lista vuelve a llamar a `doExport()`, que a su vez llama a `getDocTree()` de nuevo:

```
doExport(Raíz)
 └─ getDocTree(Raíz) → devuelve [A, B, A1, A2]     ← N llamadas HTTP
     ├─ doExport(A)
     │    └─ getDocTree(A) → devuelve [A1, A2]       ← llamadas HTTP duplicadas
     ├─ doExport(B)
     │    └─ getDocTree(B) → devuelve []              ← llamada HTTP duplicada
     ├─ doExport(A1)
     │    └─ getDocTree(A1) → devuelve []             ← llamada HTTP duplicada
     └─ doExport(A2)
          └─ getDocTree(A2) → devuelve []             ← llamada HTTP duplicada
```

Cada llamada HTTP abre una conexión TCP que entra en TIME_WAIT al cerrarse. El número de llamadas redundantes crece con la profundidad y el ancho del árbol documental.

Adicionalmente, la lógica tiene un problema de corrección: `getDocTree(Raíz)` devuelve **todos los descendientes aplanados** (A, B, A1, A2), no sólo los hijos directos. Al iterar sobre esa lista y llamar a `doExport` para cada elemento, los nodos intermedios (A) se exportan correctamente, pero los nodos hoja A1 y A2 se exportan dos veces: una como hijos de A (dentro de `doExport(A)`) y otra directamente como hijos de Raíz (en la iteración del nivel superior).

### Acción recomendada

Modificar `getDocTree` para que devuelva **solo los hijos directos** (un nivel) y dejar que la recursividad la gestione `doExport`:

```java
// ExportUtils.java — versión corregida: solo hijos directos, sin recursión interna
protected static List<DocModel> getDirectChildren(DocModel parent, DMSRestClient dms) throws ExportException {
    List<DocModel> children = new ArrayList<>();
    try (Response childrenResp = dms.getDocumentChildren(parent.getAttributes().get(DOCUMENT_ID_ATTRIBUTE))) {
        checkResponse(childrenResp, "Error getting children");
        DocListModel docListChild = childrenResp.readEntity(DocListModel.class);
        for (SimpleDocModel doc : docListChild.getDocs()) {
            try (Response docResp = dms.getDocument(doc.getId())) {
                checkResponse(docResp, "Error getting document " + doc.getId());
                children.add(docResp.readEntity(DocModel.class));
            }
        }
    } catch (Exception e) {
        throw new ExportException("Fallo al obtener hijos directos: " + e.getMessage());
    }
    return children;
}
```

```java
// ExportJob.java — doExport usa getDirectChildren en lugar de getDocTree
private void doExport(DocModel currentDoc, String idPadreV6) throws Exception {
    String idDocV5 = currentDoc.getAttributes().get(ATTRIBUTE_ID);
    try (Response createDoc = dms_v6.createDocument(idPadreV6, currentDoc)) {
        checkResponse(createDoc, "Create document in V6");
        DocModel docResV6 = createDoc.readEntity(DocModel.class);
        String idV6Creado = docResV6.getAttributes().get(ATTRIBUTE_ID);
        if (workItemIdV6 == null) workItemIdV6 = idV6Creado;

        List<DocModel> hijos = getDirectChildren(currentDoc, dms);  // solo hijos directos

        if (hijos.isEmpty()) {
            try (Response getItemResp = dms.getItem(idDocV5)) {
                checkResponse(getItemResp, "Get item from V5");
                try (InputStream itemStream = getItemResp.readEntity(InputStream.class)) {
                    try (Response uploadItem = dms_v6.uploadItem(idV6Creado, getItemResp.getMediaType().toString(), itemStream)) {
                        checkResponse(uploadItem, "Upload item to V6");
                    }
                }
            }
        } else {
            for (DocModel hijo : hijos) {
                doExport(hijo, idV6Creado);  // la recursión baja un nivel
            }
        }
    }
}
```

---

## Resumen y prioridad de acción

| # | Problema | Archivo(s) | Severidad | Impacto en puertos TCP |
|---|----------|-----------|-----------|------------------------|
| 1 | Jersey Clients sin `close()` — `closeInternalClient()` nunca se llama | `ClientsRest.java:55-91` | **CRÍTICO** | Causa principal del crecimiento exponencial |
| 2 | Respuestas de login sin cerrar (resp1, resp2, resp3) | `ClientsRest.java:42-52` | **ALTO** | +3 conexiones por ejecución, 36/minuto |
| 3 | `getDocTree` llamado redundantemente en bucle recursivo | `ExportJob.java:152`, `ExportUtils.java:96` | **MEDIO** | Multiplicador según profundidad del árbol |

Resolviendo el Issue #1 debería desaparecer el crecimiento exponencial. Los Issues #2 y #3 deben corregirse igualmente para evitar que el problema reaparezca a menor escala bajo carga alta o con expedientes con muchos documentos hijos.

---

## Verificación post-corrección

Tras desplegar los cambios, monitorizar con:

```
netstat -ano | findstr :8080 | find /c "TIME_WAIT"
```

O con TCPView (Sysinternals), filtrando por el proceso Tomcat (`java.exe`). El número de conexiones en TIME_WAIT debe estabilizarse en un valor bajo y constante en lugar de crecer con el tiempo.
