# 01 · Hello World con Spring AI + OpenAI

Mini "chat" de aprendizaje: una aplicación Spring Boot con **un único endpoint HTTP** que recibe un texto, se lo envía a OpenAI a través de Spring AI y devuelve la respuesta.

> Para entender **cómo se comunican las piezas por dentro**, lee [`docs/ARQUITECTURA.md`](docs/ARQUITECTURA.md).

## Qué es y qué NO es

| Es | No es |
|---|---|
| Un servidor web (puerto 8080) con `GET /api/chat` | Una interfaz de chat (no hay HTML ni frontend) |
| Una llamada **síncrona** a OpenAI por cada petición | Un chat con memoria: **cada petición es independiente** |
| Una demo del patrón `ChatClient` de Spring AI | Streaming: la respuesta llega completa, no por partes |

## Versiones (las del `pom.xml`)

| Componente | Versión |
|---|---|
| Java | 21 |
| Spring Boot (`spring-boot-starter-parent`) | 4.1.2 |
| Spring AI (`spring-ai-bom`) | 2.0.2 |
| Modelo de OpenAI por defecto | `gpt-5-mini` (ver nota abajo) |

## Estructura del proyecto

```
01_hello_world_openai/
├── pom.xml                                   → dependencias y versiones
├── README.md                                 → este archivo
├── docs/ARQUITECTURA.md                      → cómo se conectan las piezas
└── src/
    ├── main/
    │   ├── java/com/eazybytes/
    │   │   ├── OpenaiApplication.java        → punto de entrada (arranca Spring Boot)
    │   │   └── controller/ChatController.java→ el endpoint GET /api/chat
    │   └── resources/application.properties → configuración (aquí se enlaza la API key)
    └── test/java/com/eazybytes/
        └── OpenaiApplicationTests.java       → test "contextLoads"
```

Son solo **3 clases/archivos con lógica**: `OpenaiApplication`, `ChatController` y `application.properties`. Todo lo demás (conexión con OpenAI, creación del `ChatClient`, servidor web) lo hace Spring por **autoconfiguración** a partir de las dependencias del `pom.xml`.

## Cómo ejecutarlo

### 1. Definir la API key (variable de entorno)

`application.properties` contiene:

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
```

`${OPENAI_API_KEY}` significa: *"toma el valor de la variable de entorno `OPENAI_API_KEY`"*. **La clave nunca se escribe en el archivo.** Debe existir en el entorno del proceso que arranca la app:

```bash
export OPENAI_API_KEY="sk-..."      # OJO: sin "export" la variable NO llega a Maven/Java
mvn spring-boot:run                 # en la MISMA terminal donde hiciste el export
```

- `export` es obligatorio. `OPENAI_API_KEY="sk-..."` a secas crea una variable solo de la shell, que los programas lanzados desde ella no ven.
- La variable vive solo en esa terminal. Si abres otra, hay que volver a exportarla (o añadirla a `~/.zshrc`).
- En IntelliJ: *Run → Edit Configurations → OpenaiApplication → Environment variables → `OPENAI_API_KEY=sk-...`*.
- **Nunca** pegues la clave en `application.properties` si vas a subir el proyecto a git, ni la compartas en chats, capturas o issues. Si se expone, revócala en https://platform.openai.com/api-keys y crea otra.

### 2. Arrancar

```bash
mvn spring-boot:run
```

Debe terminar mostrando `Started OpenaiApplication`. (`mvn spring-boot:run` **compila** también los tests, pero **no los ejecuta**.)

### 3. Probar

```bash
curl "http://localhost:8080/api/chat?message=Hola"
```

o abre esa URL en el navegador. Cambia el valor de `message` para preguntar otra cosa. Para detener la app: `Ctrl+C`.

## Referencia del endpoint

| | |
|---|---|
| Método y ruta | `GET /api/chat` |
| Parámetro | `message` (query string, **obligatorio**) |
| Respuesta | Texto plano con la respuesta del modelo |

Casos de error:

| Situación | Resultado |
|---|---|
| Falta el parámetro `message` | `400 Bad Request` (lo gestiona Spring MVC) |
| `message` vacío (`?message=`) | Error `500`: `ChatClient.prompt(String)` exige texto no vacío (`IllegalArgumentException: content cannot be null or empty`) |
| API key inválida/revocada, sin saldo, o sin red | Error `500`: la excepción de la llamada a OpenAI no se captura en el controlador |
| OpenAI tarda más de 60 s | Falla por timeout (por defecto 60 s, con hasta 3 reintentos) |

## Configuración (`application.properties`)

| Propiedad | Valor actual | Qué hace |
|---|---|---|
| `spring.application.name` | `openai` | Nombre lógico de la app (informativo). |
| `logging.pattern.console` | patrón con colores | Solo cambia el **formato** de los logs en consola. |
| `spring.ai.openai.api-key` | `${OPENAI_API_KEY}` | API key usada para autenticarse con OpenAI. |

### Propiedades útiles que NO están puestas (usan valores por defecto)

Verificadas en Spring AI 2.0.2:

| Propiedad | Para qué | Por defecto |
|---|---|---|
| `spring.ai.openai.chat.model` | Elegir el modelo | `gpt-5-mini` |
| `spring.ai.openai.chat.temperature` | Creatividad de la respuesta | la del modelo |
| `spring.ai.openai.chat.max-completion-tokens` | Límite de longitud de la respuesta | sin límite configurado |
| `spring.ai.openai.base-url` | Apuntar a otro servidor compatible con OpenAI | `https://api.openai.com/v1` |
| `spring.ai.openai.timeout` | Timeout de la petición | 60 s |
| `spring.ai.openai.max-retries` | Reintentos | 3 |

> ⚠️ **Formato antiguo deprecado:** en Spring AI 2.0 la forma `spring.ai.openai.chat.options.model` (con `.options.`) está marcada como `@Deprecated(forRemoval = true)`. Usa `spring.ai.openai.chat.model`.

## Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `zsh: command not found: mvn` | Maven no instalado | `brew install maven` |
| `Could not resolve placeholder 'OPENAI_API_KEY'` al arrancar | La variable no existe en ese proceso | `export OPENAI_API_KEY=...` en la misma terminal (o configurarla en IntelliJ) |
| Error `401` de OpenAI | Clave incorrecta, revocada o con espacios/comillas de más | Revisar la clave |
| `COMPILATION ERROR ... org.junit.jupiter.api does not exist` | Faltaba `spring-boot-starter-test` en el `pom.xml` | Ya corregido: está añadido con `scope=test` |
| `mvn test` falla | El test arranca toda la app y necesita `OPENAI_API_KEY` | Exportar la variable antes de correr los tests |

## Siguientes pasos naturales (no implementados aquí)

1. **Mensaje de sistema** (`.defaultSystem(...)` en el builder) para fijar el rol/estilo del asistente.
2. **Memoria de conversación** con un *advisor* de memoria (ver "Memoria" en `docs/ARQUITECTURA.md`).
3. **Streaming** con `.stream().content()` en lugar de `.call().content()`.
4. Un frontend sencillo que consuma `/api/chat`.
