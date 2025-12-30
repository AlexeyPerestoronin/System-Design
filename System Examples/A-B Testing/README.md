# A/B Testing

Система проведения A/B тестов для Web- и мобильных приложений

## Функциональные требования
* Продуктовый аналитик (ПА) может завести А/В-тест (Тест) в нашей  Системе (с определённым параметрами)
* ПА может посмотреть статистику по заведённым тестам (получить результаты эксперимента)
* Клиентские приложения (КП) могут получить список экспериментов в которые они попали
* КП могут отгружать аналитические данные на A/B-платформу
* В Систему можно навешивать сегменты (аудиторию) на пользователей и использовать их для третирования эксперимента
* Перспективные:
    * Эксперименты должны применяться на лету

## НеФункциональные требования
* Система должна отвечать КП в предалх 200мс 99.9% времени
* Пользователи должны уметь принадлежать к экспериментам
* Система должна уметь обеспечивать условия для проведения статического эксперимента (условно равномерное распределение между тестовыми группами)
* До 1000 сегментов пользователей
* 30 млн MUA
* 10 млн DAU Mobile
* 3 млн DAU Web
* До 1000 активных экспериментов
* 100-1000 продуктовых аналитиков
* Каждый эксперимент живёт 1-4 недели


![](./a-b%20testing.drawio.png)

## Happy Path:
### Заведение нового A/B-теста продуктовым аналитиком
1. Пользователь аутентифицируется в Системе как продуктовый аналитик
2. ПА создаёт новый A/B-test и отправляет его в Систему
```txt
# ПА запрашивает пустую web-форму для A/B-теста для её заполнения
[PA's Web-App]
    GET /ab-tests/create_new/ask HTTP/1.1
    Authorization: Bearer <токен>
[EntryPoint]
| 1. [EntryPoint] --(запрос web-страницы для создания нового теста)--> [A/B-Tests Management Service]
| 2. [A/B-Tests Management Service] --(передача web-страницы)--> [EntryPoint]
[EntryPoint]
    HTTP/1.1 221 GetBlank
    заголовки ...
    пустая web-форма (вероятно, это будет WebAssembly код) ...
[PA's Web-App]

# ПА создаёт новый A/B-тест в полученной web-странице и отправляет её в Систему для сохранения
[PA's Web-App]
    POST /ab-tests/create_new/save HTTP/1.1
    Authorization: Bearer <токен>
    данные созданного a/b-теста...
[EntryPoint]
| 1. [EntryPoint] --(передача результатов созданного a/b-теста)--> [A/B-Tests Management Service]
| 2.1. [A/B-Tests Management Service] --(сохранение нового a/b-теста в БД)--> [A/B-Tests DB Service]
| 2.1. [A/B-Tests Management Service] --(запрос создания графа сопоставления пользователей)--> [A/B-Tests Users Match Service] --(результат) --> [A/B-Tests Management Service] + [A/B-Tests DB Service]
| 2.2. [A/B-Tests Management Service] --(отправка сообщения об успешном создании a/b-теста)-->[EntryPoint]
[EntryPoint]
    HTTP/1.1 222 Saved
[PA's Web-App]
```

### Применение A/B-теста на пользователя
1. Пользователь входит в Систему
```txt
# пользователь аутентифицируется в Системе
[Users's App]
    GET /ab-tests/user/log HTTP/1.1
    Content-Type: application/json
    { "name": String, "pass": String }
[EntryPoint]
| 1. [EntryPoint] --(запрос аутентификации пользователя)--> [A/B-Tests Management Service]
| 2.1. [A/B-Tests Management Service] --(запрос выдачи JWT токена)--> [A/B-Tests Users Match Service] --(результат)--> [A/B-Tests Management Service]
| 2.1.1.(?) [A/B-Tests Management Service] --(запрос идентификации пользователя с A/B-тестом, если нет в Users Match Local Cache)--> [A/B-Tests DB Service] --(результат)--> [A/B-Tests Management Service]
| 2.1.2.(?) [A/B-Tests Management Service] --(запрос целевого A/B-теста для пользователя, если нет в A/B-Tests Local Cache)--> [A/B-Tests DB Service] --(результат)--> [A/B-Tests Management Service]
| 2.2. [A/B-Tests Management Service] --(отправка пользователю JWT-токена и данных для A/B-тестирования)--> [EntryPoint]
[EntryPoint]
    HTTP/1.1 230 Success
    Content-Type: application/json
    { "JWT": String, "A/B-Test": { ... } }
[Users's App]
```
2. Пользователь пользуется системой:
```txt
# отправка в систему данных об использовании A/B-теста
[Users's App]
    PATCH /ab-tests/user/send_statistic HTTP/1.1
    Authorization: Bearer <токен>
    Content-Type: application/json
    { "A/B-Test": { ..., "Statistics": { ... } } }
[EntryPoint]
| 1. [EntryPoint] --(сохранение данных об использовании A/B-теста)--> [A/B-Tests DB Service] --(результат)--> [EntryPoint]
[EntryPoint]
    HTTP/1.1 233 Success
[Users's App]
```

### Просмотр статистики по A/B-test продуктовым аналитиком
1. Пользователь аутентифицируется в Системе как продуктовый аналитик
2. ПА отправляет запрос в Систему на получение статистических данных использование A/B-теста
```txt
# ПА запрашивает статистику использования A/B-теста
[PA's Web-App]
    GET /ab-tests/statistics/ask HTTP/1.1
    Authorization: Bearer <токен>
    Content-Type: application/json
    { "A/B-Test name": String }
[EntryPoint]
| 1. [EntryPoint] --(запрос web-страницы для создания нового теста)--> [A/B-Tests Analytics Service]
| 2.1. [A/B-Tests Analytics Service] --(запрос данных об A/B-тесте)--> [A/B-Tests DB Service] --(результат)--> [A/B-Tests Analytics Service]
| 2.1. [A/B-Tests Analytics Service] --(запрос данных об использовании)--> [A/B-Tests DB Service] --(результат)--> [A/B-Tests Analytics Service]
| 2.1. [A/B-Tests Analytics Service] --(запрос данных о пользователях)--> [A/B-Tests DB Service] --(результат)--> [A/B-Tests Analytics Service]
| 2.1.1.(?) [A/B-Tests Analytics Service] --(запрос пользователей, если необходимо)--> [A/B-Tests Users Match Service] --(результат)--> [A/B-Tests Analytics Service]
| 2.2. [A/B-Tests Management Service] --(передачи сформированной статистики об использовании)--> [EntryPoint]
[EntryPoint]
    HTTP/1.1 245 Statistic
    Authorization: Bearer <токен>
    Content-Type: application/json
    { "Statistic" { "A/B-Test name": String, ... } }
[PA's Web-App]
```

# System Services
## EntryPoint
* API Gateway
* LoadBalancing

## A/B-Tests Management Service
* управляет созданием новых тестов
* направляет пользователям данные об принадлежности к A/B-тестированию

## A/B-Tests User-Match Service
* на основании данных конкретного A/B-теста, создаёт граф целевых пользователей для тестирования

## A/B-Tests Analytics Service
* по запросу, формирует статистику использования для A/B-теста

## A/B-Tests User-Authentication Service
* регистрирует пользователей в системе
* выдаёт JWT токены доступа

## A/B-Tests DB Service
* LoadBalancing access to DB(S)
* направляет запрос на целевой сервер БД, если имеется шардирование
* удаляет тесты давность больше недели (помечает, как неактивные)

## System Users DB Service

# System Data-Bases
## A/B-Tests DB
* NoSQL → DocumentOriented → MongoDB

## A/B-Tests User-Match DB
* NoSQL → GraphLike → Neo4J

## A/B-Tests Statistics DB
* NoSQL → WideColumn → ClickHouse
* Sharding by Time-column

## System Users DB
* SQL
* Sharding by UserID
