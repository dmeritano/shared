# Connection Pooling HTTP en Jersey — ¿tiene sentido en este proyecto?

**Pregunta:** ¿Existe alguna forma de usar/configurar un pool de conexiones HTTP en Jersey para no estar abriendo y cerrando conexiones todo el tiempo? ¿O no tiene sentido en este proyecto?

---

## Cómo funciona Jersey con conexiones

Jersey soporta tres conectores HTTP con distintas capacidades de pooling:

| Conector | Pool | Configuración |
|----------|------|---------------|
| `HttpUrlConnection` (default) | No real | Ninguna |
| **Apache HTTP Client** (`jersey-apache-connector`) | Sí, completo | `PoolingHttpClientConnectionManager` |
| Grizzly | Sí | Más complejo |

Con el conector Apache se configuraría así:

```java
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(20);
cm.setDefaultMaxPerRoute(5);

ClientConfig config = new ClientConfig()
    .connectorProvider(new ApacheConnectorProvider())
    .property(ApacheClientProperties.CONNECTION_MANAGER, cm);

Client client = ClientBuilder.newClient(config);
```

---

## El problema específico de este proyecto

El obstáculo real es que `DMSRestClient` y `WMSRestClient` provienen de la librería **`atril-core`** (caja negra). No se controla cómo crean su `javax.ws.rs.client.Client` interno, y probablemente no aceptan inyección de un `ConnectionManager` externo.

Configurar un pool explícito en Jersey requeriría modificar `atril-core` o que esa librería ya lo soporte de alguna forma.

---

## La solución equivalente al pool que sí está disponible

La raíz del problema de TIME_WAIT no es la ausencia de pool — es que **se crea un cliente nuevo cada 5 segundos y el viejo nunca se cierra**. Un `javax.ws.rs.client.Client` de Jersey ya reutiliza conexiones HTTP keep-alive por defecto cuando se mantiene viva la misma instancia.

La solución equivalente a un pool, sin tocar `atril-core`, es hacer el `ClientsRest` **singleton**: crearlo una sola vez en el arranque y reutilizarlo en todas las ejecuciones del job.

```
ServiceConfiguration.init()
    └── crea ClientsRest una sola vez → se guarda como campo del scheduler/job

ExportJob.execute()  (cada 5 s)
    └── usa el ClientsRest ya existente → reutiliza los mismos Jersey Clients
    └── gestiona solo la sesión (re-login si expira)
```

Esto consigue exactamente el beneficio de un pool: misma instancia de cliente, conexiones TCP reutilizadas vía keep-alive, sin TIME_WAIT por destrucción y recreación constante.

---

## ¿Tiene sentido un pool explícito en este proyecto?

Solo tendría sentido si el job procesara documentos **en paralelo** (varios hilos simultáneos). En este proyecto el job está anotado con `@DisallowConcurrentExecution` — es estrictamente secuencial. Con un único hilo trabajando, la ventaja de un pool con múltiples conexiones simultáneas es nula.

---

## Recomendación

| Prioridad | Acción |
|-----------|--------|
| **Corto plazo** | Aplicar Issues #1 y #2 del diagnóstico (`Issues.md`): cerrar clientes y respuestas de login correctamente. Resuelve el problema de raíz. |
| **Mejora posterior** | Mover la creación de `ClientsRest` al `init()` del servlet para reutilizar instancias entre ejecuciones, con lógica de re-login si la sesión del DMS/WMS expira. Elimina completamente la apertura/cierre de conexiones por ciclo. |
| **Pool explícito (Apache Connector)** | Solo valdría la pena si en el futuro se paraleliza el procesamiento de work items, y únicamente si `atril-core` permite inyectar un `ConnectionManager` externo. |
