# KSK Management System

# Техническое задание (ТЗ)

# Версия 1.0

# Часть 2

# Архитектура базы данных PostgreSQL, таблицы, связи, модели SQLAlchemy и правила хранения данных

---

# 1. Общие требования к базе данных

## Система управления базой данных

Использовать:

```
PostgreSQL 16+
```

---

## ORM

Использовать:

```
SQLAlchemy 2.x Async
```

---

## Миграции

Использовать:

```
Alembic
```

---

## Основные требования:

База данных должна обеспечивать:

* хранение структуры домов;
* связь пользователей с квартирами;
* автоматическое начисление платежей;
* хранение всех фактических оплат;
* распределение оплат по месяцам;
* учёт доходов и расходов;
* финансовый баланс;
* аудит всех действий.

---

# 2. Главный принцип проектирования БД

Система должна разделять:

```
Объекты недвижимости

+

Людей

+

Обязательства

+

Деньги

+

Документы
```

---

Нельзя хранить всё в одной таблице.

Неправильно:

```
Apartment

amount_paid

debt
```

Почему:

* человек может платить заранее;
* может платить частями;
* может платить после нескольких месяцев долга;
* тариф может измениться.

---

Правильно:

```
Apartment

      |

      +---- Charges

      |

      +---- Payments

      |

      +---- Payment Allocations

      |

      +---- Transactions
```

---

# 3. Общая схема базы данных

Полная архитектура:

```
                         roles
                           |
                           |
                         users
                           |
                    user_apartments
                           |
                           |
                       apartments
                           |
                       entrances
                           |
                       buildings


apartments
     |
     |
  charges
     |
     |
payment_allocations
     |
     |
 payments
     |
     |
transactions


employees
     |
salary_payments
     |
expenses
     |
transactions


receipts
     |
receipt_ocr_data


audit_logs
```

---

# 4. Таблица roles

## Назначение

Хранение ролей пользователей.

---

Название:

```
roles
```

---

Структура:

```sql
id
name
description
created_at
```

---

Пример:

| id | name     |
| -- | -------- |
| 1  | admin    |
| 2  | resident |
| 3  | employee |
| 4  | manager  |

---

SQLAlchemy:

```python
class Role(Base):

    __tablename__ = "roles"

    id = Column(Integer, primary_key=True)

    name = Column(String, unique=True)

    description = Column(Text)

    created_at = Column(DateTime)
```

---

# 5. Таблица users

## Назначение

Все пользователи системы.

---

Таблица:

```
users
```

---

Поля:

```sql
id

email

password_hash

full_name

phone

role_id

is_active

created_at

updated_at
```

---

Связь:

```
Role

1

|

N

User
```

---

SQLAlchemy:

```python
class User(Base):

    __tablename__="users"


    id = Column(Integer, primary_key=True)


    email = Column(String, unique=True)


    password_hash = Column(String)


    role_id = Column(
        ForeignKey("roles.id")
    )
```

---

# 6. Таблица buildings

## Назначение

Хранение домов.

---

Таблица:

```
buildings
```

---

Поля:

```sql
id

name

address

floors_count

entrances_count

created_at
```

---

Пример:

```
Дом №15

ул. Абая

4 этажа

6 подъездов
```

---

Связь:

```
Building

1

|

N

Entrance
```

---

# 7. Таблица entrances

## Назначение

Подъезды дома.

---

Таблица:

```
entrances
```

---

Поля:

```sql
id

building_id

number

created_at
```

---

Пример:

```
Подъезд №3
```

---

Связь:

```
Building

1

|

N

Entrances
```

---

# 8. Таблица apartments

## Назначение

Квартиры.

---

Таблица:

```
apartments
```

---

Поля:

```sql
id

entrance_id

number

floor

area

monthly_fee

status

created_at

updated_at
```

---

Пример:

```
Квартира:

25


Подъезд:

4


Этаж:

3


Тариф:

5400
```

---

Статусы:

```
active

inactive

empty
```

---

Связь:

```
Entrance

1

|

N

Apartment
```

---

# 9. Таблица user_apartments

## Назначение

Связь жильцов и квартир.

---

Почему отдельная таблица:

Одна квартира может иметь:

* владельца;
* членов семьи;
* несколько пользователей.

---

Таблица:

```
user_apartments
```

---

Поля:

```sql
id

user_id

apartment_id

is_owner

created_at
```

---

Пример:

```
Иван

 |

Квартира №25

 |

owner=true
```

---

Связь:

```
User

N

|

N

Apartment
```

---

# 10. Таблица charges

# Основная финансовая таблица №1

## Назначение

Хранит обязательства квартир.

---

Очень важная таблица.

Она отвечает:

> Сколько квартира должна?

---

Таблица:

```
charges
```

---

Поля:

```sql
id

apartment_id

period_year

period_month

amount

paid_amount

status

created_at

updated_at
```

---

Пример:

```
Квартира 25

Август 2026

5400 ₸
```

---

Статусы:

```
unpaid

partial

paid

overpaid
```

---

Пример:

```
charge:

5400


paid_amount:

3000


status:

partial
```

---

Связь:

```
Apartment

1

|

N

Charges
```

---

# 11. Таблица payments

# Основная финансовая таблица №2

## Назначение

Фактические поступления денег.

---

Она отвечает:

> Сколько денег реально пришло?

---

Таблица:

```
payments
```

---

Поля:

```sql
id

apartment_id

receipt_id

amount

payment_date

payment_method

comment

created_by

created_at
```

---

Пример:

```
Квартира 25

05.08.2026

64800 ₸
```

---

Способы оплаты:

```
cash

bank_transfer

kaspi

card

other
```

---

Связь:

```
Apartment

1

|

N

Payments
```

---

# 12. Таблица payment_allocations

# Основная финансовая таблица №3

## Назначение

Связывает деньги с месяцами.

---

Она отвечает:

> За какие периоды была оплата?

---

Таблица:

```
payment_allocations
```

---

Поля:

```sql
id

payment_id

charge_id

allocated_amount

created_at
```

---

Пример:

Payment:

```
64800
```

Allocation:

```
Август:

5400


Сентябрь:

5400


Октябрь:

5400
```

---

Связь:

```
Payment

N

|

N

Charge
```

---

# 13. Таблица transactions

# Главная кассовая таблица

## Назначение

Все движения денег.

---

Она отвечает:

> Сколько денег сейчас в системе?

---

Таблица:

```
transactions
```

---

Поля:

```sql
id

type

amount

category

description

reference_type

reference_id

created_by

created_at
```

---

Типы:

```
income

expense
```

---

Пример:

Доход:

```
income

5400

Квартира 25
```

---

Расход:

```
expense

80000

Зарплата уборщика
```

---

Формула:

```
Баланс = income - expense
```

---

# 14. Таблица employees

## Назначение

Работники дома.

---

Таблица:

```
employees
```

---

Поля:

```sql
id

building_id

full_name

position

phone

salary

status

created_at
```

---

Пример:

```
Иван Петров

Уборщик

80000 ₸
```

---

# 15. Таблица salary_payments

## Назначение

История выплат зарплат.

---

Таблица:

```
salary_payments
```

---

Поля:

```sql
id

employee_id

amount

payment_date

period_month

period_year

transaction_id

created_by

created_at
```

---

Связь:

```
Employee

1

|

N

Salary Payments
```

---

# 16. Таблица expenses

## Назначение

Расходы дома.

---

Таблица:

```
expenses
```

---

Поля:

```sql
id

building_id

category

amount

date

description

employee_id

document_id

created_by

created_at
```

---

Категории:

```
salary

cleaning_materials

garbage

repair

equipment

other
```

---

# 17. Таблица receipts

## Назначение

Документы оплаты.

---

Таблица:

```
receipts
```

---

Поля:

```sql
id

payment_id

file_path

file_name

document_number

created_at
```

---

Хранение:

```
MinIO
```

---

# 18. Таблица receipt_ocr_data

## Назначение

Результаты OCR.

---

Поля:

```sql
id

receipt_id

raw_text

amount

flat_number

payment_date

bank_name

confidence

created_at
```

---

Связь:

```
Receipt

1

|

1

OCR Data
```

---

# 19. Таблица audit_logs

## Назначение

История действий пользователей.

---

Поля:

```sql
id

user_id

action

entity

entity_id

old_value

new_value

created_at
```

---

Пример:

```
05.08.2026

Admin

добавил платеж

Квартира 25

5400 ₸
```

---

# 20. Правила хранения финансовых данных

## Правило №1

Нельзя изменять старые платежи.

Использовать:

```
immutable records
```

---

Если ошибка:

Не удалять.

Создать:

```
correction transaction
```

---

# Правило №2

Каждый доход должен иметь источник.

Например:

```
Payment

↓

Transaction

↓

Balance
```

---

# Правило №3

Каждый расход должен иметь подтверждение.

Например:

```
Expense

↓

Document

↓

Transaction
```

---

# Правило №4

Баланс никогда не хранить вручную.

Нельзя:

```
balance = 500000
```

---

Баланс рассчитывается:

```
SUM(income)

-

SUM(expense)
```

---

# Правило №5

Долг квартиры не хранить отдельно.

Нельзя:

```
Apartment.debt
```

---

Долг рассчитывается:

```
SUM(charges)

-

SUM(payment_allocations)
```

---

# 21. Финальная модель данных

Итог:

```
                USER SYSTEM


roles

 |

users

 |

user_apartments

 |

apartments

 |

entrances

 |

buildings



              FINANCE SYSTEM


apartments

 |

charges

 |

payments

 |

payment_allocations

 |

transactions



              EXPENSE SYSTEM


employees

 |

salary_payments

 |

expenses

 |

transactions



              DOCUMENT SYSTEM


receipts

 |

receipt_ocr_data



              CONTROL SYSTEM


audit_logs
```

---

# 22. Критерий правильной архитектуры БД

База данных считается правильно спроектированной, если система всегда может ответить:

### По квартире:

* сколько начислено;
* сколько оплачено;
* сколько должен;
* сколько месяцев закрыто;
* есть ли переплата.

---

### По дому:

* сколько начислено всем;
* сколько собрано;
* сколько долгов;
* сколько денег в кассе;
* сколько расходов;
* куда ушли деньги.

---

**Конец Части 2.**

Следующая часть документа:

# Часть 3

## Backend архитектура, FastAPI структура проекта, API endpoints, сервисы, бизнес-логика, Celery задачи и алгоритмы расчёта.
