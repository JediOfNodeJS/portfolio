# CONTENTER

**CONTENTER** — персональная платформа для визуальной настройки и запуска веб-парсеров (сбор структурированных данных с сайтов в базу данных) с десктоп-панелью управления.

Система позволяет:

- описывать **проекты** сбора данных (например, «Фриланс биржи»);
- добавлять **источники** (сайты), **страницы** внутри источников и настраивать **запросы к странице** (CSS-селектор → поле в БД);
- описывать **структуру данных** (базы/таблицы/поля) без написания SQL;
- создавать **парсеры** со сценариями (парсинг списка ссылок из БД, слежение за страницами и т.п.) и запускать их в реальном браузере Chrome под управлением Puppeteer;
- автоматически складывать распарсенные данные в отдельную контентную базу PostgreSQL (с дедупликацией по уникальным полям).

Сейчас это **однопользовательское** приложение в активной разработке — пользователь захардкожен (`yozuul@gmail.com`), авторизации/токенов в коде нет.

---

## Состав репозитория

Монорепозиторий состоит из трёх независимых npm-пакетов.

```
contenter/
├── server/      # API-сервер (REST + WebSocket), NestJS, порт 4000
├── backend/     # «Исполнитель» парсинга: NestJS-монорепо
│                #   ├── apps/api-gateway      — шлюз, порт 3000 (socket.io-клиент к server'у)
│                #   └── apps/browser-connector — TCP-микросервис, порт 3001 (управление браузером)
└── dashboard/   # Десктоп-клиент (Electron + Vue 3 + TS), UI управления
```

Схема связей:

```
┌─────────────┐   HTTP REST (/api)    ┌─────────────┐
│  dashboard  │ ────────────────────► │   server    │  порт 4000
│ (Electron)  │                       │  (NestJS)   │  REST + socket.io-сервер
└─────────────┘                       └──────┬──────┘
                                             │ socket.io (клиент подключается к :4000)
                                             ▼
                                    ┌────────────────────┐   TCP микросервис
                                    │   backend          │  ┌──────────────────┐
                                    │   api-gateway :3000│─►│ browser-connector│ :3001
                                    └────────────────────┘  └──────────────────┘
                                             │ Puppeteer (реальный Chrome)
                                             ▼
                                        парсинг сайтов
```

> Ключевая идея: **dashboard** — интерфейс пользователя, **server** — «мозг» (бизнес-логика, задачи, хранение), **backend/api-gateway** — «руки» (поднимает настоящий браузер и выполняет парсинг), **browser-connector** — отдельный TCP-сервис для управления браузером (сейчас частично задел).

---

## Архитектура и ключевые механизмы

Этот раздел объясняет, **как** система работает изнутри и **почему** она устроена именно так. Справочник по модулям и эндпоинтам находится ниже.

### 1. Визуальный конструктор с живым превью (Electron)

Основная идея UX: пользователь не пишет код парсера. Он открывает целевой сайт прямо внутри приложения, вводит CSS-селектор и **сразу видит, что этот селектор вытащит со страницы**: уже после обработки RegExp-шагами и в том виде, в котором данные попадут в базу.

```
 Renderer (Vue, PagesView)         Main process (Electron)            Окно превью (parserWindow)
 ─────────────────────────         ───────────────────────            ──────────────────────────
 «Загрузить» ──electronAPI.loadParser(url)──► ipcMain 'open-parser-window'
                                              ├─ создаёт / переиспользует BrowserWindow
                                              ├─ переписывает CSP ответа ──────────►  сайт грузится
                                              └─ loadURL(url)                         как в браузере
 изменили селектор / атрибут /
 режим single|multiply
   └─electronAPI.querySelector({...})──► ipcMain 'querySelector'
                                              └─ webContents.executeJavaScript( ──►  document.querySelector…
                                                   сгенерированный геттер)       ◄──  результат
 parseResults[i].original  ◄──────────────────┘
   └─ findAnReplaceUpdate() → parseResults[i].edited   (RegExp-шаги прямо в UI)
```

Как это устроено:
- **Отдельное окно с реальным движком Chromium.** Страница рендерится полностью, вместе с JS, SPA-контентом и cookie. Поэтому превью совпадает с тем, что потом увидит Puppeteer, а не с «голым» HTML, который вернул бы `fetch` + `jsdom`. Вариант с jsdom пробовал, он остался закомментированным в `main/index.ts`.
- **Подмена CSP.** Через `session.webRequest.onHeadersReceived` заголовки `Content-Security-Policy` у загружаемого сайта заменяются на разрешающие, иначе многие сайты запрещают выполнять инъецированный JS.
- **Безопасная граница IPC.** Renderer не получает доступа к Node и к чужому окну. Через `contextBridge` (preload) ему открыты только три метода: `loadParser`, `querySelector` и `querySelectorAll`. Вся работа с чужой страницей выполняется в main-процессе.
- **Одно окно превью.** При повторной загрузке существующее окно переиспользуется (`focus` + `loadURL`), новые не плодятся.
- **Мгновенная обратная связь.** Любое изменение поля конструкции (`onFieldChange`) заново выполняет запрос к живой странице. Затем результат прогоняется через RegExp-шаги, и в UI показываются сразу два значения: «оригинал» и «после обработки».

### 2. Конструкция → JS-геттер: один формат для превью и для боевого парсинга

Каждый «квери» страницы хранится как декларативное описание (`pagesQueries`, JSONB):

```json
{ "htmlSelector": ".b-cat__item a", "parseMode": "multiply",
  "attributeType": "attribute", "attributeName": "href",
  "dbId": "...", "tableId": "...", "fieldId": "...",
  "regExpSteps": [{ "find": "{textstart}", "replace": "https://www.fl.ru" }] }
```

Сервер (`TasksService.generateRequestFunction`) **компилирует** это описание в строку JS, например:

```js
Array.from(document.querySelectorAll('.b-cat__item a')).map(el => el.getAttribute('href'))
```

Строка уходит исполнителю и выполняется через `page.evaluate()` в Puppeteer. Превью в Electron строит такой же геттер и выполняет его через `executeJavaScript`. Описание остаётся данными (его можно хранить, редактировать и версионировать), а исполнитель максимально «тупой»: он не знает про базы данных и поля, а просто выполняет функции на странице и возвращает результат.

### 3. Мини-DSL постобработки (RegExp-шаги)

Пользователь, не знающий регулярок, может почистить данные цепочкой шагов «найти → заменить». Кроме обычных RegExp поддерживаются маркеры:

| Маркер | Пример | Смысл |
|---|---|---|
| `{textstart}` | `{textstart}` → `https://www.fl.ru` | дописать или заменить в начале строки (относительная ссылка → абсолютная) |
| `{textend}` | `/{textend}` → `` | заменить в конце строки |
| `{skip}` | `portfolio/{skip}` → `portfolio/` | вырезать всё от «до» до «после» (отрезать query-string и якорь) |

Тот же алгоритм реализован дважды: на клиенте (`PagesView.findAnReplaceUpdate`, для превью) и на сервере (`ParserDataService.applyRegExpSteps`, для записи). Поэтому результат в превью совпадает с тем, что попадёт в БД. Дальше логику стоит вынести в общий пакет, чтобы реализации не разошлись.

### 4. Две базы: метаданные отдельно, контент отдельно

```
 contenter (Sequelize-модели)                contenter_data (динамическая схема)
 ───────────────────────────                 ───────────────────────────────────
 databases ─ database_tables ─ tables_fields   ──sync──►  "<db>___<table>___<hash8>"
 «описание того, какие таблицы нужны»                     реальные таблицы с данными
```

- Пользователь описывает структуру данных в UI: базы, таблицы и поля. Это **данные о данных**, и они хранятся в `contenter`.
- `syncStructureWithContentDatabase` превращает это описание в **реальные таблицы** во второй базе через Sequelize `QueryInterface`: `createTable`, `renameTable`, `renameColumn`. Это DDL во время работы приложения, без миграций.
- **Схема при записи (schema-on-write).** Если в данных пришло поле, которого ещё нет, `insertRow` выполняет `addColumnIfNotExists` (тип `TEXT`), и запись не падает.
- **Изоляция имён.** Имя таблицы `<db>___<table>___<первые 8 символов uuid базы>` позволяет двум проектам иметь таблицы `projects`, не мешая друг другу.
- **Зачем вторая база.** Пользовательские динамические таблицы не перемешиваются со служебными. Контентную базу можно отдельно бэкапить, очищать или отдавать во внешнюю аналитику, а служебная схема остаётся стабильной.

### 5. «Мозг» и «руки»: оркестратор и исполнитель разнесены

```
  server :4000 (оркестратор)                          backend/api-gateway (исполнитель)
  ──────────────────────────                          ─────────────────────────────────
  socket.io-СЕРВЕР  ◄──────── подключается сам ──────── socket.io-КЛИЕНТ (реконнект, backoff)
  startParser(id)
   ├─ emit updateBrowserWsEndpoint ─────────────────►  есть живой Chrome? да → его wsEndpoint
   │                         ◄── browserWsEndpointUpdated ── нет → запустить новый
   ├─ сохранить wsEndpoint в parsers.browserWsEndpoint
   ├─ собрать task (url + геттеры + адреса записи)
   └─ emit parsePage(task) ─────────────────────────►  puppeteer.connect(wsEndpoint)
                                                       page.goto → page.evaluate(геттеры)
  ParserDataService  ◄─────── app{parserResult} ─────  результат
   └─ очистка → RegExp → типизация → дедупликация → INSERT
```

Ключевые решения:
- **Исполнитель подключается к серверу сам**, а не наоборот. Поэтому его можно запустить на любой машине, даже за NAT, например на домашнем компьютере с «живым» Chrome и реальным IP, без проброса портов.
- **Запрос-ответ поверх событийного сокета.** `updateBrowserWsEndpoint` отправляет событие и ждёт ответное через `once(socket, 'browserWsEndpointUpdated')` из `node:events`, то есть получается RPC поверх socket.io.
- **Браузер живёт между запусками.** `wsEndpoint` хранится в записи парсера, и при следующем запуске исполнитель сначала пробует подключиться к уже открытому Chrome (`puppeteer.connect`). Только если это не удалось, он поднимает новый. Так сохраняются сессия, cookie и прогретый кеш, и не тратится время на старт браузера.
- **Реальный браузер вместо headless.** `puppeteer-real-browser` и `ignoreDefaultArgs: ['--enable-automation']` снижают шанс, что антибот-защита распознает автоматизацию.
- **Исполнитель не знает про БД.** Вся логика того, куда и как писать, находится на сервере. Исполнителей можно масштабировать горизонтально (это задел).

### 6. Сценарии: парсеры, которые кормят друг друга

Таблица в контентной БД работает как **очередь задач**:

1. Парсер A (например, категории на fl.ru) собирает список ссылок в таблицу `category` с полем `url` (unique) и флагом `parsed=false`.
2. Парсер B со сценарием `parseLinksFromDB` берёт из этой таблицы следующую запись с `parsed=false` (`ParseLinksScenarioService.prepareParseLink`), парсит её и помечает как `parsed=true`.
3. Когда необработанных ссылок не остаётся, парсер получает статус `finished`.

Так из простых «одностраничных» парсеров собирается многоуровневый краулер: категории → списки → карточки. При этом прогресс хранится в базе и переживает перезапуск. Связи между парсерами визуализируются на графе (vue-flow, `ParsersScenarios`).

### 7. Из колонок в строки и дедупликация

Каждый квери в режиме `multiply` возвращает массив, то есть **колонку** значений. `ParserDataService.handleMultiplyDataPrepare` склеивает колонки одной таблицы по индексу в **строки**: `[{name, url}, {name, url}, …]`. Дальше:
- `null` заменяется на маркер `-|-no__data-|-`, чтобы отличить «элемент не найден» от пустой строки;
- значения приводятся к типу поля (например, для `boolean` пустая строка становится `false`);
- перед `INSERT` выполняется проверка по полям, помеченным в структуре как `unique` (`checkUnique`). Существующие записи пропускаются, поэтому повторный запуск парсера не создаёт дублей.

### Архитектурные решения и компромиссы

| Решение | Почему | Цена / что бы сделал иначе |
|---|---|---|
| Electron для UI | Нужен встроенный полноценный браузер для живого превью; веб-приложение так не сможет из-за CORS, CSP и iframe-ограничений | Тяжёлый клиент; для многопользовательской версии превью можно перенести в удалённый браузер (CDP-стрим) |
| NestJS-модули + репозитории | Чёткие границы: структура БД, контент, задачи и сценарии разделены | Появились циклические зависимости (`forwardRef`); стоит развязать через события или отдельный оркестратор |
| socket.io между сервером и исполнителем | Двусторонняя связь, исполнитель может стоять за NAT | Нет гарантий доставки и подтверждений; для продакшена нужна очередь (BullMQ/Redis) с ретраями |
| Геттеры как строки JS | Простота: исполнитель универсальный | Селектор подставляется в строку, поэтому кавычка в селекторе ломает код. Правильнее передавать функцию с параметрами: `page.evaluate(fn, selector)` |
| Динамические таблицы | Пользователь сам проектирует схему без SQL | Сложнее с индексами и типами (по умолчанию всё `TEXT`); альтернатива — JSONB-документы в одной таблице |

---

## server — API-сервер

**Стек:** NestJS 11, Fastify (адаптер), Sequelize + `sequelize-typescript`, PostgreSQL, socket.io, `@nestjs/schedule`.

**Порт:** `4000`, глобальный префикс всех REST-эндпоинтов — `/api`.

Слушает CORS только для `http://localhost:5173` (порт dev-сервера Vite в dashboard).

### Две базы данных

| База | Назначение |
|---|---|
| `contenter` | «Метаданные»: пользователи, проекты, источники, страницы, парсеры, структура БД (таблицы, поля) |
| `contenter_data` | «Контент»: реальные распарсенные данные. Таблицы создаются динамически с именами вида `<dbName>___<tableName>___<hash>` |

Конфигурация обеих БД — в `src/db-config/db-config.module.ts`. По умолчанию: `localhost:5432`, пользователь `postgres`, пароль пустой (см. `.env` и переменные `PORT`, `DATABASE_NAME`, `DATABASE_PASS`, `PASSPHRASE`).

Контентная база работает через **динамический Sequelize-инстанс** (`DynamicDBProvider`, токен `DYNAMIC_DB`) и репозиторий `DatabaseContentRepository` — там выполняются сырые SQL-запросы: создание таблиц/колонок, вставка и обновление строк, частичная дедупликация.

### Модули и их ответственность

| Модуль | Каталог | Что делает |
|---|---|---|
| `users` | `src/users` | Пользователи. Единственный контроллер `GET /api/users/getByEmail/:email`. При старте создаёт пользователя по умолчанию (Денис, `yozuul@gmail.com`). Хранит `appWebSocketClientId` — id socket-клиента (используется как адрес доставки задач). |
| `projects` | `src/projects` | Проекты: CRUD + удаление всех проектов пользователя. При создании проекта автоматически создаётся дефолтная БД `auto` (`autoCreated: true`). |
| `sources` | `src/sources` | Источники данных (сайты) в рамках проекта: CRUD, массовое обновление. |
| `pages` | `src/pages` | Страницы источника, содержат `pagesQueries` (JSONB) — конструкцию (CSS-селектор + атрибут + режим single/multiply), привязанную к полю БД. |
| `databases` | `src/databases` | Структура данных: базы → таблицы → поля (`databases`, `database_tables`, `tables_fields`), автосоздание структуры по квери страницы, синхронизация структуры в контентную БД. |
| `parsers` | `src/parsers` | Парсеры: CRUD, сбор данных по парсеру (`fieldId → tableName/dbName`), подготовка стартовых URL из БД. |
| `tasks` | `src/tasks` | **Оркестрация парсинга**: формирование задачи (`TasksService`), обмен с клиентом через WebSocket (`TaskSenderService`), приём и запись результатов (`ParserDataService`), сценарии (`scenarios/`). |
| `watchers` | `src/watchers` | «Наблюдатели» — периодический перезапуск парсеров по cron (задел, не доделан). |
| `backup` | `src/backup` | Восстановление демо-данных при старте из `src/backup/projects.ts` (проект «Фриланс биржи» с таблицами категорий/проектов/фрилансеров). |
| `content` | `src/content` | Задел под модуль контента (модель/контроллер пустые). |

### Как работает задача парсинга (ключевой поток)

1. UI вызывает `GET /api/tasks/startParser/:parserId` (контроллер `tasks`).
2. `TasksService.startParser()`:
   - загружает парсер вместе со страницей и `pagesQueries`;
   - через `TaskSenderService.updateBrowserWsEndpoint()` просит клиента (api-gateway) поднять/вернуть WebSocket-адрес браузера;
   - формирует `readyTask` — объект с `urlToParse`, `browserWsEndpoint`, списком `queries`. Для каждого квери генерируется **JS-функция-геттер** (`generateRequestFunction`): например `document.querySelector('...').textContent` или `Array.from(document.querySelectorAll('...')).map(el => el.getAttribute('href'))`;
   - если сценарий `parseLinksFromDB`/`watchUrlsToParse` — берётся следующая нераспарсенная ссылка из контентной таблицы (`parse-links.service.ts`);
   - отправляет задачу клиенту через WebSocket-событие `parsePage`.
3. Клиент (backend/api-gateway) выполняет парсинг в реальном браузере и шлёт результат обратно по socket (событие `app`, команда `parserResult`).
4. `ParserDataService.processParserData()`:
   - заменяет `null`-значения на `-|-no__data-|-`;
   - применяет RegExp-шаги (`regExpSteps`), приводит значения к типу поля (boolean и т.д.);
   - группирует данные по таблицам-назначениям и записывает в контентную БД строки (`insertRow`/`insertUpdateData`), пропуская дубликаты по уникальным полям.

### WebSocket-гейтвей

`TaskSenderService` — `@WebSocketGateway` на порту 4000 (`cors: '*'`). Хранит подключённых клиентов в `Map<clientId, Socket>`. События:

| Событие | Кто инициирует | Что делает |
|---|---|---|
| `app` (команды `parserResult`, `updateConfig`) | клиент | обработка результата парсинга |
| `updateBrowserWsEndpoint` / `browserWsEndpointUpdated` | сервер / клиент | согласование адреса браузера |
| `parsePage` | сервер | отправка задачи на парсинг |
| `newTask` | сервер | отправка задачи (задел) |

Из-за того, что `getClientId()` берёт из БД **одного** пользователя, сервер фактически рассчитан на одного подключённого клиента.

---

## backend — «исполнитель» (браузер-парсер)

**Стек:** NestJS 11 (монорепо из двух приложений), Puppeteer + `puppeteer-real-browser`, socket.io-client, `@nestjs/microservices` (TCP).

### apps/api-gateway (порт 3000)

- HTTP-сервис NestJS, при старте подключается **как socket.io-клиент** к `http://localhost:4000`.
- Обрабатывает события от server'а:
  - `task` — логирование;
  - `updateBrowserWsEndpoint` — запускает **реальный Chrome** через `puppeteer-real-browser` (не головной!), запоминает `wsEndpoint` и возвращает его обратно (событие `browserWsEndpointUpdated`);
  - `parsePage` — через `BrowsersService.parsePage()` открывает страницу, выполняет JS-геттеры из задачи и шлёт результат назад (`app` → `parserResult`);
  - `serverAppResponse` — логирование.
- Поддерживает реконнект к server'у (до 10 попыток с экспоненциальной задержкой).
- Имеет `ClientProxy` к TCP-микросервису browser-connector (порт 3001) — пересылает команду `process_data` (демо/задел).

Задел под парсинг «из-под» авторизованного пользователя: сохранённые cookie и localStorage сайта подгружаются в браузер перед парсингом (флаг `auth` у парсера). Файлы сессии в репозиторий не входят, а применение реализовано не до конца.

### apps/browser-connector (порт 3001)

- TCP-микросервис (`Transport.TCP`). Контроллер кроме pattern `process_data` экспонирует HTTP-эндпоинты `POST /checkBrowser` и `POST /parse`, которые частично задекларированы (`startBrowser`, `parse`).
- Сервис запускает браузер через `puppeteer-real-browser` (реальная сессия Chrome). Много кода закомментировано — это ранняя экспериментальная версия.

## dashboard — десктоп-клиент

**Стек:** Electron 31, electron-vite, Vue 3 (Composition API, `<script setup>`), TypeScript, Pinia, Vue Router, Element Plus, vue-flow, SCSS, `@electron-toolkit/*`.

### Процессы

| Часть | Путь | Что делает |
|---|---|---|
| Main | `src/main/index.ts` | Создаёт безрамочное окно 1600×870 на втором мониторе (`x: 1920`), подменяет CSP-заголовки для доступа к `http://localhost:4000`, открывает DevTools; IPC-хендлеры — в `src/main/ipc-handlers.ts` (заглушки под auth), `open-parser-window` (окно-предпросмотр страницы с обходом CSP), `querySelector` / `querySelectorAll` (проверка CSS-селектора на живой странице). |
| Preload | `src/preload/index.ts`, `parse-preload.ts` | `contextBridge` — открывает renderer'у `electron.ipcRenderer` и `electronAPI` (`loadParser`, `querySelector*`). |
| Renderer | `src/renderer/` | Весь UI: роутер (`routers.ts`), 6 Pinia-сторов, представления проектов/источников/страниц/парсеров/БД/настроек/регэкспов. |

### Экранные представления и маршруты

- `/projects`, `/projects/new`, `/projects/:projectId` — проекты (карточка + список БД + добавление источников);
- `/projects/:projectId/sources`, `/sources/:sourceId`, `/sources/:sourceId/pages/new` — источники и страницы;
- `/pages/:pageId` — редактирование страницы и `pagesQueries` (FieldComponent — один кверь: имя поля, селектор, тип сбора single/multiply, атрибут, привязка к БД/таблице/полю, Regexp-шаги через `RegExpDialog`);
- `/projects/:projectId/parsers`, `/parsers/new/:projectId`, `/parsers/:parserId` — список и редактор парсеров (система сценариев `ParsersScenarios`, визуализация на vue-flow — `SpecialNode`/`SpecialEdge`);
- `/databases/:databaseId` — структура БД (`DatabaseList`, таблицы, поля);
- `/regexps` — редактор регулярных выражений;
- `/settings` — настройки (пустая страница-задел).

### Сторы

| Стор | Файл | Отвечает за |
|---|---|---|
| `projectStore` | `store/projects-store.ts` | юзер, проекты (CRUD), поиск пользователя по email |
| `sourceStore` | `store/source-store.ts` | источники |
| `pageStore` | `store/page-store.ts` | страницы |
| `databaseStore` | `store/database-store.ts` | БД: список, «полезные» (`availableDatabases`), дефолтная `auto`, структура |
| `parserStore` | `store/parser-store.ts` | парсеры |
| `taskStore` | `store/task-store.ts` | запуск парсера (`/tasks/startParser/:id`) |
| `authStore` (задел) | `store-electron.ts` | логин/рефреш токенов (не используется — нет серверных эндпоинтов `auth/*`) |

Все стора ходят в API напрямую через `fetch` на `BASE_URL` (`http://localhost:4000/api` в dev, `https://server.ru/api` в проде).

## Модели данных (база `contenter`)

```
users (id uuid PK, name, email, password, appWebSocketClientId)
 └── projects (id, name, code, description, userId FK)
      ├── databases (id, hash[8], name, type, description, autoCreated, projectId FK)
      │    └── database_tables (id, name, databaseId FK)
      │         └── tables_fields (id, name, type, unique, position, tableId FK)
      └── sources (id, name, url, description, type, projectId FK)
           └── pages (id, name, url, description, attachDatabase, pagesQueries JSONB, sourceId FK)
                └── parsers (id, name, description, scenario, parsersRules JSONB,
                             browserWsEndpoint, stopped, finished, auth, pageId FK)
```

Таблицы в контентной базе (`contenter_data`) создаются динамически с именем `<имя_бд>___<имя_таблицы>___<первые 8 символов id БД>`, например `freelance_platforms_data___projects___1b66728c`. Поля соответствуют полям структуры, плюс служебные `id`, `createdAt`, `updatedAt`.

**Сценарии парсинга** (поле `scenario` в `parsers.parsersRules`):

| Сценарий | Смысл |
|---|---|
| `parseLinksFromDB` | Брать ссылки из таблицы контентной БД (где `parsed=false`), парсить по очереди, помечать `parsed=true` |
| `watchUrlsToParse` | Как `parseLinksFromDB` (в коде обрабатывается вместе с ним) |
| `watchPage` | Слежение за конкретной страницей `parser.page.url` (задел) |

---

## Порты

| Сервис | Порт | Транспорт |
|---|---|---|
| `server` (REST + WS) | 4000 | HTTP / socket.io |
| `backend` / api-gateway | 3000 | HTTP (и socket.io-клиент на :4000) |
| `backend` / browser-connector | 3001 | TCP (микросервис) |
| dashboard (dev: Vite) | 5173 | — (CORS-friendly для server'а) |

---

## Как запускать

Требования: Node.js 18+, PostgreSQL 16+ (базы `contenter` и `contenter_data`).

**1. База данных** — создать БД `contenter` и `contenter_data`. Подключение по умолчанию: `postgres` без пароля на `localhost:5432` (правится в `server/src/db-config/db-config.module.ts` и `server/.env`).

**2. server** (`cd server`):

```bash
npm install
npm run start:dev       # NestJS с watch, REST + WS на :4000
```

При первом старте создаст пользователя по умолчанию и восстановит демо-проект «Фриланс биржи» (из `src/backup/projects.ts`).

**3. backend** (`cd backend`):

```bash
npm install
npm run start:dev       # api-gateway :3000
nest start browser-connector   # второе приложение (TCP-микросервис)
```

Пока api-gateway работает, он держит socket.io-подключение к server'у — без этого задачи парсинга не будут доставлены.

**4. dashboard** (`cd dashboard`):

```bash
npm install
npm run dev             # electron-vite dev
```

Откроется окно управления (интерфейс на русском, «CONTENTER»).

---

## Текущее состояние проекта (на момент написания README)

- **Стадия:** активная разработка, 1 разработчик; README в подпапках — стандартные шаблоны NestJS.
- **Готово:** REST API метаданных (проекты/источники/страницы/парсеры/структура БД), генерация и выполнение JS-геттеров в реальном Chrome, запись результатов в контентную БД, дедупликация, WebSocket-мост server ↔ api-gateway, UI на Element Plus.
- **Заделы/недоделки:** авторизация (JWT-зависимости подключены, эндпоинтов `auth/*` нет), watchers (cron), `watchPage`-сценарий, модуль `content`, browser-connector (частично закомментирован), определение окна по клику мыши (robotjs/ps-list/active-win — закомментировано).
- **Известные особенности:** часть данных захардкожена (email пользователя в сторах dashboard и в `TaskSenderService`), сервер предполагает одного socket-клиента.