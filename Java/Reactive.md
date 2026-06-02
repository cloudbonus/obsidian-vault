
## Spring MVC vs Spring WebFlux

| Критерий                   | Spring MVC         | Spring WebFlux                       |
| -------------------------- | ------------------ | ------------------------------------ |
| **Модель**                 | Thread-per-request | Event loop (Netty)                   |
| **Потоки**                 | Блокирующие        | Неблокирующие                        |
| **Пропускная способность** | Хорошая            | Высокая при большом числе соединений |
| **Задержка**               | Предсказуемая      | Может быть выше при малой нагрузке   |
| **Базы данных**            | Любые              | Требуются Reactive Drivers (R2DBC)   |
| **API**                    | `Servlet`          | `Reactive Streams` (Mono, Flux)      |

**`Mono<T>`** — 0 или 1 элемент.
**`Flux<T>`** — 0..N элементов.

### Netty Event Loop

Spring WebFlux по умолчанию использует Netty — асинхронный event-driven сетевой фреймворк.

**Event Loop** — это цикл обработки событий в одном потоке:
1. Ожидание событий (новое соединение, данные готовы к чтению/записи) через `Selector` (Java NIO);
2. Диспетчеризация событий в соответствующие обработчики (ChannelHandler);
3. Выполнение неблокирующих операций в том же потоке;
4. Блокирующие операции (например, вызов `Thread.sleep()` или JDBC) "замораживают" весь Event Loop и снижают пропускную способность.

```
[ Client ] --req--> [ Event Loop Thread ] --event--> [ ChannelHandler ]
                           ↑
                    Selector (NIO epoll/kqueue)
```

- **Boss Group** — потоки, принимающие входящие соединения;
- **Worker Group** — потоки, обрабатывающие I/O события;
- По умолчанию worker threads = количество ядер CPU × 2.

---

## Часть 1: Введение в реактивное программирование

### Почему реактивное программирование

Традиционный императивный подход (Spring MVC + Tomcat) имеет ограничения при высокой нагрузке:
- **Thread-per-request** — каждому запросу выделяется поток (1 МБ стека). Пул потоков ограничен, потребление памяти растёт;
- **Блокировка на I/O** — потоки простаивают во время ожидания ответа от другого сервиса, БД или файла;
- **Последовательные вызовы** — время ответа складывается из задержек всех синхронных вызовов;
- **Перегрузка клиента** — сервер может отправить слишком много данных за один раз.

### Определение

> Реактивное программирование — это неблокирующие приложения, которые являются асинхронными, управляемыми событиями и требуют небольшого количества потоков для масштабирования. Ключевой аспект — **backpressure** (противодавление), механизм, гарантирующий, что производители не перегружают потребителей.

### Reactive Streams (Java 9 Flow API)

Спецификация, стандартизирующая взаимодействие между асинхронными компонентами с backpressure. Реализована в Java 9 как `java.util.concurrent.Flow`.

```java
// Publisher — производитель данных
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}

// Subscriber — потребитель
public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);      // новый элемент
    void onError(Throwable t);  // ошибка (терминальное событие)
    void onComplete();     // завершение (терминальное событие)
}

// Subscription — управление потоком
public interface Subscription {
    void request(long n);  // запросить n элементов (backpressure)
    void cancel();         // отменить подписку
}

// Processor — трансформирует данные
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

### Backpressure

Способность потребителя сигнализировать производителю, с какой скоростью он может обрабатывать данные:

```java
Flux.range(1, 100)
    .subscribe(new Subscriber<Integer>() {
        private Subscription s;
        int counter;

        @Override
        public void onSubscribe(Subscription s) {
            this.s = s;
            s.request(10); // запрашиваем по 10 элементов за раз
        }

        @Override
        public void onNext(Integer i) {
            counter++;
            if (counter % 10 == 0) {
                s.request(10); // запрашиваем следующую партию
            }
        }
        // ...
    });
```

### Reactive Manifesto

Система считается реактивной, если она:
- **Responsive** — отзывчива (даёт своевременный ответ);
- **Resilient** — устойчива (остаётся отзывчивой при сбоях);
- **Elastic** — эластична (масштабируется под нагрузку);
- **Message Driven** — управляется сообщениями (асинхронная передача).

### История

- **2011** — Microsoft выпустила Reactive Extensions (Rx / ReactiveX) для .NET;
- **2014** — Netflix выпустила RxJava 1.0 (межъязыковой стандарт push-модели);
- **2015** — спецификация Reactive Streams (стандарт для JVM);
- **Java 9** — Flow API (реализация Reactive Streams в JDK);
- **Spring 5** — Project Reactor как основа реактивного стека Spring.

---

## Часть 2: Project Reactor

Project Reactor — реактивная библиотека для JVM, основанная на Reactive Streams. Основа Spring WebFlux.

### Модули

| Модуль | Назначение |
|--------|-----------|
| **Reactor Core** | `Mono`, `Flux`, операторы |
| **Reactor Test** | `StepVerifier`, `TestPublisher` для тестирования |
| **Reactor Netty** | Неблокирующие клиенты и серверы TCP/HTTP/UDP |
| **Reactor Adapter** | Адаптеры для RxJava2, Akka Streams |
| **Reactor Kafka** | Реактивный API для Kafka |

### Lazy-Evaluation

Реактивные типы **ленивые**: ничего не происходит без подписки.

```java
Flux<String> flux = Flux.just("red", "green", "blue")
    .map(String::toUpperCase);
// До вызова subscribe() данные не испускаются!

flux.subscribe(System.out::println);
```

### Основные операторы

```java
// Преобразование
Flux.just(1, 2, 3).map(i -> i * 2);           // 2, 4, 6

// Асинхронное преобразование (разворачивание Mono/Flux)
Flux.just(1, 2, 3).flatMap(i -> Mono.just(i * 2)); // 2, 4, 6

// Фильтрация
Flux.range(1, 10).filter(i -> i % 2 == 0);     // 2, 4, 6, 8, 10

// Объединение потоков
Flux.zip(
    Flux.just("apple", "pear"),
    Flux.just("red", "green"),
    (fruit, color) -> fruit + " is " + color
); // "apple is red", "pear is green"

// Обработка ошибок
Flux.just(1, 0, 3)
    .map(i -> 10 / i)
    .onErrorReturn(ArithmeticException.class, -1); // 10, -1

// Восстановление через fallback
Flux.just(1, 0, 3)
    .map(i -> 10 / i)
    .onErrorResume(e -> Mono.just(-1));
```

### Параллелизм: publishOn vs subscribeOn

```java
Flux.just(1)
    .map(i -> { /* выполняется в потоке A */ return i; })
    .subscribeOn(Schedulers.boundedElastic()) // весь pipeline отсюда
    .map(i -> { /* выполняется в потоке A */ return i; })
    .publishOn(Schedulers.parallel())         // с этого момента — в parallel
    .map(i -> { /* выполняется в parallel */ return i; })
    .subscribe();
```

| Scheduler | Назначение |
|-----------|-----------|
| `Schedulers.parallel()` | Фиксированный пул, количество потоков = CPU cores |
| `Schedulers.single()` | Один reusable поток |
| `Schedulers.boundedElastic()` | Динамический пул для блокирующих операций (I/O) |
| `Schedulers.immediate()` | Текущий поток без переключения |

### Cold vs Hot Publisher

- **Cold** — создаёт новые данные для каждой подписки (default). Без подписчика данные не генерируются.
- **Hot** — не зависит от подписчиков, может начать публикацию без них.

```java
// Cold: каждый подписчик получает все элементы с начала
Flux<Long> cold = Flux.interval(Duration.ofSeconds(1));
cold.subscribe(i -> System.out.println("A: " + i));
Thread.sleep(2000);
cold.subscribe(i -> System.out.println("B: " + i)); // B начнёт с 0

// Hot: подписчики получают только элементы после своей подписки
ConnectableFlux<Long> hot = Flux.interval(Duration.ofSeconds(1)).publish();
hot.connect(); // запускаем публикацию
Thread.sleep(2000);
hot.subscribe(i -> System.out.println("A: " + i)); // A получит элементы с текущего момента
```

### Тестирование: StepVerifier

```java
@Test
void testFlux() {
    StepVerifier.create(Flux.just("foo", "bar"))
        .expectNext("foo")
        .expectNext("bar")
        .verifyComplete();
}

@Test
void testWithError() {
    StepVerifier.create(
        Flux.just(1, 0, 3).map(i -> 10 / i)
    )
    .expectNext(10)
    .expectError(ArithmeticException.class)
    .verify();
}
```

### Отладка

```java
// Временно (не для production!)
Hooks.onOperatorDebug();

// Production: reactor-tools агент
public static void main(String[] args) {
    ReactorDebugAgent.init();
    SpringApplication.run(App.class, args);
}
```

### Оборачивание блокирующего кода

```java
Mono.fromCallable(() -> {
    // блокирующий вызов (JDBC, файлы)
    return jdbcTemplate.queryForObject("SELECT COUNT(*) FROM users", Long.class);
})
.subscribeOn(Schedulers.boundedElastic()); // выполняется в отдельном потоке
```

---

## Часть 3: Spring WebFlux

Реактивный веб-фреймворк Spring 5+. Полностью асинхронный и неблокирующий.

### Две модели программирования

**1. Аннотированные контроллеры** (привычный стиль):

```java
@RestController
@RequestMapping("/students")
public class StudentController {

    @GetMapping("/{id}")
    public Mono<ResponseEntity<Student>> getStudent(@PathVariable long id) {
        return studentService.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @GetMapping
    public Flux<Student> listStudents() {
        return studentService.findAll();
    }

    @PostMapping
    public Mono<Student> create(@RequestBody Student student) {
        return studentService.save(student);
    }
}
```

**2. Функциональные endpoints** (`RouterFunction` + `HandlerFunction`):

```java
@Configuration
public class StudentRouter {
    @Bean
    public RouterFunction<ServerResponse> route(StudentHandler handler) {
        return RouterFunctions
            .route(GET("/students/{id}").and(accept(APPLICATION_JSON)), handler::getStudent)
            .andRoute(GET("/students").and(accept(APPLICATION_JSON)), handler::listStudents)
            .andRoute(POST("/students").and(accept(APPLICATION_JSON)), handler::createStudent);
    }
}

@Component
public class StudentHandler {
    public Mono<ServerResponse> getStudent(ServerRequest request) {
        long id = Long.parseLong(request.pathVariable("id"));
        return studentService.findById(id)
            .flatMap(s -> ServerResponse.ok().bodyValue(s))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
}
```

### WebClient — реактивный HTTP-клиент

Замена `RestTemplate` (который блокирующий):

```java
@Service
public class StudentWebClient {
    private final WebClient client = WebClient.create("http://localhost:8080");

    public Mono<Student> getStudent(long id) {
        return client.get()
            .uri("/students/" + id)
            .retrieve()
            .bodyToMono(Student.class);
    }

    public Flux<Student> getAllStudents() {
        return client.get()
            .uri("/students")
            .retrieve()
            .bodyToFlux(Student.class);
    }

    public Mono<Student> create(Student student) {
        return client.post()
            .uri("/students")
            .body(Mono.just(student), Student.class)
            .retrieve()
            .bodyToMono(Student.class);
    }
}
```

### Тестирование: WebTestClient

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class StudentControllerTest {
    @Autowired
    private WebTestClient webClient;

    @Test
    @WithMockUser(roles = "USER")
    void shouldReturnStudents() {
        webClient.get().uri("/students")
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(APPLICATION_JSON)
            .expectBodyList(Student.class);
    }
}
```

### WebFlux Security

```java
@EnableWebFluxSecurity
public class SecurityConfig {
    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http.authorizeExchange()
            .pathMatchers("/students/admin").hasAuthority("ROLE_ADMIN")
            .anyExchange().authenticated()
            .and().httpBasic()
            .and().build();
    }
}
```

---

## Часть 4: R2DBC

**R2DBC** (Reactive Relational Database Connectivity) — API для работы с SQL-базами через неблокирующий реактивный интерфейс. Альтернатива блокирующему JDBC.

### Отличие от JPA/Hibernate

| Критерий | JPA/Hibernate | Spring Data R2DBC |
|----------|---------------|-------------------|
| **API** | Блокирующее (JDBC) | Неблокирующее (Reactive Streams) |
| **ORM** | Полноценный (кэш, lazy loading) | Простой маппинг объектов |
| **Lazy loading** | Да | Нет |
| **Кэш** | First/Second level | Нет |
| **Типы возврата** | `List<T>`, `Optional<T>` | `Flux<T>`, `Mono<T>` |

### ReactiveCrudRepository

```java
public interface StudentRepository extends ReactiveCrudRepository<Student, Long> {
    Flux<Student> findByName(String name);

    @Query("SELECT * FROM student WHERE address = :address")
    Flux<Student> findByAddress(String address);
}
```

### Entity

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table("student")
public class Student {
    @Id
    private Long id;
    private String name;
    private String address;
}
```

### DatabaseClient (прямой SQL)

```java
@Service
public class StudentService {
    private final DatabaseClient client;

    public Flux<Student> findAll() {
        return client.sql("SELECT * FROM student")
            .map((row, metadata) -> new Student(
                row.get("id", Long.class),
                row.get("name", String.class),
                row.get("address", String.class)
            ))
            .all();
    }
}
```

### R2dbcEntityTemplate

```java
@Service
public class StudentService {
    @Autowired
    private R2dbcEntityTemplate template;

    public Flux<Student> findAll() {
        return template.select(Student.class).all();
    }

    public Mono<Void> delete(Student student) {
        return template.delete(student).then();
    }
}
```

### Транзакции и блокировки

```java
// Оптимистическая блокировка
public class Student {
    @Id
    private Long id;
    @Version
    private Long version;
}

// Реактивные транзакции
@Transactional
public Mono<Student> updateStudent(long id, Student student) {
    return repository.findById(id)
        .flatMap(s -> {
            s.setName(student.getName());
            return repository.save(s);
        });
}
```

### Зависимости

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Project Loom (упоминание)

Project Loom (OpenJDK) вводит **виртуальные потоки** (Virtual Threads) — легковесные потоки, управляемые JVM, а не ОС. В будущем это может устранить необходимость в сложном реактивном программировании для масштабирования I/O, поскольку блокировка виртуального потока не блокирует поток ОС.
