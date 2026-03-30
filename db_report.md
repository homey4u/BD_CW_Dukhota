# Контрольна робота: База даних платформи продажу квитків

> **Предмет:** Бази даних  
> **Завдання:** Спроектувати базу даних для платформи продажу квитків на події

---

## Зміст

1. [Опис предметної області](#1-опис-предметної-області)
2. [Схема бази даних](#2-схема-бази-даних)
3. [DDL: Створення таблиць](#3-ddl-створення-таблиць)
4. [Наповнення даними (Seed)](#4-наповнення-даними-seed)
5. [OLTP запити](#5-oltp-запити)
6. [OLAP запити](#6-olap-запити)

---

## 1. Опис предметної області

Платформа керує продажем квитків на різноманітні події. Система охоплює:

- **Майданчики** — місця проведення подій (стадіони, клуби, арени)
- **Події** — концерти, фестивалі, спортивні матчі, конференції тощо
- **Типи квитків** — VIP, стандарт, студентський з різними цінами для кожної події
- **Клієнти** — зареєстровані покупці квитків
- **Покупки** — записи про придбані квитки (кількість, ціна, дата)

---

## 2. Схема бази даних

<img width="915" height="717" alt="image" src="https://github.com/user-attachments/assets/dd90f88c-516f-4584-834c-6e8b651522b5" />

---

## 3. DDL: Створення таблиць

```sql
-- 1. Таблиця майданчиків (Venues)
CREATE TABLE venues (
    venue_id  SERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    location  VARCHAR(255) NOT NULL,
    capacity  INTEGER      NOT NULL CHECK (capacity > 0)
);

-- 2. Таблиця клієнтів (Customers)
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name  VARCHAR(50)  NOT NULL,
    last_name   VARCHAR(50)  NOT NULL,
    email       VARCHAR(100) NOT NULL UNIQUE,
    phone       VARCHAR(20)
);

-- 3. Таблиця подій (Events)
CREATE TABLE events (
    event_id        SERIAL PRIMARY KEY,
    venue_id        INTEGER      NOT NULL REFERENCES venues(venue_id),
    event_name      VARCHAR(150) NOT NULL,
    event_datetime  TIMESTAMP    NOT NULL,
    event_type      VARCHAR(50)  NOT NULL
);

-- 4. Таблиця типів квитків (Ticket_types)
CREATE TYPE ticket_type_enum AS ENUM ('VIP', 'standard', 'student');

CREATE TABLE ticket_types (
    ticket_id  SERIAL           PRIMARY KEY,
    event_id   INTEGER          NOT NULL REFERENCES events(event_id) ON DELETE CASCADE,
    type_name  ticket_type_enum NOT NULL,
    price      NUMERIC(8,2)     NOT NULL CHECK (price >= 0)
);

-- 5. Таблиця покупок (Purchases)
CREATE TABLE purchases (
    purchase_id   BIGSERIAL     PRIMARY KEY,
    ticket_id     INTEGER       NOT NULL REFERENCES ticket_types(ticket_id),
    customer_id   INTEGER       NOT NULL REFERENCES customers(customer_id),
    purchase_date TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    quantity      SMALLINT      NOT NULL CHECK (quantity > 0),
    total_price   NUMERIC(10,2) NOT NULL CHECK (total_price >= 0)
);

-- 6. Індекси на зовнішні ключі
CREATE INDEX idx_events_venue       ON events(venue_id);
CREATE INDEX idx_ticket_types_event ON ticket_types(event_id);
CREATE INDEX idx_purchases_ticket   ON purchases(ticket_id);
CREATE INDEX idx_purchases_customer ON purchases(customer_id);
```

> **Примітка:** Для `ticket_types` використовується тип `ticket_type_enum`, що гарантує цілісність даних і обмежує значення лише допустимими категоріями. Каскадне видалення (`ON DELETE CASCADE`) забезпечує автоматичне очищення типів квитків при видаленні події.

---


## 4. OLTP запити

### INSERT запити

<details>
<summary>Розгорнути INSERT-запити для наповнення таблиць</summary>
    
```sql
-- Майданчики
INSERT INTO venues (name, location, capacity) VALUES
    ('Олімпійський',  'Київ, вул. Спортивна 1',       70000),
    ('Палац Спорту',  'Київ, просп. Перемоги 2',        5000),
    ('Lviv Arena',    'Львів, вул. Стрийська 199',      8000),
    ('Meridian',      'Одеса, вул. Канатна 83',         1200),
    ('Atlas Weekend', 'Київ, НСК Олімпійський',        50000);

-- Клієнти
INSERT INTO customers (first_name, last_name, email, phone) VALUES
    ('Олексій', 'Коваленко', 'kovalenko@gmail.com', '+380671234567'),
    ('Марія',   'Шевченко',  'shevchenko@ukr.net',  '+380931234567'),
    ('Іван',    'Бойко',     'boyko@gmail.com',      '+380501234567'),
    ('Наталія', 'Мельник',   'melnyk@gmail.com',     '+380661234567'),
    ('Дмитро',  'Лисенко',   'lysenko@ukr.net',      '+380731234567'),
    ('Оксана',  'Гриценко',  'grytsenko@gmail.com',  '+380961234567'),
    ('Андрій',  'Савченко',  'savchenko@gmail.com',  '+380671112233'),
    ('Юлія',    'Ткаченко',  'tkachenko@ukr.net',    '+380931112233');

-- Події
INSERT INTO events (venue_id, event_name, event_datetime, event_type) VALUES
    (1, 'Концерт Океан Ельзи',     '2025-06-15 20:00:00', 'concert'),
    (1, 'Фінал Ліги Чемпіонів',    '2025-07-01 21:00:00', 'sport'),
    (2, 'Вечір Stand-Up Комедії',   '2025-05-20 19:00:00', 'standup'),
    (2, 'Концерт Kalush Orchestra', '2025-06-28 20:00:00', 'concert'),
    (3, 'Lviv Jazz Festival',       '2025-07-10 18:00:00', 'concert'),
    (3, 'IT-конференція TechUA',    '2025-08-05 09:00:00', 'conference'),
    (4, 'Вечірка Meridian NYE',     '2025-12-31 22:00:00', 'party'),
    (5, 'Atlas Weekend 2025',       '2025-07-18 14:00:00', 'festival');

-- Типи квитків
INSERT INTO ticket_types (event_id, type_name, price) VALUES
    (1, 'VIP', 2500.00), (1, 'standard', 800.00), (1, 'student', 400.00),
    (2, 'VIP', 5000.00), (2, 'standard', 1500.00),
    (3, 'standard', 350.00), (3, 'student', 200.00),
    (4, 'VIP', 3000.00), (4, 'standard', 900.00), (4, 'student', 500.00),
    (5, 'VIP', 1800.00), (5, 'standard', 600.00),
    (6, 'VIP', 4000.00), (6, 'standard', 1200.00), (6, 'student', 700.00),
    (7, 'VIP', 2000.00), (7, 'standard', 800.00),
    (8, 'VIP', 6000.00), (8, 'standard', 1500.00), (8, 'student', 900.00);
```

</details>

---

### UPDATE запити

**Запит 1.** Оновлення ціни VIP-квитків для концерту Океан Ельзи.

```sql
UPDATE ticket_types
SET price = 2500.00
WHERE event_id = 1
  AND type_name = 'VIP';
```

**Запит 2.** Оновлення номера телефону клієнта Олексія Коваленка.

```sql
UPDATE customers
SET phone = '+380991234567'
WHERE customer_id = 1;
```

---

### DELETE запити

**Запит 1.** Видалення окремої покупки (скасування замовлення).

```sql
DELETE FROM purchases
WHERE purchase_id = 1;
```

**Запит 2.** Видалення події. Завдяки `ON DELETE CASCADE` всі пов'язані типи квитків видаляються автоматично.

```sql
DELETE FROM events
WHERE event_id = 1;
```

---

### SELECT запити

**Запит 1.** Знайти всі події на конкретному майданчику (наприклад, Олімпійський, `venue_id = 1`).

```sql
SELECT
    e.event_id,
    e.event_name,
    e.event_datetime,
    e.event_type,
    v.name     AS venue_name,
    v.location
FROM events e
JOIN venues v ON e.venue_id = v.venue_id
WHERE v.venue_id = 1
ORDER BY e.event_datetime;
```

**Запит 2.** Порахувати доступні квитки для події за типом квитка. Доступність розраховується як різниця між місткістю майданчика та вже проданими квитками.

```sql
SELECT
    tt.type_name,
    tt.price,
    v.capacity                              AS total_capacity,
    COALESCE(SUM(p.quantity), 0)            AS tickets_sold,
    v.capacity - COALESCE(SUM(p.quantity), 0) AS tickets_available
FROM ticket_types tt
JOIN events  e  ON tt.event_id  = e.event_id
JOIN venues  v  ON e.venue_id   = v.venue_id
LEFT JOIN purchases p ON p.ticket_id = tt.ticket_id
WHERE tt.event_id = 1
GROUP BY tt.type_name, tt.price, v.capacity
ORDER BY tt.type_name;
```

---

## 5. OLAP запити

### Запит 1. Загальний дохід за типом події

Об'єднуємо таблиці подій, типів квитків та покупок, щоб підсумувати виручку по кожному типу події (концерт, фестиваль, спорт тощо).

```sql
SELECT
    e.event_type,
    SUM(p.total_price) AS total_revenue
FROM events e
JOIN ticket_types tt ON e.event_id  = tt.event_id
JOIN purchases    p  ON tt.ticket_id = p.ticket_id
GROUP BY e.event_type
ORDER BY total_revenue DESC;
```

---

### Запит 2. Топ-10 найбільш продаваних подій

Рахуємо загальну кількість проданих квитків (`SUM(p.quantity)`) для кожної події. Колонка `total_revenue` додана для повноти аналізу; основне сортування — за кількістю квитків.

```sql
SELECT
    e.event_name,
    SUM(p.quantity)    AS total_tickets_sold,
    SUM(p.total_price) AS total_revenue
FROM events e
JOIN ticket_types tt ON e.event_id  = tt.event_id
JOIN purchases    p  ON tt.ticket_id = p.ticket_id
GROUP BY e.event_id, e.event_name
ORDER BY total_tickets_sold DESC
LIMIT 10;
```

---

### Запит 3. Середня ціна квитка за майданчиком

Для кожного майданчика рахуємо середнє арифметичне значення поля `price` по всіх типах квитків усіх подій, що проходять на цьому майданчику. `ROUND(..., 2)` обмежує результат двома знаками після коми.

```sql
SELECT
    v.name                    AS venue_name,
    ROUND(AVG(tt.price), 2)   AS avg_ticket_price
FROM venues v
JOIN events      e  ON v.venue_id  = e.venue_id
JOIN ticket_types tt ON e.event_id = tt.event_id
GROUP BY v.venue_id, v.name
ORDER BY avg_ticket_price DESC;
```

---

### Запит 4. Клієнти, які купили квитки на декілька подій в один день

**Запит 1.** Переносимо Stand-Up на дату Океану Ельзи, щоб Олексій міг купити квиток на обидві події в один день (підготовка до перевірки OLAP-запиту №4).

```sql
-- Змінюємо дату Stand-Up (event_id = 3) на 15 червня 2025 року
UPDATE events
SET event_datetime = '2025-06-15 16:00:00'
WHERE event_id = 3;
```

**Запит 2.** Олексій (customer_id = 1) купує 2 квитки на Stand-Up (ticket_id = 6).

```sql
INSERT INTO purchases (ticket_id, customer_id, purchase_date, quantity, total_price)
VALUES (6, 1, CURRENT_TIMESTAMP, 2, 700.00);
```

Функція `DATE()` зрізає час з `event_datetime`, залишаючи лише дату. `HAVING COUNT(DISTINCT e.event_id) > 1` відфільтровує лише тих клієнтів, у яких більше однієї унікальної події припадає на ту саму дату.

```sql
SELECT
    c.first_name,
    c.last_name,
    DATE(e.event_datetime)    AS event_date,
    COUNT(DISTINCT e.event_id) AS events_count
FROM customers c
JOIN purchases    p  ON c.customer_id = p.customer_id
JOIN ticket_types tt ON p.ticket_id   = tt.ticket_id
JOIN events       e  ON tt.event_id   = e.event_id
GROUP BY c.customer_id, c.first_name, c.last_name, DATE(e.event_datetime)
HAVING COUNT(DISTINCT e.event_id) > 1;
```

