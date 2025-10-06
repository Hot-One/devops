# PostgreSQL Кластер с Репликацией и PgPool

Этот проект настраивает PostgreSQL кластер, состоящий из одного основного сервера (primary), двух реплик (replica) и балансировщика нагрузки PgPool в качестве прокси между клиентами и кластером.

## 🏗️ Архитектура

```
┌─────────────────┐     ┌──────────────────┐
│     Client      │────▶│     PgPool       │
│                 │     │  (Port: 9999)    │
└─────────────────┘     └──────────┬───────┘
                                   │
                        ┌──────────▼───────────┐
                        │                      │
                        ▼                      ▼
               ┌─────────────────┐    ┌─────────────────┐
               │ PostgreSQL      │    │ PostgreSQL      │
               │ Primary         │───▶│ Replica 1       │
               │ (Port: 5432)    │    │ (Port: 5433)    │
               └─────────────────┘    └─────────────────┘
                        │
                        ▼
               ┌─────────────────┐
               │ PostgreSQL      │
               │ Replica 2       │
               │ (Port: 5434)    │
               └─────────────────┘
```

## 📋 Компоненты

### PostgreSQL Primary (Основной сервер)
- **Порт**: 5432
- **База данных**: mydb
- **Пользователь**: admin
- **Пароль**: secret
- **Версия**: PostgreSQL 16 Alpine

### PostgreSQL Replica 1 & 2 (Реплики)
- **Порты**: 5433, 5434
- **Тип**: Физические реплики (streaming replication)
- **Слоты репликации**: replication_slot_1, replication_slot_2

### PgPool (Балансировщик нагрузки)
- **Порт**: 9999
- **Функции**: 
  - Балансировка нагрузки чтения
  - Проверка состояния серверов
  - Проксирование подключений

## 🚀 Быстрый запуск

### Предварительные требования
- Docker
- Docker Compose

### Запуск кластера

```bash
# Клонирование репозитория
git clone <repository-url>
cd postgres-replica-pgpoll

# Запуск всех сервисов
docker-compose up -d

# Проверка статуса сервисов
docker-compose ps
```

### Проверка работы репликации

```bash
# Подключение к primary через PgPool
psql -h localhost -p 9999 -U admin -d mydb

# Создание тестовой таблицы
CREATE TABLE test_replication (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO test_replication (data) VALUES ('test data');

# Проверка данных на репликах
psql -h localhost -p 5433 -U admin -d mydb -c "SELECT * FROM test_replication;"
psql -h localhost -p 5434 -U admin -d mydb -c "SELECT * FROM test_replication;"
```

## ⚙️ Конфигурация

### Переменные окружения

#### PostgreSQL Primary
```env
POSTGRES_USER=admin
POSTGRES_PASSWORD=secret
POSTGRES_DB=mydb
```

#### PostgreSQL Replicas
```env
PGUSER=replicator
PGPASSWORD=secret
```

#### PgPool
- Балансировка нагрузки включена
- Проверка состояния каждые 10 секунд
- 32 дочерних процесса
- Максимум 4 подключения в пуле

### Порты
- **PgPool**: 9999 → 5432 (основная точка подключения)
- **PostgreSQL Primary**: 5432 → 5432
- **PostgreSQL Replica 1**: 5433 → 5432
- **PostgreSQL Replica 2**: 5434 → 5432

## 🔧 Настройка репликации

Репликация настраивается автоматически через:

1. **init-primary.sh** - создает пользователя репликации и слоты
2. **pg_basebackup** - создает базовую копию для реплик
3. **Streaming replication** - синхронизация в реальном времени

### Параметры репликации

```sql
wal_level=replica
hot_standby=on
max_wal_senders=10
max_replication_slots=10
hot_standby_feedback=on
```

## 📊 Мониторинг и управление

### Проверка статуса репликации

```sql
-- На primary сервере
SELECT client_addr, state, sync_state FROM pg_stat_replication;

-- Проверка слотов репликации
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
```

### Проверка через PgPool

```bash
# Статус PgPool
docker exec pgpool pcp_node_info -h localhost -p 9898 -U admin -w

# Показать конфигурацию PgPool
docker exec pgpool cat /etc/pgpool2/pgpool.conf
```

### Логи

```bash
# Логи primary сервера
docker logs postgres-primary

# Логи реплик
docker logs postgres-replica1
docker logs postgres-replica2

# Логи PgPool
docker logs pgpool
```

## 🛠️ Устранение неполадок

### Проблемы с правами доступа к Docker

Если вы получаете ошибку "permission denied" при работе с Docker:

```bash
# Добавить пользователя в группу docker
sudo usermod -aG docker $USER

# Перезайти в систему или обновить группы
newgrp docker

# Альтернативно, перезапустить сессию
# logout и login снова
```

### Проблемы с подключением к репликам

1. Проверьте, что primary сервер запущен и доступен
2. Убедитесь, что пользователь replicator создан
3. Проверьте pg_hba.conf на primary сервере

### Проблемы с PgPool

1. Проверьте health check конфигурацию
2. Убедитесь, что все backend серверы доступны
3. Проверьте настройки аутентификации

### Сброс репликации

```bash
# Остановка сервисов
docker-compose down

# Удаление volumes (выберите один из вариантов)
# Вариант 1: Удаление с помощью docker-compose
docker-compose down -v

# Вариант 2: Ручное удаление volumes
docker volume ls | grep postgres-replica-pgpoll
docker volume rm postgres-replica-pgpoll_pg_primary_data postgres-replica-pgpoll_pg_replica1_data postgres-replica-pgpoll_pg_replica2_data

# Вариант 3: Удаление всех неиспользуемых volumes (будьте осторожны!)
docker volume prune

# Перезапуск
docker-compose up -d
```

## 📁 Структура проекта

```
postgres-replica-pgpoll/
├── docker-compose.yml      # Основная конфигурация Docker Compose
├── init-primary.sh         # Скрипт инициализации primary сервера
└── README.md               # Этот файл
```

## 🔒 Безопасность

⚠️ **Внимание**: Данная конфигурация предназначена для разработки и тестирования. Для production среды необходимо:

- Изменить пароли по умолчанию
- Настроить SSL/TLS
- Ограничить сетевой доступ
- Настроить мониторинг и алерты
- Регулярно обновлять образы Docker

## 📚 Дополнительные ресурсы

- [Документация PostgreSQL по репликации](https://www.postgresql.org/docs/current/high-availability.html)
- [Документация PgPool-II](https://www.pgpool.net/docs/latest/en/html/)
- [Docker образы PostgreSQL](https://hub.docker.com/_/postgres)
- [Docker образы PgPool](https://hub.docker.com/r/pgpool/pgpool)
