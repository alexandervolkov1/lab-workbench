# Поэтапный план реализации lab-workbench

Статус: **план после архитектурного ревью, не разрешение начать реализацию**.
Основание: [architecture-proposal.md](architecture-proposal.md).

## 0. Зафиксировать решения ревью

Подтвердить один Runtime/одно окно, отдельные scope, документ без scripts,
внешний API без Runtime gateway, bootstrap с domain fences, отсутствие persistent
GUI state и автоматического повторения mutations. Разобрать
[open-questions.md](open-questions.md).

Результат: короткие изменения архитектурного документа по принятым решениям.
Не начинать сразу «всю систему» после первого обсуждённого пункта.

## 1. Пустое штатное окно

Создать один Cargo package. Выбрать совместимые `eframe/egui/egui_plot`, закрепить
lockfile и MSRV. `eframe::run_native`, одна системная строка, стандартная панель,
обычный пустой Plot. Не включать persistence. Ввести отдельный Tokio IO-поток
и ограниченное штатное завершение.

Проверка: окно запускается/сворачивается/закрывается без зависания; `logic` получает
repaint при свёрнутом окне. Ни Runtime child process, ни GPU host helpers.
`CARGO_TARGET_DIR=D:\lab-workbench-sandbox\cargo-target`.

## 2. Read-only подключение и discovery

Собственные Application wire DTO, bounded framing, hello и capabilities.
Прочитать discovery pages последовательно; показать instruments/signals,
descriptor и recording_status. Сформировать typed ObservedRuntime и состояния
Disconnected/Connecting/Synchronizing/Live. Пока не включать управление.

Проверка: тестовый socket делит JSON на части, посылает CRLF, malformed/oversized
frame, response error. Workbench не зависает и не импортирует Runtime crates.
На реальном Runtime проверить обе композиции: Recorder absent и enabled.

## 3. Один живой график с корректным bootstrap

Реализовать subscription от hello cursor, D/M fences, контрольный F, bounded
staging и установку снимков; добавить один signal source и сегментированный
ограниченный buffer. Температура виртуальной печи обновляется непосредственно
Runtime events. Штатные zoom/drag/follow latest.

Проверка: непрерывные точки без polling latest на каждый sample; тест гонки
D=100 → Reference event 101 → M=105; timeout/gap не оставляет Live.
Последовательное чтение frozen families не уничтожает свои continuation tokens.

## 4. Reconnect, время и границы данных

Backoff, новый connection epoch, сохранение cached plots, полный bootstrap после
reconnect, separate boot/generation/authority-epoch segments, stale indicators.
Добавить optional `measurement_window` для короткого начального заполнения.
Calendar axis и durable history пока не обязательны.

Проверка: kill/restore тестового socket; новый Runtime boot; unavailable sample;
Runtime ring и локальная очередь переполнены; линия никогда не пересекает разрыв.
Превышение memory budget обрезает старые данные с видимой пометкой.

## 5. Декларативный документ и UI API

Строгие schema v1 DTO, стабильные IDs, стандартный layout. Реализовать loopback
hello/get/replace/status и, если утверждено, шесть patch operations.
Установку документа и локальные edits проводить одним CAS-путём в главном потоке.
Сначала поддержать графики и индикаторы; editor widgets пока только меняют drafts.

Проверка: Clojure создаёт два графика/панель, затем отключается — GUI остаётся.
Два клиента конфликтуют на base revision без частичных изменений.
Ошибочный документ, oversized frame и slow client не блокируют Runtime reader.
Старый instance ID после рестарта отвергается. API работает при свёрнутом окне.

## 6. Полная read-only печь

Добавить typed bindings Reference, controller, output, Recorder и connection.
Для power реализовать `output_observation` с readback timestamp и ясной подписью.
Recorder lifecycle events инвалидируют query, counters обновляются редкими queries.

Проверка: второй независимый Runtime-клиент меняет setpoint/PID/Recorder,
Workbench узнаёт об этом сам. Мощность не запрашивается через несуществующий signal
и не подменяется `controller.latest_output`. Новых readback нет — точек не добавляется.

## 7. Операторские действия и неопределённые исходы

Сначала Reference и controller lifecycle, затем PID configuration и Recorder.
Ввести actual/draft/pending, подтверждения, captured revision/run/boot,
один mutation sequencer и limited RAM evidence. До first write сохранить exact
request identity. Отдельная кнопка operation_status; автоматического resubmit нет.

Проверка: redraw, drag и focus loss не отправляют команд. Чужое изменение revision
не перезаписывается старым draft. Одна confirm-активация даёт одну mutation.
Обрыв на каждой границе accepted/terminal даёт корректный unknown. После reconnect
актуальное значение не считается результатом потерянной команды.

## 8. Явное ручное управление выходом

Добавить acquire/renew/write/safe, только по собственному scope и Runtime profile.
Показать owner, lease expiry, safe_pending/safe_confirmed и delivery evidence.
Не включать автоматическое продление. Если short-lease UX не принят, оставить
ручные output actions disabled и закончить обсуждение до объявления полной версии.

Проверка: Clojure и Workbench имеют разные sessions; чужая lease блокирует
write/renew/safe, PID и manual authority конфликтуют по Runtime.
Safe completed ещё не означает safe_confirmed. Истёкшее подтверждение не вызывает
reacquire. После known admission долгого write явный safe не ждёт его terminal;
при unknown admission новый seq не угадывается.
Закрытие Workbench не посылает команды Runtime автоматически.

## 9. Приёмка первой полноценной версии

Выполнить двухклиентский сценарий из архитектуры на виртуальной печи:
температура, output readback, Reference, PID, Recorder, ошибки, disconnect/reconnect
и новый boot. Проверить выключенный Recorder и ограниченную RAM history.
Прогнать protocol/model/action tests и короткий GUI smoke; `fmt`, `clippy`, тесты
применительно к созданным исходникам.

Документировать фактические wire examples UI API и Clojure transcript.
Никаких скрытых дополнительных listeners, Runtime dependencies или дискового
recovery state. Все технические артефакты и test DB внутри sandbox.

## Отдельные необязательные улучшения после этой версии

- Durable history через публичный API с явными gap/coverage, без чтения SQLite.
- Optional UTC axis на time_anchor.
- Min/max downsampling после измерений производительности.
- Более удобная layout-настройка на штатных egui widgets.

Для них нужен конкретный пользовательский сценарий. Multi-runtime, plugins,
внешний Runtime gateway, автопродление manual leases и сложная recovery-система
не являются автоматическим продолжением этого плана.
