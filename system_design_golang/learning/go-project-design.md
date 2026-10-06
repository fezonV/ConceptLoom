# Проектирование проектов на Go

_Created by Concept Loom on 2026-10-05T09:25:55.494Z_

> [!note] Начальная диагностика
>
> Цель: самостоятельно проектировать структуры, функции и пакеты Go. Различает состояние Task и действие над ним. Верно разместил общую проверку названия в общей функции: «shared, так как вся бизнес логика у нас в service». Знаком с термином service; понимание зависимостей пакетов пока проверяется.

> [!note] Предложенный маршрут
>
> Выявленный пробел: связь логики с хранилищем через интерфейс. Маршрут ожидает согласования.
> 
> 1. Сценарии пользователя → требования: описать создание и завершение задачи; опора — различение данных и действий.
> 
> 2. Требования → данные и правила: вывести поля Task, допустимые состояния и ошибки; зависит от шага 1.
> 
> 3. Данные и правила → операции: определить функции, параметры и результаты; зависит от шагов 1–2.
> 
> 4. Операции → обязанности: разделить ввод, правила и хранение; зависит от шага 3 и понимания общих правил.
> 
> 5. Обязанности → интерфейс хранения: связать логику с реализацией через минимальный набор методов; зависит от шагов 3–4.
> 
> 6. Обязанности и зависимости → пакеты, папки и main: разместить код, проверить отсутствие циклов, собрать приложение; зависит от шагов 4–5.

> [!note] Интерфейс: первый шаг
>
> По выбору ученика начинаем с простого примера интерфейса Sender { Send(string) error }. Конкретный тип Console имеет метод Send(string) error. Соответствие неявное: требуются методы с совпадающими именами и типами параметров и результатов. Имена параметров не важны. Проверка: соответствует ли Email с Send(string) (int, error).

> [!note] Вызов через интерфейс
>
> Закреплено совпадение сигнатур методов. Следующая связь: Notify(sender Sender) вызывает sender.Send("Привет!"). При передаче Console{} выполняется Console.Send; при передаче LogSender{} — LogSender.Send. Notify работает с требованием Send(string) error, а конкретное поведение определяется переданным значением. Ожидается ответ на checkpoint о Notify(LogSender{}).

> [!note] Конкретное поведение и общая логика
>
> Checkpoint LogSender — accurate. Ученик понимает, что вызов через интерфейс выполняет метод переданного значения. Польза для проектирования: Notify определяет содержание сообщения, конкретный Sender — способ отправки, вызывающий код — выбор реализации. Следующий checkpoint: подключение EmailSender с совместимым Send.

> [!note] Простой пример интерфейса закреплён
>
> Ответ inject — accurate. Закреплены: совпадение сигнатур методов; неявное соответствие интерфейсу; вызов метода конкретного переданного значения; выбор реализации вызывающим кодом без изменения Notify. Направления импортов и перенос на Store ещё не закреплены. Следующий шаг выбирает ученик: хранение задач или самостоятельный пример интерфейса.

> [!note] Перенос интерфейса на хранение задач
>
> Ученик согласовал переход к Store и зависимостям пакетов. Пример: Task { Title string; Done bool }; Store { Save(Task) error }; CreateTask(store Store, title string) проверяет title == "", возвращает ошибку либо вызывает store.Save(Task{Title: title}). Соответствие Sender → Store, Send → Save, Notify → CreateTask. Store содержит требование к методу, а конкретное хранилище реализует сохранение. Ожидается checkpoint о размещении SQL INSERT.

> [!note] Конкретная реализация MemoryStore
>
> MemoryStore { Tasks []Task }; метод (s *MemoryStore) Save(task Task) error добавляет задачу через s.Tasks = append(s.Tasks, task) и возвращает nil. Передаём &MemoryStore{} в CreateTask: метод определён на *MemoryStore, указатель позволяет менять поле исходного экземпляра. Хранилище живёт в памяти процесса. Checkpoint: два последовательных вызова CreateTask с непустым и пустым названием; проследить, где Save вызывается, а где нет.

> [!note] Импорты и сборка приложения
>
> Пример размещения: task/task.go — Task, Store, CreateTask; memory/store.go — MemoryStore; main.go — создание экземпляра и вызов CreateTask.
> 
> ```text
> main ---> memory ---> task
>   \--------------------^
> 
> Стрелка A ---> B означает: пакет A импортирует пакет B.
> ```
> 
> Store лежит рядом с использующей его логикой. memory импортирует task для типа task.Task. main импортирует оба и передаёт &memory.MemoryStore{} в task.CreateTask. Стрелки импортов отличаются от последовательности вызовов: во время работы CreateTask вызывает MemoryStore.Save через интерфейс.

> [!note] Почему Store находится в пакете task
>
> Ученик верно выбрал direction. Закреплено: Store — контракт требуемой способности Save(Task) error; лежит рядом с потребителем CreateTask в пакете task, потому что тот использует этот контракт. task.go и store.go — файлы, а граница импорта — папка-пакет. Реализация postgres использует task.Task, поэтому postgres импортирует task; task вызывает Save через интерфейс и не импортирует postgres. Следующий шаг: спроектировать структуру пакетов по памяти для PostgreSQL или перейти к полному маршруту от требований.

> [!question] Самостоятельно восстановить схему пакетов
>
> Learner requested to reconstruct the task/Postgres/main package arrangement independently. Checkpoint checks responsibilities and import direction.

> [!note] Маршрут: от требований к проекту
>
> Checkpoint восстановления пакетов выбран верно целиком: task, postgres, main и направления зависимостей. Предлагаемый маршрут основной цели:
> 
> 1. Сценарии пользователя → требования. Для списка задач описать создание, завершение и просмотр; опираемся на уже понятный пример CreateTask.
> 
> 2. Требования → данные и правила. Вывести поля Task, допустимые значения и ошибки; зависит от шага 1.
> 
> 3. Данные и правила → операции. Сформулировать функции и методы, которые нужны сценариям, с параметрами и результатами; зависит от шагов 1–2.
> 
> 4. Операции → обязанности и зависимости. Разделить ввод, правила и хранение, не создавая циклов; зависит от шага 3 и закреплённого понимания Store.
> 
> 5. Обязанности и зависимости → пакеты и папки. Сгруппировать код по связным обязанностям, решить, где полезны отдельные пакеты, а какие файлы остаются вместе; зависит от шага 4.
> 
> 6. Собрать приложение в main и проверить проект на новом сценарии. main соединяет конкретные реализации; ученик объясняет выбор структур, функций и расположения.

> [!question] Сценарий создания задачи
>
> Сценарий: пользователь вводит название новой задачи. Пустое название нужно отклонить; непустое — принять и сохранить. Первый шаг — отделить наблюдаемое требование от возможных деталей структуры, СУБД и каталогов.

> [!question] Вывод минимальных данных
>
> После фиксации поведения выводим минимальные данные: непустое название должно быть принято и сохранено, значит его значение нужно представить в модели. ID, время и владелец сценарием пока не заданы.

> [!note] Из сценария к минимальным данным
>
> Ученик верно вывел, что сохранённое значение — title. Минимальная модель из сценария: type Task struct { Title string }. ID, CreatedAt и Owner не добавляются, пока для них нет требований. Следующий шаг — вывести порядок проверки пустого названия до Store.Save.

> [!note] От сценария к сигнатуре CreateTask
>
> Минимальная модель: type Task struct { Title string }. Операция следует из сценария: CreateTask(store Store, title string) error. title — пользовательский ввод; store — зависимость, необходимая для сохранения; error — сценарий может быть отклонён или не сохраниться. Алгоритм: отвергнуть пустой title, иначе собрать Task{Title:title}, вызвать store.Save(task). Проверить, где располагается общая проверка при разных реализациях Store.

> [!note] Отдельный пакет service
>
> В предыдущем компактном примере Task, Store и CreateTask были в пакете task. Это допустимый небольшой вариант. Чтобы отразить роль service и правило создания, используем task.Task в пакете task; service содержит интерфейс TaskStore и CreateTask; postgres содержит PostgresStore.Save(task.Task), импортирует task и не импортирует service; main импортирует service и postgres и соединяет их. Граф импортов:
> 
> ```text
> main ──> service ──> task
>   └──> postgres ──> task
> ```
> 
> Следующая проверка — какие пакеты используют task.Task.

> [!note] HTTP, use case, хранилище
>
> Checkpoint: learner correctly assigned HTTP request parsing to the handler, the empty-title rule to task creation logic, and SQL INSERT to the Postgres Store implementation. Reason: transport format, application rule, and persistence mechanism are different responsibilities.

> [!note] Зависимости пакетов
>
> ```text
> cmd/app (main) ──соединяет──▶ httpapi / service / postgres
> httpapi ──передаёт title──▶ service.CreateTask
> service ──использует──▶ task.Task + интерфейс TaskStore
> postgres ──реализует TaskStore, работает с──▶ task.Task
> ```

> [!note] Интерфейс рядом с потребителем
>
> Checkpoint: learner correctly chose consumerneed. service declares the smallest TaskStore behavior it needs; postgres satisfies the interface structurally by matching method names and signatures; main connects the concrete PostgresStore to the service.

> [!note] Пакеты и зависимости
>
> ```text
> cmd/app (main)
>   ├── соединяет httpapi, service и postgres
> 
> httpapi ──▶ service
> service ──▶ task
> postgres ──▶ task
> 
> service: TaskStore interface + CreateTask
> postgres: SQL implementation of TaskStore
> task: Task model
> ```

> [!success] Различать допустимую схему и цикл
>
> Learner selected the clean composition as a likely valid graph and noted that the bothdomain option could also work even though it omits main. Clarify that bothdomain and clean are both acyclic; only reciprocal imports create a cycle. The earlier prompt asked which graph creates a cycle.

> [!success] Цикл импортов и валидные графы
>
> Learner clarified that their previous choice targeted a valid package arrangement, not a cycle; they said they would have selected reciprocal had they focused on the question. Fresh checkpoint correctly identified service → postgres → service as a cycle. Both clean and bothdomain are acyclic.

> [!note] Реализация интерфейса без импорта потребителя
>
> Checkpoint: learner correctly chose that postgres imports task for task.Task; it does not need to import service solely to implement service.TaskStore because Go checks method compatibility structurally.

> [!note] TaskStore и SQL-драйвер
>
> Checkpoint: learner correctly distinguished the application-level TaskStore contract from the lower-level SQL API/driver. The postgres adapter implements TaskStore using the driver; main selects and injects the concrete adapter.
