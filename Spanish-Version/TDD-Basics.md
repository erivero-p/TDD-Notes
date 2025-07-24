# TDD (Testing Driven Development)

### Qué es el Test-Driven Development (TDD)

Es una metodología de desarrollo en la que **las pruebas automatizadas se escriben antes del código funcional**. Esto ayuda a definir claramente el comportamiento esperado desde el principio.

El objetivo es diseñar la funcionalidad basándose en lo que debe hacer (y no en cómo hacerlo internamente).

> Es una forma de diseñar software más robusto y menos propenso a errores.
>

### La Pirámide del Testeo

Los tests pueden organizarse en varios niveles:

- Unit tests: Constituyen la base de la pirámide, son test aislados del resto del sistema, simples y rápidos. Tendremos, por tanto, una mayor cantidad de los mismos.
- Integration tests: Ponen a prueba un subconjunto del sistema, implicando varios grupos o unidades en un único test. Verifican cómo interactúan varios componentes entre sí.  Son más complicados de escribir y mantener, y más lentos que los anteriores.
- End-to-end tests: Utilizarán la misma interfaz que el usuario use para testear el sistema, como un buscador web. Están en la cúspide de la pirámide, ya que, son más lentos y frágiles, al simular la interacción de los usuarios.

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

> Cuanto más subimos en la pirámide, más costosos y lentos son los tests, así que hay que usarlos con cabeza.
>

### The Red-Green-Refactor Loop

Este ciclo guía el desarrollo en TDD:

1. **Red**: Escribimos una prueba que falla (porque la funcionalidad aún no existe).
2. **Green**: Implementamos el código mínimo necesario para que la prueba pase.
3. **Refactor**: Mejoramos el código manteniendo todas las pruebas en verde.

> Esto permite avanzar paso a paso con seguridad, manteniendo el sistema limpio y funcionando.
>

---

# Tests Unitarios

Los test unitarios son pruebas automáticas que verifican unidades de código, por ejemplo, un método de forma aislada al resto del sistema.

Principalmente, contamos con dos herramientas para hacer los test unitarios: `JUnit`, que nos ayudará a crear y ejecutar los test, y `Mockito`, que servirá para simular dependencias externas.

Algunas anotaciones clave serán:

- `@Test`: marca un método como test
- `@BeforeEach` / `@AfterEach`: setup y teardown antes o después de cada test
- `@Mock`: crea un mock (objeto simulado)
- `@InjectMocks`: inyecta mocks en la clase a testear
- `@ExtendWith(MockitoExtension.class)`: activa soporte de Mockito. Esta anotación se usará para la clase completa que hagamos los test.

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("Book Service Tests")
class BookServiceTest { 
	(...)
}
```

Por si no estaba claro, con `@Mock` creamos un mock de una clase o interfaz que vayamos a necesitar a la hora de testear. Por ejemplo, para testear un service, necesitaremos un mock de repositorio, para no usar el real. Por otro lado, con `@InjectMocks` creamos una instancia real del objeto a testear (como un service) e inyecta automáticamente los mocks que necesita.

### Patrón AAA: Arrange - Act - Assert

Es una convención muy común en testing para estructurar los métodos de test:

1. **Arrange (Preparar):** Configuramos el entorno del test (datos de entrada, mocks, objetos).
2. **Act (Actuar):** Ejecutamos el método que queremos testear.
3. **Assert (Afirmar):** Verificamos que el resultado sea el esperado (comparación o verificación de efectos).

Esto ayuda a que el código del test sea más legible y mantenible.

## Test Unitarios Simples

En este caso, estaremos viendo ejemplos fuera del contexto de una  API, en ejercicios simples como un conversor de Celsius a Farenheit, un FizzBuzz, o un detector de palíndromos. Usamos **JUnit puro**, sin Mockito (porque no hay dependencias que simular).

### Test parametrizados

Para poder testear distintos imputs de forma automática usamos: `@ParameterizedTest`.Permite ejecutar el mismo test varias veces con distintos valores. Algunas fuentes de datos para los test parametrizados son:

- `@ValueSource`: valores simples (strings, ints, etc.)
- `@CsvSource`: listas de pares o más valores, separados por comas.
- `@NullSource`: pasa `null` como argumento.
- `@EmptySource`: pasa valores vacíos (como `""` o `Collections.emptyList()`).
- `@NullAndEmptySource`: combina ambas anteriores.

Otras anotaciones útiles son:

- `@RepeatedTest(n)`: ejecuta el mismo test `n` veces (útil para detectar flakiness).
- `@DisplayName("...")`: mejora la legibilidad del test en la consola.

Respecto al test en sí, podremos utilizar diferentes aserciones:

- `assertEquals(expected, actual, msg)`: para asegurarnos de la igualdad entre el resultado esperado y el obtenido. El tercer parámetro `msg` es opcional.
- `assertTrue(condición)` / `assertFalse(condición)`
- `assertThrows(Exception.class, () -> {...})`
- `assertNotNull(obj)` / `assertNull(obj)`
- `assertArrayEquals(...)`

### Ejemplos

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
    public void testToFarenheit(double temperature, double expected) {
        double result = converter.toFahrenheit(temperature);
        assertEquals(expected, result, 0.01);
    }
    @RepeatedTest(50)
    void testToFarenheitRepeatedly() {
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

## Testing de Service

Se testeará la lógica interna del service sin llamar a repositorios reales. Para esto, contamos con los siguientes métodos:

| Método | Descripción |
| --- | --- |
| `when(...).thenReturn(...)` | Define el comportamiento simulado de un mock |
| `verify(...)` | Verifica que un método fue llamado |
| `verify(..., never())` | Verifica que **no** se llamó un método |
| `verify(..., times(n))` | Verifica que se llamó **exactamente n veces** |
| `assertEquals(...)` | Compara valores esperados y reales |
| `assertThrows(...)` | Verifica que se lanza una excepción |
| `any(Class.class)` | Acepta cualquier instancia de una clase (matcher de Mockito) |

### Ejemplo con `BookServiceTest`

En el siguiente ejemplo, para testear los distintos métodos de la clase `BookService`, crearemos una clase `BookServiceTest` con los siguientes atributos:

- `private BookRepository bookRepository` - para simular el acceso a la base de datos
- `private BookService bookService` -  la instancia real del service con dependencias mockeadas

(Estos son modelos auxiliares):

- `private BookRequest bookRequest;` - el input que recibimos
- `private Book book;` - el modelo de entidad persistida
- `private BookResponse bookResponse;` - el modelo de respuesta

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

Será una práctica común anidar clases para organizar los test. Por ejemplo, si fuesemos hacer varios test para una misma operación. En este caso, el profesor ha decidido anidarlos todos dentro de la clase `CrudOperations`  no me preguntéis por qué si total, los va a meter todos ahí pero bueno xd.

Y define el método setUp, con la anotación BeforeEach, para señalar que es un método que será triggereado antes de cada test. Así, antes de cada uno de los test, contaremos con una `bookRequest` ya inicializada con cada uno de sus campos cumplimentados, y lo mismo con una entidad de `Book` que luego utiliza para inicializar `bookResponse`.  De esta forma, ya estarían inicializados los trés atributos auxiliares de la clase `BookServiceTest`.

```java
@Nested
    class CrudOperations {
		@BeforeEach
		
		 (...)
		}
```

Finalmente, añadiremos tantos test como necesitemos. Habitualmente, uno para cada método de la clase a testear.

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

Además será buena práctica testear los casos negativos, por ejemplo, para el anterior método, comprobar que no se cree un libro si ya existe uno con su identificador:

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

## Testing de Controller

A la hora de testear los controllers, probaremos los endpoints HTTP expuestos por nuestros controladores, con la idea de verificar lo siguiente:

- Que la **respuesta HTTP** (código, cuerpo, cabeceras) sea la esperada.
- Que se **interactúe correctamente con los servicios** subyacentes.
- Que se **simulen escenarios reales** sin necesidad de levantar todo el contexto de la aplicación.

Nuestra herramienta principal en este caso, será `MockMvc`, que permite simular peticiones HTTP, y verificar las respuestas sin necesidad de lanzar el servidor web. Para utilizarlo, deberemos anotar la clase test con:

```java
@WebMvcTest(ControllerName.class)
@AutoConfigureMockMvc
```

Y deberemos también definir como atributos principales lo siguiente:

```java
@MockBean
private BookService bookService;  // Simulación del service inyectado

@Autowired
private MockMvc mockMvc;          // Cliente para simular peticiones
```

En este caso, utilizamos `@MockBean` en lugar de `@Mock` como hacíamos en el service. En este caso, con MockBean SpringBoot inyecta el mock dentro del ApplicationContext. (no sé muy bien lo que esto implica pero ahí lo llevo)

Alguno de los métodos claves serán:

| Método | Descripción |
| --- | --- |
| `mockMvc.perform(...)` | Ejecuta una petición HTTP simulada |
| `andExpect(...)` | Verifica el resultado (código, body, etc.) |
| `content()` | Permite acceder al contenido del body |
| `jsonPath(...)` | Permite acceder a partes del JSON |
| `doNothing().when(...)` | Simula una llamada `void` (útil para `delete`) |
| `status().isOk()` | Verifica código 200 |
| `status().isCreated()` | Verifica código 201 |
| `status().isNotFound()` | Verifica código 404 |
| `status().isBadRequest()` | Verifica código 400 |

### Cómo usar `.perform()`

El método `.perform(...)` de `MockMvc` es el que utilizaremos para simular las llamadas HTTP. Lo podremos configurar con:

- `post("/endpoint/path")`, `get(...)`, `put(...)`, `delete(...)` para definir el tipo de HTTP.
- `.content(...)` para enviar un body (JSON).
- `.contentType(...)` para especificar el tipo del body (por ejemplo, `application/json`).
- `.accept(...)` para indicar qué tipo de respuesta espera el cliente.

> Siempre que se envíe un JSON en el cuerpo, deberemos utilizar  `contentType(MediaType.APPLICATION_JSON)` y serializa con `objectMapper.writeValueAsString(...)`.
>

### Ejemplo con `BookControllerTest`

En este caso, los atributos que necesitaremos serán:

- `private BookService bookService` anotado con `@MockitoBean` (o  `@MockBean` en Java21)
- `private MockMvc mockMvc`

Y estos atributos auxiliares:

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

Definiremos el método `setUp()` de forma similar a como hicimos en `BookServiceTest`

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

Y ya estaremos listos para definir los métodos para cada test.

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

Habrá tests para los que necesitemos simular listas:

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

En el caso de delete, dado que lo que se espera es respuesta vacía, utilizaremos `doNothing`:

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

# Tests de Integración

Los **tests de integración** validan que varias partes del sistema funcionan correctamente **en conjunto**, por ejemplo: que un `Service` pueda interactuar correctamente con un `Repository` o que un mensaje enviado por Kafka sea recibido y procesado correctamente. Generalmente, van a cubrir todo el camino de una request, desde el controller, hasta el repositorio.

Mientras que en los **tests unitarios** aislamos la lógica con mocks, en los de integración **usamos los componentes reales** (o simulaciones muy cercanas, como una base de datos embebida o un broker de Kafka embebido).

En el contexto de Spring Boot, los tests de integración suelen lanzar una **parte o todo el contexto de la aplicación**, usando anotaciones como `@SpringBootTest`.

Algunas anotaciones comunes para esto son:

| Anotación | Descripción |
| --- | --- |
| `@SpringBootTest` | Arranca todo el contexto de Spring (como si se ejecutara la app real) |
| `@AutoConfigureMockMvc` | Configura `MockMvc` para testear controladores sin arrancar el servidor |
| `@DataJpaTest` | Levanta solo la capa JPA (repositorios + datasource embebido) |
| `@Testcontainers` | Para usar contenedores Docker reales en los tests (como PostgreSQL o Kafka) |
| `@EmbeddedKafka` | Crea un broker Kafka embebido para testear producción y consumo de mensajes |
| `@Transactional` | No es exactamente para esto, pero será muy útil. (Luego lo explico mejor) |

La estructura básica de los test de integración va a ser muy similar a la que venimos viendo hasta ahora, ya que también sigue el patrón AAA (Arrange, Act, Assert, pa los que no tengan capacidad de retentiva).

Un ejemplo simple podría tener la siguiente forma:

```java
@SpringBootTest
@AutoConfigureMockMvc
class BookControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldCreateBook() throws Exception {
		    // Given - simulamos un objeto Json
        String json = """
            {
                "title": "Clean Code",
                "author": "Robert C. Martin"
            }
        """;
				// When & Then: performamos la llamada y esperamos que el objeto se haya creado
        mockMvc.perform(post("/books")
                .contentType(MediaType.APPLICATION_JSON)
                .content(json))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.title").value("Clean Code"));
    }
}

```

Este test lanza el contexto real y prueba la API de libros como si se hiciera desde Postman.

## Conexiones reales entre capas

En un test de integración típico podrías querer:

- Verificar que un `Service` guarda correctamente una entidad en la base de datos.
- Comprobar que un `Controller` responde con datos traídos del `Repository`.
- Validar que un mensaje Kafka enviado desde un publisher es recibido por el consumidor y procesado correctamente.

Por eso, es **normal que en los tests de integración no se usen mocks**, o se usen solo en casos muy específicos.

### Herramientas que complementan

- `Testcontainers`: si necesitas probar con servicios reales como PostgreSQL, Redis o Kafka. Usa contenedores Docker en tests.
- `EmbeddedKafka`: si tu microservicio produce o consume de Kafka, puedes simular el broker Kafka en memoria.
- `@SpyBean`: si necesitas verificar que un método real fue llamado durante un flujo real.
- `JdbcTemplate`: puede ser útil para verificar directamente el estado de la base de datos después de ejecutar un test.

> Veremos sobre embedded kafka y SpyBean en reactive TDD. (Estaría genial que me acordara de poner el enlace aquí, pero si no, lo buscáis vosotros)
>

## Buenas prácticas

### Usar perfiles de test con `application-test.yml`

Cuando corremos tests (especialmente de integración), **no queremos usar la misma configuración que en desarrollo o producción**. Por ejemplo, no sería buena idea correr los tests contra una base de datos real o enviar mensajes reales por Kafka.

Para esto, Spring Boot permite usar **perfiles específicos**, como el perfil `test`, que activa configuraciones alternativas. Esto se hace con un archivo separado llamado:

```
src/test/resources/application-test.yml
```

En el que podremos redefinir cualquier propiedad de nuestra aplicación

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

Después, simplemente activamos ese perfil en tus tests con la anotación `@ActiveProfiles`:

```java
@SpringBootTest
@ActiveProfiles("test")
class BookServiceIntegrationTest {
    // ...
}
```

> Esto asegura que todos los beans, configuraciones y recursos cargados sean los del entorno de test: base de datos embebida, broker de Kafka embebido, endpoints mockeados, etc.
>

### Anotación `@Transactional` en tests

Cuando escribimos tests de integración que interactúan con la base de datos, **es común que los datos persistan si no los eliminamos manualmente**. Para evitar tener que limpiar la base de datos después de cada test, podemos usar la anotación `@Transactional` directamente en la clase o método de test.

```java
@SpringBootTest
@Transactional
class BookRepositoryIntegrationTest {
    // ...
}
```

Con esto lo que conseguimos es ejecutar cada test dentro de una transacción de base de datos **que se revierte automáticamente al finalizar el test**, como si no hubiera pasado nada.

> Pro tip: Si por alguna razón necesitáramos que los datos persistan (por ejemplo, para hacer una segunda verificación), podemos anotar ese test con @Commit o quitarle @Transactional.
>

Y si queremos asegurarnos de que se limpie la base de datos entre tests, siempre podemos  preparar un método con `@BeforeEach` que borre las tablas:

```java
@BeforeEach
void cleanDatabase() {
    repository.deleteAll();
}

```

### Ejemplo con  `BookIntegrationTest`

En nuestro ejemplo, vamos a marcar la clase como un **test de integración** utilizando `@SpringBootTest`, que **arranca el contexto completo de Spring**, como si se ejecutara la aplicación real.

También añadimos:

- `@AutoConfigureWebMvc`: que prepara internamente las configuraciones necesarias del stack web (MVC). Es decir, **carga los beans relacionados con controladores, convertidores, resolvers de vistas, validadores, etc.** Básicamente, todo lo que Spring necesita para procesar una request HTTP.
- `@AutoConfigureMockMvc`: crea automáticamente un bean `MockMvc` listo para ser inyectado con `@Autowired`, lo que nos permite **simular llamadas HTTP sin arrancar un servidor real**.
- `@ActiveProfiles("test") y @Transactional`: de las que acabamos de hablar.

Cuando usamos `@SpringBootTest`, hay **dos formas principales** de preparar `MockMvc`, la primera es la que estamos viendo en este ejemplo, con la configuración automática. Si quisiéramos hacer alguna personalización (filtros, configuraciones de seguridad, etc), necesitaríamos un objeto de tipo `WebApplicationContext`, que luego setearíamos con:

```java
@Autowired
private WebApplicationContext webApplicationContext;
(...)
mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
```

Los miembros de nuestra clase serán: un `ObjectMapper` para poder serializar los objetos a JSON,  el `MockMvc`, y el repositorio, para poder preparar y verificar el estado de la base de datos. En este caso, estamos usando el repositorio real, no un mock, inyectado con `@Autowired`, porque queremos testear el funcionamiento completo de los endpoints.

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

En nuestro método setUp, podremos vaciar el repositorio para asegurarnos de que los tests no se afecten entre sí.

```java
   @BeforeEach
    void setUp() {
        bookRepository.deleteAll();
    }
    
```

Y ya estaremos listos para declarar los tests.  Como se puede ver, es bastante parecido a los test que hicimos en el controller. Creamos un objeto de la request, que luego serializaremos con `ObjectMapper` para añadirlo al contenido de la petición, y performaremos la llamada, para finalmente checkear el resultado:

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
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(createdBook.getId()))
                .andExpect(jsonPath("$.title").value("Test Book"));
    }
```