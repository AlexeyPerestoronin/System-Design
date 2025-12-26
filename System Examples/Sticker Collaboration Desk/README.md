# Sticker Collaboration Desk

Система для совместной работы над стикерами на доске.

## Функциональные требования
* Дать возможность любому пользователю создать новую доску с уникальными ссылкой
* Созданную доску можно править любому пользователю без ограничений
* Пользователи должны иметь возможность создавать/изменять/удалять стикеры на доске

## НеФункциональные требования
* Время жизни одной доски 1 неделя
* Пользователи должны видеть актуальное содержимое доски в режиме реального времени
* Оценка объёма требуемой памяти для хранения всех досок:
    * $7GB \simeq DeskSize * DeskCount * 7_{ДнейВНеделе}$, где:
      * $DeskSize \simeq 10KB$ - средний размер памяти необходимый для сохранения доски
      * $DeskCount \simeq 1.000.000$ - среднее количество новых досок, созданный в течении одного дня
* Оценка сверху пропускной способности системы в секунду:
    * $50MB \simeq Users * Diff$, где:
        * $Users \simeq 500.000$ - максимальное ожидаемое количество активных пользователей в секунду
        * $Diff \simeq DeskSize * 0.01 \simeq 0.1KB$ - diff-информации для доски, генерируемый пользователем в секунду
* Дополнительно:
    * Должна быть возможность работать offline, а также в ситуации с плохим соединением.


## Happy Path:
1. Пользователь создаёт новую доску, или подключается к уже созданной:
    ```txt
    # Клиент-1 запрашивает доску
    [User's App]
        POST /api/desk/init HTTP/1.1
        Content-Type: application/json
        { "name": STRING, "create_new": BOOL }
    [Entry Point]  
    | 1. [Entry Point] --(запрос создания новой доски/запрос поиска доски)--> [Desk Management Service]
    | 2. [Desk Management Service] --(запусти сессию коллаборации для созданной доски)--> [Collaboration Service]
    | 3. [Collaboration Service] --(запустить сессию коллаборации для созданной доски и отправить пользователю URL для подключения)--> [Entry Point]
    # Сервис отвечает, что доска найдена/создана и возвращает её данные и токен сессии коллаборации
    [Entry Point]
        HTTP/1.1 200 OK
        Content-Type: application/json
        { "desk": Desk Binary Data as STRING, "session_key": XXX-XXX-XXX }
    [User's App]
    # Клиент-1 запрашивает переход на WebSocket пля работы в режиме коллаборации
    [User's App]
        GET /api/desk/collaboration/init HTTP/1.1
        Upgrade: websocket
        WebSocket-Session-Key: XXX-XXX-XXX
    [Entry Point]
    # Сервис подтверждает переход на обмен данными через WebSocket в рамках созданной сессии коллаборации
    [Entry Point]
        HTTP/1.1 101 Switching Protocols
        Upgrade: websocket
        WebSocket-Session-Accept: XXX-XXX-XXX
    [User's App]
    ```
2. Работает с доской внося изменения (возможно в режиме коллаборации с другими пользователями)
    ```txt
    # Клиент-1 отправляет на Сервер diff рабочей доски
    [User's App]
        PATCH /api/desk/collaboration/sync HTTP/1.1
        Upgrade: websocket
        WebSocket-Session-Key: XXX-XXX-XXX
        R-User-ID: client-1-ID
    [Collaboration Service]
    # Сервер уведомляет Клиента, что diff получен
    [Entry Point]
        HTTP/1.1 221 Desk Updated
        Upgrade: websocket
        Content-Type: application/json
        { "collision_status": Status as STRING }
    [User's App]
    # Сервер отправляет изменения остальным Клиентам, участвующим в сессии коллаборации, данные об изменениях внесённых Клиентом-1
    [Entry Point]
        PATCH /api/desk/collaboration/sync HTTP/1.1
        Upgrade: websocket
        WebSocket-Session-Key: XXX-XXX-XXX
        R-User-ID: client-1-ID
    [Users' App]
    ```
3. Завершил работу над  доской  
    Сессия коллаборации завершается либо автоматически по прошествии определённого времени без внесения изменений со стороны какого-либо из пользователей, либо, если пользователь создатель доски отправил HTTP запрос на закрытие сессии коллаборации:
    ```txt
    [User's App]
        PATCH /api/desk/collaboration/finish HTTP/1.1
        WebSocket-Session-Key: XXX-XXX-XXX
        Content-Type: application/json
        { "diff": STRING }
    [Collaboration Service]
    | 1. [Collaboration Service] --(завершение сессии коллаборации + передача доски над которой завершена работа под дальнейшее управление)-->[Desk Management Service]
    | 2. [Desk Management Service] --(запрос на разрешение конфликтов, если если таковые есть)--> [Desk Collisions Service]
    | 3. [Desk Management Service] --(отправка доски на долгосрочное хранение)--> [Desk DB]
    [Collaboration Service]
        HTTP/1.1 200 OK
        WebSocket-Session-Key: XXX-XXX-XXX
        Content-Type: application/json
        { "status": STRING }
    [User's App]
    ```

---

![](./stiсker%20collaboration%20desk.drawio.png)

---

# System Services
## Entry Point
Сервис внешнего доступа к Системе:
* **Load Balancer**:
    * управляет количеством запущенных экземпляров Collaboration Services в зависимости от количества активных сессий
* **API Gateway**

## Message Queue
Глобальная  очередь сообщений системы с возможностью отслеживания статуса сообщений с нотификацией отправителя.

## Business Logic Services:
### Collaboration Service (CS)
* управляет сессией (возможно несколькими) коллаборации:
    * инициации сессии
    * отслеживание и синхронизация изменений пользователей в процессе работы над доской
    * завершение сессии в случае разрыва всех WebSocket соединений с пользователями (либо по timeout в случае потери связи)

### Desk Collisions Service (DCS)
* разрешает коллизии, возникающие в процессе работы над доской или после

### Desk Management Service (DMS)
* управляет досками

## Data Base Services:
### Desk DB
* хранит доски
