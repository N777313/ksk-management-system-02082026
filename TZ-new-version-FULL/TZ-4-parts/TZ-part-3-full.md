# KSK Management System

# Техническое задание (ТЗ)

# Версия 1.0

# Часть 3

# Backend архитектура, FastAPI, API, бизнес-логика, сервисы, Celery и алгоритмы расчёта

---

# 1. Общая архитектура Backend

Backend разрабатывается как REST API сервис.

Основные требования:

* асинхронная обработка запросов;
* модульная архитектура;
* независимость бизнес-логики от API;
* возможность масштабирования;
* возможность подключения мобильных клиентов;
* возможность подключения нескольких домов.

---

## Используемые технологии

Backend:

```text
Python 3.12+

FastAPI

SQLAlchemy 2.x Async

PostgreSQL

Alembic

Pydantic v2

JWT

Celery

Redis

MinIO

OCR Engine
```

---

# 2. Архитектура проекта

Рекомендуемая структура проекта:

```text
app/

│

├── api/

│     ├── auth.py
│     ├── buildings.py
│     ├── entrances.py
│     ├── apartments.py
│     ├── residents.py
│     ├── charges.py
│     ├── payments.py
│     ├── expenses.py
│     ├── employees.py
│     ├── reports.py
│     ├── receipts.py
│     └── admin.py

│

├── models/

├── schemas/

├── services/

├── repositories/

├── workers/

├── core/

├── utils/

├── storage/

├── database/

└── main.py
```

---

# 3. Принципы архитектуры

Использовать многослойную архитектуру:

```text
API

↓

Service Layer

↓

Repository Layer

↓

Database
```

Каждый слой отвечает только за свою область.

---

## API

Отвечает за:

* получение запросов;
* проверку авторизации;
* валидацию данных;
* возврат ответа.

API не должно содержать бизнес-логику.

---

## Service Layer

Главное место реализации бизнес-логики.

Именно здесь:

* создаются начисления;
* принимаются платежи;
* рассчитываются долги;
* распределяются оплаты;
* рассчитывается баланс.

---

## Repository Layer

Отвечает только за работу с базой данных.

Repository не должен выполнять финансовые расчёты.

---

# 4. Основные сервисы

Создать следующие сервисы:

```text
AuthService

BuildingService

EntranceService

ApartmentService

ResidentService

ChargeService

PaymentService

PaymentAllocationService

TransactionService

ExpenseService

EmployeeService

SalaryService

ReceiptService

OCRService

ReportService

DashboardService

AuditService
```

---

# 5. AuthService

Функции:

```text
Регистрация

Авторизация

JWT

Refresh Token

Получение текущего пользователя

Смена пароля
```

---

# 6. ApartmentService

Отвечает за:

* создание квартиры;
* изменение квартиры;
* изменение тарифа;
* получение карточки квартиры;
* список жильцов.

---

# 7. ChargeService

Это главный сервис начислений.

Основные функции:

```text
Создать начисление

Получить начисления квартиры

Получить начисления дома

Пересчитать статус начисления

Проверить задолженность
```

---

Основной метод:

```python
create_monthly_charges()
```

Алгоритм:

```text
Берём все активные квартиры

↓

Создаём Charge

↓

Сумма = текущий тариф

↓

Статус = unpaid
```

---

# 8. PaymentService

Главный сервис приёма денег.

Функции:

```text
Создать оплату

Проверить сумму

Создать чек

Передать оплату AllocationService

Создать Transaction
```

---

Алгоритм:

```text
Получена оплата

↓

Создать Payment

↓

Передать в AllocationService

↓

Создать Income Transaction

↓

Обновить Charges

↓

Вернуть чек
```

---

# 9. PaymentAllocationService

Самый важный сервис системы.

Именно он определяет:

> За какие месяцы была произведена оплата.

---

Правило №1

Всегда закрываются самые старые начисления.

Например:

```text
Январь

Февраль

Март

Апрель
```

При оплате:

```text
10000 ₸
```

Результат:

```text
Январь

5400

↓

Закрыт


Остаток:

4600


Февраль

5400

↓

Оплачено:

4600

↓

Частично
```

---

Правило №2

Если долгов нет —

начинают закрываться будущие начисления.

Например:

Оплата:

```text
64800 ₸
```

Результат:

```text
Август

↓

Июль следующего года
```

---

Метод:

```python
allocate_payment(payment_id)
```

Алгоритм:

```text
Получить сумму оплаты

↓

Получить все незакрытые Charges

↓

Отсортировать по периоду

↓

Распределять деньги

↓

Создать Allocation

↓

Обновить Charge

↓

Если деньги остались

↓

Создать авансовое покрытие будущих месяцев
```

---

# 10. TransactionService

Главная задача:

Ведение кассы.

Функции:

```text
Добавить приход

Добавить расход

Получить баланс

История операций

Экспорт
```

---

Правило:

Любой платёж автоматически создаёт:

```text
Income Transaction
```

---

Любой расход автоматически создаёт:

```text
Expense Transaction
```

---

Баланс никогда не хранится.

Всегда рассчитывается.

---

# 11. ExpenseService

Функции:

```text
Создать расход

Изменить расход

Прикрепить документ

Получить расходы

Получить расходы по категории
```

---

После создания расхода:

```text
Expense

↓

Transaction

↓

Баланс уменьшается
```

---

# 12. EmployeeService

Функции:

```text
Создать сотрудника

Изменить сотрудника

Уволить

Получить карточку

История зарплат
```

---

# 13. SalaryService

Функции:

```text
Выплатить зарплату

История выплат

Получить задолженность по зарплате
```

---

После выплаты:

Создаются автоматически:

```text
SalaryPayment

↓

Expense

↓

Transaction
```

---

# 14. DashboardService

Главный сервис администратора.

Возвращает:

```json
{
  "apartments":48,
  "monthly_expected":259200,
  "charged":259200,
  "received":216000,
  "debt":43200,
  "expenses":150000,
  "balance":66000
}
```

---

# 15. ReportService

Формирует отчёты.

Отчёты:

```text
По квартире

По дому

Доходы

Расходы

Долги

Переплаты

Сотрудники

Зарплаты
```

---

# 16. ReceiptService

Функции:

```text
Загрузка PDF

Удаление

Получение

Связь с Payment

Хранение MinIO
```

---

# 17. OCRService

Функции:

```text
OCR

Парсинг PDF

Извлечение суммы

Извлечение даты

Извлечение квартиры

Извлечение банка

Связь с Receipt
```

---

# 18. AuditService

Записывает любые действия пользователей.

Например:

```text
Создана квартира

↓

Добавлен платёж

↓

Удалён сотрудник

↓

Изменён тариф
```

---

# 19. REST API

## Authentication

```http
POST /auth/register

POST /auth/login

POST /auth/refresh

GET /auth/me

POST /auth/logout

POST /auth/change-password
```

---

## Buildings

```http
GET /buildings

POST /buildings

GET /buildings/{id}

PATCH /buildings/{id}

DELETE /buildings/{id}
```

---

## Entrances

```http
GET /entrances

POST /entrances

PATCH /entrances/{id}

DELETE /entrances/{id}
```

---

## Apartments

```http
GET /apartments

POST /apartments

GET /apartments/{id}

PATCH /apartments/{id}

DELETE /apartments/{id}
```

Дополнительно:

```http
GET /apartments/{id}/card

GET /apartments/{id}/charges

GET /apartments/{id}/payments

GET /apartments/{id}/balance
```

---

## Residents

```http
GET /resident/profile

GET /resident/apartment

GET /resident/dashboard

GET /resident/payments

GET /resident/charges

GET /resident/receipts
```

---

## Charges

```http
GET /charges

POST /charges

GET /charges/{id}

PATCH /charges/{id}
```

Служебный endpoint:

```http
POST /charges/generate
```

Доступен только администраторам и запускается автоматически по расписанию.

---

## Payments

```http
GET /payments

POST /payments

GET /payments/{id}

GET /payments/{id}/receipt
```

---

## Transactions

```http
GET /transactions

GET /transactions/balance

GET /transactions/income

GET /transactions/expenses
```

---

## Employees

```http
GET /employees

POST /employees

PATCH /employees/{id}

DELETE /employees/{id}

GET /employees/{id}/salary
```

---

## Salary

```http
POST /salary/pay

GET /salary/history
```

---

## Expenses

```http
GET /expenses

POST /expenses

PATCH /expenses/{id}

DELETE /expenses/{id}
```

---

## Reports

```http
GET /reports/dashboard

GET /reports/debtors

GET /reports/prepayments

GET /reports/income

GET /reports/expenses

GET /reports/financial

GET /reports/apartment/{id}
```

---

## Receipts

```http
POST /receipts/upload

GET /receipts/{id}

DELETE /receipts/{id}
```

---

# 20. Celery задачи

Система должна выполнять ряд операций автоматически.

## Ежемесячное начисление

Каждое первое число месяца:

```text
CreateMonthlyChargesTask
```

Алгоритм:

```text
Получить все активные квартиры

↓

Создать Charge

↓

Записать Audit
```

---

## OCR обработка

После загрузки PDF:

```text
Receipt Uploaded

↓

OCR Worker

↓

Extract Data

↓

Apartment Matcher

↓

Payment Binding
```

---

## Ежедневная проверка долгов

Каждую ночь:

```text
DebtCheckTask
```

Проверяет:

* новые должники;
* просроченные начисления;
* квартиры с переплатой.

---

## Пересчёт финансового дашборда

После каждого:

* платежа;
* расхода;
* выплаты зарплаты.

Запускается:

```text
DashboardUpdateTask
```

---

## Генерация отчётов

По расписанию:

```text
MonthlyReportsTask
```

Создаются:

* финансовый отчёт;
* отчёт по долгам;
* отчёт по переплатам;
* отчёт по доходам и расходам.

---

# 21. Алгоритмы расчёта

## Алгоритм расчёта долга квартиры

```text
Долг квартиры =
Сумма всех начислений -
Сумма распределённых оплат
```

Важно:

Используются **распределённые суммы (`payment_allocations`)**, а не общая сумма платежей, чтобы корректно учитывать частичные оплаты и авансы.

---

## Алгоритм расчёта переплаты

```text
Переплата =
Остаток нераспределённых средств
или
Сумма оплат,
распределённая на будущие начисления
```

Переплата не является отдельным полем в таблице `apartments`.

---

## Алгоритм расчёта общей задолженности дома

```text
Сумма долгов всех квартир
```

---

## Алгоритм расчёта ежемесячного начисления

```text
Количество активных квартир

×

Тариф,
действующий на дату начисления
```

Если тариф изменился, новые начисления создаются по новому тарифу, а существующие не изменяются.

---

## Алгоритм расчёта баланса дома

```text
Баланс =
Все доходные транзакции -
Все расходные транзакции
```

Баланс не хранится отдельным полем и вычисляется агрегатным запросом.

---

## Алгоритм определения статуса начисления

Для каждого `charge`:

* `unpaid` — оплачено 0 ₸;
* `partial` — оплачено больше 0 ₸, но меньше суммы начисления;
* `paid` — оплачено полностью;
* `overpaid` не используется как статус начисления. Переплата отражается через распределение платежей на будущие начисления.

---

# 22. Требования к производительности

Система должна поддерживать:

* не менее 10 000 квартир;
* не менее 1 000 000 платежей;
* не менее 10 лет истории операций.

Обязательные индексы:

* `charges(apartment_id, period_year, period_month)`
* `payments(apartment_id, payment_date)`
* `payment_allocations(payment_id)`
* `payment_allocations(charge_id)`
* `transactions(created_at)`
* `expenses(date)`
* `audit_logs(created_at)`
* `receipts(payment_id)`

Все операции по приёму платежа должны выполняться в одной транзакции базы данных с возможностью отката (`rollback`) при любой ошибке.

---

# 23. Критерии готовности Backend

Backend считается реализованным, если:

### Жилец может:

* зарегистрироваться и авторизоваться;
* увидеть свою квартиру;
* увидеть начисления;
* увидеть историю оплат;
* скачать чек;
* узнать текущий долг или переплату.

### Администратор может:

* управлять домами, подъездами и квартирами;
* принимать платежи;
* автоматически распределять их по начислениям;
* вести кассу;
* учитывать расходы и зарплаты;
* получать финансовые отчёты;
* видеть баланс дома в любой момент.

### Система автоматически:

* создаёт ежемесячные начисления;
* обрабатывает OCR-квитанции;
* распределяет платежи;
* ведёт журнал действий;
* поддерживает неизменяемую историю финансовых операций.

---

**Конец Части 3.**

Следующая часть документа:

**Часть 4 — Frontend архитектура, пользовательские интерфейсы (UI/UX), личный кабинет жильца, административная панель, навигация, сценарии работы и требования к пользовательскому опыту.**
