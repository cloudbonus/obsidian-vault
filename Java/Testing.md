
## JUnit 5

JUnit 5 = JUnit Platform + JUnit Jupiter + JUnit Vintage.

### Основные аннотации

| Аннотация            | Назначение                                                              |
| -------------------- | ----------------------------------------------------------------------- |
| `@Test`              | Тестовый метод                                                          |
| `@BeforeEach`        | Выполняется перед каждым тестом (замена `@Before`)                      |
| `@AfterEach`         | Выполняется после каждого теста (замена `@After`)                       |
| `@BeforeAll`         | Выполняется один раз перед всеми тестами класса (замена `@BeforeClass`) |
| `@AfterAll`          | Выполняется один раз после всех тестов класса (замена `@AfterClass`)    |
| `@DisplayName`       | Человекочитаемое имя теста                                              |
| `@ParameterizedTest` | Параметризованный тест                                                  |
| `@ValueSource`       | Источник простых значений для параметризации                            |
| `@CsvSource`         | Источник CSV-данных                                                     |
| `@Disabled`          | Отключение теста (замена `@Ignore`)                                     |
| `@Nested`            | Вложенные тестовые классы для группировки                               |
| `@ExtendWith`        | Регистрация расширений (замена `@RunWith`)                              |

### @ExtendWith (JUnit 5) vs @RunWith (JUnit 4)

| Критерий | `@RunWith` (JUnit 4) | `@ExtendWith` (JUnit 5) |
|----------|----------------------|-------------------------|
| **Архитектура** | Один раннер на класс | Модульные расширения, можно несколько |
| **Пример** | `@RunWith(SpringRunner.class)` | `@ExtendWith(SpringExtension.class)` |
| **Комбинация** | Невозможно комбинировать раннеры | `@ExtendWith({MockitoExtension.class, SpringExtension.class})` |

```java
// JUnit 4
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = TestConfig.class)
public class OldStyleTest { }

// JUnit 5
@ExtendWith(SpringExtension.class)
@SpringBootTest
class ModernTest { }

// JUnit 5: несколько расширений
@ExtendWith({MockitoExtension.class, SpringExtension.class})
class CombinedTest { }
```

### AssertJ (рекомендуемая альтернатива стандартным assert)

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

assertThat(user.getName()).isEqualTo("John");
assertThat(user.getAge()).isPositive();
assertThat(users).hasSize(3).extracting("name").contains("John", "Jane");

assertThatThrownBy(() -> service.divide(1, 0))
    .isInstanceOf(ArithmeticException.class)
    .hasMessageContaining("zero");
```

---

## Mockito

Фреймворк для создания mock-объектов.

### Основные аннотации

| Аннотация | Назначение |
|-----------|-----------|
| `@Mock` | Создаёт mock-объект |
| `@Spy` | Оборачивает реальный объект (частичный mock) |
| `@InjectMocks` | Внедряет моки в тестируемый объект |
| `@Captor` | Захват аргументов |

### verify

Проверка, что метод был вызван с нужными аргументами:

```java
verify(userRepository).findById(1L);                    // вызван ровно 1 раз
verify(userRepository, times(2)).findAll();            // вызван 2 раза
verify(userRepository, never()).delete(any());          // никогда не вызывался
verify(userRepository, atLeastOnce()).save(any());      // минимум 1 раз
```

### Spy и частичный mock

```java
// Для spy используем doReturn, иначе реальный метод вызовется при stubbing
@Spy
private List<String> spyList = new ArrayList<>();

doReturn(10).when(spyList).size(); // правильно
// when(spyList.size()).thenReturn(10); // вызовет реальный size() перед stubbing!
```

### Примеры

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserById() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User(1L, "John")));

        User user = userService.findById(1L);

        assertThat(user.getName()).isEqualTo("John");
        verify(userRepository).findById(1L);
    }

    @Test
    void shouldThrowWhenUserNotFound() {
        when(userRepository.findById(1L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.findById(1L))
            .isInstanceOf(UserNotFoundException.class);
    }
}
```

### Mockito vs WireMock

| Критерий | Mockito | WireMock |
|----------|---------|----------|
| **Уровень** | Unit-тесты | Интеграционные тесты |
| **Что мокируется** | Java-классы | HTTP-сервисы |
| **Применение** | Сервисы, репозитории | Внешние REST API |

---

## WireMock

Фреймворк для мокирования внешних HTTP-сервисов в интеграционных тестах.

```java
@WireMockTest(httpPort = 8089)
class PaymentServiceIT {

    @Test
    void shouldCallExternalApi() {
        // Настраиваем stub для внешнего сервиса
        stubFor(post("/api/payments")
            .withRequestBody(matchingJsonPath("$.amount"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"status\": \"APPROVED\"}")));

        // Вызываем наш сервис, который обращается к http://localhost:8089/api/payments
        PaymentResult result = paymentService.process(new PaymentRequest(100));

        assertThat(result.getStatus()).isEqualTo("APPROVED");
        verify(postRequestedFor(urlEqualTo("/api/payments")));
    }
}
```

**Основные методы:**
- `stubFor(...)` — определить поведение мок-сервера;
- `verify(...)` — проверить, что запрос был отправлен;
- `reset()` — сбросить все stubs между тестами.

---

## Spring Boot Test

### Аннотации

| Аннотация | Назначение |
|-----------|-----------|
| `@SpringBootTest` | Полный интеграционный тест с поднятым контекстом Spring |
| `@WebMvcTest(Controller.class)` | Тест только web-слоя (контроллеры) |
| `@DataJpaTest` | Тест JPA-репозиториев с in-memory БД |
| `@RestClientTest` | Тест REST-клиентов |
| `@JsonTest` | Тест JSON-сериализации |
| `@AutoConfigureMockMvc` | Автоконфигурация MockMvc |
| `@TestConfiguration` | Тестовая конфигурация (переопределяет основную) |

### @SpringBootTest

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class FullIntegrationTest {
    // RANDOM_PORT — поднимает встроенный сервер на случайном порту
    // MOCK — web-слой мокируется (default)
    // DEFINED_PORT — фиксированный порт из application.properties
    // NONE — не поднимает web-сервер
}
```

### @WebMvcTest

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "John"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

### @DataJpaTest

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class UserRepositoryTest {
    @Autowired
    private UserRepository repository;

    @Test
    void shouldSaveAndFindUser() {
        User saved = repository.save(new User("John"));
        Optional<User> found = repository.findById(saved.getId());
        assertThat(found).isPresent();
    }
}
```

### TestProfile

Изоляция тестовой конфигурации через `@ActiveProfiles`:

```java
@SpringBootTest
@ActiveProfiles("test")
class MyIntegrationTest {
    // application-test.yml — отдельные настройки (in-memory БД, mock endpoints)
}
```

### TestRestTemplate

Клиент для интеграционных тестов REST API:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerIT {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateUser() {
        ResponseEntity<User> response = restTemplate.postForEntity(
            "/api/users", new UserDto("John"), User.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

---

## Testcontainers

JUnit-расширение для запуска Docker-контейнеров в тестах.

```java
@Testcontainers
class UserRepositoryTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

**Преимущества:**
- Тесты работают с реальной БД (PostgreSQL, MySQL, MongoDB);
- Изолированность между тестами;
- Нет различий между тестовой и продакшен БД.

### Kafka и Redis в Testcontainers

```java
@Testcontainers
class MessagingIntegrationTest {
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
    }
}
```

---

## Hamcrest

Библиотека матчеров для читаемых assert:

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

assertThat(user.getName(), is(equalTo("John")));
assertThat(users, hasSize(greaterThan(2)));
assertThat(user.getEmail(), containsString("@example.com"));
assertThat(user.getRoles(), hasItem("ADMIN"));
```

---

## Cucumber

BDD-фреймворк, позволяющий писать тесты на естественном языке (Gherkin).

```gherkin
Feature: User registration
  Scenario: Successful registration
    Given the user is on the registration page
    When the user enters valid credentials
    And clicks the register button
    Then the user should see a success message
```

```java
public class RegistrationSteps {
    @Given("the user is on the registration page")
    public void openRegistrationPage() { /* ... */ }

    @When("the user enters valid credentials")
    public void enterCredentials() { /* ... */ }

    @Then("the user should see a success message")
    public void verifySuccess() { /* ... */ }
}
```

---

## Пирамида тестирования

```
        /\
       /  \
      / E2E \          <- Немного тестов, дорогие, медленные
     /--------\
    /Integration\       <- Средний слой (API, DB, сервисы)
   /------------\
  /    Unit      \     <- Большинство тестов, быстрые, дешёвые
 /----------------\
```

| Уровень | Стоимость | Скорость | Покрытие |
|---------|-----------|----------|----------|
| Unit | Низкая | Быстро | Отдельные классы |
| Integration | Средняя | Средне | Взаимодействие компонентов |
| E2E | Высокая | Медленно | Полный пользовательский сценарий |

**Пирамида тестирования Борисова:** большинство тестов — unit (быстрые, дешёвые), меньше — интеграционные, совсем немного — E2E (дорогие, медленные). Это обеспечивает баланс между скоростью обратной связи и уверенностью в работе системы.

---

## CompletableFuture: тестирование

```java
@Test
void shouldCompleteAsyncOperation() {
    CompletableFuture<String> future = service.asyncOperation();

    String result = assertDoesNotThrow(() -> future.get(5, TimeUnit.SECONDS));
    assertThat(result).isEqualTo("success");
}
```

**Полезные методы:**
- `CompletableFuture.completedFuture(value)` — завершённый future;
- `CompletableFuture.failedFuture(exception)` — завершённый с ошибкой;
- `join()` — блокирующее получение результата (без checked exception).

---

## Kafka: тестирование

**Embedded Kafka (Spring Kafka Test):**

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, brokerProperties = {"listeners=PLAINTEXT://localhost:9092"})
class KafkaIntegrationTest {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private ConsumerFactory<String, String> consumerFactory;

    @Test
    void shouldSendAndReceiveMessage() {
        kafkaTemplate.send("test-topic", "hello");

        Consumer<String, String> consumer = consumerFactory.createConsumer();
        consumer.subscribe(List.of("test-topic"));
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));

        assertThat(records).hasSize(1);
        assertThat(records.iterator().next().value()).isEqualTo("hello");
    }
}
```

**Testcontainers для Kafka:**

```java
@Container
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
```


