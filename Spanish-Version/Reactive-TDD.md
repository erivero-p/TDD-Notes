# Kafka testing and reactive programing

## Testing Flux y Mono

### Por qué testear reactive streams es diferente

Testear código reactivo con `Flux` y `Mono` es distinto al testing clásico porque:

- Son **asincrónicos y no bloqueantes**: los valores se emiten en el tiempo, por lo que no podemos simplemente “comparar valores”.
- Tienen **orden e intervalo de emisión**: no solo importa *qué* se emite, sino *cuándo* y *en qué orden*.
- La suscripción es **perezosa (lazy)**: nada ocurre hasta que alguien se suscribe.
- El stream puede **emitir elementos, completarse o lanzar errores**.

Por eso necesitamos una herramienta especializada: `StepVerifier`.

### Usando StepVerifier para las aserciones

`StepVerifier` es parte de **Reactor Test**, una librería que permite testear `Flux` y `Mono` declarando paso a paso qué esperas que ocurra.

Para usarlo, necesitaremos añadir su dependencia:

```xml
<dependency>
  <groupId>io.projectreactor</groupId>
  <artifactId>reactor-test</artifactId>
  <scope>test</scope>
</dependency>
```

La sintaxis básica es la siguiente:

```java
StepVerifier.create(miFluxOMono)
    .expectNext(...)        // Espera un valor emitido
    .expectComplete()       // Espera que el stream termine bien
    .verify();              // Ejecuta la verificación
```

Algunas aserciones comunes con StepVerifier son:

| Método | Qué hace |
| --- | --- |
| `.expectNext(T... values)` | Espera que esos valores se emitan en orden |
| `.expectNextCount(n)` | Espera que se emitan `n` elementos, sin verificar valores |
| `.expectComplete()` | Espera que el `Flux` o `Mono` finalice correctamente |
| `.expectError()` | Espera que ocurra un error (de cualquier tipo) |
| `.expectError(Class<?>)` | Espera un error de un tipo específico |
| `.thenCancel()` | Cancela el stream antes de que termine |
| `.verify(Duration timeout)` | Verifica con un timeout para evitar bloqueos |

### Ejemplo con `ReactiveBookStatisticsServiceTest`

Como en el test no reactivo de service, declararemos el mock del repositorio, e inyectaremos el del servicio.

En cuanto a los miembros auxiliares, esta vez,
Como vamos a realizar varios tests con distintos datos, inicializamos múltiples objetos `Review` en el `setUp()`.En este caso, de Review.

```java
@ExtendWith(MockitoExtension.class)
class ReactiveBookStatisticsServiceTest {

    @Mock
    private ReviewRepository reviewRepository;

    @InjectMocks
    private ReactiveBookStatisticsService reactiveBookStatisticsService;

    private Review review1;
    private Review review2;
    private Review review3;
    private Review review4;
    private Review review5;
    
     @BeforeEach
    void setUp() {
        review1 = new Review(1L, 1L, "user1", 5, "Excellent book!");
        review2 = new Review(2L, 1L, "user2", 4, "Very good book");
        review3 = new Review(3L, 1L, "user3", 3, "Good book");
        review4 = new Review(4L, 1L, "user4", 2, "Average book");
        review5 = new Review(5L, 1L, "user5", 1, "Poor book");
    }
    (...)
 }
```

Con esto, ya podremos declarar los tests. Seguiremos la misma estructura Given / when / then. Sólo que, en esta ocasión, el resultado de llamar al método  a testear será un Mono o un Flux, y las aserciones en Then se harán con StepVerifier:

```java
    @Test
    void getAverageRating_WhenBookHasMultipleReviews_ShouldReturnCorrectAverage() {
        // Given
        Long bookId = 1L;
        List<Review> reviews = Arrays.asList(review1, review2, review3, review4, review5);
        when(reviewRepository.findByBookId(bookId)).thenReturn(reviews);

        // Expected average: (5+4+3+2+1)/5 = 3.0
        double expectedAverage = 3.0;

        // When
        Mono<Double> result = reactiveBookStatisticsService.getAverageRating(bookId);

        // Then
        StepVerifier.create(result)
                .expectNext(expectedAverage)
                .verifyComplete();
    }
```

Ejemplo de un caso negativo:

```java
    @Test
    void getAverageRating_WhenBookIdIsNull_ShouldThrowException() {
        // Given
        Long bookId = null;

        // When & Then
        StepVerifier.create(reactiveBookStatisticsService.getAverageRating(bookId))
                .expectError(IllegalArgumentException.class)
                .verify();
    }
```
---

# Testing con Kafka y Spring WebFlux

## Consumers y publishers con Flux  KafkaTemplate

### ¿Cómo se modelan los streams Kafka en código reactivo?

- En una arquitectura reactiva con Spring WebFlux y Kafka, los productores y consumidores se modelan con `Flux` o `Mono`.
- Un consumidor puede exponer un `Flux<String>` o `Flux<Event>` que reacciona a nuevos mensajes entrantes.
- Un productor puede tomar un `Flux<Event>` y publicarlos en Kafka mediante `KafkaTemplate`.

**Ejemplo de publicación reactiva:**

```java
@Autowired
private ReactiveKafkaProducerTemplate<String, MyEvent> producer;

public Mono<Void> sendEvent(MyEvent event) {
    return producer.send("my-topic", event.getId(), event)
                   .then(); // devuelve Mono<Void> tras publicar
}

```

**Ejemplo de consumidor reactivo:**

```java
@KafkaListener(topics = "my-topic", groupId = "group-id")
public void listen(String message) {
    // Procesamiento reactivo aquí
}

```

### Consideraciones para pruebas unitarias

- El `KafkaTemplate` y otros componentes de Kafka se pueden mockear para testear lógicamente sin conexión real.
- No se testea el envío real del mensaje, solo que se invoca correctamente.
- Usar `StepVerifier` si se devuelve `Mono` o `Flux`.

```java
@Test
void shouldSendEventSuccessfully() {
    MyEvent event = new MyEvent("1", "test");

    when(producer.send(anyString(), anyString(), any(MyEvent.class)))
        .thenReturn(Mono.just(SenderResult.success()));

    StepVerifier.create(service.sendEvent(event))
        .verifyComplete();

    verify(producer, times(1)).send("my-topic", "1", event);
}
```

## Tests de integración con `@EmbeddedKafka`

### Configuración básica

- `@EmbeddedKafka` inicia un broker Kafka en memoria.
- Útil para simular envío y consumo real en tests de integración.
- Necesitas configurar `spring.kafka.bootstrap-servers` para apuntar al embedded broker.

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"test-topic"})
class KafkaIntegrationTest {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private KafkaConsumerService consumerService;

    @Test
    void testKafkaMessageFlow() throws InterruptedException {
        kafkaTemplate.send("test-topic", "key", "Hello World");

        // Esperamos con algún mecanismo como CountDownLatch
        boolean received = consumerService.awaitMessage("Hello World");
        assertTrue(received);
    }
}

```

> Con esto, podremos enviar mensajes al topic embebido, y utilizar listeners como los que se usarían en producción para recibir mensajes.
>

## Verificación de mensajes enviados y recibidos
Cuando trabajamos con Kafka y Spring WebFlux, **verificar que un mensaje fue realmente enviado y procesado** puede ser complejo por la naturaleza asíncrona del sistema. Algunas técnicas comunes para esto son:

### 1. `CountDownLatch`

Para esperar eventos de manera controlada usaremos `CountDownLatch`, que es  es una herramienta de sincronización de Java muy útil en tests integrados con Kafka. Permite **esperar a que un mensaje sea procesado antes de hacer las aserciones**.

**Ejemplo**: un `@KafkaListener` que procesa una notificación y actualiza un servicio:

```java
@Component
public class BookKafkaConsumer {

    private final BookService bookService;
    private final CountDownLatch latch = new CountDownLatch(1);

    public BookKafkaConsumer(BookService bookService) {
        this.bookService = bookService;
    }

    @KafkaListener(topics = "book-notifications", groupId = "test-group")
    public void listen(String message) {
        bookService.process(message); // lógica del negocio
        latch.countDown(); // señal de que el mensaje fue recibido
    }

    public CountDownLatch getLatch() {
        return latch;
    }
}

```

En el test:

```java
@Autowired
private BookKafkaConsumer consumer;

@Test
void shouldConsumeKafkaMessage() throws Exception {
    kafkaTemplate.send("book-notifications", "123", "test message");

    boolean processed = consumer.getLatch().await(5, TimeUnit.SECONDS);

    assertTrue(processed, "El mensaje no fue procesado a tiempo");
}

```

### 2. `StepVerifier`

Cuando el procesamiento de mensajes se hace de forma reactiva (por ejemplo, `Flux<String>`), podremos utilizar `StepVerifier` para verificar que el flujo se comporta como esperas:

```java
Flux<String> messageFlux = kafkaReceiver.receive()
    .map(ReceiverRecord::value)
    .take(1); // suponiendo que esperamos solo un mensaje

StepVerifier.create(messageFlux)
    .expectNext("Mensaje esperado")
    .expectComplete()
    .verify(Duration.ofSeconds(5));

```

### 3. `@SpyBean` o `@MockBean`

Podemos usar estas anotaciones para verificar llamadas internas, porque permiten **espiar** o **mockear** los beans dentro del contexto de Spring y luego verificar interacciones.

Por ejemplo, si esperamos que el `BookService` procese el mensaje recibido:

```java
@SpyBean
private BookService bookService;

@Test
void shouldCallBookServiceWhenKafkaMessageReceived() throws Exception {
    kafkaTemplate.send("book-notifications", "123", "some book json");

    Thread.sleep(2000); // espera breve al listener

    verify(bookService, times(1)).process(any(String.class));
}

```

> `@SpyBean` mantiene el comportamiento real del bean, pero permite usar Mockito.verify().
>

### Serialización y deserialización

En el contexto de una arquitectura basada en **Kafka**, cada mensaje que se envía o recibe a través de un tópico debe **serializarse** (convertirse a un array de bytes) para que pueda ser transportado por el sistema de mensajería. Al recibirlo, el mensaje debe **deserializarse** para reconstruir el objeto original en memoria.

Cuando trabajamos con **Spring WebFlux** y Kafka, esta serialización suele hacerse con bibliotecas como:

- **JSON**: mediante Jackson (`ObjectMapper`) o serializers de Kafka integrados.
- **Avro**: un sistema de serialización más eficiente y estructurado, útil en entornos donde se requiere validación de esquemas y alto rendimiento.

### ¿Por qué es importante testear esto?

- Porque una mala configuración del serializer o deserializer puede hacer que el consumidor falle silenciosamente o descarte mensajes.
- Porque si cambias el modelo del mensaje, es fácil romper compatibilidad con versiones anteriores.
- Porque puedes garantizar que los mensajes cumplen con ciertos esquemas (especialmente en Avro).

Los tests de serialización y deserialización nos permiten asegurarnos de que:

- Los mensajes **se transforman correctamente** de objetos Java a bytes y viceversa.
- No se pierden ni corrompen datos.
- Se cumplen los requisitos de esquema (estructura, campos obligatorios, tipos, etc.).

### Testing con Avro

- Si usas Avro, asegúrate de tener los esquemas generados y los serializers correctamente configurados.
- Puedes testear serialización individual con un test unitario:

```java
@Test
void shouldSerializeAndDeserializeWithAvro() {
    AvroSerializer serializer = new AvroSerializer();
    AvroDeserializer<MyEvent> deserializer = new AvroDeserializer<>(MyEvent.class);

    MyEvent original = new MyEvent("id123", "test");
    byte[] bytes = serializer.serialize("topic", original);
    MyEvent result = deserializer.deserialize("topic", bytes);

    assertEquals(original, result);
}

```

### Testing con JSON

- Similar al caso de Avro, puedes testear la serialización con Jackson:

```java
@Test
void shouldSerializeEventToJson() throws JsonProcessingException {
    ObjectMapper objectMapper = new ObjectMapper();
    MyEvent event = new MyEvent("id123", "test");

    String json = objectMapper.writeValueAsString(event);
    MyEvent deserialized = objectMapper.readValue(json, MyEvent.class);

    assertEquals(event, deserialized);
}

```

### Validaciones adicionales

- Es buena práctica verificar que los campos requeridos estén presentes.
- Puedes usar `@JsonInclude`, validaciones de Bean Validation (`@NotNull`) o `JsonSchema` para validar estructura.
- En el caso de Avro, verifica que el `schema` esté alineado con las clases generadas.

### Ejemplo con `KafkaIntegrationTest`

En este ejemplo, validamos que una **notificación enviada a Kafka** se procese correctamente y cree un libro en el sistema. Utilizamos:

- `@EmbeddedKafka`: para crear un **cluster Kafka en memoria** solo para los tests.
- `KafkaTemplate`: para **enviar mensajes** manualmente durante el test.
- `BookService`: para verificar si el mensaje ha sido procesado correctamente.
- `ObjectMapper`: para serializar el objeto `BookNotification` a JSON.

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {
    "book-notifications", "book-stock-updates", "book-events"
})
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}",
    "spring.kafka.consumer.group-id=test-group",
    "spring.kafka.consumer.auto-offset-reset=earliest",
    "spring.kafka.consumer.enable-auto-commit=false"
})
@ActiveProfiles("test")
@Transactional
class KafkaIntegrationTest {
(...)
```

Con `@EmbeddedKafka` haremos un “mock” del broker de Kafka, en este caso, con tres topics. Y podremos configurar las propiedades  del broker  con `@TestPropertySource`. Por último, definiremos como test el perfil activo con `@ActiveProfiles`("test"), y haremos que las operaciones sobre la base de datos se reviertan al terminar los tests usando `@Transactional`.

Después, declararemos los atributos que vamos a necesitar. En este caso:

- un objeto KafkaTemplate para poder enviar mensajes a cada tópic
- una instancia del BookService y otra de BookKafkaProducerService. No usamos mokcs porque friendly reminder de que esto es un test de integración.
- ObjectMapper para la serialización
- Dos miembros auxiliares: uno para la request, y otro con la entidad, que en este caso es una notificación, ya que estamos testeando eventos.

```java
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private BookService bookService;
    
    @Autowired
    private BookKafkaProducerService kafkaProducerService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    private BookRequest testBookRequest;
    private BookNotification testNotification;
    private static int isbnCounter = 0;

```

Después, definiremos nuestro método `setUp()` para inicializar los dos atributos auxiliares:

```java
    @BeforeEach
    void setUp() {
        // Use unique ISBNs for each test to avoid conflicts
        String uniqueIsbn = generateUniqueIsbn();
        testBookRequest = new BookRequest(
            "Test Book for Kafka",
            "Test Author",
            uniqueIsbn,
            "Test book description for Kafka integration testing",
            new BigDecimal("29.99"),
            10,
            BookCategory.FICTION
        );
        
        testNotification = new BookNotification(
            generateUniqueIsbn(),
            "Kafka Test Book",
            "Kafka Test Author",
            "Book for Kafka integration testing",
            new BigDecimal("39.99"),
            25,
            BookCategory.NON_FICTION,
            BookNotification.NotificationType.NEW_BOOK,
            LocalDateTime.now()
        );
    }
```

Y ya estaremos listos para declarar los tests. En este caso, el test `shouldSendAndReceiveBookNotification` lo que busca validar es que al enviar una notificación a un topic Kafka, el sistema:

- **Consuma el mensaje correctamente** (desde un `@KafkaListener`)
- **Procese la lógica de negocio asociada** (crear un libro)
- **Y lo refleje en la base de datos** (a través de `bookService`)

Seguiremos la estructura Given when then como venimos haciendo hasta ahora.

Given: En este caso, para la preparación de los datos serializaremos la notificación a JSON con el objectMapper, para poder enviarlo como string al topic  (reminder de que el `KafkaTemplate<String, String>` trabaja con mensajes tipo texto en este caso).

When: Enviamos el mensaje al topic utilizando la kafkaTemplate y el método send. En este caso, cuando la enviamos, lo que esperamos es una confirmación por parte de Kafka, que tiene que llegar antes de 5 segundos en este caso  `.get(5, TimeUnit.SECONDS)`.

Then: Esperaremos con sleep a que consumidor Kafka procese el mensaje, activando la lógia para crear el libro en el sistema.

Tras esto, hacemos retries para por si, incluso habiéndole dado un margen con sleep, el proceso no se hubiera llevado a cabo. En este caso, estamos aplicando una estrategia de reintento progresivo, de hasta 10 retries con pausas de 1s. El bucle de retries se romperá si ya no es necesario.

Finalmente, hacemos las aserciones necesarias para comprobar que el libro se haya creado correctamente.

```java
@Test
@DisplayName("Should send and receive book notification successfully")
void shouldSendAndReceiveBookNotification() throws Exception {
    // Given - Convertimos la notificación a JSON
    String notificationJson = objectMapper.writeValueAsString(testNotification);

    // When - Enviamos el mensaje a Kafka
    kafkaTemplate.send("book-notifications", testNotification.getIsbn(), notificationJson)
        .get(5, TimeUnit.SECONDS); // espera a que Kafka confirme

    // Then - Esperamos a que se procese y verificamos el resultado
    Thread.sleep(3000); // permitimos que el listener consuma y procese

    // Retry para evitar flakiness en ambientes lentos
    BookResponse createdBook = null;
    for (int i = 0; i < 10; i++) {
        try {
            createdBook = bookService.getBookByIsbn(testNotification.getIsbn());
            if (createdBook != null && testNotification.getTitle().equals(createdBook.getTitle())) {
                break;
            }
        } catch (Exception e) {
            // No encontrado aún
        }
        Thread.sleep(1000);
    }
    // Assertions
    assertNotNull(createdBook);
    assertEquals(testNotification.getTitle(), createdBook.getTitle());
    assertEquals(testNotification.getAuthor(), createdBook.getAuthor());
    assertEquals(testNotification.getIsbn(), createdBook.getIsbn());
    assertEquals(testNotification.getStock(), createdBook.getStock());
    assertEquals(testNotification.getCategory(), createdBook.getCategory());
}

```

Sobre la vaina de flakiness: En entornos asíncronos como Kafka, el procesamiento no es inmediato. Por eso, se recomienda hacer reintentos cortos antes de fallar directamente el test.

### Buenas prácticas:

- **Evita sleeps fijos** cuando puedas, pero en Kafka embebido a veces es necesario usar `Thread.sleep()` + reintentos para dar tiempo al procesamiento.
- Es importante verificar **todos los campos** recibidos para asegurar que el mensaje se ha deserializado correctamente.
- Si usas Avro en lugar de JSON, cambiarás el serializador y probablemente no necesitarás `ObjectMapper`.