# Часть VII. Concurrent Rust

## Глава 23. Потоки

До этого момента все наши программы выполнялись в одном потоке. Это означало, что код выполнялся последовательно: одна инструкция за другой. Но современные компьютеры имеют несколько ядер, и для эффективного использования их мощности нам нужно выполнять код **параллельно**.

Rust предоставляет безопасную модель работы с потоками, основанную на тех же принципах владения и заимствования, которые мы изучали ранее. Благодаря этому Rust предотвращает гонки данных (data races) на уровне компиляции.

В этой главе мы научимся создавать потоки, передавать данные между ними и использовать параллелизм для ускорения программ.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 23.1. Что такое поток?

**Поток (thread)** — это независимая последовательность выполнения инструкций внутри процесса.

До сих пор наши программы выполнялись последовательно в одном потоке. Теперь мы можем создать дополнительный поток и позволить ему выполнять работу одновременно с основным потоком.

Например:

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        println!("Hello from the new thread!");
    });

    println!("Hello from the main thread!");

    handle.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Hello%20from%20the%20new%20thread%21%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20println%21%28%22Hello%20from%20the%20main%20thread%21%22%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

Порядок первых двух сообщений **не гарантирован**. Новый поток и основной поток могут выполняться в разном порядке.

Однако последняя строка:

```rust
handle.join().unwrap();
```

гарантирует, что основной поток дождётся завершения созданного потока.

Это важное отличие от использования `sleep()`:

```rust
thread::sleep(...);
```

Задержка лишь предполагает, что другой поток успеет что-то сделать. Она не является механизмом синхронизации.

У `thread::spawn` есть важные ограничения: передаваемое замыкание должно реализовывать `Send + 'static`, а его результат — `Send + 'static`. Это связано с тем, что обычный поток может продолжать существовать независимо от места, где он был создан. ([Rust Documentation][1])

---

## 23.2. `JoinHandle` — управление созданным потоком

Функция `thread::spawn` возвращает значение типа:

```rust
JoinHandle<T>
```

где `T` — тип значения, возвращаемого потоком.

Полученный объект можно использовать для ожидания завершения потока:

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        println!("Thread is running...");

        for number in 1..=5 {
            println!("Thread: {number}");
        }

        42
    });

    println!("Main thread continues...");

    match handle.join() {
        Ok(result) => println!("Thread returned: {result}"),
        Err(_) => println!("Thread panicked"),
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Thread%20is%20running...%22%29%3B%0A%0A%20%20%20%20%20%20%20%20for%20number%20in%201..%3D5%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22Thread%3A%20%7Bnumber%7D%22%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%2042%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20println%21%28%22Main%20thread%20continues...%22%29%3B%0A%0A%20%20%20%20match%20handle.join%28%29%20%7B%0A%20%20%20%20%20%20%20%20Ok%28result%29%20%3D%3E%20println%21%28%22Thread%20returned%3A%20%7Bresult%7D%22%29%2C%0A%20%20%20%20%20%20%20%20Err%28_%29%20%3D%3E%20println%21%28%22Thread%20panicked%22%29%2C%0A%20%20%20%20%7D%0A%7D)

Метод:

```rust
handle.join()
```

возвращает:

```rust
Result<T>
```

Если поток завершился нормально, мы получаем:

```rust
Ok(value)
```

Если поток завершился с паникой:

```rust
Err(...)
```

Поэтому запись:

```rust
handle.join().unwrap();
```

означает: «дождаться потока и получить результат; если поток запаниковал — запаниковать и здесь».

`JoinHandle` также является механизмом владения правом присоединиться к потоку. Если `JoinHandle` уничтожить, поток становится **detached**: у программы больше нет этого handle, чтобы вызвать `join()`. ([Rust Documentation][2])

---

## 23.3. Передача данных в поток с `move`

До этого мы изучали `move` в контексте closures. Теперь эта возможность становится особенно важной.

Рассмотрим:

```rust
use std::thread;

fn main() {
    let message = String::from("Hello from main!");

    let handle = thread::spawn(move || {
        println!("{message}");
    });

    handle.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20message%20%3D%20String%3A%3Afrom%28%22Hello%20from%20main%21%22%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bmessage%7D%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

Ключевое слово:

```rust
move
```

говорит closure захватить используемые переменные **по значению**, а не по ссылке.

В данном случае `String` перемещается во владение closure, а затем closure передаётся новому потоку.

После этого исходная переменная больше не может использоваться:

```rust
// println!("{message}"); // ошибка: value moved
```

Почему это важно?

Поток, созданный через обычный `thread::spawn`, может продолжить работу после того, как функция, создавшая его, завершится. Поэтому Rust не разрешает такому потоку хранить обычную ссылку на локальную переменную.

Именно поэтому `thread::spawn` требует от closure `'static`.

Важно понимать: **`move` и `'static` — не одно и то же**.

`move` определяет способ захвата переменных closure.

`'static` означает, что захваченные данные не должны зависеть от локального времени жизни, которое может закончиться раньше потока. ([Rust Documentation][1])

---

## 23.4. Возврат результата из потока

`JoinHandle` возвращает результат, который можно получить через `join()`:

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        // Имитация вычислений
        let result = 42 * 2;
        result // возвращаем значение
    });

    let result = handle.join().unwrap();
    println!("Result from thread: {}", result);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20let%20result%20%3D%2042%20*%202%3B%0A%20%20%20%20%20%20%20%20result%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20result%20%3D%20handle.join%28%29.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22Result%20from%20thread%3A%20%7B%7D%22%2C%20result%29%3B%0A%7D)

**Вывод:** `Result from thread: 84`

---

## 23.5. Владение и потоки

При передаче данных в поток действуют те же правила владения:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5];

    let handle = thread::spawn(move || {
        println!("Data in thread: {:?}", data);
        // data перемещён в поток
    });

    // println!("{:?}", data); // ❌ Ошибка! data больше не принадлежит main

    handle.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20data%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Data%20in%20thread%3A%20%7B%3A%3F%7D%22%2C%20data%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

---

## 23.6. Время жизни потока

Обычный:

```rust
thread::spawn(...)
```

создаёт поток, который не привязан к области видимости вызывающей функции.

Например:

```rust
use std::thread;
use std::time::Duration;

fn start_worker() {
    thread::spawn(|| {
        thread::sleep(Duration::from_millis(100));
        println!("Worker finished");
    });
}

fn main() {
    start_worker();

    println!("Main continues...");
    thread::sleep(Duration::from_millis(200));
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0Ause%20std%3A%3Atime%3A%3ADuration%3B%0A%0Afn%20start_worker%28%29%20%7B%0A%20%20%20%20thread%3A%3Aspawn%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20thread%3A%3Asleep%28Duration%3A%3Afrom_millis%28100%29%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22Worker%20finished%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20start_worker%28%29%3B%0A%0A%20%20%20%20println%21%28%22Main%20continues...%22%29%3B%0A%20%20%20%20thread%3A%3Asleep%28Duration%3A%3Afrom_millis%28200%29%29%3B%0A%7D)

Здесь `start_worker()` завершает работу, но созданный поток продолжает существовать.

Это и есть причина требования `'static` для `thread::spawn`.

Если `JoinHandle` не нужен, его можно отбросить:

```rust
thread::spawn(|| {
    // работа
});
```

В этом случае поток становится detached.

Если же программа завершается раньше этого потока, процесс завершится вместе с основным потоком, поэтому нельзя использовать такой подход как способ гарантировать выполнение фоновой работы до конца. Для управляемого завершения используйте `join()`.

---

## 23.7. Scoped threads — потоки с ограниченным временем жизни

Иногда нам необходимо обратное.

Мы хотим создать поток, который использует **обычные локальные данные**, но не хотим передавать эти данные во владение потоку.

Для этого используется:

```rust
thread::scope(...)
```

Например:

```rust
use std::thread;

fn main() {
    let data = vec![10, 20, 30, 40];

    thread::scope(|scope| {
        scope.spawn(|| {
            println!("First element: {}", data[0]);
        });

        scope.spawn(|| {
            println!("Last element: {}", data[data.len() - 1]);
        });

        println!("Main thread is inside the scope");
    });

    println!("All scoped threads have finished");
    println!("Data is still available: {data:?}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+vec%21%5B10%2C+20%2C+30%2C+40%5D%3B%0A%0A++++thread%3A%3Ascope%28%7Cscope%7C+%7B%0A++++++++scope.spawn%28%7C%7C+%7B%0A++++++++++++println%21%28%22First+element%3A+%7B%7D%22%2C+data%5B0%5D%29%3B%0A++++++++%7D%29%3B%0A%0A++++++++scope.spawn%28%7C%7C+%7B%0A++++++++++++println%21%28%22Last+element%3A+%7B%7D%22%2C+data%5Bdata.len%28%29+-+1%5D%29%3B%0A++++++++%7D%29%3B%0A%0A++++++++println%21%28%22Main+thread+is+inside+the+scope%22%29%3B%0A++++%7D%29%3B%0A%0A++++println%21%28%22All+scoped+threads+have+finished%22%29%3B%0A++++println%21%28%22Data+is+still+available%3A+%7Bdata%3A%3F%7D%22%29%3B%0A%7D)

Здесь потоки **заимствуют** `data`.

Нам не требуется:

```rust
move
```

и не требуется передавать `data` во владение потокам.

Почему это безопасно?

Потому что `thread::scope` гарантирует, что все созданные внутри него потоки будут завершены до выхода из `scope`. Поэтому ссылки на локальные данные не могут пережить эти данные. ([Rust Documentation][3])

Это очень важная идея:

```text
thread::spawn
    ↓
поток может жить независимо
    ↓
нужен 'static

thread::scope
    ↓
потоки ограничены scope
    ↓
можно заимствовать локальные данные
```

---

## 23.8. Параллельная обработка коллекции

Теперь можно соединить несколько изученных нами возможностей:

- коллекции;
- итераторы;
- closures;
- ownership;
- потоки;
- `join()`.

Например, разделим массив чисел на части и обработаем каждую часть отдельным потоком:

```rust
use std::thread;

fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8];

    let results: Vec<_> = thread::scope(|scope| {
        let handles: Vec<_> = numbers
            .chunks(2)
            .map(|chunk| {
                scope.spawn(move || {
                    chunk.iter().map(|number| number * number).sum::<i32>()
                })
            })
            .collect();

        handles
            .into_iter()
            .map(|handle| handle.join().unwrap())
            .collect()
    });

    println!("Results: {results:?}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+numbers+%3D+vec%21%5B1%2C+2%2C+3%2C+4%2C+5%2C+6%2C+7%2C+8%5D%3B%0A%0A++++let+results%3A+Vec%3C_%3E+%3D+thread%3A%3Ascope%28%7Cscope%7C+%7B%0A++++++++let+handles%3A+Vec%3C_%3E+%3D+numbers%0A++++++++++++.chunks%282%29%0A++++++++++++.map%28%7Cchunk%7C+%7B%0A++++++++++++++++scope.spawn%28move+%7C%7C+%7B%0A++++++++++++++++++++chunk.iter%28%29.map%28%7Cnumber%7C+number+*+number%29.sum%3A%3A%3Ci32%3E%28%29%0A++++++++++++++++%7D%29%0A++++++++++++%7D%29%0A++++++++++++.collect%28%29%3B%0A%0A++++++++handles%0A++++++++++++.into_iter%28%29%0A++++++++++++.map%28%7Chandle%7C+handle.join%28%29.unwrap%28%29%29%0A++++++++++++.collect%28%29%0A++++%7D%29%3B%0A%0A++++println%21%28%22Results%3A+%7Bresults%3A%3F%7D%22%29%3B%0A%7D)

Здесь происходит несколько важных вещей.

Сначала:

```rust
numbers.chunks(2)
```

создаёт итератор по частям исходного вектора.

Затем для каждой части создаётся поток:

```rust
scope.spawn(move || {
    ...
})
```

Замыкание получает владение **ссылкой на chunk**, а не самим исходным `Vec`.

Поскольку используется `thread::scope`, эта ссылка может быть обычной ссылкой на локальный массив.

Каждый поток возвращает результат:

```rust
sum::<i32>()
```

а `join()` собирает эти результаты обратно в основной поток.

Таким образом, ownership и lifetime не являются препятствиями для параллелизма. Наоборот, именно они позволяют компилятору проверить, что разные потоки работают с допустимыми данными.

---

## 23.9. Конкурентность и параллелизм

Два понятия часто используются вместе, но означают разные вещи.

**Конкурентность (concurrency)** означает, что несколько независимых задач могут продвигаться вперёд в рамках одной программы.

**Параллелизм (parallelism)** означает, что несколько задач действительно выполняются одновременно — например, на разных ядрах процессора.

Упрощённо:

```text
Concurrency:

Task A ─────┐     ┌─────
            └─────┘
Task B       ┌─────┐
        ─────┘     └─────


Parallelism:

Task A ─────────────────
Task B ─────────────────
        одновременно
```

Конкурентная программа не обязательно выполняется одновременно на нескольких ядрах.

Например, операционная система может быстро переключаться между задачами:

```text
A → B → A → B → A → B
```

Параллельная программа может выполнять:

```text
CPU 1: A A A A
CPU 2: B B B B
```

Потоки Rust позволяют строить программы, использующие оба подхода.

При этом важно помнить: создание большого количества потоков не означает автоматического ускорения программы. Потоки имеют стоимость создания, планирования и синхронизации. Для большого количества коротких задач обычно применяются другие модели — например, thread pools, task runtimes или async.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Почему обычная ссылка не работает с `thread::spawn`

```rust
use std::thread;

fn main() {
    let message = String::from("Hello");

    let handle = thread::spawn(|| {
        println!("{message}"); // ошибка: message не захвачен по значению
    });

    handle.join().unwrap();
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20message%20%3D%20String%3A%3Afrom%28%22Hello%22%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bmessage%7D%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

Компилятор сообщит: `closure may outlive the current function`. Дело в том, что без `move` замыкание по умолчанию пытается захватить `message` по ссылке — но `thread::spawn` требует, чтобы переданное замыкание было `'static`, поскольку созданный поток может продолжать работать после того, как функция `main`, создавшая его, теоретически могла бы уже завершиться (в данном случае мы вызываем `join()`, но компилятор рассуждает в общем случае, а не для конкретного порядка вызовов в этой программе).

Исправление:

```rust
use std::thread;

fn main() {
    let message = String::from("Hello");

    let handle = thread::spawn(move || {
        println!("{message}");
    });

    handle.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20message%20%3D%20String%3A%3Afrom%28%22Hello%22%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bmessage%7D%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

Теперь `message` перемещается во владение замыкания через `move`, и замыкание больше не зависит от времени жизни переменной `message` в `main` — оно само владеет данными, поэтому удовлетворяет требованию `'static`.

---

### Эксперимент 2: Почему `Rc<T>` нельзя передать потоку

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);

    let handle = thread::spawn(move || {
        println!("{data:?}");
    });

    handle.join().unwrap();
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Arc%3A%3ARc%3B%0Ause%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20data%20%3D%20Rc%3A%3Anew%28vec%21%5B1%2C%202%2C%203%5D%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bdata%3A%3F%7D%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

В отличие от эксперимента 1, здесь проблема не в `move` — `move` присутствует и честно передаёт владение `Rc` замыканию. Дело в самом типе: `Rc<T>` **не реализует `Send`**, потому что его счётчик ссылок не синхронизирован — если бы клоны `Rc` разошлись по разным потокам, увеличение и уменьшение счётчика могло бы происходить одновременно из разных потоков без защиты, что привело бы к гонке данных и некорректному подсчёту ссылок. Компилятор не разрешает передавать такой тип между потоками в принципе, независимо от `move`.

Для многопоточного разделяемого владения используется атомарный аналог:

```rust
Arc<T>
```

`Arc<T>` (глава 21) использует атомарные операции для изменения счётчика ссылок и поэтому реализует `Send` (при условии, что `T: Send + Sync`). Мы подробно разберём `Send` и `Sync` в следующей главе.

---

## Практика

### Задание 1

Создайте 5 потоков, каждый из которых выводит число от 1 до 5. Используйте `move` для передачи числа в каждый поток.

### Задание 2

Напишите программу, которая создаёт 10 потоков, каждый вычисляет сумму чисел в своём диапазоне, а затем суммирует результаты.

### Задание 3

Используйте `scoped threads` для параллельной обработки вектора чисел: каждый поток умножает свой элемент на 2.

### Задание 4

Создайте поток, который возвращает `String`. Выведите результат в основном потоке.

### Задание 5

🔨 **Эксперимент с компилятором.**

Почему следующий код не работает?

```rust
use std::thread;

fn main() {
    let x = 42;
    thread::spawn(|| {
        println!("{}", x);
    }).join().unwrap();
}
```

### Задание 6

🔨 **Эксперимент с компилятором.**

Почему следующий код не работает?

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);
    let handle = thread::spawn(move || {
        println!("{:?}", data);
    });
    handle.join().unwrap();
}
```

---

## Главное из этой главы

После этой главы мы понимаем:

- **`thread::spawn`** создаёт новый поток.
- **`JoinHandle`** позволяет управлять созданным потоком и получить его результат.
- **`join()`** дожидается завершения потока и возвращает `Result<T>`.
- **`move`** заставляет closure захватывать используемые значения по значению.
- Обычный `thread::spawn` требует от closure и результата подходящие ограничения `Send + 'static`.
- **`thread::scope`** позволяет потокам заимствовать локальные данные, потому что гарантирует завершение потоков до выхода из области scope.
- **`Send`** определяет возможность безопасно передавать значение между потоками.
- **`Sync`** определяет возможность безопасно использовать ссылку на значение из другого потока.
- **Concurrency** и **parallelism** — связанные, но разные понятия.
- Потоки не являются бесплатным способом ускорить любую программу: параллелизм требует разумного разделения работы и синхронизации.

**Самая важная идея:**

> В Rust потоки не являются отдельной системой безопасности. Те же ownership, borrowing, lifetimes и type system, которые защищают обычную программу, продолжают работать и при параллельном выполнении. Именно поэтому компилятор может обнаружить многие ошибки многопоточной программы ещё до её запуска. ([Rust Documentation][1])

[1]: https://doc.rust-lang.org/beta/std/thread/fn.spawn.html 'spawn in std::thread - Rust'
[2]: https://doc.rust-lang.org/nightly/std/thread/struct.JoinHandle.html 'JoinHandle in std::thread - Rust'
[3]: https://doc.rust-lang.org/std/thread/fn.scope.html 'scope in std::thread - Rust'
