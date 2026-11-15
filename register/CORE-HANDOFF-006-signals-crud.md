# Отчёт о завершении: Логика сигналов и обновление CRUD приборов

**Проект:** KIP_register
**Статус:** Завершено
**Дата:** 15.11.2026
**Задача:** `CORE-TASK-006-signals-crud.md` — nullable поля (CORE-ADR-014), автосоздание/пересоздание сигналов, GET/PUT сигналов.

---

## 1. Что сделано

**Модель и миграции (CORE-ADR-014).**
- `device.project_standard_id` — nullable (прибор-черновик).
- `signal.control_system_id` — nullable (при автосоздании NULL, заполняется вручную).
- Миграция `77df9a477560`: `ALTER COLUMN ... DROP NOT NULL` для обеих колонок. Применена, проверена в `information_schema`.
- CASCADE для зависимых и связующих таблиц (7 FK): `signal.device_id`, `signal_override_history.signal_id`, `segment_signal.signal_id`/`segment_id`, `device_override_history.device_id`, `device_document.device_id`/`document_id`. Миграция `a1c2e3f4b5d6` (вручную: autogenerate не видит `ondelete` в существующих FK). Применена, проверена в `referential_constraints`.
- `segment.address_a_*`/`address_b_*` — NO ACTION (сегмент самостоятельный физический объект; удаление прибора-адреса блокируется 409). Зафиксировано в `REG-ARCH-db.md`.

**CRUD приборов (`devices/service.py`).**
- `create_item`: `project_standard_id` необязателен. Задан → сигналы создаются по `project_standard_signal` (в той же транзакции); NULL → черновик, сигналов нет.
- `update_item`: при фактическом изменении `project_standard_id` (NULL→ID, ID1→ID2, ID→NULL) сигналы пересоздаются: старые удаляются, новые создаются по шаблону (или не создаются при NULL). No-op (тот же ID или поле не передано) → сигналы не трогаются.
- `_recreate_signals`: `DELETE` + `INSERT` в одной транзакции; `control_system_id = NULL`, переопределения сигналов сбрасываются.

**Сигналы (`signals/service.py`, `api/signals.py`).**
- `GET /api/devices/<id>/signals` → `{items, total}`; наследование трёх полей (`electrical_implementation_id`, `setpoint_level_id`, `setpoint_value`), `overrides`; нет стандарта → `[]`.
- `PUT /api/signals/<id>` → редактирование 4 полей (`electrical_implementation_id`, `setpoint_level_id`, `setpoint_value`, `control_system_id`). `position`/`device_id` — unknown (400). Без POST/DELETE.
- `updated_at` — явно (no-op PUT).

**Контракты.**
- `REG-CONTRACT-devices.md` обновлён: nullable стандарт, автосоздание/пересоздание/no-op, CASCADE, `project_standard_id` не обязателен, 404-нецелый id.
- `REG-CONTRACT-signals.md` — новый: GET/PUT, наследование, ошибки, пробелы.
- `REG-ARCH-db.md` обновлён: nullable, CASCADE, NO ACTION адресов.

**Тесты.**
- Наши: 29 новых (`test_devices_signals.py` — 10, `test_signals.py` — 16, фикстура `standard_with_signals`). Итого 286.
- Независимые: 61 (`tests/independent/test_signals_api.py`), 39 строгих + 6 характеризационных функций (55/6 кейсов). Итого независимых 198 (137 v2 + 61 v3).
- Полный прогон: **347 passed**, 0 failed, 0 skipped, 122.98 s.
- Отдельный прогон нового файла: 61 passed, 18.74 s.

**Независимое тестирование (Level 2).**
- Задание `REG-TASK-TEST-004-signals-api.md`, включая обзор старых тестов (раздел 2.1).
- 8 вопросов тестировщика, все закрыты ответами (без правок контрактов).
- Отчёт `docs/testing/REG-REPORT-004-signals-api-bc134bb.md`: находок нет; удаление 2 старых тестов (`test_devices_missing_project_standard`, `test_devices_null_project_standard`) признано обоснованным; других устаревших после ADR-014 не найдено.

**CHANGELOG** — запись 15.11.2026 (вечер): черновик прибора, автосоздание сигналов, GET/PUT сигналов, каскадное удаление.

**Коммиты:** `bc134bb` (код, миграции, тесты), `31ebb1b` (`REG-ARCH-db.md`), `cd03e21` (контракты), `e8d3a12` (CHANGELOG), `64666fb` (задание тестировщику), `8eaa8f9` (независимые тесты, отчёт).

## 2. Что найдено в процессе (важно)

**Запрос в Core до кода — `REG-REQ-006`.** До реализации обнаружено противоречие: задание требовало «новые сигналы создаются по шаблону», но `control_system_id` (NOT NULL, не наследуется) взять неоткуда. Запрос → `CORE-ADR-014` (nullable). Код начат только после решения — не гадали.

**CASCADE — расширение объёма по решению арбитра.** Задание не упоминало каскад. Обнаружено: DELETE прибора с сигналами упал бы на FK. Решение: CASCADE для 5 зависимых FK, включая связующие `segment_signal`, `device_document` (симметрия: связь не существует без родителя). Адреса сегмента — NO ACTION. Миграция `a1c2e3f4b5d6`.

**Два старых независимых теста устарели — удалены.** `test_devices_missing_project_standard`, `test_devices_null_project_standard` проверяли «POST без стандарта → 400»; после ADR-014 → 201. Править независимые тесты разработчиком — подгонка; удалить — честно. Новое поведение покрыто `test_draft_device_creation_contract` (тестировщик, v3). Обоснованность удаления проверена независимо.

**Тестировщик отметил:** арбитр запустил только полный `pytest tests -q`, не отдельно новый файл. Закрыто раздельным прогоном (61 passed).

## 3. Что требует внимания

- **`REG-CONTRACT-devices.md` с BOM** — с прошлой итерации, не критично, при правке перезаписать в UTF-8 без BOM.
- **Отсутствие `GET /api/signals/<id>`** — тестировщик отметил: чтобы получить сигнал по id, приходится брать из списка (`_signal_by_position`). Не входит в задание (только GET списка и PUT), но для UI понадобится. Отдельная задача.
- **CRUD `project_standard`** отсутствует — тестировщик создавал второй стандарт через `session` (fixture-level setup). Когда появится CRUD стандарта, тесты можно перевести на API.
- **Открытые пункты с прошлых заданий:** история переопределений (`device_override_history` — таблица есть, не заполняется), `device_document` CRUD, race-ветки 409 (невоспроизводимы в Black-box).
- **`segment.address_*` NO ACTION** — при появлении CRUD сегментов удаление прибора-адреса будет 409 `in_use`. Это по решению арбитра; UI должен предупреждать.
- **CHANGELOG «(вечер)»** — самодеятельность, не по правилу (дата, версия, категория). Оставлено по решению арбитра; впредь — спрашивать время или дописывать в существующую запись.

## 4. Доказательства

- **Коммиты:** `bc134bb` (код, миграции, наши тесты), `31ebb1b` (`REG-ARCH-db.md`), `cd03e21` (контракты), `e8d3a12` (CHANGELOG), `64666fb` (задание тестировщику), `8eaa8f9` (независимые тесты, отчёт).
- **Миграции:** `migrations/versions/77df9a477560_...py` (nullable), `a1c2e3f4b5d6_cascade_fks.py` (CASCADE).
- **Код:** `src/kip_register/devices/service.py` (автосоздание), `src/kip_register/signals/service.py`, `src/kip_register/api/signals.py`, `app.py`, модели (`device.py`, `signal.py`, `audit.py`, `segment.py` — `ondelete`).
- **Тесты:** `tests/test_devices_signals.py`, `tests/test_signals.py`, `tests/independent/test_signals_api.py`.
- **Прогон:** `pytest tests -q` → 347 passed; `pytest tests/independent/test_signals_api.py -q` → 61 passed.
- **Контракты:** `docs/api/REG-CONTRACT-devices.md`, `docs/api/REG-CONTRACT-signals.md`, `docs/architecture/REG-ARCH-db.md`.
- **Отчёт тестировщика:** `docs/testing/REG-REPORT-004-signals-api-bc134bb.md`.
- **CHANGELOG:** `CHANGELOG.md`, запись 15.11.2026 (вечер).

## 5. Вывод для управляющего чата

Готово к приёмке: `CORE-ADR-014` реализован (nullable стандарт и control_system), автосоздание/пересоздание/сброс сигналов при назначении/смене/сбросе стандарта, GET/PUT сигналов, CASCADE для зависимых FK, 347 тестов проходят (286 наших + 61 независимых), независимое тестирование v3 — находок нет, удаление 2 устаревших тестов обосновано и проверено. Требует внимания — отсутствие `GET /api/signals/<id>` (отдельная задача), CRUD `project_standard` (для перевода тестов на API), открытые пункты с прошлых заданий.