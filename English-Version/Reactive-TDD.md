# Kafka Testing and Reactive Programming

## Testing Flux and Mono

### Why testing reactive streams is different

Testing reactive code with `Flux` and `Mono` is different from classic testing because:

- They are **asynchronous and non-blocking**: values are emitted over time, so we can't simply "compare values".
- They have **order and emission interval**: not only *what* is emitted matters, but *when* and *in what order*.
- Subscription is **lazy**: nothing happens until someone subscribes.
- The stream can **emit elements, complete, or throw errors**.

That's why we need a specialized tool: `StepVerifier`.

### Using StepVerifier for assertions

`StepVerifier` is part of **Reactor Test**, a library that allows testing `Flux` and `Mono` by declaring step by step what you expect to happen.

To use it, we'll need to add its dependency:

```xml
<dependency>
  <groupId>io.projectreactor</groupId>
  <artifactId>reactor-test</artifactId>
  <scope>test</scope>
</dependency>
```

The basic syntax is as follows:

```java
StepVerifier.create(myFluxOrMono)
    .expectNext(...)        // Expects an emitted value
    .expectComplete()       // Expects the stream to terminate successfully
    .verify();              // Executes the verification
```

Some common assertions with StepVerifier are:

| Method | What it does |
| --- | --- |
| `.expectNext(T... values)` | Expects those values to be emitted in order |
| `.expectNextCount(n)` | Expects `n` elements to be emitted, without verifying values |
| `.expectComplete()` | Expects the `Flux` or `Mono` to complete successfully |
| `.expectError()` | Expects an error to occur (of any type) |
| `.expectError(Class<?>)` | Expects an error of a specific type |
| `.thenCancel()` | Cancels the stream before it completes |
| `.verify(Duration timeout)` | Verifies with a timeout to avoid blocking |

### Example with `ReactiveBookStatisticsServiceTest`

As in the non-reactive service test, we'll declare the repository mock and inject the service one.

Regarding auxiliary members, this time, since we're going to perform several tests with different data, we initialize multiple `Review` objects in the `setUp()`. In this case, Review objects.

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

With this, we'll be able to declare the tests. We'll follow the same Given / When / Then structure. Only this time, the result of calling the method to be tested will be a Mono or Flux, and the assertions in Then will be made with StepVerifier:

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

Example of a negative case:

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

# Testing with Kafka and Spring WebFlux

## Consumers and publishers with Flux KafkaTemplate

### How are Kafka streams modeled in reactive code?

- In a reactive architecture with Spring WebFlux and Kafka, producers and consumers are modeled with `Flux` or `Mono`.
- A consumer can expose a `Flux<String>` or `Flux<Event>` that reacts to new incoming messages.
- A producer can take a `Flux<Event>` and publish them to Kafka using `KafkaTemplate`.

**Example of reactive publishing:**

```java
@Autowired
private ReactiveKafkaProducerTemplate<String, MyEvent> producer;

public Mono<Void> sendEvent(MyEvent event) {
    return producer.send("my-topic", event.getId(), event)
                   .then(); // returns Mono<Void> after publishing
}
```

**Example of reactive consumer:**

```java
@KafkaListener(topics = "my-topic", groupId = "group-id")
public void listen(String message) {
    // Reactive processing here
}
```

### Considerations for unit tests

- The `KafkaTemplate` and other Kafka components can be mocked to test logically without real connection.
- The actual message sending is not tested, only that it's invoked correctly.
- Use `StepVerifier` if `Mono` or `Flux` is returned.

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

## Integration tests with `@EmbeddedKafka`

### Basic configuration

- `@EmbeddedKafka` starts an in-memory Kafka broker.
- Useful for simulating real sending and consumption in integration tests.
- You need to configure `spring.kafka.bootstrap-servers` to point to the embedded broker.

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

        // We wait with some mechanism like CountDownLatch
        boolean received = consumerService.awaitMessage("Hello World");
        assertTrue(received);
    }
}
```

> With this, we can send messages to the embedded topic, and use listeners like those used in production to receive messages.

## Verification of sent and received messages

When working with Kafka and Spring WebFlux, **verifying that a message was actually sent and processed** can be complex due to the asynchronous nature of the system. Some common techniques for this are:

### 1. `CountDownLatch`

To wait for events in a controlled manner we'll use `CountDownLatch`, which is a Java synchronization tool very useful in tests integrated with Kafka. It allows **waiting for a message to be processed before making assertions**.

**Example**: a `@KafkaListener` that processes a notification and updates a service:

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
        bookService.process(message); // business logic
        latch.countDown(); // signal that the message was received
    }

    public CountDownLatch getLatch() {
        return latch;
    }
}
```

In the test:

```java
@Autowired
private BookKafkaConsumer consumer;

@Test
void shouldConsumeKafkaMessage() throws Exception {
    kafkaTemplate.send("book-notifications", "123", "test message");

    boolean processed = consumer.getLatch().await(5, TimeUnit.SECONDS);

    assertTrue(processed, "The message was not processed in time");
}
```

### 2. `StepVerifier`

When message processing is done reactively (for example, `Flux<String>`), we can use `StepVerifier` to verify that the flow behaves as expected:

```java
Flux<String> messageFlux = kafkaReceiver.receive()
    .map(ReceiverRecord::value)
    .take(1); // assuming we expect only one message

StepVerifier.create(messageFlux)
    .expectNext("Expected message")
    .expectComplete()
    .verify(Duration.ofSeconds(5));
```

### 3. `@SpyBean` or `@MockBean`

We can use these annotations to verify internal calls, because they allow **spying** or **mocking** beans within the Spring context and then verify interactions.

For example, if we expect the `BookService` to process the received message:

```java
@SpyBean
private BookService bookService;

@Test
void shouldCallBookServiceWhenKafkaMessageReceived() throws Exception {
    kafkaTemplate.send("book-notifications", "123", "some book json");

    Thread.sleep(2000); // brief wait for the listener

    verify(bookService, times(1)).process(any(String.class));
}
```

> `@SpyBean` maintains the real behavior of the bean, but allows using Mockito.verify().

### Serialization and deserialization

In the context of an architecture based on **Kafka**, each message that is sent or received through a topic must **be serialized** (converted to a byte array) so it can be transported by the messaging system. When receiving it, the message must **be deserialized** to reconstruct the original object in memory.

When working with **Spring WebFlux** and Kafka, this serialization is usually done with libraries like:

- **JSON**: using Jackson (`ObjectMapper`) or integrated Kafka serializers.
- **Avro**: a more efficient and structured serialization system, useful in environments requiring schema validation and high performance.

### Why is it important to test this?

- Because a bad serializer or deserializer configuration can make the consumer fail silently or discard messages.
- Because if you change the message model, it's easy to break compatibility with previous versions.
- Because you can guarantee that messages comply with certain schemas (especially in Avro).

Serialization and deserialization tests allow us to ensure that:

- Messages **are correctly transformed** from Java objects to bytes and vice versa.
- No data is lost or corrupted.
- Schema requirements are met (structure, mandatory fields, types, etc.).

### Testing with Avro

- If you use Avro, make sure you have the generated schemas and serializers correctly configured.
- You can test individual serialization with a unit test:

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

### Testing with JSON

- Similar to the Avro case, you can test serialization with Jackson:

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

### Additional validations

- It's good practice to verify that required fields are present.
- You can use `@JsonInclude`, Bean Validation validations (`@NotNull`) or `JsonSchema` to validate structure.
- In the case of Avro, verify that the `schema` is aligned with the generated classes.

### Example with `KafkaIntegrationTest`

In this example, we validate that a **notification sent to Kafka** is processed correctly and creates a book in the system. We use:

- `@EmbeddedKafka`: to create an **in-memory Kafka cluster** only for tests.
- `KafkaTemplate`: to **send messages** manually during the test.
- `BookService`: to verify if the message has been processed correctly.
- `ObjectMapper`: to serialize the `BookNotification` object to JSON.

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

With `@EmbeddedKafka` we'll make a "mock" of the Kafka broker, in this case, with three topics. And we can configure the broker properties with `@TestPropertySource`. Finally, we'll define test as the active profile with `@ActiveProfiles`("test"), and make database operations revert when tests finish using `@Transactional`.

Then, we'll declare the attributes we're going to need. In this case:

- a KafkaTemplate object to be able to send messages to each topic
- an instance of BookService and another of BookKafkaProducerService. We don't use mocks because friendly reminder that this is an integration test.
- ObjectMapper for serialization
- Two auxiliary members: one for the request, and another with the entity, which in this case is a notification, since we're testing events.

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

Then, we'll define our `setUp()` method to initialize the two auxiliary attributes:

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

And we'll be ready to declare the tests. In this case, the `shouldSendAndReceiveBookNotification` test seeks to validate that when sending a notification to a Kafka topic, the system:

- **Consumes the message correctly** (from a `@KafkaListener`)
- **Processes the associated business logic** (create a book)
- **And reflects it in the database** (through `bookService`)

We'll follow the Given When Then structure as we've been doing so far.

Given: In this case, for data preparation we'll serialize the notification to JSON with the objectMapper, to be able to send it as a string to the topic (reminder that `KafkaTemplate<String, String>` works with text-type messages in this case).

When: We send the message to the topic using kafkaTemplate and the send method. In this case, when we send it, what we expect is a confirmation from Kafka, which has to arrive within 5 seconds in this case `.get(5, TimeUnit.SECONDS)`.

Then: We'll wait with sleep for the Kafka consumer to process the message, activating the logic to create the book in the system.

After this, we do retries in case, even having given a margin with sleep, the process hadn't been carried out. In this case, we're applying a progressive retry strategy, up to 10 retries with 1s pauses. The retry loop will break if it's no longer necessary.

Finally, we make the necessary assertions to check that the book has been created correctly.

```java
@Test
@DisplayName("Should send and receive book notification successfully")
void shouldSendAndReceiveBookNotification() throws Exception {
    // Given - Convert the notification to JSON
    String notificationJson = objectMapper.writeValueAsString(testNotification);

    // When - Send the message to Kafka
    kafkaTemplate.send("book-notifications", testNotification.getIsbn(), notificationJson)
        .get(5, TimeUnit.SECONDS); // wait for Kafka to confirm

    // Then - Wait for it to be processed and verify the result
    Thread.sleep(3000); // allow the listener to consume and process

    // Retry to avoid flakiness in slow environments
    BookResponse createdBook = null;
    for (int i = 0; i < 10; i++) {
        try {
            createdBook = bookService.getBookByIsbn(testNotification.getIsbn());
            if (createdBook != null && testNotification.getTitle().equals(createdBook.getTitle())) {
                break;
            }
        } catch (Exception e) {
            // Not found yet
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

About the flakiness thing: In asynchronous environments like Kafka, processing is not immediate. Therefore, it's recommended to do short retries before directly failing the test.

### Best practices:

- **Avoid fixed sleeps** when you can, but in embedded Kafka sometimes it's necessary to use `Thread.sleep()` + retries to give time for processing.
- It's important to verify **all received fields** to ensure the message has been deserialized correctly.
- If you use Avro instead of JSON, you'll change the serializer and probably won't need `ObjectMapper`.
