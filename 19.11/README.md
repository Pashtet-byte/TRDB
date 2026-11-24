# Практическая работа №3

**Тема:** Структуры данных СУБД, общий подход к организации представлений, таблиц, индексов и кластеров

**Вариант №13. Предметная область: Сеть АЗС**

| Параметр | Задание варианта |
| --- | --- |
| Основная таблица и нагрузка | `Fuel_Sales` — чеки отпуска топлива с колонок |
| Представления (View / MatView) | View смены кассира; MatView: расход марок топлива по дням |
| Индексирование и кластеризация | Составной индекс `(station_id, sale_time)`; кластеризация по `sale_time` |

**СУБД:** PostgreSQL 16.

---

## 1. Цель работы

Изучить внутренние структуры хранения данных в реляционных СУБД и на примере сети АЗС научиться проектировать оптимизированные таблицы, создавать обычные и материализованные представления, подбирать типы индексов (составные, частичные, покрывающие) и применять кластерное упорядочение таблицы (`CLUSTER`) для ускорения доступа и защиты данных.

**Характер нагрузки:** таблица `fuel_sales` — это журнал чеков, который только пополняется (append-only): каждая заправка на каждой колонке даёт одну строку. Для сети из сотен АЗС это десятки миллионов строк в год. Типовые запросы:

- выручка и литры по конкретной АЗС за период (оперативные отчёты, сверка смен);
- продажи конкретного кассира за текущую смену (работа кассира);
- расход марок топлива по дням (аналитика закупок и прогноз остатков);
- аннулированные чеки (контроль злоупотреблений).

---

## 2. Проектирование базовых таблиц (PostgreSQL)

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE SCHEMA IF NOT EXISTS azs_core;
SET search_path TO azs_core, public;

-- Справочник АЗС
CREATE TABLE fuel_stations (
    station_id   SERIAL PRIMARY KEY,
    station_code VARCHAR(10)  NOT NULL UNIQUE,
    city         VARCHAR(50)  NOT NULL,
    address      VARCHAR(150) NOT NULL,
    pumps_count  SMALLINT     NOT NULL CHECK (pumps_count > 0)
);

-- Справочник марок топлива
CREATE TABLE fuel_types (
    fuel_type_id SMALLSERIAL PRIMARY KEY,
    fuel_name    VARCHAR(20) NOT NULL UNIQUE,
    fuel_group   VARCHAR(10) NOT NULL CHECK (fuel_group IN ('petrol', 'diesel', 'gas'))
);

-- Высоконагруженная таблица чеков отпуска топлива
CREATE TABLE fuel_sales (
    sale_id            BIGSERIAL PRIMARY KEY,
    receipt_no         VARCHAR(20)   NOT NULL,
    station_id         INT           NOT NULL
        REFERENCES fuel_stations (station_id) ON DELETE RESTRICT,
    pump_no            SMALLINT      NOT NULL CHECK (pump_no BETWEEN 1 AND 50),
    fuel_type_id       SMALLINT      NOT NULL
        REFERENCES fuel_types (fuel_type_id) ON DELETE RESTRICT,
    liters             NUMERIC(6,2)  NOT NULL CHECK (liters > 0 AND liters <= 1000),
    price_per_liter    NUMERIC(6,2)  NOT NULL CHECK (price_per_liter > 0),
    -- Сумма чека вычисляется СУБД: исключает расхождение litres * price и total
    total_rub          NUMERIC(10,2) GENERATED ALWAYS AS (ROUND(liters * price_per_liter, 2)) STORED,
    payment_type       VARCHAR(10)   NOT NULL CHECK (payment_type IN ('cash', 'card', 'corp_card')),
    card_pan_masked    VARCHAR(19),               -- маскированный номер карты
    card_pan_encrypted BYTEA,                     -- зашифрованный полный номер карты
    cashier_login      VARCHAR(30)   NOT NULL,    -- учётная запись кассира в БД
    is_voided          BOOLEAN       NOT NULL DEFAULT FALSE,   -- чек аннулирован
    sale_time          TIMESTAMPTZ   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_receipt UNIQUE (station_id, receipt_no),
    -- Данные карты обязательны при безналичной оплате и запрещены при наличной
    CONSTRAINT chk_card_data CHECK (
        (payment_type = 'cash'
            AND card_pan_encrypted IS NULL AND card_pan_masked IS NULL)
        OR
        (payment_type <> 'cash'
            AND card_pan_encrypted IS NOT NULL AND card_pan_masked IS NOT NULL)
    )
);
```

### Обоснование типов данных

| Поле | Тип | Обоснование |
| --- | --- | --- |
| `sale_id` | `BIGSERIAL` (8 байт) | таблица растёт на десятки миллионов строк в год, `INT` со временем переполнится |
| `station_id` | `INT` (4 байта) | количество АЗС небольшое, `BIGINT` не нужен; ключ повторяется в каждой строке чеков, поэтому экономия важна |
| `fuel_type_id`, `pump_no` | `SMALLINT` (2 байта) | марок топлива единицы, колонок не больше нескольких десятков; значение повторяется в каждой строке самой большой таблицы |
| `liters`, `price_per_liter`, `total_rub` | `NUMERIC` | точная десятичная арифметика, `FLOAT` для денег и литров недопустим |
| `fuel_name`, `fuel_group` | `VARCHAR` в справочнике | название марки хранится один раз, в чеках только `SMALLINT`-ключ |
| `payment_type` | `VARCHAR(10)` + `CHECK` | короткий справочный признак без отдельной таблицы |
| `is_voided` | `BOOLEAN` (1 байт) | флаг аннулирования |
| `sale_time` | `TIMESTAMPTZ` | учёт часовых поясов (АЗС в разных регионах), 8 байт |
| `card_pan_encrypted` | `BYTEA` | зашифрованное значение (`pgcrypto`), полный номер карты в открытом виде не хранится |

---

## 3. Тестовые данные

Для проверки индексов и кластеризации генерируется 500 000 чеков за последние 90 дней. Чеки вставляются в **случайном порядке времени**, чтобы физическое расположение строк было неупорядоченным и эффект `CLUSTER` был наглядным (в реальной системе строки поступают в порядке времени).

```sql
INSERT INTO fuel_stations (station_code, city, address, pumps_count)
SELECT 'AZS-' || LPAD(g::text, 3, '0'),
       (ARRAY['Москва', 'Казань', 'Самара', 'Тверь'])[1 + g % 4],
       'ул. Трассовая, ' || g,
       8
FROM generate_series(1, 20) g;

INSERT INTO fuel_types (fuel_name, fuel_group) VALUES
    ('АИ-92', 'petrol'), ('АИ-95', 'petrol'), ('АИ-98', 'petrol'),
    ('ДТ', 'diesel'), ('Газ', 'gas');

INSERT INTO fuel_sales (receipt_no, station_id, pump_no, fuel_type_id, liters,
                        price_per_liter, payment_type, card_pan_masked,
                        card_pan_encrypted, cashier_login, is_voided, sale_time)
SELECT g::text,
       1 + g % 20,
       1 + g % 8,
       1 + g % 5,
       ROUND((10 + random() * 50)::numeric, 2),
       ROUND((50 + (g % 5) * 4)::numeric, 2),
       CASE WHEN g % 3 = 0 THEN 'cash' ELSE 'card' END,
       CASE WHEN g % 3 = 0 THEN NULL ELSE '4276 **** **** 1234' END,
       -- заглушка вместо шифртекста (в реальной системе: pgp_sym_encrypt(...))
       CASE WHEN g % 3 = 0 THEN NULL ELSE decode(md5(g::text), 'hex') END,
       CASE WHEN g % 10 = 0 THEN 'cashier_demo' ELSE 'cashier_' || (g % 10) END,
       (g % 100 = 0),
       CURRENT_TIMESTAMP - random() * INTERVAL '90 days'
FROM generate_series(1, 500000) g;

ANALYZE;
```

---

## 4. Организация представлений

### 4.1. Обычное представление — «смена кассира» (защита данных)

Кассир видит только **свои** неаннулированные чеки за текущую смену (последние 12 часов). Из представления исключены зашифрованный номер карты и учётная запись кассира. Фильтр по `current_user` встроен в представление, поэтому кассир физически не может увидеть чужие чеки. Параметр `security_barrier` не позволяет пользовательским функциям в запросе к представлению обойти фильтр.

```sql
CREATE OR REPLACE VIEW v_cashier_shift_sales
WITH (security_barrier = true) AS
SELECT
    fs.sale_id,
    fs.receipt_no,
    st.station_code,
    fs.pump_no,
    ft.fuel_name,
    fs.liters,
    fs.price_per_liter,
    fs.total_rub,
    fs.payment_type,
    fs.card_pan_masked,
    fs.sale_time
FROM fuel_sales fs
JOIN fuel_stations st ON st.station_id   = fs.station_id
JOIN fuel_types    ft ON ft.fuel_type_id = fs.fuel_type_id
WHERE fs.cashier_login = current_user
  AND NOT fs.is_voided
  AND fs.sale_time >= CURRENT_TIMESTAMP - INTERVAL '12 hours';
```

Что даёт обычное представление:

- **безопасность:** скрыты чувствительные столбцы и чужие строки;
- **логическая независимость:** приложение кассира работает с представлением и не зависит от физической структуры таблиц (можно добавить партиционирование или переименовать столбцы, не меняя приложение);
- представление не хранит данные, поэтому всегда актуально.

### 4.2. Материализованное представление — расход марок топлива по дням

Агрегация `SUM` и `GROUP BY` по всей таблице чеков тяжёлая, поэтому результат считается заранее и хранится физически.

```sql
CREATE MATERIALIZED VIEW mv_daily_fuel_consumption AS
SELECT
    (fs.sale_time AT TIME ZONE 'Europe/Moscow')::date AS sale_date,
    st.station_code,
    ft.fuel_name,
    COUNT(*)            AS receipts_count,
    SUM(fs.liters)      AS total_liters,
    SUM(fs.total_rub)   AS total_revenue_rub
FROM fuel_sales fs
JOIN fuel_stations st ON st.station_id   = fs.station_id
JOIN fuel_types    ft ON ft.fuel_type_id = fs.fuel_type_id
WHERE NOT fs.is_voided
GROUP BY 1, 2, 3
WITH DATA;

-- Уникальный индекс: обязателен для REFRESH ... CONCURRENTLY
CREATE UNIQUE INDEX idx_mv_fuel_day
    ON mv_daily_fuel_consumption (sale_date, station_code, fuel_name);
```

**Стратегия обновления:**

```sql
-- Регламентное обновление без блокировки чтения (читатели видят старую версию до конца обновления)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_fuel_consumption;
```

- **Ночью (03:00)** — полное обновление по расписанию, когда нагрузка на кассы минимальна. Для отчётов по закрытым дням этого достаточно.
- **Днём — раз в час** — если нужны данные текущего дня для оперативного контроля остатков. Данные в представлении всегда отстают от таблицы на период между обновлениями, это осознанный компромисс «актуальность против нагрузки».
- Запуск по расписанию: расширение `pg_cron` (`SELECT cron.schedule('0 3 * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY azs_core.mv_daily_fuel_consumption');`) или `cron` на сервере с вызовом `psql -c "..."`.
- `CONCURRENTLY` требует уникального индекса и уже заполненного представления; он медленнее обычного обновления, но не блокирует аналитиков.
- Если таблица вырастет до сотен миллионов строк, полное пересчитывание станет слишком дорогим: тогда агрегаты обновляют инкрементально (отдельная сводная таблица, в которую добавляются только новые сутки).

---

## 5. Проектирование индексов под нагрузку

```sql
-- 1. Составной + покрывающий B-Tree индекс (основной по заданию варианта).
--    Ключ (station_id, sale_time) обслуживает фильтр по АЗС и диапазону времени
--    с готовой сортировкой; INCLUDE добавляет столбцы, которые читает отчёт,
--    что позволяет выполнять запрос Index Only Scan без обращения к таблице.
CREATE INDEX idx_fs_station_time
    ON fuel_sales (station_id, sale_time DESC)
    INCLUDE (fuel_type_id, liters, total_rub);

-- 2. Частичный индекс только по аннулированным чекам (около 1% таблицы).
--    Остальные 99% строк в него не попадают: индекс очень компактный.
CREATE INDEX idx_fs_voided
    ON fuel_sales (sale_time)
    WHERE is_voided;

-- 3. Частичный составной индекс для представления смены кассира:
--    поиск по логину и времени, аннулированные чеки исключены.
CREATE INDEX idx_fs_cashier_shift
    ON fuel_sales (cashier_login, sale_time DESC)
    WHERE NOT is_voided;
```

### Обоснование выбора индексов

| Типовой запрос | Индекс | Почему именно он |
| --- | --- | --- |
| Выручка и литры по АЗС за период: `WHERE station_id = ? AND sale_time BETWEEN ? AND ?` | `idx_fs_station_time` (составной, покрывающий) | порядок столбцов: сначала равенство (`station_id`), затем диапазон (`sale_time`); `INCLUDE` даёт Index Only Scan |
| Контроль аннулированных чеков: `WHERE is_voided AND sale_time > ?` | `idx_fs_voided` (частичный) | выборка редкая, индекс на ~1% строк занимает минимум места и почти не замедляет вставку |
| Представление смены кассира: `WHERE cashier_login = current_user AND NOT is_voided AND sale_time >= ?` | `idx_fs_cashier_shift` (частичный составной) | условие индекса совпадает с условием представления, поэтому планировщик его использует |
| Поиск чека по номеру на АЗС | автоматический индекс `uq_receipt` (`station_id`, `receipt_no`) | создаётся ограничением `UNIQUE` |

**Почему не другие типы индексов.** `Hash`-индекс не поддерживает диапазоны и сортировку, а основные запросы именно диапазонные по времени. `GIN` нужен для `JSONB`, массивов и полнотекстового поиска; таких полей в таблице чеков нет. Индекс по `fuel_type_id` не создаётся: в справочнике всего 5 значений, избирательность столбца низкая, а все аналитические запросы по маркам топлива и так читают большую часть таблицы (их обслуживает материализованное представление). Каждый лишний индекс замедляет `INSERT`, а таблица чеков принимает запись непрерывно.

---

## 6. Кластерная организация таблицы

Большинство выборок читают чеки за период («за последние сутки», «за месяц»), а при случайном порядке строк на диске каждая нужная строка лежит в отдельной 8-килобайтной странице. Это приводит к большому числу случайных чтений (Random I/O).

**Решение:** физически упорядочить таблицу `fuel_sales` по времени продажи.

```sql
-- 1. B-Tree индекс по времени продажи (для CLUSTER нужен именно B-Tree индекс)
CREATE INDEX idx_fs_sale_time ON fuel_sales (sale_time);

-- 2. Физическая перестройка таблицы в порядке индекса
CLUSTER fuel_sales USING idx_fs_sale_time;

-- 3. Обновление статистики планировщика
ANALYZE fuel_sales;
```

**Эффект для дискового ввода-вывода.** Чеки одного дня или часа оказываются в соседних страницах, поэтому диапазонный запрос `WHERE sale_time BETWEEN ...` читает непрерывный участок файла (Sequential I/O) вместо разрозненных страниц, а чтение одной страницы сразу даёт много нужных строк. Это также ускоряет построение материализованного представления и частые отчёты по дням.

**Ограничения, которые нужно учитывать:**

- `CLUSTER` — разовая операция: новые строки вставляются в конец и упорядоченность со временем нарушается. Для журнала чеков это почти не проблема, потому что строки и так поступают в порядке времени; в учебном примере порядок специально нарушен.
- Степень упорядоченности проверяется запросом `SELECT correlation FROM pg_stats WHERE tablename = 'fuel_sales' AND attname = 'sale_time';` (значение, близкое к 1, означает почти идеальный порядок).
- Во время `CLUSTER` таблица блокируется эксклюзивной блокировкой (чтение и запись невозможны), поэтому операцию выполняют в окно обслуживания; без остановки работы используют расширение `pg_repack`.
- `CLUSTER` полностью перезаписывает таблицу и требует свободного места, равного её размеру.
- Для таблицы, которая растёт бесконечно, следующий шаг — партиционирование по месяцам (`PARTITION BY RANGE (sale_time)`): старые партиции упорядочены и неизменны, а новая всегда пополняется в порядке времени.

---

## 7. Защита данных: роли и привилегии

```sql
CREATE ROLE cashier_role;                          -- кассиры АЗС
CREATE ROLE analyst_role;                          -- аналитики сети
CREATE ROLE cashier_demo LOGIN IN ROLE cashier_role;   -- пример учётной записи кассира

-- Прямой доступ к таблицам закрыт для всех
REVOKE ALL ON fuel_sales, fuel_stations, fuel_types FROM PUBLIC;

GRANT USAGE ON SCHEMA azs_core TO cashier_role, analyst_role;

-- Кассир работает только через представление своей смены
GRANT SELECT ON v_cashier_shift_sales TO cashier_role;

-- Аналитик видит только агрегированные данные
GRANT SELECT ON mv_daily_fuel_consumption TO analyst_role;
```

- Кассир не имеет прав на таблицу `fuel_sales`: он не видит `card_pan_encrypted`, чужие чеки и не может изменять данные.
- Аналитики работают только с материализованным представлением: в нём нет ни номеров карт, ни логинов кассиров, ни отдельных чеков.
- Номер карты шифруется функциями `pgcrypto` (`pgp_sym_encrypt`), ключ шифрования хранится вне БД; в чеках остаётся только маскированное значение.

---

## 8. Выводы

Для сети АЗС спроектированы три связанные таблицы (`fuel_stations`, `fuel_types`, `fuel_sales`) с экономными типами данных и ограничениями целостности, включая вычисляемую сумму чека. Создано обычное представление смены кассира, которое скрывает чувствительные столбцы и чужие строки, и материализованное представление расхода марок топлива по дням с регламентным обновлением `REFRESH ... CONCURRENTLY`. Подобраны три индекса разных типов: составной покрывающий `(station_id, sale_time)`, частичный по аннулированным чекам и частичный составной под представление кассира; выбор обоснован типовыми запросами. Таблица чеков физически упорядочена по времени продажи командой `CLUSTER`, что превращает диапазонные выборки из случайного чтения в последовательное; описаны ограничения кластеризации и путь развития через партиционирование.
