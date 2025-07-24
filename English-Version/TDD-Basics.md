# TDD (Test-Driven Development)

### What is Test-Driven Development (TDD)

It's a development methodology where **automated tests are written before functional code**. This helps clearly define expected behavior from the beginning.

The goal is to design functionality based on what it should do (not on how to implement it internally).

> It's a way to design more robust software that's less prone to errors.

### The Testing Pyramid

Tests can be organized at several levels:

- Unit tests: They form the base of the pyramid, are isolated tests from the rest of the system, simple and fast. Therefore, we'll have a greater quantity of them.
- Integration tests: They test a subset of the system, involving several groups or units in a single test. They verify how various components interact with each other. They're more complicated to write and maintain, and slower than the previous ones.
- End-to-end tests: They use the same interface that the user would use to test the system, like a web browser. They're at the top of the pyramid, as they are slower and more fragile, simulating user interactions.

```
                  ▲
                  │  More Integration
      Slower ◄────┤
                 /  \
                /    \
               /      \
              /    E2E \
             /____Tests_\
            /            \
           /   Integration\
          /_________Tests__\
         /                  \
        /               Unit \
       /________________Tests_\
     Faster ◄─────┤
                  │  More Isolation
                  ▼

               

```

> The higher we go in the pyramid, the more expensive and slower the tests are, so we need to use them wisely.

### The Red-Green-Refactor Loop

This cycle guides TDD development:

1. **Red**: We write a test that fails (because the functionality doesn't exist yet).
2. **Green**: We implement the minimum code necessary for the test to pass.
3. **Refactor**: We improve the code while keeping all tests green.

> This allows us to advance step by step safely, keeping the system clean and working.

---

# Unit Tests

Unit tests are automated tests that verify code units, for example, a method in isolation from the rest of the system.

Mainly, we have two tools for unit testing: `JUnit`, which will help us create and run tests, and `Mockito`, which will serve to simulate external dependencies.

Some key annotations will be:

- `@Test`: marks a method as a test
- `@BeforeEach` / `@AfterEach`: setup and teardown before or after each test
- `@Mock`: creates a mock (simulated object)
- `@InjectMocks`: injects mocks into the class to be tested
- `@ExtendWith(MockitoExtension.class)`: activates Mockito support. This annotation will be used for the complete test class.

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("Book Service Tests")
class BookServiceTest { 
	(...)
}
```

In case it wasn't clear, with `@Mock` we create a mock of a class or interface that we'll need when testing. For example, to test a service, we'll need a repository mock, so we don't use the real one. On the other hand, with `@InjectMocks` we create a real instance of the object to be tested (like a service) and automatically inject the mocks it needs.

### AAA Pattern: Arrange - Act - Assert

It's a very common convention in testing to structure test methods:

1. **Arrange (Prepare):** We configure the test environment (input data, mocks, objects).
2. **Act (Execute):** We execute the method we want to test.
3. **Assert (Verify):** We verify that the result is as expected (comparison or effect verification).

This helps make test code more readable and maintainable.

## Simple Unit Tests

In this case, we'll be seeing examples outside the context of an API, in simple exercises like a Celsius to Fahrenheit converter, a FizzBuzz, or a palindrome detector. We use **pure JUnit**, without Mockito (because there are no dependencies to simulate).

### Parameterized Tests

To be able to test different inputs automatically we use: `@ParameterizedTest`. It allows executing the same test multiple times with different values. Some data sources for parameterized tests are:

- `@ValueSource`: simple values (strings, ints, etc.)
- `@CsvSource`: lists of pairs or more values, separated by commas.
- `@NullSource`: passes `null` as argument.
- `@EmptySource`: passes empty values (like `""` or `Collections.emptyList()`).
- `@NullAndEmptySource`: combines both previous ones.

Other useful annotations are:

- `@RepeatedTest(n)`: executes the same test `n` times (useful for detecting flakiness).
- `@DisplayName("...")`: improves test readability in the console.

Regarding the test itself, we can use different assertions:

- `assertEquals(expected, actual, msg)`: to ensure equality between expected and obtained result. The third parameter `msg` is optional.
- `assertTrue(condition)` / `assertFalse(condition)`
- `assertThrows(Exception.class, () -> {...})`
- `assertNotNull(obj)` / `assertNull(obj)`
- `assertArrayEquals(...)`

### Examples

```java
    @ParameterizedTest
    @CsvSource({
            "0, 32",
            "100, 212",
            "-40, -40",
            "25, 77",
            "37, 98.6",
            "-10, 14",
            "15, 59"
    })
    public void testToFahrenheit(double temperature, double expected) {
        double result = converter.toFahrenheit(temperature);
        assertEquals(expected, result, 0.01);
    }
    @RepeatedTest(50)
    void testToFahrenheitRepeatedly() {
        double result = converter.toFahrenheit(0);
        assertEquals(32, result, 0.0);
    }
```

```java
        @ParameterizedTest
        @DisplayName("Should return true for valid palindromes")
        @ValueSource(strings = {
                "racecar",
                "a man a plan a canal Panama",
                "was it a car or a cat I saw",
                "madam",
                "noon",
                "level",
                "a",
                ""
        })
        void shouldReturnTrueForPalindromes(String palindrome) {
            assertTrue(StringUtils.isPalindrome(palindrome),
                    "'" + palindrome + "' should be recognized as a palindrome");
        }
```

```java
@ParameterizedTest
@DisplayName("Should handle null or empty inputs")
@NullAndEmptySource
void shouldHandleNullOrEmpty(String input) {
    assertFalse(StringUtils.isPalindrome(input));
}
```

## Service Testing

We'll test the internal service logic without calling real repositories. For this, we have the following methods:

| Method | Description |
| --- | --- |
| `when(...).thenReturn(...)` | Defines simulated behavior of a mock |
| `verify(...)` | Verifies that a method was called |
| `verify(..., never())` | Verifies that a method was **not** called |
| `verify(..., times(n))` | Verifies that it was called **exactly n times** |
| `assertEquals(...)` | Compares expected and actual values |
| `assertThrows(...)` | Verifies that an exception is thrown |
| `any(Class.class)` | Accepts any instance of a class (Mockito matcher) |

### Example with `BookServiceTest`

In the following example, to test the different methods of the `BookService` class, we'll create a `BookServiceTest` class with the following attributes:

- `private BookRepository bookRepository` - to simulate database access
- `private BookService bookService` - the real service instance with mocked dependencies

(These are auxiliary models):

- `private BookRequest bookRequest;` - the input we receive
- `private Book book;` - the persisted entity model
- `private BookResponse bookResponse;` - the response model

```java
class BookServiceTest {
    
    @Mock
    private BookRepository bookRepository;
    
    @InjectMocks
    private BookService bookService;
    
    private BookRequest bookRequest;
    private Book book;
    private BookResponse bookResponse;
    (...)
  }  

```

It will be a common practice to nest classes to organize tests. For example, if we were to make several tests for the same operation. In this case, the teacher decided to nest them all within the `CrudOperations` class - don't ask me why if he's going to put them all there anyway, but oh well xd.

And defines the setUp method, with the BeforeEach annotation, to indicate that it's a method that will be triggered before each test. So, before each test, we'll have a `bookRequest` already initialized with each of its fields filled out, and the same with a `Book` entity that then uses to initialize `bookResponse`. This way, all three auxiliary attributes of the `BookServiceTest` class would be initialized.

```java
@Nested
    class CrudOperations {
		@BeforeEach
		
		 (...)
		}
```

Finally, we'll add as many tests as we need. Usually, one for each method of the class to be tested.

```java
@Test
		@DisplayName("Should create a book successfully")
		void shouldCreateBook() {
			//Arrange
			when(bookRepository.existsByIsbn(bookRequest.getIsbn())).thenReturn(false);
			when(bookRepository.save(any(Book.class))).thenReturn(book);
			
			//Act
			BookResponse createdBook = bookService.createBook(bookRequest);

			//Assert
			assertNotNull(createdBook);
			assertEquals(bookRequest.getTitle(), createdBook.getTitle());
			assertEquals(bookRequest.getAuthor(), createdBook.getAuthor());
			assertEquals(bookRequest.getIsbn(), createdBook.getIsbn());
			assertEquals(bookRequest.getDescription(), createdBook.getDescription());
			verify(bookRepository, times(1)).existsByIsbn(bookRequest.getIsbn());
			verify(bookRepository, times(1)).save(any(Book.class));
			
		}
```

It will also be good practice to test negative cases, for example, for the previous method, check that a book is not created if one already exists with its identifier:

```java
    @Test
    @DisplayName("Should throw DuplicateIsbnException when creating book with existing ISBN")
    void shouldThrowDuplicateIsbnExceptionWhenCreatingBookWithExistingIsbn() {
        // Given
        when(bookRepository.existsByIsbn(bookRequest.getIsbn())).thenReturn(true);
        
        // When & Then
        assertThrows(DuplicateIsbnException.class, () -> {
            bookService.createBook(bookRequest);
        });
        
        verify(bookRepository).existsByIsbn(bookRequest.getIsbn());
        verify(bookRepository, never()).save(any(Book.class));
    }
```

## Controller Testing

When testing controllers, we'll test the HTTP endpoints exposed by our controllers, with the idea of verifying the following:

- That the **HTTP response** (code, body, headers) is as expected.
- That there's **correct interaction with underlying services**.
- That **real scenarios are simulated** without needing to launch the entire application context.

Our main tool in this case will be `MockMvc`, which allows simulating HTTP requests and verifying responses without needing to launch the web server. To use it, we should annotate the test class with:

```java
@WebMvcTest(ControllerName.class)
@AutoConfigureMockMvc
```

And we should also define the following as main attributes:

```java
@MockBean
private BookService bookService;  // Simulation of the injected service

@Autowired
private MockMvc mockMvc;          // Client to simulate requests
```

In this case, we use `@MockBean` instead of `@Mock` as we did in the service. In this case, with MockBean SpringBoot injects the mock within the ApplicationContext. (I don't really know what this implies but there you go)

Some key methods will be:

| Method | Description |
| --- | --- |
| `mockMvc.perform(...)` | Executes a simulated HTTP request |
| `andExpect(...)` | Verifies the result (code, body, etc.) |
| `content()` | Allows access to body content |
| `jsonPath(...)` | Allows access to parts of the JSON |
| `doNothing().when(...)` | Simulates a `void` call (useful for `delete`) |
| `status().isOk()` | Verifies code 200 |
| `status().isCreated()` | Verifies code 201 |
| `status().isNotFound()` | Verifies code 404 |
| `status().isBadRequest()` | Verifies code 400 |

### How to use `.perform()`

The `.perform(...)` method of `MockMvc` is what we'll use to simulate HTTP calls. We can configure it with:

- `post("/endpoint/path")`, `get(...)`, `put(...)`, `delete(...)` to define the HTTP type.
- `.content(...)` to send a body (JSON).
- `.contentType(...)` to specify the body type (for example, `application/json`).
- `.accept(...)` to indicate what type of response the client expects.

> Whenever a JSON is sent in the body, we should use `contentType(MediaType.APPLICATION_JSON)` and serialize with `objectMapper.writeValueAsString(...)`.

### Example with `BookControllerTest`

In this case, the attributes we'll need will be:

- `private BookService bookService` annotated with `@MockitoBean` (or `@MockBean` in Java21)
- `private MockMvc mockMvc`

And these auxiliary attributes:

- `private ObjectMapper objectMapper;`
- `private BookRequest bookRequest;`
- `private BookResponse bookResponse;`
- `private Book book;`

```java
@WebMvcTest(BookController.class)
@DisplayName("Book Controller Tests with WebMvcTest")
class BookControllerTest {
    
    @MockitoBean
    private BookService bookService;

    @Autowired
    private MockMvc mockMvc;

    private ObjectMapper objectMapper;
    private BookRequest bookRequest;
    private BookResponse bookResponse;
    private Book book;
    (...)
  }
```

We'll define the `setUp()` method similarly to how we did in `BookServiceTest`

```java
  @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper();

        bookRequest = new BookRequest(
            "Test Book",
            "Test Author",
            "1234567890",
            "Test Description",
            new BigDecimal("29.99"),
            10,
            BookCategory.FICTION
        );

        book = new Book(
            "Test Book",
            "Test Author",
            "1234567890",
            "Test Description",
            new BigDecimal("29.99"),
            10,
            BookCategory.FICTION
        );
        book.setId(1L);

        bookResponse = new BookResponse(book);
    }
```

And we'll be ready to define methods for each test.

```java
@Test
    @DisplayName("Should create book successfully")
    void shouldCreateBook() throws Exception {
		// Given
		when(bookService.createBook(any(BookRequest.class))).thenReturn(bookResponse);

		// When & Then
		mockMvc.perform(post("/api/books")
				.contentType(MediaType.APPLICATION_JSON)
				.content(objectMapper.writeValueAsString(bookRequest)))
			.andExpect(status().isCreated())
			.andExpect(content().contentType(MediaType.APPLICATION_JSON))
			.andExpect(jsonPath("$.title").value(book.getTitle()))
			.andExpect(jsonPath("$.author").value(book.getAuthor()))
			.andExpect(jsonPath("$.isbn").value(book.getIsbn()))
			.andExpect(jsonPath("$.description").value(book.getDescription()))
			.andExpect(jsonPath("$.price").value(book.getPrice().doubleValue()))
			.andExpect(jsonPath("$.stock").value(book.getStock()))
			.andExpect(jsonPath("$.category").value(book.getCategory().name()));
    }

```

There will be tests for which we need to simulate lists:

```java

    @Test
    @DisplayName("Should get all books")
    void shouldGetAllBooks() throws Exception {
        // Given
        List<BookResponse> books = Arrays.asList(bookResponse, bookResponse);
        when(bookService.getAllBooks()).thenReturn(books);

        // When & Then
        mockMvc.perform(get("/api/books")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$").isArray())
            .andExpect(jsonPath("$.length()").value(2))
            .andExpect(jsonPath("$[0].id").value(bookResponse.getId()))
            .andExpect(jsonPath("$[0].title").value(bookResponse.getTitle()))
            .andExpect(jsonPath("$[0].author").value(bookResponse.getAuthor()))
            .andExpect(jsonPath("$[0].isbn").value(bookResponse.getIsbn()))
            .andExpect(jsonPath("$[0].description").value(bookResponse.getDescription()))
            .andExpect(jsonPath("$[0].price").value(bookResponse.getPrice().doubleValue()))
            .andExpect(jsonPath("$[0].stock").value(bookResponse.getStock()))
            .andExpect(jsonPath("$[0].category").value(bookResponse.getCategory().name()))
            .andExpect(jsonPath("$[1].id").value(bookResponse.getId()))
            .andExpect(jsonPath("$[1].title").value(bookResponse.getTitle()))
            .andExpect(jsonPath("$[1].author").value(bookResponse.getAuthor()))
            .andExpect(jsonPath("$[1].isbn").value(bookResponse.getIsbn()))
            .andExpect(jsonPath("$[1].description").value(bookResponse.getDescription()))
            .andExpect(jsonPath("$[1].price").value(bookResponse.getPrice().doubleValue()))
            .andExpect(jsonPath("$[1].stock").value(bookResponse.getStock()))
            .andExpect(jsonPath("$[1].category").value(bookResponse.getCategory().name()));
    }
    
```

In the case of delete, since what's expected is an empty response, we'll use `doNothing`:

```java
   @Test
    @DisplayName("Should delete book successfully")
    void shouldDeleteBook() throws Exception {
        // Given
        doNothing().when(bookService).deleteBook(1L);

        // When & Then
        mockMvc.perform(delete("/api/books/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isNoContent());
    }
```

---

# Integration Tests

**Integration tests** validate that various parts of the system work correctly **together**, for example: that a `Service` can interact correctly with a `Repository` or that a message sent by Kafka is received and processed correctly. Generally, they'll cover the entire path of a request, from the controller to the repository.

While in **unit tests** we isolate logic with mocks, in integration tests we **use real components** (or very close simulations, like an embedded database or embedded Kafka broker).

In the Spring Boot context, integration tests usually launch **part or all of the application context**, using annotations like `@SpringBootTest`.

Some common annotations for this are:

| Annotation | Description |
| --- | --- |
| `@SpringBootTest` | Starts the complete Spring context (as if the real app was running) |
| `@AutoConfigureMockMvc` | Configures `MockMvc` to test controllers without starting the server |
| `@DataJpaTest` | Launches only the JPA layer (repositories + embedded datasource) |
| `@Testcontainers` | To use real Docker containers in tests (like PostgreSQL or Kafka) |
| `@EmbeddedKafka` | Creates an embedded Kafka broker to test message production and consumption |
| `@Transactional` | Not exactly for this, but it will be very useful. (I'll explain better later) |

The basic structure of integration tests will be very similar to what we've been seeing so far, as it also follows the AAA pattern (Arrange, Act, Assert, for those without retention capacity).

A simple example could have the following form:

```java
@SpringBootTest
@AutoConfigureMockMvc
class BookControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldCreateBook() throws Exception {
		    // Given - we simulate a Json object
        String json = """
            {
                "title": "Clean Code",
                "author": "Robert C. Martin"
            }
        """;
				// When & Then: we perform the call and expect the object to have been created
        mockMvc.perform(post("/books")
                .contentType(MediaType.APPLICATION_JSON)
                .content(json))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.title").value("Clean Code"));
    }
}

```

This test launches the real context and tests the books API as if it were done from Postman.

## Real connections between layers

In a typical integration test you might want to:

- Verify that a `Service` correctly saves an entity in the database.
- Check that a `Controller` responds with data brought from the `Repository`.
- Validate that a Kafka message sent from a publisher is received by the consumer and processed correctly.

Therefore, it's **normal that mocks are not used in integration tests**, or they're used only in very specific cases.

### Complementary tools

- `Testcontainers`: if you need to test with real services like PostgreSQL, Redis or Kafka. Uses Docker containers in tests.
- `EmbeddedKafka`: if your microservice produces or consumes from Kafka, you can simulate the Kafka broker in memory.
- `@SpyBean`: if you need to verify that a real method was called during a real flow.
- `JdbcTemplate`: can be useful to directly verify the database state after executing a test.

> We'll see about embedded kafka and SpyBean in reactive TDD. (It would be great if I remembered to put the link here, but if not, you can search for it yourselves)

## Best practices

### Using test profiles with `application-test.yml`

When we run tests (especially integration ones), **we don't want to use the same configuration as in development or production**. For example, it wouldn't be a good idea to run tests against a real database or send real messages through Kafka.

For this, Spring Boot allows using **specific profiles**, like the `test` profile, which activates alternative configurations. This is done with a separate file called:

```
src/test/resources/application-test.yml
```

In which we can redefine any property of our application

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  kafka:
    bootstrap-servers: ${spring.embedded.kafka.brokers}

```

Then, we simply activate that profile in your tests with the `@ActiveProfiles` annotation:

```java
@SpringBootTest
@ActiveProfiles("test")
class BookServiceIntegrationTest {
    // ...
}
```

> This ensures that all beans, configurations and resources loaded are those of the test environment: embedded database, embedded Kafka broker, mocked endpoints, etc.

### `@Transactional` annotation in tests

When we write integration tests that interact with the database, **it's common for data to persist if we don't delete it manually**. To avoid having to clean the database after each test, we can use the `@Transactional` annotation directly on the test class or method.

```java
@SpringBootTest
@Transactional
class BookRepositoryIntegrationTest {
    // ...
}
```

With this, what we achieve is to execute each test within a database transaction **that automatically reverts when the test finishes**, as if nothing had happened.

> Pro tip: If for some reason we needed the data to persist (for example, to make a second verification), we can annotate that test with @Commit or remove @Transactional.

And if we want to make sure the database is cleaned between tests, we can always prepare a method with `@BeforeEach` that deletes the tables:

```java
@BeforeEach
void cleanDatabase() {
    repository.deleteAll();
}

```

### Example with `BookIntegrationTest`

In our example, we're going to mark the class as an **integration test** using `@SpringBootTest`, which **starts the complete Spring context**, as if the real application were running.

We also add:

- `@AutoConfigureWebMvc`: which internally prepares the necessary configurations of the web stack (MVC). That is, **loads beans related to controllers, converters, view resolvers, validators, etc.** Basically, everything Spring needs to process an HTTP request.
- `@AutoConfigureMockMvc`: automatically creates a `MockMvc` bean ready to be injected with `@Autowired`, which allows us **to simulate HTTP calls without starting a real server**.
- `@ActiveProfiles("test") and @Transactional`: which we just talked about.

When we use `@SpringBootTest`, there are **two main ways** to prepare `MockMvc`, the first is what we're seeing in this example, with automatic configuration. If we wanted to make some customization (filters, security configurations, etc), we would need an object of type `WebApplicationContext`, which we would then set with:

```java
@Autowired
private WebApplicationContext webApplicationContext;
(...)
mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
```

The members of our class will be: an `ObjectMapper` to be able to serialize objects to JSON, the `MockMvc`, and the repository, to be able to prepare and verify the database state. In this case, we're using the real repository, not a mock, injected with `@Autowired`, because we want to test the complete functioning of the endpoints.

```java
@SpringBootTest
@AutoConfigureWebMvc
@ActiveProfiles("test")
@Transactional
@AutoConfigureMockMvc
@DisplayName("Book Integration Tests")
class BookIntegrationTest {
        
    @Autowired
    private BookRepository bookRepository;
    @Autowired
    private ObjectMapper objectMapper;
     @Autowired
    private MockMvc mockMvc;
    (...)
}
```

In our setUp method, we can empty the repository to make sure tests don't affect each other.

```java
   @BeforeEach
    void setUp() {
        bookRepository.deleteAll();
    }
    
```

And we'll be ready to declare the tests. As you can see, it's quite similar to the tests we did in the controller. We create a request object, which we'll then serialize with `ObjectMapper` to add it to the request content, and we'll perform the call, to finally check the result:

```java
    @Test
    @DisplayName("Should create and retrieve a book")
    void shouldCreateAndRetrieveBook() throws Exception {
        // Given
        BookRequest bookRequest = new BookRequest(
            "Test Book",
            "Test Author",
            "1234567890",
            "Test Description",
            new BigDecimal("29.99"),
            10,
            BookCategory.FICTION
        );
        
        // When & Then - Create book
        String createResponse = mockMvc.perform(post("/api/books")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bookRequest)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.title").value("Test Book"))
                .andExpect(jsonPath("$.author").value("Test Author"))
                .andExpect(jsonPath("$.isbn").value("1234567890"))
                .andReturn()
                .getResponse()
                .getContentAsString();
        
        BookResponse createdBook = objectMapper.readValue(createResponse, BookResponse.class);
        
        // When & Then - Retrieve book
        mockMvc.perform(get("/api/books/" + createdBook.getId()))
                .andExpected(status().isOk())
                .andExpect(jsonPath("$.id").value(createdBook.getId()))
                .andExpect(jsonPath("$.title").value("Test Book"));
    }
```
