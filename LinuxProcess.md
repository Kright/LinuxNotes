##Процессы Linux

Процесс - исполняемый экземпляр программы.
Процессы могут быть порождены другими процессами, породить ещё процессы,
умереть или быть убитыми. Когда процесс создаётся он почти идентичен своему
родителю: получает копию адресного пространства процесса-родителя на момент
создания ребёнка. У отца и ребёнка свои собственные данные(стек, куча,
структуры данных, код(возможно) - копия), потому изменение ребёнком своих
данных не видно родителю и наоборот.
Во время эволюции Unix систем появились пользовательские POSIX потоки -
pthread, которые совместно используют большое количество структур данных.
Однако первоначальная реализация потоков была не очень удачна, что связано
с тем что их планировкой занимались пользователи. Тогда были созданы
облегчённые процессы. По сути те же потоки, только их планирование
осуществляется ядром и, как следствие, их проще синхронизировать. Теперь я
каждым потоком ассоциирован облегчённый процесс.
В Linux есть такое понятие, как группа потоков - набор облегчённых потоков,
которые действуют, как единое целое по отношению к системам вызовам:
getpid(), kill(), _exit().

Каждый процесс(облегчённый тоже) описывается с помощью структуры struct
task_struct(task_t). Структура полностью описывает всё состояние, атрибуты
процесса и многое другое, и она ОЧЕНЬ большая, её описание можно найти в
файле include/linux/sched.h.

|task_struct|
|---|
|   thread_info                                                     |
|   state -1 unrunnable, 0 runnable, >0 stopped  |
|   \*stack                                                          |
|   usage                                                           |
|   flags                                                           |
|   sched_info sched_info                                           |
|   mm_struct *mm   /* Указатели на дескрипторы областей памяти*/   |
|   mm_struct *active_mm                                            |
|   pid_t pid       /* Идентификатор процесса*/                     |
|   pid_t tgid      /* Идентификатор группы процесса*/              |
|   *parent         /* recipient of SIGCHLD, wait4() reports */     |
|   *real_parent    /* real parent process */                       |
|   list_head children  /* list of my children */                   |
|   fs_struct *fs   /* filesystem information */                    |
|   files_struct *files /* open file information */                 |
|   /* signal handlers */                                           |
|   list_head sibling;  /* linkage in my parent's children list */  |
| struct task_struct, tasks /* Общий список процессов*/ | 
|……………|

(include/linux/sched.h)

Процесс может находится в различных состояниях во время своего
существования. Например:
```c
#define TASK_RUNNING    0 // либо выполняется, либо готов в любой момент
```
выполняться и ждёт своего времени.
```
#define TASK_INTERRUPTIBLE  1 //процесс приостановлен, пока не произойдёт
```
определённое событие, которое должно его разбудить.
```
#define TASK_UNINTERRUPTIBLE   2 //похож на предыдущий, но приход сигнала
```

не пробуждает процесс, процесс в этом состоянии прервать нельзя.
Используется редко, когда ожидаемое событие долго ждать, но оно
гарантировано конечно. Например чтение файлов устройств(IO), времени,
взятие mutex.
```
#define __TASK_STOPPED      4 // переходит в это состояние после получения
```
сигнала SIGSTOP и похожих.
```
#define __TASK_TRACED 8 // выполнение приостановлено отладчиком ptrace().
```
У процесса так же есть - int exit_state - содержит состояние после
завершения выполнения.
```
#define EXIT_ZOMBIE 32 // процесс завершил своё выполнение, но его родитель
```
ещё не прочитал его состояние с помощью wait4(), waitpid().
```
#define EXIT_DEAD 16 // процесс переходит сюда после того, как кто-нибудь
```
выполнит к нему wait4, waitpid().

Схема состояний и переходов между ними процесса; о состоянии процесса можно
узнать с помощью утилиты ps:
```
_____                     _______
| R | <—————————————————> | r/q | <—————————————————————————\
————— ———\______________  ———————       /\                   \
  | exit/kill           \_____________|________________      |
  \/                  read \/         |      IO/mutex |      |
_____                   ________ _____|               |      |
| Z | <————————kill()———|  W/S | interruptible        \/     |
—————       /\          ————————                    _____ ___/
  | wait()   | ———————————————— NO_WAY ———————————— | D | uninterruptible
  \/                                                —————
________
| dead |
-———————
```

Максимальное количество процессов обычно 2^15 = 32768. По умолчанию можно
получить из макроса #define PID_MAX_DEFAULT (CONFIG_BASE_SMALL ? 0x1000 :
0x8000) (include/linux/threads.h). Так же количество доступных процессов
может быть изменено системным администратором в /proc/sys/kernel/pid_max/.
На 64 разрядных машинах максимальное значение можно увеличить до 4 * 1024 *
1024.
Для повторного использования pid’ов ядро поддерживает pidmap(include/linux/
pid_namespace.h)
```
struct pidmap {
       atomic_t nr_free;
       void *page;
};
```

task_struct существует для каждого процесса, даже облегчённого, pid у них у
всех разный. Однако с потоками надо работать, как с единой сущностью,
например, посылая сигнал. Потому у каждого потока в группе есть общий
идентификатор - pid лидера группы и хранится он в поле tgid (thread group
id), и системный вызов getpid() возвращает именно его.

Для каждого процесса ядро хранила struct thread_info, содержащую указатель
на task_struct, и стэк режима ядра. thread_info - платформозависимая
структура, потому искать её в <asm/thread_info.h>. Для x86 для ядра 2.6
выглядела вот так:
```
struct thread_info {
        struct task_struct          *task;
        struct exec_domain      *exec_domain;
        unsigned long               flags;
        unsigned long               status;
        __u32                           cpu;
        __s32                           preempt_count;
        mm_segment_t            addr_limit;
        struct restart_block    restart_block;
        unsigned long               previous_esp;
        __u8                            supervisor_stack[0];
};
```

Раньше размер стэка ядерного режима был 4Кб и task_struct хранился в нём.
```
LOW                             HIGH
_________________________________
|             |        |        |
| task_struct |  <———— | stack  |
|             |        |        |
—————————————————————————————————
```
Для 32битной системы sizeof(task_struct) ~ 1.7 Kb, что было довольно
критично, поэтому ввели thread_info, который указывает на task_struct,
также увеличили стэк ядерного режима до 8Kb.
```
LOW                                                   HIGH
__________________________________________________________
|             |                                 |        |
| thread_info |            <———————————————————-| stack  |
|             |                                 |        |
——————————————————————————————————————————————————————————
```
Однако в процессе эволюции thread_info переместилось в окончательно в
task_struct, а на структуру указывает регистр gs.
```
LOW                                                   HIGH   ______
__________________________________________________________   | gs |
|                                               |        |   ——————
|                          <———————————————————-| stack  |      \-> current
|                                               |        |
——————————————————————————————————————————————————————————
```
Текущий процесс, запущенный на процессоре можно получить с помощью макроса
current() <asm/current.h>:
```
#define current get_current()
static __always_inline struct task_struct *get_current(void)
{
    return this_cpu_read_stable(current_task);
}
#define this_cpu_read_stable(var)   percpu_stable_op("mov", var)
```

Так же надо заметить, что thread_info должна быть самой первой в
task_struct, чтобы имея адрес одной в регистре fs, иметь адрес двух
структур сразу, что можно заметить в макросе:
```
#define current_thread_info() ((struct thread_info *)current)
```
(include/linux/thread_info.h)

Списки процессов:
Для дальнейшего понимания происходящего необходимо разобраться со списками
Linux и технологией RCU. Вы сможете это сделать прочитав, например,
соседние файлы: Linux Lists.txt и Linux Synchronization.txt

Все процессы выстроены в двунаправленный список, для этого каждая структура
task_struct включает в себя поле — struct list_head tasks. В голове списка
находится дескриптор init_task с pid = 0.
Пройтись по списку процессов можно с помощью макроса for_each_process(p)
(include/linux/sched.h):
```c
#define next_task(p) \
    list_entry_rcu((p)->tasks.next, struct task_struct, tasks)

#define for_each_process(p) \
    for (p = &init_task ; (p = next_task(p)) != &init_task ; )

(include/linux/rculist.h)
/**
 * list_entry_rcu - get the struct for this entry
 * @ptr:        the &struct list_head pointer.
 * @type:       the type of the struct this is embedded in.
 * @member:     the name of the list_head within the struct.
 *
 * This primitive may safely run concurrently with the _rcu list-mutation
 * primitives such as list_add_rcu() as long as it's guarded by rcu_read_lock().
 */
#define list_entry_rcu(ptr, type, member) \
    container_of(lockless_dereference(ptr), type, member)
```

При создании процесса, новый добавляется в конец списка процессов
(kernel/fork.c —> copy_procces):
```
list_add_tail_rcu(&p->tasks, &init_task.tasks);
```
Изучив структуру task_struct, мы можем отыскать ещё списки в её составе:
```
    struct list_head children;  /* list of my children */
    struct list_head sibling;   /* linkage in my parent's children list */
```

Данные списки описывают какие отношения связывают наш процесс с другими
процессами. Список — children содержит детей нашего процесса, а sibling —
братьев — других процессов созданных нашим родителем.

Процесс 0 — предок всех процессов. Его описывает статическая структура:
struct task_struct init_task = INIT_TASK(init_task); (/init/init_task.c и
макрос в/linux/init_task.h)

Структура инициализируется при старте функцией — start_kernel(void)
(/init/main.c). После чего процесс 0 инициализирует подсистемы, создаёт
процесс pid = 1 — int:
Функция kernel_thread(kernel_init, NULL, CLONE_FS) - создаст клона, который
будет исполнять функцию kernel_init. После этого процесс 0 вызовет
cpu_idle() и будет работать в холостую, просыпаясь только если другие
процессы не могут быть исполнены. Созданный поток ядра завершает
инициализацию ядра, а после загружает исполняемую программу init.

Управление процессами.
Как мы видели раньше у каждого процесса может быть группа потоков
исполнения. У каждого потока свой собственный pid, но все потоки должны
реагировать на событие, как одна сущность потому есть tgid = pid лидера
группы потоков.
Различные процессы объединяются в группы процессов, для того чтобы
взаимодействовать в ними, как с единой сущностью есть pgid = pig процесса
лидера. В группу процессов входят не только родственники, но процессы
объединённые с помощью конвейера process | process …
Группы процессов в свою очередь объединяются в сессию, серии связаны с
терминалом. И, как можно догадаться, sid = pid лидера сессии.
```
            _________session________
           /            |           \
 process groupe(pg)     pg          pg
    /       |      \
process(p)  p       p
          / | \
 thread(t)  t  t
```
Для удобства и скорости управления процессами на этапе инициализации ядра
создаётся 4 хэш-таблицы: для каждого потока (различные pid), для каждого
лидера группы(tgid = pid), для каждого лидера группы процессов(pgid = pid),
для каждого лидера сессии(sid = pid).
Хэш таблица эффективнее массива работает с памятью, так как количество
процессов в системе зачастую меньше чем их максимальное количество.
```
pid_hash(hlist_head)
__________
|        |    pid                    pid
——————————    ______________         ______________
|        | —> |            |    / —> |            |
——————————    ——————————————   /     ——————————————
|        |    | hlist_node | —/      | hlist_node |
| …      |    ——————————————         ——————————————
——————————    | hlist_head | ———\
              ——————————————     \————> список task_struct tgid = nr
                                  \ ————> список task_struct pgid = nr
                                   \ ————> список task_struct sid = nr
```
(/include/linux/pid.h)

```c
enum pid_type
{
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
};

struct upid {
    /* Try to keep pid_chain in the same cacheline as nr for find_vpid */
    int nr;
    struct pid_namespace *ns;
    struct hlist_node pid_chain;
};

struct pid
{
    atomic_t count;                 // счётчик ссылок на структуру
    unsigned int level;
    /* lists of tasks that use this pid */
    struct hlist_head tasks[PIDTYPE_MAX];
    struct rcu_head rcu;
    struct upid numbers[1];
};
```