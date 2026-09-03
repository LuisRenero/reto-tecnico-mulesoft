# TConectaApp

API de integración desarrollada con **MuleSoft Anypoint Studio**, orientada al registro, persistencia y distribución de solicitudes mediante una API REST.

La solución implementa la recepción de solicitudes HTTP, validación de información, persistencia en MySQL, generación de archivos JSON, envío mediante SFTP y consulta de solicitudes previamente registradas.

---

## Descripción

**TConectaApp** expone una API REST para la creación y consulta de solicitudes.

El flujo de creación permite:

1. Recibir una solicitud HTTP.
2. Validar la información recibida.
3. Generar y propagar un `correlationId` para trazabilidad.
4. Persistir la solicitud en MySQL.
5. Obtener el identificador generado por la base de datos.
6. Generar un archivo JSON asociado a la solicitud.
7. Almacenar el archivo localmente.
8. Enviar el archivo mediante SFTP.
9. Responder al consumidor con el resultado de la operación.

La consulta permite recuperar una solicitud por su identificador y validar la existencia de su archivo asociado.

---

## Tecnologías

| Tecnología      | Versión                          |
| --------------- | -------------------------------- |
| Java            | 17                               |
| Mule Runtime    | 4.12.2                           |
| Anypoint Studio | 7.28.x                           |
| DataWeave       | 2.0                              |
| MySQL           | 8.1                              |
| SFTP            | OpenSSH                          |
| RAML            | RAML 1.0                         |


---

## Funcionalidades

### Crear solicitud

```http
POST /api/v1/solicitudes
```

Permite registrar una nueva solicitud.

Ejemplo:

```json
{
  "clienteId": 12345,
  "tipo": "ALTA",
  "descripcion": "Solicitud de prueba"
}
```

Respuesta exitosa:

```http
HTTP/1.1 201 Created
```

```json
{
    "status": "SUCCESS",
    "mensaje": "Solicitud procesada y almacenada correctamente",
    "idSolicitud": 49,
    "idCorrelacion": "ac1f0c50-a7d3-11f1-9a2f-fa3dc6d9953e",
    "clienteId": "101",
    "tipo": "ALTA",
    "descripcion": "Solicitud de alta",
    "estatus": "PROCESADO",
    "fechaCreacion": "2026-09-03T14:11:42.4770033-06:00"
}
```

---

### Consultar solicitud

```http
GET /api/v1/solicitudes/{id}
```

Ejemplo:

```http
GET /api/v1/solicitudes/31
```

Respuesta exitosa:

```http
HTTP/1.1 200 OK
```

```json
{
    "status": "SUCCESS",
    "mensaje": "Solicitud consultada correctamente",
    "idSolicitud": 39,
    "clienteId": 101,
    "tipo": "ALTA",
    "descripcion": "Solicitud de alta",
    "fechaCreacion": "2026-09-03T11:02:06",
    "archivoLocal": {
        "existe": true,
        "nombre": "solicitud-39.json"
    }
}
```

---

El archivo generado utiliza el identificador de la solicitud:

```text
solicitud-{id}.json
```

Ejemplo:

```text
solicitud-31.json
```

---

## Base de datos

La solución utiliza MySQL.

Base de datos:

```text
reto_tconecta
```

Tabla:

```text
solicitudes
```

Estructura principal:

| Campo            | Tipo         | Descripción                   |
| ---------------- | ------------ | ----------------------------- |
| `id`             | BIGINT       | Identificador autogenerado    |
| `cliente_id`     | BIGINT       | Identificador del cliente     |
| `tipo`           | VARCHAR(50)  | Tipo de solicitud             |
| `descripcion`    | VARCHAR(500) | Descripción                   |
| `fecha_registro` | TIMESTAMP    | Fecha de registro             |
| `correlation_id` | VARCHAR(100) | Identificador de trazabilidad |
| `estado`         | VARCHAR(100) | Estado de registro            |

La fecha de registro es generada automáticamente por MySQL mediante `CURRENT_TIMESTAMP`.

---

## Configuración

Las propiedades de conexión se mantienen fuera del código fuente.

Ejemplo:

```yaml
db:
  host: "localhost"
  port: 3306
  user: "CHANGE_ME"
  password: "CHANGE_ME"

sftp:
  host: "localhost"
  port: 22
  user: "CHANGE_ME"
  password: "CHANGE_ME"
```

> **Nota:** Las credenciales reales no deben almacenarse en el repositorio.

```text
src/main/resources/config.yaml
```

y mantener la configuración real fuera del control de versiones.

---

## Estructura del proyecto

```text
TConectaApp/
│
├── src/
│   └── main/
│       ├── mule/
│       │   └── *.xml
│       │
│       └── resources/
│           └── *.yaml
│
├── api/
│   └── tconectaapp.raml
│
├── docs/
│   └── Guia-Implementacion.docx
│
├── pom.xml
├── mule-artifact.json
├── README.md
└── .gitignore
```

---

## RAML / API Design

La API se encuentra documentada mediante **RAML 1.0**.

Especificación:

```text
api/solicitudes.raml
```

La especificación fue diseñada utilizando **Anypoint Design Center**.

### Recursos documentados

```text
POST /api/v1/solicitudes
GET  /api/v1/solicitudes/{id}
```

La especificación incluye los principales códigos HTTP y estructuras de request/response.

### Anypoint Design Center

> https://anypoint.mulesoft.com/designcenter/designer/#/exchange/7dbcc70c-b7dd-4687-8493-76e960dd9a86/tconectaapp/1.0.0

---

## Manejo de errores

La aplicación implementa manejo centralizado de errores para responder de manera consistente ante errores funcionales y técnicos.

### Bad Request

```http
400 Bad Request
```

```json
{
    "error": "InternalError",
    "detail": "El campo \"tipo\" debe contener información"
}
```

### Solicitud no encontrada

```http
404 Not Found
```

```json
{
    "error": "Solicitud no encontrada",
    "mensaje": "No existe una solicitud con el identificador proporcionado."
}
```

### Error interno

```http
500 Internal Server Error
```

```json
{
  "error": "InternalError",
  "detail": "Descripción del error"
}
```

Los errores relacionados con base de datos, archivos y SFTP son capturados y procesados mediante el mecanismo de manejo de errores de MuleSoft.

---

## Trazabilidad

La aplicación utiliza el `correlationId` proporcionado por MuleSoft para facilitar el seguimiento de una solicitud durante su procesamiento.

Ejemplo de log:

```text
Solicitud enviada a sftp. idSolicitud=50 correlationId=bbee57d0-a7d8-11f1-bb59-fa3dc6d9953e
```
---

## Pruebas funcionales

### POST exitoso

```text
POST /api/v1/solicitudes
→ 201 Created
→ Registro creado en MySQL
→ Archivo JSON generado
→ Archivo enviado a SFTP
```

### GET exitoso

```text
GET /api/v1/solicitudes/{id}
→ 200 OK
→ Registro encontrado
→ Archivo asociado encontrado
```

```text
GET /api/v1/solicitudes/{id}
→ 200 OK
→ Registro encontrado
→ Archivo asociado no encontrado
```

### Solicitud inexistente

```text
GET /api/v1/solicitudes/{id}
→ 404 Not Found
```

### Error técnico

```text
Error DB / File System / SFTP
→ 500 Internal Server Error
```

---

## Ejecución local

### Prerrequisitos

* Java 17
* Maven
* Anypoint Studio
* MySQL 8.1
* Servidor SFTP
* Configuración de propiedades correspondiente al ambiente

### Ejecución

1. Clonar el repositorio:

```bash
git clone https://github.com/LuisRenero/reto-tecnico-mulesoft.git
```

2. Importar el proyecto en Anypoint Studio.

3. Configurar las propiedades de ambiente.

4. Verificar la disponibilidad de MySQL.

5. Verificar la disponibilidad del servidor SFTP.

6. Ejecutar la aplicación desde Anypoint Studio.

7. Probar los endpoints utilizando Postman, curl u otra herramienta HTTP.

---

## Checklist de despliegue

* [ ] Configuración de base de datos validada
* [ ] Credenciales configuradas de forma segura
* [ ] Conectividad con MySQL validada
* [ ] Conectividad con SFTP validada
* [ ] RAML actualizado
* [ ] Endpoints probados
* [ ] Manejo de errores validado
* [ ] Logs revisados
* [ ] `correlationId` validado
* [ ] Archivo BAR generado
* [ ] Variables de ambiente configuradas
* [ ] No existen credenciales en el repositorio

---

## Consideraciones de seguridad

* No almacenar credenciales en el código fuente.
* No publicar contraseñas en Git.
* Utilizar variables de entorno o mecanismos seguros de gestión de secretos.
* Evitar registrar payloads completos cuando puedan contener información sensible.
* Mantener actualizadas las dependencias utilizadas.
* Utilizar HTTPS/TLS en ambientes donde corresponda.

---

## Documentación adicional

* **Guía de Implementación:** `docs/Documentacion_e_Implementacion_API_Mule.docx`
* **Especificación RAML:** `api/solicitudes.raml`
* **Código fuente:** este repositorio
* **Anypoint Design Center:** https://anypoint.mulesoft.com/designcenter/designer/#/exchange/7dbcc70c-b7dd-4687-8493-76e960dd9a86/tconectaapp/1.0.0

---

## Autor

**Elaboró:** Luis Alberto Ramírez Renero

**Proyecto:** tconectaapp - Reto Técnico MuleSoft
**Versión:** 1.0
**Fecha:** 03/09/2026

````
