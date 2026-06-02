
## Паттерны проектирования в Spring-приложениях

### Порождающие

| Паттерн | Где в Spring | Пример |
|---------|-------------|--------|
| **Singleton** | Scope бина по умолчанию (`@Scope("singleton")`) | Один экземпляр `UserService` на контекст |
| **Factory Method** | `FactoryBean<T>`, `@Bean` | `ColorFactory` создаёт бин по кастомной логике |
| **Builder** | Lombok `@Builder`, `WebClient.Builder` | Конфигурируемое создание сложных объектов |
| **Prototype** | `@Scope("prototype")` | Новый экземпляр бина при каждом запросе |

### Структурные

| Паттерн | Где в Spring | Пример |
|---------|-------------|--------|
| **Adapter** | `WebMvcConfigurer`, `HandlerAdapter` | Адаптация разных типов контроллеров к общему интерфейсу |
| **Proxy** | Spring AOP, `@Transactional` | Прокси перехватывает вызовы методов для транзакций |
| **Decorator** | `HttpServletRequestWrapper`, `HandlerInterceptor` | Добавление поведения (логирование, сжатие) к существующему объекту |
| **Facade** | `@Service`, `JpaRepository` | Упрощённый интерфейс над сложной подсистемой (БД, внешние API) |
| **Composite** | `CompositeCacheManager` | Группировка нескольких менеджеров кэша как одного |

### Поведенческие

| Паттерн | Где в Spring | Пример |
|---------|-------------|--------|
| **Observer** | `ApplicationEventPublisher`, `@EventListener` | Публикация и подписка на события (`UserRegisteredEvent`) |
| **Strategy** | DI + интерфейсы | Инжект разных реализаций `PaymentStrategy` (Card, PayPal) |
| **Template Method** | `JdbcTemplate`, `RestTemplate` | Алгоритм в шаблонном классе, кастомизация через callback |
| **Command** | `@Async`, `Runnable`, `Callable` | Инкапсуляция запроса как объекта для выполнения |
| **Chain of Responsibility** | Spring Security Filter Chain, Servlet Filters | Запрос проходит через цепочку обработчиков, каждый решает — обработать или передать дальше |
