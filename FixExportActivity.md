# ExportActivity — Análisis y Diagnóstico TCP

**Fecha:** 2026-04-18  
**Proyecto:** `C:\Desarrollo\Addalia\RepositoriosSvn\ExportActivity`

---

## ¿Qué hace el proyecto?

**ExportActivityFactory** es una aplicación web Java (WAR) que automatiza la migración de documentos entre dos sistemas de gestión documental (DMS) de Addalia.

Ejecuta un **job periódico** (Quartz, cada 5 segundos) que:
1. Se conecta a **DMS/WMS Factory V5** (sistema origen)
2. Obtiene items de trabajo pendientes del flujo ("Exportacion")
3. Por cada item: crea el documento en **DMS V6** (sistema destino), sube los archivos, actualiza atributos y libera el item en el WMS

### Stack
- Java 1.8 / Maven / WAR
- **Quartz Scheduler** para la ejecución periódica
- **Jersey REST** para las llamadas HTTP a DMS/WMS
- **Log4j** para logging

### Sistemas externos

| Sistema | Rol |
|---|---|
| DMS Factory V5 | Origen de documentos |
| WMS Factory (Atril) | Workflow — entrega y liberación de items |
| DMS V6 | Destino de la migración |

---

## Problema: Puertos TCP Idle (pid 0) en Tomcat 9

### Síntoma
Durante la ejecución en Tomcat 9 para Windows, se abren muchos puertos TCP que quedan en estado **Idle con pid 0**.

### Causa raíz
En Jersey, aunque se llame a `readEntity()`, el objeto `Response` mantiene una referencia al pool de conexiones HTTP hasta que se llama explícitamente a `.close()`. Sin ese cierre, la conexión queda en estado **TIME_WAIT/Idle** — exactamente lo que aparece con `pid 0`.

Como el job Quartz corre cada 5 segundos, las conexiones se acumulan continuamente.

---

## Problemas encontrados

### 1. `ExportUtils.java` — el más grave (recursivo)

- Líneas ~49, 74, 110, 116: cuatro objetos `Response` que se usan con `readEntity()` pero **nunca se cierran**
- El método `getDocTree()` es **recursivo** — por cada documento hijo se abre un nuevo `Response`. Con jerarquías profundas esto escala exponencialmente:
  - 1 padre con 10 hijos = 11 Responses sin cerrar
  - Esos 10 hijos con 5 hijos cada uno = 61 Responses sin cerrar

### 2. `ExportJob.java` — InputStream sin cerrar

- Líneas ~158-163: se obtiene un `InputStream` del response y se pasa a `uploadItem()` pero nunca se cierra explícitamente

### 3. `ClientsRest.java` — login/logout sin cerrar

- Las respuestas de `logIn()` y `logOut()` (líneas ~72, 78, 85, 96-105) nunca se cierran — ocurre en **cada ejecución del job**

---

## Tabla de prioridad de corrección

| Archivo | Líneas | Impacto |
|---|---|---|
| `ExportUtils.java` | ~49, 74, 110, 116 | **Crítico** (recursivo) |
| `ClientsRest.java` | ~72, 78, 85, 96-105 | **Alto** (cada ejecución) |
| `ExportJob.java` | ~158-163 | **Medio** |

---

## Fix requerido

El patrón correcto es usar `try-with-resources` ya que `Response` en Jersey 2.x implementa `Closeable`:

```java
// ANTES (con fuga)
Response resp = dms.getDocument(id);
SomeType obj = resp.readEntity(SomeType.class);
// resp nunca se cierra → fuga de conexión TCP

// DESPUÉS (correcto con try-with-resources)
try (Response resp = dms.getDocument(id)) {
    SomeType obj = resp.readEntity(SomeType.class);
    // usar obj...
}

// ALTERNATIVA con try-finally
Response resp = dms.getDocument(id);
try {
    SomeType obj = resp.readEntity(SomeType.class);
    // usar obj...
} finally {
    resp.close();
}
```

Aplicar este patrón a **todos** los objetos `Response` en `ExportUtils.java`, `ExportJob.java` y `ClientsRest.java`.
