# Arquitectura: cómo funciona y cómo se comunican las piezas

Este documento explica **qué hace cada pieza** del proyecto y **cómo se relacionan**. Los datos técnicos fueron verificados leyendo el código fuente de Spring AI 2.0.2 (y la configuración del proyecto); donde algo es estándar de Spring Boot y no específico de Spring AI, se indica.

---

## 1. Visión general en una imagen

```
 Navegador / curl
       │  GET /api/chat?message=Hola
       ▼
┌────────────────────────────────────────────────────────────────────┐
│  TU APLICACIÓN (proceso Java, Spring Boot, puerto 8080)            │
│                                                                    │
│  ChatController            ← TU código                             │
│     │  chatClient.prompt(message).call().content()                 │
│     ▼                                                              │
│  ChatClient                ← Spring AI (spring-ai-client-chat)     │
│     │  (cadena de advisors, vacía de memoria en este proyecto)     │
│     ▼                                                              │
│  OpenAiChatModel           ← Spring AI (spring-ai-openai)          │
│     │  convierte el Prompt a una petición de OpenAI                │
│     ▼                                                              │
│  OpenAIClient (SDK oficial de OpenAI para Java, HTTP con OkHttp)   │
└─────┬──────────────────────────────────────────────────────────────┘
      │  HTTPS  (POST, API "Chat Completions", cabecera con la API key)
      ▼
  https://api.openai.com/v1      ← servidores de OpenAI (modelo gpt-5-mini)
      │
      └── respuesta ──► vuelve por el mismo camino ──► String ──► HTTP 200
```

**Idea clave:** tu código solo toca la capa `ChatClient`. Todo lo que hay debajo (`OpenAiChatModel`, el cliente HTTP, la API key, el modelo) lo crea y conecta Spring **automáticamente**.

---

## 2. Qué hace cada archivo del proyecto

### 2.1 `pom.xml` — qué librerías entran y de qué versión

| Elemento | Qué hace |
|---|---|
| `spring-boot-starter-parent` 4.1.2 | Fija las versiones de Spring Boot y de sus librerías. |
| `spring-ai-bom` 2.0.2 (en `dependencyManagement`, `scope=import`) | "Lista de versiones" de Spring AI: por eso las dependencias de Spring AI **no llevan `<version>`**. No añade librerías por sí solo. |
| `spring-boot-starter-web` | Servidor web embebido (Tomcat) en el puerto 8080 + Spring MVC (`@RestController`, etc.). |
| `spring-ai-starter-model-openai` | **La dependencia central.** Trae todo lo necesario para hablar con OpenAI (ver 2.1). |
| `spring-boot-devtools` (`runtime`, `optional`) | Reinicio automático al cambiar clases durante el desarrollo. No afecta a la lógica. |
| `spring-boot-starter-test` (`test`) | JUnit y utilidades de test. Necesaria para que compile `OpenaiApplicationTests`. |
| `spring-boot-maven-plugin` | Permite `mvn spring-boot:run` y empaquetar el JAR ejecutable. |

#### 2.1 Qué trae realmente `spring-ai-starter-model-openai`

Un *starter* es un "paquete de dependencias". Este arrastra (verificado en su `pom` 2.0.2):

| Dependencia interna | Función |
|---|---|
| `spring-ai-openai` | Contiene `OpenAiChatModel`: la clase que implementa la conversación con OpenAI. |
| `spring-ai-client-chat` | Contiene `ChatClient`: la API de alto nivel que usa tu controlador. |
| `spring-ai-autoconfigure-model-openai` | Crea automáticamente el `OpenAiChatModel` leyendo tus propiedades. |
| `spring-ai-autoconfigure-model-chat-client` | Crea automáticamente el `ChatClient.Builder`. |
| `spring-ai-autoconfigure-model-chat-memory` | Registra beans de memoria de chat (ver sección 6; **no se usan** por defecto). |
| `spring-boot-starter-webclient`, `spring-boot-starter-restclient`, `kotlin-reflect` | Soporte HTTP/Kotlin de apoyo. |

### 2.2 `OpenaiApplication.java` — el arranque

```java
@SpringBootApplication
public class OpenaiApplication {
    public static void main(String[] args) { SpringApplication.run(OpenaiApplication.class, args); }
}
```

`@SpringBootApplication` combina tres cosas:
1. **`@Configuration`** — esta clase puede definir beans.
2. **`@ComponentScan`** — Spring busca clases anotadas (`@RestController`, etc.) en el paquete `com.eazybytes` **y todos sus subpaquetes**. Por eso encuentra `controller/ChatController` sin que lo registres a mano.
3. **`@EnableAutoConfiguration`** — activa la **autoconfiguración**: Spring revisa qué librerías hay en el classpath y crea beans por ti. **Aquí es donde "aparece" la conexión con OpenAI.**

> **Importante:** por eso la ubicación importa. `OpenaiApplication` está en `com.eazybytes`; el controlador en `com.eazybytes.controller`. Si el controlador estuviera fuera de `com.eazybytes`, Spring no lo vería.

### 2.3 `application.properties` — la configuración

```properties
spring.application.name=openai
logging.pattern.console=%green(%d{HH:mm:ss.SSS}) %blue(%-5level) %red([%thread]) %yellow(%logger{15}) - %msg%n
spring.ai.openai.api-key=${OPENAI_API_KEY}
```

- `spring.application.name` y `logging.pattern.console` son de **Spring Boot** (nombre y formato de logs).
- `spring.ai.openai.api-key` es la **única propiedad de Spring AI**. El prefijo `spring.ai.openai` corresponde a la clase `OpenAiCommonProperties`.
- `${OPENAI_API_KEY}` es un *placeholder* de Spring: se sustituye por el valor de la variable de entorno. Si no existe, la app **falla al arrancar** con `Could not resolve placeholder 'OPENAI_API_KEY'`.

> **Dato verificado:** aun sin esa línea, Spring AI buscaría por sí mismo la variable de entorno `OPENAI_API_KEY` (`OpenAiSetup.detectApiKey`). La línea explícita hace el enlace visible y obligatorio; hace que la ausencia de la variable sea un error claro en el arranque.

### 2.4 `ChatController.java` — tu único código de negocio

```java
@RestController
@RequestMapping("/api")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam("message") String message) {
        return chatClient.prompt(message).call().content();
    }
}
```

Línea por línea:

| Código | Qué significa |
|---|---|
| `@RestController` | Es un controlador web; lo que devuelven sus métodos va directo en el cuerpo de la respuesta HTTP (no se busca una vista HTML). |
| `@RequestMapping("/api")` | Prefijo de ruta para todos sus métodos. |
| `@GetMapping("/chat")` | Este método atiende `GET /api/chat` (ruta completa = prefijo + esto). |
| `@RequestParam("message")` | Toma `message` de la query string (`?message=...`). Obligatorio. |
| Constructor con `ChatClient.Builder` | **Inyección de dependencias:** Spring ve que el constructor pide un `ChatClient.Builder`, lo busca entre sus beans y lo entrega. Tú nunca haces `new` de él. |
| `chatClientBuilder.build()` | Construye el `ChatClient` una sola vez y se guarda en un campo `final`. |
| `chatClient.prompt(message)` | Crea una petición con `message` como mensaje de usuario. |
| `.call()` | Ejecuta la petición de forma **síncrona** (bloquea hasta que OpenAI responde). |
| `.content()` | Extrae solo el **texto** de la respuesta (descarta metadatos como tokens usados, etc.). |
| `return <String>` | Spring lo escribe como cuerpo de la respuesta (texto plano). |

> **¿Por qué se inyecta el *Builder* y no el `ChatClient`?** Porque Spring AI registra un `ChatClient.Builder`, no un `ChatClient` listo. Es la forma intencional: así cada clase puede personalizar su cliente (mensaje de sistema, advisors…) antes de `build()`. El builder es de ámbito **`prototype`**: cada punto de inyección recibe **una copia nueva**, de modo que personalizar uno no afecta a los demás.

### 2.5 `OpenaiApplicationTests.java` — el test

`@SpringBootTest` + un método vacío `contextLoads()`: verifica que **el contexto de Spring arranca sin errores**. Como arranca toda la app, necesita `OPENAI_API_KEY` definida. No llama a OpenAI.

---

## 3. Qué pasa al ARRANCAR (autoconfiguración, paso a paso)

Spring Boot lee, dentro de los JAR de Spring AI, el archivo `META-INF/spring/…AutoConfiguration.imports`, que lista las clases de autoconfiguración. Se activan las siguientes (verificado en 2.0.2):

```
1. application.properties se lee
        │   spring.ai.openai.api-key  → OpenAiCommonProperties
        │   (spring.ai.openai.chat.*  → OpenAiChatProperties, vacío aquí → valores por defecto)
        ▼
2. OpenAiChatAutoConfiguration crea el bean  OpenAiChatModel
        │   - construye un OpenAIClient (SDK oficial de OpenAI) con la API key y
        │     la URL base https://api.openai.com/v1 (también un cliente asíncrono)
        │   - le pasa las opciones (modelo por defecto: gpt-5-mini)
        ▼
3. ChatClientAutoConfiguration crea el bean  ChatClient.Builder  (prototype)
        │   necesita un ChatModel → recibe el OpenAiChatModel del paso 2
        ▼
4. Spring crea ChatController
        │   su constructor pide ChatClient.Builder → Spring se lo inyecta
        ▼
5. Constructor: this.chatClient = builder.build()      → app lista en :8080
```

**Condiciones relevantes** (qué hace que esto se active):
- `OpenAiChatAutoConfiguration` está condicionada a `spring.ai.model.chat` = `openai`, y **se activa si la propiedad no existe** (`matchIfMissing = true`). Por eso funciona sin configurar nada más.
- El bean `OpenAiChatModel` lleva `@ConditionalOnMissingBean`: si tú definieras el tuyo, Spring AI no crearía el suyo.
- `ChatClientAutoConfiguration` se activa por defecto (`spring.ai.chat.client.enabled` = `true`) y requiere un `ChatModel` en el contexto.

---

## 4. Qué pasa en CADA PETICIÓN (flujo en tiempo de ejecución)

Ejemplo: `GET /api/chat?message=Hola`

| # | Quién | Qué ocurre |
|---|---|---|
| 1 | Tomcat + Spring MVC | Recibe la petición, ve que `/api/chat` coincide con `ChatController.chat(...)` y pasa `message="Hola"`. |
| 2 | `ChatController` | Llama `chatClient.prompt("Hola")`. Internamente `DefaultChatClient` exige texto no vacío y crea un `Prompt` con **un único mensaje de usuario**. |
| 3 | `.call()` | Marca la petición como síncrona. |
| 4 | `.content()` | Dispara la ejecución: la petición pasa por la **cadena de advisors** y llega al `ChatModel`. Luego se extrae el texto de la respuesta. |
| 5 | `OpenAiChatModel` | Traduce el `Prompt` de Spring AI a una petición propia de OpenAI (`ChatCompletionCreateParams`) con el modelo y las opciones. |
| 6 | `OpenAIClient` | Hace la llamada HTTPS a `https://api.openai.com/v1` usando la API **Chat Completions** (`chat().completions().create(...)`), con tu API key en la autenticación. |
| 7 | OpenAI | El modelo genera la respuesta y la devuelve. |
| 8 | `OpenAiChatModel` | Convierte la respuesta de OpenAI a un `ChatResponse` de Spring AI (texto + metadatos: tokens, límites de uso…). |
| 9 | `.content()` | Devuelve solo el texto (`String`). |
| 10 | Spring MVC | Escribe ese `String` como cuerpo de la respuesta HTTP 200. |

**Qué NO ocurre:** no se guarda nada entre peticiones. Si preguntas "Me llamo Laura" y luego "¿Cómo me llamo?", el modelo **no lo sabrá**: el paso 2 siempre crea un `Prompt` con un solo mensaje.

---

## 5. Cómo se relacionan las capas de Spring AI

| Capa | Tipo | Responsabilidad | ¿Quién la usa aquí? |
|---|---|---|---|
| `ChatClient` | API fluida de alto nivel | Construir peticiones (`prompt`), añadir mensaje de sistema, advisors, y obtener la salida (`content`, `entity`, `chatResponse`, `stream`). | **Tu controlador** |
| `ChatModel` (interfaz) | Abstracción de bajo nivel | Contrato "dado un `Prompt`, devuelve un `ChatResponse`". Independiente del proveedor. | Lo usa `ChatClient` internamente |
| `OpenAiChatModel` | Implementación para OpenAI | Implementa `ChatModel` hablando con OpenAI. | Creada por la autoconfiguración |
| `OpenAIClient` | SDK oficial de OpenAI (Java) | HTTP, autenticación, reintentos, timeouts. | Lo usa `OpenAiChatModel` |

**Por qué existe esta separación:** `ChatClient` y `ChatModel` son agnósticos del proveedor. Para cambiar de OpenAI a otro (Anthropic, Ollama…) se cambia el starter y las propiedades; **el controlador no cambia**. Esa es la razón de ser de Spring AI.

Opciones de salida de `ChatClient` disponibles en 2.0.2 (para ampliar el proyecto):

| Método | Devuelve |
|---|---|
| `.call().content()` | Solo el texto (**el que usamos**) |
| `.call().chatResponse()` | La respuesta completa con metadatos (tokens usados, etc.) |
| `.call().entity(Clase.class)` | La respuesta convertida a un objeto Java (salida estructurada) |
| `.stream().content()` | El texto llegando por partes (streaming) |

---

## 6. Memoria: por qué el "chat" no recuerda nada

El starter incluye `spring-ai-autoconfigure-model-chat-memory`, que **sí registra** dos beans en el contexto: un `ChatMemoryRepository` (en memoria RAM, `InMemoryChatMemoryRepository`) y un `ChatMemory` (ventana de mensajes, `MessageWindowChatMemory`).

Pero **tener los beans no equivale a usarlos.** `ChatClientAutoConfiguration` no hace referencia a `ChatMemory` (verificado), y el controlador no añade ningún advisor de memoria. Resultado: el historial **no** se envía al modelo.

Para dar memoria habría que añadir un *advisor* de memoria al `ChatClient` y enviar un identificador de conversación en cada petición. Es el siguiente paso lógico del curso.

---

## 7. Valores por defecto que "no ves" pero están actuando

| Qué | Valor | Origen |
|---|---|---|
| Modelo | `gpt-5-mini` | `OpenAiChatOptions.DEFAULT_CHAT_MODEL` |
| URL de OpenAI | `https://api.openai.com/v1` | `OpenAiSetup.OPENAI_URL`; también admite la variable de entorno `OPENAI_BASE_URL` |
| Timeout | 60 s | `AbstractOpenAiProperties.DEFAULT_TIMEOUT` |
| Reintentos | 3 | `AbstractOpenAiProperties.DEFAULT_MAX_RETRIES` |
| Puerto web | 8080 | Estándar de Spring Boot |
| Tool calling | Habilitado, sin herramientas registradas | `spring.ai.chat.client.tool-calling.enabled=true` (por defecto) |

---

## 8. Glosario

| Término | Significado |
|---|---|
| **Bean** | Objeto creado y gestionado por Spring (lo "guarda" en el contexto). |
| **Inyección de dependencias** | Spring entrega a una clase los objetos que pide en su constructor, en lugar de que ella los cree. |
| **Autoconfiguración** | Spring Boot crea beans automáticamente según las librerías presentes y las propiedades. |
| **Starter** | Dependencia "paquete" que agrupa todo lo necesario para una función. |
| **BOM** | *Bill of Materials*: lista de versiones compatibles. No añade librerías por sí misma. |
| **Prompt** | La entrada que se envía al modelo (uno o varios mensajes). |
| **Advisor** | Interceptor del `ChatClient` que puede modificar la petición/respuesta (p. ej., para memoria). |
| **Síncrono** (`call`) | El hilo espera hasta tener la respuesta completa. |
| **Placeholder** `${X}` | Marcador que Spring sustituye por una propiedad o variable de entorno. |

---

## 9. Qué se verificó y cómo

Esta documentación se contrastó con el código fuente de Spring AI 2.0.2 descargado en el repositorio Maven local, concretamente: `OpenAiChatAutoConfiguration`, `ChatClientAutoConfiguration`, `ChatClientBuilderProperties`, `OpenAiChatProperties`, `AbstractOpenAiProperties`, `OpenAiSetup`, `OpenAiChatOptions`, `OpenAiChatModel`, `DefaultChatClient` y `ChatMemoryAutoConfiguration`.

**No se verificó** (es comportamiento estándar de Spring Boot/MVC, no leído en el código de este proyecto): el formato exacto de las respuestas de error HTTP, y el tipo exacto de excepción lanzada por el SDK de OpenAI ante una clave inválida (la documentación solo afirma que se propaga como error 500 al no haber manejo de excepciones en el controlador).
