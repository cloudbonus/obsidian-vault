
## CI/CD

| Критерий | CI (Continuous Integration) | CD (Continuous Delivery/Deployment) |
|----------|-----------------------------|-------------------------------------|
| **Цель** | Автоматическая сборка и тестирование | Автоматическое развёртывание |
| **Триггер** | Push / Merge Request | Успешный CI pipeline |
| **Результат** | Проверенный артефакт | Артефакт в нужном окружении |

**Pipeline этапы:**
1. Build — компиляция, сборка артефакта;
2. Test — unit, integration, contract tests;
3. Security scan — SAST/DAST, dependency check;
4. Deploy to staging — развёртывание в тестовое окружение;
5. Integration/E2E tests — приёмочное тестирование;
6. Deploy to production — развёртывание в прод (может быть с approval).

---

## Docker

**Контейнер** — изолированный процесс с собственной файловой системой, сетью и ресурсами.

**Основные команды:**

```bash
docker build -t myapp:1.0 .          # Сборка образа
docker run -p 8080:8080 myapp:1.0    # Запуск контейнера
docker ps                            # Список запущенных
docker stop <id>                     # Остановка
docker logs <id>                     # Логи
docker exec -it <id> /bin/sh         # Вход в контейнер
```

### Docker Compose

Определяет и запускает мультиконтейнерные приложения:

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

### Dockerfile best practices

- Использовать минимальный базовый образ (`eclipse-temurin:17-jre-alpine`);
- Многоэтапная сборка (multi-stage) — отдельный этап для компиляции;
- Запускать не от root (`USER 1000:1000`);
- Минимизировать слои (объединять RUN-команды);
- Использовать `.dockerignore`.

---

## Kubernetes

Платформа оркестрации контейнеров.

### Основные объекты

| Объект | Назначение |
|--------|-----------|
| **Pod** | Минимальная deployable единица (1+ контейнеров) |
| **Deployment** | Управление репликами Pod, rolling updates |
| **Service** | Стабильный сетевой доступ к Pod (ClusterIP, NodePort, LoadBalancer) |
| **Ingress** | HTTP/HTTPS маршрутизация внешнего трафика |
| **ConfigMap** | Конфигурационные данные |
| **Secret** | Чувствительные данные (пароли, токены) |
| **PersistentVolume** | Хранилище данных вне Pod |
| **Namespace** | Логическое разделение кластера |

### Масштабирование в Kubernetes

**Horizontal Pod Autoscaler (HPA):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Мониторинг и логирование

### ELK Stack

- **Elasticsearch** — поиск и аналитика;
- **Logstash** — сбор и обработка логов;
- **Kibana** — визуализация данных.

**Альтернативы:** Grafana + Loki + Prometheus (для метрик).

---

## Maven

### Жизненный цикл Maven

| Фаза | Описание |
|------|----------|
| **validate** | Проверка корректности проекта |
| **compile** | Компиляция исходников |
| **test** | Запуск unit-тестов |
| **package** | Упаковка в JAR/WAR |
| **verify** | Проверка готовности (интеграционные тесты) |
| **install** | Установка в локальный репозиторий |
| **deploy** | Развёртывание в удалённый репозиторий |

---

## Git

| Команда | Назначение |
|---------|-----------|
| `fetch` | Загрузка изменений с удалённого репозитория (без слияния) |
| `pull` | `fetch` + `merge` |
| `push` | Отправка изменений на удалённый репозиторий |
| `merge` | Слияние веток с сохранением истории |
| `rebase` | Перемещение коммитов на новое основание (линейная история) |
| `cherry-pick` | Применение отдельного коммита из другой ветки |
| `squash` | Объединение нескольких коммитов в один |

**Merge vs Rebase:**
- **Merge** — сохраняет полную историю, создаёт merge-коммит;
- **Rebase** — переписывает историю в линейную, не создаёт merge-коммитов. Не использовать для публичных веток.

---

## Linux: основные команды

```bash
# Навигация
ls, cd, pwd, find, locate

# Файлы
cat, less, head, tail, grep, awk, sed

# Процессы
ps, top, htop, kill, systemctl

# Диск
 df, du, mount

# Сеть
netstat, ss, curl, wget, telnet, nc

# Права
chmod, chown, sudo

# Архивы
tar, gzip, unzip
```
