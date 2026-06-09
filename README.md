# Резервное копирование баз данных - Юртаева Софья Вячеславовна

---

## Задание 1. Резервное копирование

### 1.1. Восстановление данных в полном объёме за предыдущий день

Подходит **полное (full) резервное копирование**, выполняемое раз в сутки — например, ежедневно в 00:00.

**Принцип работы:**
- Каждую ночь создаётся полный дамп всей базы данных.
- При необходимости восстановления берётся дамп за прошлую ночь и разворачивается целиком.
- Потеря данных (RPO) — до 24 часов (все изменения за текущий день будут утеряны).

**Инструменты:** `pg_dump` / `pg_dumpall` для PostgreSQL, `mysqldump` для MySQL, физические снапшоты (LVM snapshot, облачные снапшоты дисков).

**Пример расписания (cron):**
```bash
0 0 * * * pg_dump -U postgres mydb > /backups/mydb_$(date +%F).sql
```

---

### 1.2. Восстановление данных за час до предполагаемой поломки

Подходит комбинация **полного резервного копирования + инкрементное/журнальное резервирование (Point-in-Time Recovery, PITR)**.

**Принцип работы:**
- Раз в сутки (или реже) делается полный бэкап — база восстановления.
- Непрерывно или с коротким интервалом (каждые 15–60 минут) архивируются журналы транзакций (WAL для PostgreSQL, binary log для MySQL).
- При аварии восстанавливается последний полный бэкап, затем поверх него воспроизводятся журналы вплоть до нужного момента времени.
- RPO — минуты (зависит от частоты архивации журналов).

**Инструменты:**
- PostgreSQL: `pg_basebackup` + архивирование WAL (`archive_command`) + `pg_restore` с параметром `--recovery-target-time`.
- MySQL: `mysqldump` / `xtrabackup` (full) + бинарные логи (`mysqlbinlog`) для воспроизведения до нужного момента.

**Пример параметра восстановления PostgreSQL:**
```
recovery_target_time = '2026-06-08 14:30:00'
```
---

## Задание 2. PostgreSQL

### 2.1. Резервное копирование и восстановление с помощью pg_dump / pg_restore

#### Резервное копирование — `pg_dump`

Создание дампа базы данных `mydb` в формате custom (бинарный, сжатый, поддерживает выборочное восстановление):

```bash
pg_dump -U postgres -h localhost -p 5432 -F c -b -v -f /backups/mydb.dump mydb
```

| Флаг | Описание |
|------|----------|
| `-U postgres` | пользователь PostgreSQL |
| `-h localhost` | хост |
| `-p 5432` | порт |
| `-F c` | формат custom (рекомендуется для pg_restore) |
| `-b` | включить большие объекты (blobs) |
| `-v` | verbose — подробный вывод |
| `-f /backups/mydb.dump` | путь к файлу дампа |
| `mydb` | имя базы данных |

Дамп в plain SQL (текстовый):
```bash
pg_dump -U postgres -h localhost mydb > /backups/mydb.sql
```
---

#### Восстановление — `pg_restore`

Сначала создаём пустую базу-получатель:
```bash
createdb -U postgres mydb_restored
```

Затем восстанавливаем из дампа формата custom:
```bash
pg_restore -U postgres -h localhost -p 5432 -d mydb_restored -v /backups/mydb.dump
```

| Флаг | Описание |
|------|----------|
| `-d mydb_restored` | целевая база данных |
| `-v` | verbose |
| `/backups/mydb.dump` | файл дампа |

Восстановление из plain SQL дампа:
```bash
psql -U postgres -d mydb_restored -f /backups/mydb.sql
```

---

## Задание 3. MySQL

### 3.1. Инкрементное резервное копирование MySQL с помощью MySQL Enterprise Backup / XtraBackup

Стандартный инструмент для инкрементных бэкапов MySQL — **Percona XtraBackup** (open source) или **MySQL Enterprise Backup** (официальный).

#### Шаг 1. Полный (full) бэкап — база для инкрементов

```bash
xtrabackup --backup \
  --user=root \
  --password=secret \
  --target-dir=/backups/full
```

#### Шаг 2. Первый инкрементный бэкап (изменения после full)

```bash
xtrabackup --backup \
  --user=root \
  --password=secret \
  --target-dir=/backups/inc1 \
  --incremental-basedir=/backups/full
```

`--incremental-basedir` — указывает на директорию предыдущего бэкапа (full или предыдущего инкремента). XtraBackup читает LSN (Log Sequence Number) из `xtrabackup_checkpoints` и копирует только изменённые страницы.

#### Шаг 3. Второй инкрементный бэкап (изменения после inc1)

```bash
xtrabackup --backup \
  --user=root \
  --password=secret \
  --target-dir=/backups/inc2 \
  --incremental-basedir=/backups/inc1
```

---

#### Восстановление из инкрементного бэкапа

**1. Подготовить full бэкап (без применения инкрементов):**
```bash
xtrabackup --prepare --apply-log-only --target-dir=/backups/full
```

**2. Применить inc1:**
```bash
xtrabackup --prepare --apply-log-only \
  --target-dir=/backups/full \
  --incremental-dir=/backups/inc1
```

**3. Применить inc2 (последний инкремент — без `--apply-log-only`):**
```bash
xtrabackup --prepare \
  --target-dir=/backups/full \
  --incremental-dir=/backups/inc2
```

**4. Скопировать файлы на место:**
```bash
xtrabackup --copy-back --target-dir=/backups/full
chown -R mysql:mysql /var/lib/mysql
```
