# ДЗ 6. Проектная работа — решение

> Условие: [../Задание.md](../Задание.md). Схемы — в [diagrams/](diagrams/).

## Выбранная тема

_Система управления мероприятиями — билеты на мероприятия (Tiketing System)._

Система управления мероприятиями и продажи билетов — это система, которая позволяет организаторам создавать мероприятия(события), настраивать интерактивные схемы залов, управлять билетами на мероприятие. Покупатели могут быстро находить концерты, выбирать места на схеме и оплачивать билеты онлайн. Система обеспечивает  проверку билетов контролерами с помощью сканеров на входе на площадку.


## 1. Требования

### Функциональные

Основные роли в системе:
- Покупатель;
- Организатор;
- Контролер;


#### Для покупателя 

Поиск и фильтрация: Возможность искать мероприятия по жанру, городу, дате, цене и исполнителю.

Интерактивная схема зала: Выбор конкретного места на динамической карте площадки (партер, амфитеатр, ложи) с отображением статуса занятости в реальном времени.

Покупка и оплата: Корзина, интеграция со СБП, картами, частями (сплит) и экосистемными баллами (например, Яндекс Плюс). 

Личный кабинет и Билеты: Хранение купленных билетов в виде QR-кодов. 


#### Для организатора мероприятий

Создание конфигураций площадок (схема зала) .
Формирование мероприятий.
ФОрмирование билетов на мероприятия.


#### Для администрации и контроля на площадке 

Мобильное приложение или СКУД-терминалы для валидации QR-кодов билетов на входе.

### Нефункциональные

#### Производительность и масштабируемость 

Требования к обрабатываемому системой количеству запросов предъявляемые к различным компонентам системы.

| Компонент | Стандарт RPS | Пиковые показатели RPS |
| --------------- | --------------- | --------------- |
| Показ мероприятий (списки событий, поиск, фильтры) | 800 – 1 200 | 8 000 – 12 000 |
| Показ схем залов (интерактивная карта, загрузка сетки мест и их статусов) | 300 - 500 | 5 000 - 7 000 |
| Бронирование мест (выбор места и резервирование его на 10 мин) | 30 – 50 | 1 500 – 2 500 |
| Создание заказа и оплата (списки событий, поиск, фильтры) | 5 – 10 | 200 – 400 |


#### Время отклика (Latency)

Поиск событий не более 500 мс.

Отображение схемы зала — не более 1.5 секунд.

Проверка доступности места и блокировка под сессию покупки — не более 200 мс.

Время валидации билета сканером на входе — не более 0.3–0.5 секунд для предотвращения очередей.

#### Консистентность данных 

Строгая транзакционность. 

При выборе места пользователем оно блокируется на 10 минут для оплаты. 

Ни один другой пользователь не должен иметь возможность заблокировать или купить это же место параллельно.

#### Доступность 

Коэффициент доступности системы (SLA) — 99.9%. 

Архитектура должна быть отказоустойчивой.


### Риски и ограничения

1. Ажиотажный спрос. (пик трафика в момент публикации события).

Риск падения системы при старте продаж билетов на сверхпопулярных артистов (например, финалы крупных мероприятий на подобии чемпионата мира по футболу или туры топ-исполнителей). 95% трафика за сутки может прийтись на первые 10 минут.

Последствия: Отказ API, недовольство пользователей, репутационные потери.

2. Овербукинг (Двойные продажи):

Риск продажи одного и того же места двум разным пользователям.

3. Атаки ботов и перекупщиков

Использование скриптов для моментального выкупа лучших мест в первые секунды продаж с целью перепродажи на сторонних площадках.

4. Сбои платежных шлюзов 

Отказ со стороны банков-эквайеров или СБП в моменты пиковых нагрузок.

5. Законодательство о возврате билетов

Четко регламентирует процент возврата средств в зависимости от того, за сколько дней до мероприятия оформлен возврат (например, за 10 дней — 100%, за 5 дней — 50%, менее чем за 3 дня — 0%, за исключением болезни). Логика возвратов в коде должна строго соответствовать этим правилам.

6. Фискализация

Обязательное формирование онлайн-чеков в момент оплаты и отправка их в ОФД. Система должна быть интегрирована с облачными кассами (АТОЛ Онлайн и др.), способными пробивать тысячи чеков в минуту.

7. Интеграция со сторонними сервисами.

При использовании сторонних интеграционных сервисов наша система в какой-то мере зависит от скорости работы стороннего API.

### Backlog 

Управление ценовыми категориями (тарифами) и квотами билетов. 
Настройка динамического ценообразования. 
Отчетность по продажам и аналитика.
Оформление возврата.
Выполнение требований законов о защите персональных данных.
Фискализация.
Защита от перекупщиков и ботов.
Интеграция со сторонними сервисами площадок.
Возможность проверять QR-коды в отсутствии интернета.


## 2. Концептуальная архитектура

```mermaid
C4Context
    title Контекст системы бронирования билетов на мероприятия
        Enterprise_Boundary(b0, "") {
        Person(personUser, "Покупатель", "Покупатель билетов.<br/>Пользователь, ищущий и покупающий билеты.")
        Person(personOrganizer, "Организатор", "Организатор мероприятий.<br/>Добавляет мероприятия и управляет залами.")
        Person(personController, "Контролер", "Контролер билетов.<br/>Проверяет и валидирует билеты на входе.")
        Person(personAdministrator, "Администратор", "Администратор системы<br/>Сотрудник Tiketing System, настраивает лимиты и решает споры.")

        System(systemTicketingSystem, "Tiketing System", "Платформа бронирования билетов.<br/>'Позволяет искать мероприятия, безопасно покупать билеты без риска двойных продаж и валидировать их на входе.")
        }

        Boundary(b1, "") {
        System_Ext(extSystemPayment, "Payment Provider", "Инициирует списание средств и возвраты клиентам.")
        System_Ext(extSystemIdentity, "Identity Provider", "Внешний сервис авторизации. Yandex, Google, Facebook, etc...")
        System_Ext(extSystemNotification, "Notification Provider", "Передает Email / SMS / Push сообщения с электронными билетами для доставки.")
        }      

        Rel(personUser, systemTicketingSystem, "Ищет концерты,<br/>резервирует места,<br/> оплачивает заказы.")
        Rel(personOrganizer, systemTicketingSystem, "Заводит мероприятия,<br/> настраивает схемы залов.")
        Rel(personController, systemTicketingSystem, "Сканирует QR-коды<br/> для проверки<br/> статуса билета.")
        Rel(personAdministrator, systemTicketingSystem, "Модерирует события,<br/>меняет лимиты,<br/> разрешает спорные ситуации.")

        Rel(systemTicketingSystem, extSystemPayment, "Списание и возвраты<br/>средств клиента.")
        Rel(systemTicketingSystem, extSystemIdentity, "Внешний сервис авторизации.")
        Rel(systemTicketingSystem, extSystemNotification, "Передает Email / SMS / Push<br/>сообщения с электронными билетами для доставки.")

        UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="1")
       
```

```mermaid
C4Container
    title Container diagram for Tiketing System

        Person(personUser, "Покупатель", "Покупатель билетов.<br/>Пользователь, ищущий и покупающий билеты.")
        Person(personOrganizer, "Организатор", "Организатор мероприятий.<br/>Добавляет мероприятия и управляет залами.")
        Person(personController, "Контролер", "Контролер билетов.<br/>Проверяет и валидирует билеты на входе.")
        Person(personAdministrator, "Администратор", "Администратор системы<br/>Сотрудник Tiketing System, настраивает лимиты и решает споры.")

    Container_Boundary(c1, "Tiketing System") {
        Container(webApp, "Веб-сайт", "JavaScript", "Веб-сайт для бронирования билетов покупателями.")
        Container(mobileApp, "Мобильное приложение", "Android/iOS", "Мобильное приложение для бронирования билетов покупателями.")
        Container(controllerApp, "Мобильное приложение", "Android/iOS", "Мобильное приложение для проверки валидноти билетов.")
        Container(webAdminPortal, "Веб-сайт", "", "Веб-портал для администрирования.")
    }
```


```mermaid
C4Container
    title Container diagram for Internet Banking System

    System_Ext(email_system, "E-Mail System", "The internal Microsoft Exchange system", $tags="v1.0")
    Person(customer, Customer, "A customer of the bank, with personal bank accounts", $tags="v1.0")

    Container_Boundary(c1, "Internet Banking") {
        Container(spa, "Single-Page App", "JavaScript, Angular", "Provides all the Internet banking functionality to customers via their web browser")
        Container_Ext(mobile_app, "Mobile App", "C#, Xamarin", "Provides a limited subset of the Internet banking functionality to customers via their mobile device")
        Container(web_app, "Web Application", "Java, Spring MVC", "Delivers the static content and the Internet banking SPA")
        ContainerDb(database, "Database", "SQL Database", "Stores user registration information, hashed auth credentials, access logs, etc.")
        ContainerDb_Ext(backend_api, "API Application", "Java, Docker Container", "Provides Internet banking functionality via API")

    }

    System_Ext(banking_system, "Mainframe Banking System", "Stores all of the core banking information about customers, accounts, transactions, etc.")

    Rel(customer, web_app, "Uses", "HTTPS")
    UpdateRelStyle(customer, web_app, $offsetY="60", $offsetX="90")
    Rel(customer, spa, "Uses", "HTTPS")
    UpdateRelStyle(customer, spa, $offsetY="-40")
    Rel(customer, mobile_app, "Uses")
    UpdateRelStyle(customer, mobile_app, $offsetY="-30")

    Rel(web_app, spa, "Delivers")
    UpdateRelStyle(web_app, spa, $offsetX="130")
    Rel(spa, backend_api, "Uses", "async, JSON/HTTPS")
    Rel(mobile_app, backend_api, "Uses", "async, JSON/HTTPS")
    Rel_Back(database, backend_api, "Reads from and writes to", "sync, JDBC")

    Rel(email_system, customer, "Sends e-mails to")
    UpdateRelStyle(email_system, customer, $offsetX="-45")
    Rel(backend_api, email_system, "Sends e-mails using", "sync, SMTP")
    UpdateRelStyle(backend_api, email_system, $offsetY="-60")
    Rel(backend_api, banking_system, "Uses", "sync/async, XML/HTTPS")
    UpdateRelStyle(backend_api, banking_system, $offsetY="-50", $offsetX="-140")
```

- Схема C4: Context + Container (в diagrams/).
- Описание компонентов (1–2 предложения на каждый).
- Обоснование архитектурного стиля.
- ADR для 2–3 ключевых решений.
- Пользователи и внешние интеграции.

## 3. Сайзинг

_RPS, хранилище, bandwidth, ресурсы и стоимость._

## 4. Хранение данных

_Выбор БД с обоснованием, шардирование, кэширование._

## 5. Взаимодействие

_Схема взаимодействия и API-контракты._
