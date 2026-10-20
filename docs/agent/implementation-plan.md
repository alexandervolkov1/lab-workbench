# Поэтапный план реализации lab-workbench

Статус: **общая архитектура одобрена, поправки ревью внесены; текущее задание
только на документы, реализация не начата**.
Основание: [architecture-proposal.md](architecture-proposal.md).

## 0. Решения ревью зафиксированы

Сохранены один Runtime/одно окно, отдельные scope, документ без scripts,
внешний API без Runtime gateway, корректный bootstrap с domain fences,
ограниченные очереди, RAM-only GUI state и запрет повторения неопределённых mutations.

Приняты порт UI API `7430`, ранний `document.replace`, отложенный typed patch,
дискретный `output_observation` и Apply без дополнительного modal для обычной
уставки. Длительный ручной режим с видимым автопродлением — отдельное предложение
для обсуждения, не утверждённая реализация. Оставшиеся вопросы:
[open-questions.md](open-questions.md).

Следующие этапы описывают будущую работу по отдельному заданию. Сейчас не создавать
исходники и Cargo.toml.

## 1. Пустое штатное окно

Создать один Cargo package. Выбрать совместимые `eframe/egui/egui_plot`, закрепить
lockfile и MSRV. `eframe::run_native`, одна системная строка, стандартная панель,
обычный пустой Plot. Не включать persistence. Ввести отдельный Tokio IO-поток
и ограниченное штатное завершение.

Проверка: окно запускается/сворачивается/закрывается без зависания; `logic` получает
repaint при свёрнутом окне. Ни Runtime child process, ни GPU host helpers.
`CARGO_TARGET_DIR=D:\lab-workbench-sandbox\cargo-target`.

## 2. Минимальный внешний UI API и document.replace

Рано ввести TCP/NDJSON listener `127.0.0.1:7430`: hello/get/replace/status,
строгие DTO, стабильные IDs, instance ID и CAS revision. Runtime TCP `7420`
и WebSocket `7421` не занимать. Первые документы содержат график с signal binding;
пока Runtime не подключён, показывать unresolved, без выдуманных значений.
Документ устанавливается атомарно главным потоком и сохраняется только в RAM.

Проверка: Clojure через replace задаёт график/кривую, меняет подпись и отключается —
конфигурация остаётся. Проверить conflict, невалидный/oversized document, partial
NDJSON/CRLF и старый instance. API работает при свёрнутом окне. `document.patch`
не объявляется в hello.operations и возвращает `unsupported_operation`.
Нет отдельного этапа полного patch API перед живыми данными.

## 3. Первый вертикальный срез: Clojure → документ → Runtime → живой график

Собственные Application wire DTO, bounded framing, hello/capabilities, discovery
со всеми страницами и describe. Реализовать subscription от hello cursor,
D/M fences (P при необходимости), контрольный F, последовательные frozen families,
bounded staging и установку снимков. Добавить ObservedRuntime и состояния
Disconnected/Connecting/Synchronizing/Live, один сегментированный ограниченный
signal buffer. Температура виртуальной печи приходит непосредственно из Runtime.

Критерий готовности: Clojure подключается к `7430`, делает `document.replace`
из раздела 4.3 архитектуры; Workbench сам подключается к Runtime и показывает
живую температуру через `egui_plot`. Clojure не пересылает измерения; после его
отключения график продолжает обновляться. Пример не зависит от patch и управлений.

Проверки: partial/malformed Runtime frames, ответы вперемежку с events, гонка
D=100 → Reference event 101 → M=105; timeout/gap немедленно снимает Live.
Страницы разных frozen families не уничтожают друг друга. Штатные zoom/drag/follow
latest, конечные memory/queue budgets, никаких Runtime crates как dependencies.
После успешного Clojure transcript можно отдельно планировать typed patch,
но он не становится обязательным для дальнейших этапов.

## 4. Reconnect, время и границы данных

Backoff, новый connection epoch, сохранение cached plots, полный bootstrap после
reconnect, separate boot/generation/authority-epoch segments, stale indicators.
Добавить optional `measurement_window` для короткого начального заполнения.
Calendar axis и durable history пока не обязательны.

Проверка: kill/restore тестового socket; новый Runtime boot; unavailable sample;
Runtime ring и локальная очередь переполнены; линия никогда не пересекает разрыв.
Превышение memory budget обрезает старые данные с видимой пометкой.

## 5. Расширение документа: панели, индикаторы и drafts

Расширить уже работающий replace-документ до нескольких графиков, панелей,
индикаторов и editor widgets. Стандартный layout и общий CAS-путь для внешних
и локальных edits. Editors пока только меняют drafts. Никаких физических команд
из replace, redraw, drag или focus loss.

Проверка: Clojure создаёт два графика/панель, затем отключается — GUI остаётся.
Два клиента конфликтуют на base revision без частичных изменений.
Ошибочный документ, oversized frame и slow client не блокируют Runtime reader.
Старый instance ID после рестарта отвергается. API работает при свёрнутом окне.

## 6. Полная read-only печь

Добавить typed bindings Reference, controller, output, Recorder и connection.
Для power реализовать `output_observation` с readback timestamp и ясной подписью.
Это отдельные маркеры подтверждённых дискретных значений `output.readback.value/at`,
а не независимое непрерывное измерение физической мощности. Не соединять их
наклонной линией, не дорисовывать значения по таймеру и не интерполировать пропуски.
Recorder lifecycle events инвалидируют query, counters обновляются редкими queries.

Проверка: второй независимый Runtime-клиент меняет setpoint/PID/Recorder,
Workbench узнаёт об этом сам. Мощность не запрашивается через несуществующий signal
и не подменяется `controller.latest_output`. Новых readback нет — точек не добавляется.

## 7. Операторские действия и неопределённые исходы

Сначала Reference и controller lifecycle, затем PID configuration и Recorder.
Ввести actual/draft/pending, captured revision/run/boot,
один mutation sequencer и limited RAM evidence. До first write сохранить exact
request identity. Отдельная кнопка operation_status; автоматического resubmit нет.
Обычное изменение уставки в прежнем режиме — явный Apply с проверкой revision,
без дополнительного modal. Смена режима/темпа ramp, включение или настройка PID
и другие потенциально опасные действия требуют дополнительного подтверждения.
Pause/Safe — явные кнопки без задерживающего modal.

Проверка: redraw, drag и focus loss не отправляют команд. Чужое изменение revision
не перезаписывается старым draft. Один Apply/требуемое подтверждение даёт одну mutation.
Обрыв на каждой границе accepted/terminal даёт корректный unknown. После reconnect
актуальное значение не считается результатом потерянной команды.

## 8. Разовые ручные операции; длительный режим — только после отдельного решения

Добавить acquire/renew/write/safe, только по собственному scope и Runtime profile.
Показать owner, lease expiry, safe_pending/safe_confirmed и delivery evidence.
Разовые операции пригодны для коротких проверок; ручное нажатие renew каждые
2 секунды не считать пригодным UX длительного эксперимента.

Предложение длительного режима зафиксировано в разделе 6.4 архитектуры:
явный вход и согласие на видимое автопродление, новые плановые renew после
подтверждённого успеха, выход через явный Safe, остановка при stale/ошибке/unknown.
**Пока этот режим не реализовать.** Его подтверждение и отдельное задание —
условия добавления, а не препятствие для раннего вертикального среза.

Проверка: Clojure и Workbench имеют разные sessions; чужая lease блокирует
write/renew/safe, PID и manual authority конфликтуют по Runtime.
Safe completed ещё не означает safe_confirmed. Истёкшее подтверждение не вызывает
reacquire. После known admission долгого write явный safe не ждёт его terminal;
при unknown admission новый seq не угадывается.
Закрытие Workbench не посылает команды Runtime автоматически.
После отдельного утверждения длительного режима дополнительно проверить idle-only
renew, write дольше lease, неизвестный/поздний renew, отсутствие backlog/resubmit/
reacquire, удаление document-панели, сворачивание, сон и Safe при pending admission.

## 9. Приёмка первой полноценной версии

Выполнить двухклиентский сценарий из архитектуры на виртуальной печи:
температура, output readback, Reference, PID, Recorder, ошибки, disconnect/reconnect
и новый boot. Проверить выключенный Recorder и ограниченную RAM history.
Typed patch не является условием этой приёмки: документ можно полностью заменять.
Длительный manual mode не объявлять готовым без его отдельного утверждения,
реализации и проверок; разовые manual operations оценивать отдельно.
Прогнать protocol/model/action tests и короткий GUI smoke; `fmt`, `clippy`, тесты
применительно к созданным исходникам.

Документировать фактические wire examples UI API и Clojure transcript.
Никаких скрытых дополнительных listeners, Runtime dependencies или дискового
recovery state. Все технические артефакты и test DB внутри sandbox.

## Отложенные и необязательные этапы

- Typed `document.patch`: шесть сохранённых операций с общим CAS. Его можно взять
  отдельным этапом **после работающего вертикального среза этапа 3**, необязательно
  дожидаться всей версии; до этого не реализовать. Проверки transactional patch,
  ссылок и удаления элементов добавляются вместе с операцией, не раньше.
- Durable history через публичный API с явными gap/coverage, без чтения SQLite.
- Optional UTC axis на time_anchor.
- Min/max downsampling после измерений производительности.
- Более удобная layout-настройка на штатных egui widgets.

Для них нужен конкретный пользовательский сценарий. Multi-runtime, plugins,
внешний Runtime gateway и сложная recovery-система не являются автоматическим
продолжением этого плана. Для предложенного автопродления manual leases нужна
отдельная UX-договорённость из раздела 6.4, а не повторное ревью всей архитектуры.
