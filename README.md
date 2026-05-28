        # temporal — Ретраи и таймауты

        Homework-шаблон для урока **l1_retries_and_timeouts** (Ретраи и таймауты) на платформе Vibe Learn.

        ## Что делать

        Реализуй PaymentWorkflow на Go SDK (go.temporal.io/sdk) с активностью ChargeCard,
которая флакает: с заданной вероятностью возвращает транзиентную ошибку (503),
иногда — бизнес-ошибку CardDeclinedError. Настрой ActivityOptions: StartToCloseTimeout,
RetryPolicy с BackoffCoefficient 2.0, MaximumAttempts и NonRetryableErrorTypes так,
чтобы транзиентные ошибки ретраились с backoff, а CardDeclinedError проваливал workflow
сразу. Тесты через TestWorkflowEnvironment проверят: транзиентный сбой не роняет
workflow (дожимается ретраями), а CardDeclinedError завершает его без лишних попыток.

## Контекст (из transfer-задачи урока)

У тебя workflow обработки платежа. Активность `ChargeCard` дёргает внешний платёжный
шлюз. Бывает три типа сбоев: (1) шлюз временно недоступен (503), (2) таймаут сети,
(3) карта отклонена банком ("card_declined"). Сейчас разработчик задал
`MaximumAttempts: 5` без таймаутов и жалуется: при отклонённой карте workflow всё равно
делает 5 попыток и теряет 20 секунд, а при зависшем шлюзе одна попытка висит «вечно».

**Вопрос:** как настроить RetryPolicy и таймауты правильно? Опиши:
(a) какой таймаут ограничит одну зависшую попытку;
(b) как сделать, чтобы "card_declined" НЕ ретраился, а 503/таймаут — ретраились;
(c) почему ручной retry-цикл внутри workflow здесь хуже декларативной политики.

## Recap из урока

- **Ретраи бесплатны для программиста**: ты конфигурируешь RetryPolicy, движок ретраит сам — durable, переживая падения воркеров. Никаких ручных retry-циклов в workflow.
- RetryPolicy: InitialInterval, BackoffCoefficient (**дефолт 2.0**, экспоненциальный backoff), MaximumInterval (потолок), MaximumAttempts (**0 = бесконечно**), NonRetryableErrorTypes.
- Четыре таймаута активности: ScheduleToStart (ожидание в очереди), **StartToClose** (одна попытка — задают почти всегда), ScheduleToClose (весь срок с ретраями), Heartbeat (детект зависшего воркера).
- Три таймаута workflow: WorkflowExecutionTimeout (вся жизнь, включая ContinueAsNew), WorkflowRunTimeout (один run), WorkflowTaskTimeout (одна порция решений, дефолт 10s).
- Таймаут и количество попыток — **ортогональны**. Для бизнес-дедлайнов используй таймер, а не WorkflowExecutionTimeout (тот жёстко терминирует без компенсации).

        ## Как работать

        1. Платформа Vibe Learn создаёт копию этого репо в твоём GitHub-аккаунте по клику «Начать домашку» на странице урока (через GitHub `/generate`, codecrafters-pattern).
        2. Склонируй копию локально, реализуй TODO в `main.go` (workflow + активности), прогони тесты, запушь.
        3. CI (`.github/workflows/ci.yml`) запускает `go vet` + `go test ./...` на каждый push. Платформа слушает результат через webhook от GitHub Actions и обновляет статус домашки на странице урока.

        ## Локальное окружение

        - Go 1.22+
        - SDK: `go.temporal.io/sdk`
        - Docker + docker-compose — `docker compose up` поднимает Temporal dev server на `:7233` + Web UI на `:8233`. Адрес переопределяется через env `TEMPORAL_ADDRESS` (дефолт `localhost:7233`).
        - Юнит-тесты на `testsuite.TestWorkflowEnvironment` (активности замоканы) бегут в CI БЕЗ сервера; интеграционный тест включается через `TEMPORAL_INTEGRATION=1`.

        ## Запуск

        ```bash
        # Поднять локальный Temporal dev server + UI
        docker compose up -d
        # Web UI: http://localhost:8233

        # Прогнать тесты (юнит на TestWorkflowEnvironment — без сервера;
        # интеграционный включается через TEMPORAL_INTEGRATION=1)
        go test ./...
        TEMPORAL_INTEGRATION=1 go test ./...

        # Запустить воркер (регистрирует workflow + активности, слушает task queue)
        go run .
        ```

        ## Заметка автора

        Это baseline-шаблон, сгенерированный платформой. Бизнес-сущность задачи (что конкретно реализовать в `main.go`, какие тесты сделать строгими) расширяется по ходу итераций — параллельно с углублением теории урока.
