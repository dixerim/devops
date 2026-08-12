# Linux signal handling: пример `SIGTERM`

Сигнал в Linux лучше думать не как «мгновенный прыжок в handler», а как событие,
которое сначала становится pending, а потом доставляется конкретному `task_struct`
перед возвратом этого task в userspace.

```mermaid
flowchart TD
    A["SIGTERM generated"] --> B{"Как адресован сигнал?"}

    B -->|"kill(pid, SIGTERM)"| C["Process-directed signal"]
    C --> D["Кладётся в shared pending<br/>thread group / signal_struct"]
    D --> E["Kernel выбирает один подходящий task<br/>у которого SIGTERM не blocked"]

    B -->|"tgkill(tgid, tid, SIGTERM)<br/>pthread_kill(thread, SIGTERM)"| F["Thread-directed signal"]
    F --> G["Кладётся в private pending<br/>конкретного task_struct"]
    G --> H["Target task будет проверен<br/>перед возвратом в userspace"]

    E --> I["Delivery в конкретный task"]
    H --> I

    I --> J{"Disposition для SIGTERM<br/>в shared sighand_struct"}

    J -->|"handler установлен"| K["Kernel строит signal frame<br/>на stack выбранного task"]
    K --> L["Меняет userspace registers:<br/>следующая инструкция = handler"]
    L --> M["Task выполняет handler<br/>как обычный userspace-код"]
    M --> N{"Что сделал handler?"}
    N -->|"return"| O["Task продолжает выполнение<br/>с места прерывания"]
    N -->|"pthread_exit()"| P["Завершается текущий task/thread"]
    N -->|"exit() / exit_group()"| Q["Завершается вся thread group"]

    J -->|"default action = terminate"| R["Handler не запускается"]
    R --> S["Kernel инициирует group exit"]
    S --> T["Завершается вся thread group"]

    J -->|"ignored"| U["Signal отбрасывается<br/>или остаётся pending по правилам mask"]
```

## Короткая модель

```text
pending signal:
  - group-level: shared pending в signal_struct
  - task-level: private pending в конкретном task_struct

handler/disposition:
  - общий для thread group через sighand_struct

handler execution:
  - всегда в одном конкретном task
  - на stack этого task

default terminate/stop/continue/core:
  - действует на всю thread group
```

## Примеры

```text
kill(pid, SIGTERM), handler есть
→ signal pending на thread group
→ kernel выбирает один task
→ handler выполняется в этом task

kill(pid, SIGTERM), handler нет
→ default terminate
→ завершается вся thread group

pthread_kill(thread, SIGTERM), handler есть
→ signal pending на конкретный task
→ handler выполняется в этом task

pthread_kill(thread, SIGTERM), handler нет
→ default terminate
→ завершается вся thread group
```

Главная формула:

```text
pending бывает group-level или task-level;
handler/disposition общий для thread group;
handler выполняется в одном task;
fatal/job-control default action действует на всю thread group.
```
