
## Типы индексов

Индекс — структура данных для повышения скорости поиска. Наиболее популярные реализации — B-деревья и хеш-таблицы.

**PostgreSQL:**

| Тип | Описание | Применение |
|-----|----------|------------|
| **B-tree** (default) | Сбалансированное дерево | Операторы сравнения (`=`, `>`, `<`, `BETWEEN`, `LIKE 'abc%'`). Поддерживает уникальность. |
| **Hash** | Хеш-таблица | Только `=`. Не поддерживает уникальные столбцы. Обычно выигрывает B-tree при сравнении через равно. |
| **GiST** | Generalized Search Tree | Геометрические типы, диапазоны, полнотекстовый поиск. |
| **GIN** | Generalized Inverted Index | Полнотекстовый поиск, массивы, JSONB. |
| **BRIN** | Block Range Index | Большие таблицы с естественным порядком (timestamp, ID). Компактный. |

**Кластеризованный vs Некластеризованный:**
- **Кластеризованный** — данные физически упорядочены на диске. Минус: сортировка при обновлении. Только один на таблицу.
- **Некластеризованный** — отдельная структура с указателями на данные. Может быть несколько.

### Составные индексы

При создании составного индекса важен **порядок столбцов**:
- Первым должно идти поле с самой высокой селективностью (больше уникальных значений);
- Индекс `CREATE INDEX idx ON table (a, b, c)` будет использован для запросов:
  - `WHERE a = 1`
  - `WHERE a = 1 AND b = 2`
  - `WHERE a = 1 AND b = 2 AND c = 3`
  - `WHERE a = 1 AND c = 3` (только по `a`)
- НЕ будет использован для: `WHERE b = 2`, `WHERE c = 3`, `WHERE b = 2 AND c = 3`.

### Ключи

**Primary Key (PK)**
- Столбец (или группа столбцов), используемый для обеспечения уникальности данных в таблице;
- Уникально идентифицирует каждую строку;
- Не допускает `NULL`;
- Автоматически создаёт кластеризованный (B-tree) индекс;
- Таблица может иметь только один PK.

**Foreign Key (FK)**
- Обеспечивает ссылочную целостность между таблицами;
- Поддерживает каскадные действия:

| Действие | Описание |
|----------|----------|
| `ON DELETE/UPDATE CASCADE` | При изменении/удалении родительской строки изменения распространяются на дочерние |
| `SET NULL` | В дочерней строке значение FK устанавливается в `NULL` |
| `SET DEFAULT` | Устанавливается значение по умолчанию |
| `RESTRICT` / `NO ACTION` | Запрещает изменение/удаление, если есть зависимые строки |

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    CONSTRAINT fk_customer
        FOREIGN KEY (customer_id) REFERENCES customers(id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```

**Покрывающий индекс (Covering Index)**
- Индекс, который содержит все столбцы, необходимые запросу;
- PostgreSQL: `INCLUDE` — добавляет неключевые столбцы в индекс, чтобы избежать обращения к таблице (Index Only Scan);
- Уменьшает случайный I/O, но увеличивает размер индекса.

```sql
CREATE INDEX idx_orders_customer_date
ON orders (customer_id, order_date)
INCLUDE (total_amount, status);
```

### Constraints (ограничения)

Правила целостности данных на уровне таблицы:

| Constraint | Описание |
|------------|----------|
| `NOT NULL` | Столбец не может содержать `NULL` |
| `UNIQUE` | Каждое значение в столбце должно быть уникальным |
| `PRIMARY KEY` | Комбинация `NOT NULL` + `UNIQUE`. Одна на таблицу |
| `FOREIGN KEY` | Обеспечивает ссылочную целостность между таблицами |
| `DEFAULT` | Значение по умолчанию, если при `INSERT` не указано |
| `CHECK` | Проверка условия на значение (например, `CHECK (age > 0)`) |

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INT CHECK (age >= 0),
    status VARCHAR(50) DEFAULT 'ACTIVE',
    department_id INT REFERENCES departments(id)
);
```

### JOIN

| Тип JOIN | Результат |
|----------|-----------|
| **INNER JOIN** | Строки, имеющие совпадения в обеих таблицах |
| **LEFT JOIN** | Все строки из левой таблицы + совпадающие из правой (NULL где нет совпадений) |
| **RIGHT JOIN** | Все строки из правой таблицы + совпадающие из левой |
| **FULL JOIN** | Все строки из обеих таблиц (NULL где нет совпадений) |
| **CROSS JOIN** | Декартово произведение — все комбинации строк |

### Категории SQL-команд

| Категория | Назначение | Примеры |
|-----------|------------|---------|
| **DDL** (Data Definition Language) | Определение и изменение структуры БД | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** (Data Manipulation Language) | Манипуляция данными | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| **DCL** (Data Control Language) | Управление доступом | `GRANT`, `REVOKE` |
| **TCL/DTL** (Transaction Control) | Управление транзакциями | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

### Базовые конструкции запросов

**ORDER BY**
- Сортировка результата по одному или нескольким столбцам;
- `ASC` (по умолчанию) / `DESC`;
- В PostgreSQL: `NULLS FIRST` / `NULLS LAST`.

```sql
SELECT * FROM products
WHERE category_id = 5
ORDER BY price DESC, name ASC NULLS LAST;
```

**GROUP BY и HAVING**
- `GROUP BY` — группировка строк по значениям столбцов для агрегации;
- `HAVING` — фильтрация уже сгруппированных данных (в отличие от `WHERE`, который фильтрует строки до группировки);
- В `SELECT` с `GROUP BY` можно использовать только столбцы из `GROUP BY` и агрегатные функции.

```sql
SELECT category_id, COUNT(*) as cnt, AVG(price) as avg_price
FROM products
WHERE is_active = true
GROUP BY category_id
HAVING COUNT(*) > 5
ORDER BY avg_price DESC;
```

**Агрегатные функции**

| Функция | Описание |
|---------|----------|
| `COUNT(*)` | Количество строк |
| `COUNT(column)` | Количество не-NULL значений |
| `SUM(column)` | Сумма |
| `AVG(column)` | Среднее значение |
| `MIN(column)` / `MAX(column)` | Минимум / максимум |
| `STRING_AGG(column, delimiter)` | Конкатенация строк (PostgreSQL) |

```sql
-- Количество заказов, общая и средняя сумма по клиентам
SELECT
    customer_id,
    COUNT(*) as order_count,
    SUM(total_amount) as total_spent,
    AVG(total_amount) as avg_order_value,
    MAX(created_at) as last_order_date
FROM orders
GROUP BY customer_id;
```

### Транзакции

**ACID:**
- **Atomicity** — атомарность: транзакция либо выполняется полностью, либо не выполняется вовсе. Достигается через **WAL (Write-Ahead Log)** — сначала изменения пишутся в лог, потом в данные. При сбое БД восстанавливается по логу;
- **Consistency** — согласованность: транзакция переводит БД из одного согласованного состояния в другое. Достигается через ограничения (constraints), триггеры, каскадные операции;
- **Isolation** — изоляция: результаты незавершённой транзакции не видны другим. Достигается через **MVCC (Multi-Version Concurrency Control)** — каждая транзакция видит снимок данных на момент своего старта, и **блокировки** (row-level, table-level);
- **Durability** — долговечность: результаты сохраняются даже при сбоях. Достигается через синхронную запись WAL на диск (`fsync`) и репликацию.

**Уровни изоляции (от меньшей к большей изоляции):**

| Уровень | Грязное чтение | Неповторяющееся чтение | Фантомы |
|---------|---------------|------------------------|---------|
| READ UNCOMMITTED | Да | Да | Да |
| READ COMMITTED | Нет | Да | Да |
| REPEATABLE READ | Нет | Нет | Да |
| SERIALIZABLE | Нет | Нет | Нет |

- **MySQL default**: `REPEATABLE READ`
- **PostgreSQL default**: `READ COMMITTED`

**Проблемы параллельного выполнения:**

1. **Dirty read** — чтение незафиксированных данных (другая транзакция ещё не commit):

```sql
-- Транзакция 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- ещё не COMMIT

-- Транзакция 2 (READ UNCOMMITTED):
SELECT balance FROM accounts WHERE id = 1; -- видит -100, но Т1 может ROLLBACK
```

2. **Non-repeatable read** — при повторном чтении той же строки в рамках одной транзакции она изменилась (другая транзакция сделала commit):

```sql
-- Транзакция 1:
BEGIN;
SELECT balance FROM accounts WHERE id = 1; -- 1000

-- Транзакция 2:
UPDATE accounts SET balance = 900 WHERE id = 1;
COMMIT;

-- Транзакция 1:
SELECT balance FROM accounts WHERE id = 1; -- 900 (уже другое значение!)
```

3. **Phantom read** — при повторном чтении по условию появились новые строки:

```sql
-- Транзакция 1:
BEGIN;
SELECT * FROM orders WHERE status = 'PENDING'; -- 5 строк

-- Транзакция 2:
INSERT INTO orders (status) VALUES ('PENDING');
COMMIT;

-- Транзакция 1:
SELECT * FROM orders WHERE status = 'PENDING'; -- 6 строк (новая "фантомная" строка)
```

### Оптимизация SQL-запросов

Основные подходы к повышению производительности:

1. **Индексы** — ускоряют поиск, но замедляют запись. Покрывающие индексы (`INCLUDE`) позволяют избежать обращения к таблице;
2. **Кэширование** — Redis / in-memory кэш для часто запрашиваемых данных;
3. **Профилирование запросов** — `EXPLAIN ANALYZE` для анализа плана выполнения;
4. **Денормализация** — объединение таблиц для сокращения JOIN в read-heavy сценариях;
5. **Пагинация** — `LIMIT`/`OFFSET` или keyset pagination вместо выборки всей таблицы;
6. **Batch-операции** — массовая вставка/обновление вместо поэлементных.

```sql
-- Пример: покрывающий индекс + LIMIT для пагинации
EXPLAIN ANALYZE
SELECT id, name, price
FROM products
WHERE category_id = 5 AND is_active = true
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

### Блокировки

**Пессимистическая блокировка:**
- Блокировка строки/таблицы на время транзакции;
- `SELECT ... FOR UPDATE` — блокировка строк для чтения;
- `SELECT ... FOR SHARE` — разделяемая блокировка;
- Гарантирует отсутствие конфликтов, но снижает параллелизм.

**Оптимистическая блокировка:**
- Нет физической блокировки;
- Проверка версии при коммите (`@Version` / `version` поле);
- Если версия изменилась — `OptimisticLockException`;
- Подходит при низкой вероятности конфликтов.

**Дедлок (Deadlock):**
- Взаимная блокировка транзакций;
- Пример: Т1 захватила A, ждёт B; Т2 захватила B, ждёт A;
- Решение: упорядоченное получение блокировок, `tryLock()` с таймаутом.

### План выполнения запроса

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

- `EXPLAIN` — показывает план без выполнения;
- `EXPLAIN ANALYZE` — выполняет запрос и показывает реальные времена и количество строк.

### Оконные функции

Вычисляют значение для каждой строки на основе набора строк (окна), не требуя `GROUP BY`.

**Где применяется:** в финансовом приложении оконная функция используется для вычисления скользящей средней цены акции за последние 30 дней для каждого торгового дня.

```sql
SELECT
    stock_symbol,
    trade_date,
    closing_price,
    AVG(closing_price) OVER (
        PARTITION BY stock_symbol
        ORDER BY trade_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) as moving_avg_30d
FROM stock_prices;
```

### Представления (Views) и материализованные представления

- **View** — виртуальная таблица на основе SQL-запроса. Данные не хранятся, запрос выполняется при каждом обращении;
- **Materialized View** — хранит результат запроса физически. Требует обновления (`REFRESH`).

**Где применяется:** использование вьюх для создания агрегированного отчёта по продажам за день, чтобы не повторять сложные SQL-запросы в каждом отчёте.

```sql
CREATE VIEW daily_sales_report AS
SELECT
    DATE_TRUNC('day', created_at) as sale_date,
    COUNT(*) as total_orders,
    SUM(total_amount) as revenue
FROM orders
GROUP BY DATE_TRUNC('day', created_at);
```

### Хранимые процедуры

Набор SQL-операторов, сохранённых на сервере и выполняемых как единый вызов.

**Где применяется:** в интернет-магазине хранимая процедура используется для обработки заказа и обновления связанных таблиц транзакционно (например, уменьшение количества товара на складе).

```sql
CREATE PROCEDURE process_order(p_order_id BIGINT, p_items JSONB)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Уменьшаем остатки на складе
    UPDATE products p
    SET stock = stock - i.qty
    FROM jsonb_to_recordset(p_items) AS i(product_id BIGINT, qty INT)
    WHERE p.id = i.product_id;

    -- Создаём заказ
    INSERT INTO orders (id, status) VALUES (p_order_id, 'CONFIRMED');
END;
$$;

### Триггеры (Triggers)

Хранимая процедура, которая автоматически выполняется при наступлении события (`INSERT`, `UPDATE`, `DELETE`).

**Пример:** автоматическое обновление поля `updated_at` при изменении строки.

```sql
CREATE TRIGGER set_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();

CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Типы триггеров:**
- `BEFORE` / `AFTER` — до или после события;
- `INSTEAD OF` — выполняется вместо события (обычно для представлений);
- `FOR EACH ROW` / `FOR EACH STATEMENT` — для каждой строки или для всей операции.

### Сиквенсы (Sequences)

Механизм генерации уникальных значений.

**Где применяется:** в CRM-системе сиквенс (`BIGSERIAL`) применяется для автоматической генерации уникальных идентификаторов новых клиентов.

```sql
CREATE TABLE clients (
    id BIGSERIAL PRIMARY KEY,
    full_name VARCHAR(255) NOT NULL
);
```

- `SERIAL` / `BIGSERIAL` в PostgreSQL — автоинкремент на основе sequence;
- `AUTO_INCREMENT` в MySQL.

### Суррогатный и естественный ключ

**Суррогатный ключ**
- Искусственно созданный столбец для обеспечения уникальности (например, автоинкремент `id`, UUID);
- Не несёт бизнес-смысла;
- Преимущества: неизменность, компактность, единообразие для всех таблиц;
- Недостатки: дополнительный индекс, нечитаемость для пользователя.

**Естественный ключ**
- Основан на реальных данных (email, ИНН, паспортные данные);
- Может изменяться со временем (смена email), что нарушает ссылочную целостность;
- Рекомендуется использовать суррогатный ключ как PK, а естественные поля индексировать как `UNIQUE`.

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,          -- суррогатный ключ
    email VARCHAR(255) NOT NULL UNIQUE -- естественный ключ
);
```

### Партиционирование в PostgreSQL

Декларативное разделение больших таблиц на более мелкие физические части (партиции) для повышения производительности и упрощения управления данными.

**Типы партиционирования:**

| Тип | Принцип | Применение |
|-----|---------|------------|
| **RANGE** | Диапазон значений | Логи, заказы по датам (`order_date`), события |
| **LIST** | Список значений | Данные по регионам, статусам |
| **HASH** | Хеш-функция от ключа | Равномерное распределение по партициям, когда нет естественного диапазона |

**Пример: партиционирование заказов по месяцам**

```sql
CREATE TABLE orders (
    order_id BIGSERIAL,
    customer_id BIGINT NOT NULL,
    total_amount DECIMAL(12,2),
    status VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (order_id, created_at) -- PK должен включать ключ партиционирования
) PARTITION BY RANGE (created_at);

-- Создание партиций
CREATE TABLE orders_2025_01 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE TABLE orders_2025_03 PARTITION OF orders
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

-- Партиция по умолчанию для данных вне диапазонов
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

**Преимущества:**
- Быстрое удаление старых данных (`DROP PARTITION` вместо `DELETE`);
- Ускорение запросов с фильтром по ключу партиционирования (pruning — чтение только нужных партиций);
- Возможность хранить старые партиции на медленных дисках.

**Ограничения:**
- Ключ партиционирования должен входить в PK и UNIQUE индексы;
- FK из других таблиц на партиционированную таблицу не поддерживаются (до PostgreSQL 11 ограничения, сейчас частично разрешено, но есть нюансы);
- Перекрёстные партиции не поддерживаются (нельзя партиционировать по `RANGE` и `LIST` одновременно).

### Репликация и шардинг

| Критерий | Репликация | Шардинг |
|----------|------------|---------|
| **Назначение** | Отказоустойчивость, чтение | Масштабирование записи |
| **Принцип** | Копия данных на несколько серверов | Разделение данных по серверам |
| **Типы** | Master-Slave, Master-Master | Horizontal partitioning |

### Нормализация и денормализация

**Нормализация** — уменьшение избыточности, разделение таблиц:
- 1NF — атомарные значения;
- 2NF — нет частичных зависимостей от составного ключа;
- 3NF — нет транзитивных зависимостей.

**Денормализация** — объединение таблиц для улучшения производительности чтения за счёт уменьшения JOIN.

### Реляционные vs Нереляционные (NoSQL) базы данных

| Критерий | Реляционные (SQL) | Нереляционные (NoSQL) |
|----------|-------------------|----------------------|
| **Структура** | Таблицы со строгой схемой | Документы, ключ-значение, графы, столбцы |
| **Связи** | Через Foreign Key (JOIN) | Вложенные документы или ссылки |
| **Транзакции** | ACID | BASE (Eventually Consistent) |
| **Масштабирование** | Вертикальное + шардинг | Горизонтальное (естественно) |
| **Примеры** | PostgreSQL, MySQL, Oracle | MongoDB, Redis, Cassandra, Neo4j |

**NoSQL: принципы BASE**
- **Basically Available** — базовая доступность (система отвечает на запросы);
- **Soft state** — состояние может меняться без внешнего вмешательства (из-за репликации);
- **Eventually consistent** — в конечном счёте согласованность (данные станут согласованными через время).

**Типы NoSQL:**
- **Документные** — MongoDB (JSON-подобные документы);
- **Ключ-значение** — Redis;
- **Столбцовые** — Cassandra;
- **Графовые** — Neo4j.

---

## Liquibase

Liquibase — система управления миграциями схемы БД.

### Основные концепции

- **ChangeLog** — файл с описанием изменений (XML, YAML, JSON, SQL);
- **ChangeSet** — единица изменения, аналог коммита. Имеет составной ID: `id`, `author`, `filename`;
- **databasechangelog** — таблица с историей выполненных changeSet;
- **databasechangelock** — таблица для блокировки при параллельном запуске.

### Контрольная сумма

Liquibase вычисляет MD5-хэш каждого changeSet. **Нельзя изменять уже выполненный changeSet** — это приведёт к ошибке при старте.

### Откат

- `rollbackCount N` — откат N последних changeSet;
- `rollback TAG` — откат до указанного тега;
- Некоторые операции требуют явного описания rollback-логики в changeSet.

### Spring Boot интеграция

```yaml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.xml
  jpa:
    hibernate:
      ddl-auto: none
```

---

## Flyway

Flyway — инструмент миграций схемы БД, основанный на **конвенции именования файлов**.

### Основные концепции

- **Migration** — SQL- или Java-файл, описывающий изменение схемы;
- **Versioned migration** — `V<version>__<description>.sql` (например, `V1__init_schema.sql`, `V2__add_users_table.sql`). Выполняются один раз, строго по порядку;
- **Repeatable migration** — `R__<description>.sql`. Выполняются каждый раз при изменении их контрольной суммы;
- **flyway_schema_history** — таблица с историей выполненных миграций.

### Ключевые отличия от Liquibase

| Критерий | Flyway | Liquibase |
|----------|--------|-----------|
| **Формат** | SQL, Java | XML, YAML, JSON, SQL |
| **Именование** | Строгая конвенция по имени файла | ChangeSet ID внутри changelog |
| **Откат** | Требует явного `UNDO`-файла (Flyway Teams) | Встроенная поддержка rollback |
| **Сложность** | Проще, меньше бойлерплейта | Гибче для сложных сценариев |

### Spring Boot интеграция

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
  jpa:
    hibernate:
      ddl-auto: none
```

---

## MySQL: движки хранения

| Движок | Транзакции | Блокировка | Назначение |
|--------|-----------|------------|------------|
| **InnoDB** | Да | Строковая | Транзакционные приложения, дефолт с MySQL 5.5+ |
| **MyISAM** | Нет | Табличная | Только чтение, полнотекстовый поиск |
| **MEMORY** | Нет | Табличная | Временные таблицы в RAM |
| **Archive** | Нет | Строковая | Хранение больших объёмов без индексов |
| **NDB** | Да | Строковая | MySQL Cluster, высокая доступность |

---

## JDBC

JDBC (Java Database Connectivity) — стандартный API Java для взаимодействия с реляционными базами данных.

**Основные компоненты:**
- `DriverManager` — загрузка JDBC-драйвера и создание соединения;
- `Connection` — сессия работы с БД;
- `Statement` / `PreparedStatement` — выполнение SQL-запросов;
- `ResultSet` — результат SELECT-запроса.

```java
// Базовый JDBC-цикл
Connection conn = DriverManager.getConnection(url, user, password);
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setLong(1, 1L);
ResultSet rs = stmt.executeQuery();
while (rs.next()) {
    String name = rs.getString("name");
}
rs.close(); stmt.close(); conn.close();
```

---

## Пул соединений (Connection Pool)

Набор предварительно созданных соединений к БД, которые переиспользуются между запросами.

**Зачем нужен:**
- Создание TCP-соединения с БД — дорогая операция (handshake, аутентификация);
- Пул держит соединения "теплыми" и выдаёт готовые по запросу;
- Снижает нагрузку на БД и улучшает latency приложения.

**Жизненный цикл соединения в пуле:**
1. Создание (warm-up) — при старте пула;
2. Выдача приложению — при запросе `getConnection()`;
3. Возврат в пул — `connection.close()` (не закрывает физически, а возвращает в пул);
4. Проверка валидности — при простое;
5. Уничтожение — при превышении `max-lifetime` или при закрытии пула.

---

## HikariCP

HikariCP — высокопроизводительный пул соединений JDBC, используемый по умолчанию в Spring Boot.

**Преимущества:**
- Быстрее c3p0, DBCP, Tomcat JDBC Pool;
- Минимальный оверхед;
- Проверка соединений в фоне (`connectionTestQuery` не требуется для JDBC 4+).

**Ключевые настройки:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20      # Макс. соединений в пуле
      minimum-idle: 5            # Мин. незанятых соединений
      connection-timeout: 30000  # Макс. время ожидания соединения (ms)
      idle-timeout: 600000       # Время жизни незанятого соединения (ms)
      max-lifetime: 1800000    # Макс. время жизни соединения (ms)
      leak-detection-threshold: 60000  # Диагностика утечек
```

**Правило:** `maximum-pool-size` должен быть меньше или равен максимальному количеству соединений, которое может обработать БД, иначе возникнут блокировки.

---

## OLAP vs OLTP

| Критерий | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---|--------------------------------------|-----------------------------------|
| **Назначение** | Операционная обработка транзакций | Аналитика, отчётность, data mining |
| **Характер запросов** | Простые, короткие (CRUD) | Сложные, агрегирующие, сканирующие большие объёмы |
| **Данные** | Текущие, детальные | Исторические, агрегированные |
| **Нормализация** | Высокая (3NF+) | Низкая (денормализация, звёздная схема) |
| **Примеры систем** | CRM, ERP, банковские транзакции | Data Warehouse, BI-отчёты |
| **Базы данных** | PostgreSQL, MySQL, Oracle | ClickHouse, Apache Druid, Snowflake |
