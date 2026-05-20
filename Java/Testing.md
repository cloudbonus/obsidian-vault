# Тестирование

## JUnit 5

JUnit 5 = JUnit Platform + JUnit Jupiter + JUnit Vintage.

### Основные аннотации

| Аннотация | Назначение |
|-----------|-----------|
| `@Test` | Тестовый метод |
| `@BeforeEach` | Выполняется перед каждым тестом (замена `@Before`) |
| `@AfterEach` | Выполняется после каждого теста (замена `@After`) |
| `@BeforeAll` | Выполняется один раз перед всеми тестами класса (замена `@BeforeClass`) |
| `@AfterAll` | Выполняется один раз после всех тестов класса (замена `@AfterClass`) |
| `@DisplayName` | Человекочитаемое имя теста |
| `@ParameterizedTest` | Параметризованный тест |
| `@ValueSource` | Источник простых значений для параметризации |
| `@CsvSource` | Источник CSV-данных |
| `@Disabled` | Отключение теста (замена `@Ignore`) |
| `@Nested` | Вложенные тестовые классы для группировки |
| `@ExtendWith` | Регистрация расширений (замена `@RunWith`) |

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

| | Mockito | WireMock |
|---|---------|----------|
| **Уровень** | Unit-тесты | Интеграционные тесты |
| **Что мокируется** | Java-классы | HTTP-сервисы |
| **Применение** | Сервисы, репозитории | Внешние REST API |

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

### Профили

```java
@TestPropertySource(properties = "spring.profiles.active=test")
class MyIntegrationTest {
    // ...
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

---

## Повторение: ключевые темы для подготовки

### Unit-тестирование
1. Пирамида тестирования Борисова;
2. JUnit 5: `@ExtendWith` vs `@RunWith` (JUnit 4);
3. Mockito: `@Mock`, `@Spy`, `@InjectMocks`, `verify`;
4. Hamcrest / AssertJ.

### Интеграционное тестирование
1. `@SpringBootTest` — поднятие полного контекста;
2. `TestProfile` — изоляция конфигурации;
3. `TestRestTemplate` — REST-клиент для тестов;
4. `@WebMvcTest` — изолированное тестирование контроллеров;
5. `@DataJpaTest` — тестирование репозиториев.

### Контрактное и E2E тестирование
1. WireMock — мокирование внешних HTTP-сервисов;
2. Cucumber — BDD-сценарии;
3. Testcontainers — интеграция с Docker (БД, Kafka, Redis).
