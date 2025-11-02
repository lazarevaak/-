# Study Plan App — Security NFR Report

**Автор:** Лазарева Александра Константиновна  
**Группа:** БПИ‑234  
**Пакет:** «Требования безопасности + Модель угроз» (в этой версии — **только NFR**; без ADR/рисков/STRIDE/DFD)

---

## 1. Описание проекта

**Study Plan App** — мини‑API для планирования учёбы: создание тем, дедлайнов и трекинга прогресса.

- **Цель:** учебный стенд для практик DevSecOps — формулировка измеримых NFR, негативные/нагрузочные тесты, CI.
- **Технологии:** FastAPI, SQLAlchemy (SQLite), Pydantic v2.
- **Хранение:** локальная `studyplan.db` (только ORM, без raw SQL).
- **Безопасность в коде:** CORS‑allowlist, единый формат ошибок (RFC7807‑style problem JSON), лимит тела запроса, простой per‑IP rate‑limit, `X‑Request‑ID`, маскирование чувствительных полей в логах, безопасная загрузка файлов (PNG/JPEG ≤ 2 MB, защита от path traversal и symlink).  
  Outbound‑запросы: безопасный HTTP‑клиент с таймаутами и ретраями (`safe_get`).
- **Тесты:** pytest (включая негативные), проверка покрытия, базовый нагрузочный замер.
- **Границы и потоки:** Клиент → API (HTTPS/JSON). API → БД через ORM. API → логи (корреляция по `X‑Request‑ID`). Отдельный эндпоинт `/upload` для изображений.
- **Эндпоинты:**
  - `POST /topics` — создать тему
  - `GET /topics` — список тем
  - `GET /topics/{id}` — получить тему
  - `PUT /topics/{id}/progress` — обновить прогресс (0..100)
  - `DELETE /topics/{id}` — удалить
  - `POST /upload` — загрузка PNG/JPEG (≤ 2 MB)

**Фикстуры тестов:**  
Отдельная тестовая БД через `DATABASE_URL=sqlite:///./test_studyplan.db`, автосброс таблицы `topics` перед каждым тестом, общий `TestClient` на сессию.

---

## 2. Сводная таблица NFR (что сделано, где и как проверено)

| ID | Название | Цель / Описание | Метрика / Порог | Как проверено (pytest) | Где в коде (ключевые места) | Компонент | Статус |
|---|---|---|---|---|---|---|---|
| **NFR‑01** | Время ответа API | Быстрый отклик `/topics` | p95 ≤ **400 мс** | `test_topics_response_time_manual()` | `app/main.py` (эндпоинты), sqlite/ORM | FastAPI backend | ✔️ |
| **NFR‑02** | Единый формат ошибок | 404/422 в формате problem+json | ≥ 95 % корректных ответов | `test_not_found_topic_format()`, `test_validation_error_format()`, `test_rfc7807_404_and_422` | `http_exc_handler`, `validation_exc_handler`, `app.utils.errors.problem_json` | API Layer | ✔️ |
| **NFR‑03** | Валидация входных данных | `title` обязателен, `progress ∈ [0;100]` | 100 % ошибок ловятся валидатором | `test_create_topic_without_title()`, `test_progress_out_of_range()` | `TopicCreate`, `ProgressUpdate`, проверки в CRUD | Validation | ✔️ |
| **NFR‑04** | Безопасность БД | Только ORM (без raw SQL) | 100 % ORM‑вызовов | `test_no_raw_sql_in_app_code()` (regex‑скан кода) | `app/database.py`, `Topic` в `app/main.py` | DB Layer | ✔️ |
| **NFR‑05** | Логирование операций | Все `/topics` логируются на INFO | ≥ 90 % запросов | `test_logging_of_requests()` (`caplog`) | `log_requests` + `mask_sensitive` | Logging middleware | ✔️ |
| **NFR‑06** | Устойчивость под нагрузкой | Ошибок при ~50 RPS ≤ 2 % | error_rate ≤ **2 %** | `test_stability_under_load()` (500 req, 50 потоков) | Эндпоинты `/topics`, sqlite/ORM | API backend | ✔️ |
| **NFR‑07** | Покрытие тестами | ≥ 80 % строк | coverage ≥ **80 %** | `test_coverage_threshold()` | Весь модуль `app/*` | CI / Backend | ✔️ |
| **NFR‑08** | Консистентность данных | После DELETE запись недоступна | 100 % корректных транзакций | `test_topic_consistency_after_deletion()` | `DELETE /topics/{id}`, `GET /topics/{id}` | DB Layer | ✔️ |

**Доп. проверки контролей:**  
- **CORS:** `test_cors_preflight_allowed_and_denied()`, `test_preflight_*`, `test_simple_get_includes_cors_header_for_allowed_origin()`  
- **Лимиты/Rate‑limit:** `test_body_too_large_returns_413()`, `test_limits_payload_and_rate_limit()`  
- **Дубликаты:** `test_duplicate_conflict()`, `test_post_deduplicates_by_title_deadline()`  
- **CRUD Happy‑path:** `test_crud_happy()`

---

## 3. Детализация NFR (что сделал → зачем → как проверил)

### NFR‑01 — Время ответа API (p95 ≤ 400 мс)
- **Что сделал:** реализованы CRUD‑эндпоинты `/topics` на FastAPI + SQLite через SQLAlchemy ORM.
- **Зачем:** обеспечить приемлемую отзывчивость пользовательских сценариев списка/добавления тем.
- **Как проверил:** синтетический замер 50 запросов с расчётом p95 — `test_topics_response_time_manual()`.

### NFR‑02 — Единый формат ошибок (RFC7807‑style problem JSON)
- **Что сделал:** единые обработчики ошибок: `http_exc_handler` (включая 404/409/422) и `validation_exc_handler` (422). Все ответы — `application/problem+json` с полями `title/status/detail` (+ `errors` для валидации).
- **Зачем:** консистентность клиентской обработки ошибок и читаемые кейсы в логах.
- **Как проверил:** `test_not_found_topic_format()`, `test_validation_error_format()`, `test_rfc7807_404_and_422` проверяют коды/заголовки/структуру тела.

### NFR‑3 — Валидация входных данных
- **Что сделал:** схемы Pydantic v2: `TopicCreate` (обязательный `title`, валидный `deadline`), `ProgressUpdate` (целое 0..100). В `create_topic` — доменная проверка «deadline не в прошлом».
- **Зачем:** предсказуемые сообщения об ошибках и защита от некорректного ввода.
- **Как проверил:** `test_create_topic_without_title()` (422) и `test_progress_out_of_range()` (422).

### NFR‑04 — Безопасность БД (только ORM)
- **Что сделал:** все операции через SQLAlchemy ORM; уникальность `(title, deadline)` на уровне схемы; **нет** `.execute()` и raw SQL.
- **Зачем:** снизить риск SQL‑инъекций и обеспечить переносимость кода.
- **Как проверил:** `test_no_raw_sql_in_app_code()` сканирует репозиторий на запрещённые шаблоны; `test_post_deduplicates_by_title_deadline()` и `test_duplicate_conflict()` проверяют уникальность.

### NFR‑05 — Логирование (INFO + маскирование)
- **Что сделал:** middleware `log_requests` логирует метод/путь/код ответа; тело запроса проходит через `mask_sensitive` (поля `password/token/secret` заменяются на `****`). Корреляция по `X‑Request‑ID` в отдельном middleware.
- **Зачем:** воспроизводимость инцидентов и наблюдаемость без утечки чувствительных данных.
- **Как проверил:** `test_logging_of_requests()` с `caplog` убеждается, что операции `/topics` пишутся на уровне INFO.

### NFR‑06 — Устойчивость под нагрузкой (~50 RPS, error‑rate ≤ 2 %)
- **Что сделал:** проверка стабильности обработчиков `/topics` в многопоточном режиме клиента.
- **Зачем:** убедиться, что при параллельной нагрузке не возникает деградаций/ошибок приложения.
- **Как проверил:** `test_stability_under_load()` — пул из 50 потоков, 500 запросов; считается error‑rate и средняя RPS.

### NFR‑07 — Покрытие тестами (≥ 80 %)
- **Что сделал:** unit/интеграционные тесты для CRUD, валидации, ошибок, CORS, лимитов и логирования.
- **Зачем:** регрессионная устойчивость и прозрачность качества.
- **Как проверил:** `test_coverage_threshold()` запускает pytest с `--cov-fail-under=80`.

### NFR‑08 — Консистентность данных (после DELETE запись недоступна)
- **Что сделал:** удаление темы атомарно коммитится; чтение удалённой — 404.
- **Зачем:** целостность данных и предсказуемость пользовательских сценариев.
- **Как проверил:** `test_topic_consistency_after_deletion()` (create → get → delete → get=404).

---

## 4. Где находится реализация контролей (быстрые ссылки по модулям)

- **Эндпоинты и middleware:** `app/main.py`  
  - Лимит размера тела (`MAX_BODY_BYTES`) — `body_size_limit_middleware`  
  - Простейший rate‑limit (`RATE_LIMIT_RPM`) — `rate_limit_middleware`  
  - Единый формат ошибок — `http_exc_handler`, `validation_exc_handler`  
  - Структурные логи + маскирование — `log_requests` (+ `mask_sensitive`)  
- **ORM‑модель и соединение:** `Topic` в `app/main.py`, база и сессии — `app/database.py`
- **Безопасная загрузка:** `app/secure_files.py::secure_save` (PNG/JPEG ≤ 2 MB, traversal/symlink guard)
- **Тестовые фикстуры:** общий клиент и очистка БД — в файле с фикстурами/тестами (см. данный пакет)


