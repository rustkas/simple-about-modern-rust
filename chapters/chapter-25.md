# Глава 25. Shared State

В предыдущих главах мы научились создавать потоки и узнали о трейтах `Send` и `Sync`. Теперь пришло время решить ключевую задачу параллельного программирования: **как безопасно разделять изменяемое состояние между потоками**.

В этой главе мы рассмотрим все основные инструменты для работы с разделяемым состоянием в Rust: `Arc`, `Mutex`, `RwLock` и атомарные типы. Мы разберёмся, как они работают, когда какой использовать, и как избежать распространённых проблем, таких как взаимоблокировки (deadlocks).

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 25.1. Проблема: разделяемое изменяемое состояние

Когда несколько потоков должны работать с **одними и теми же изменяемыми данными**, обычного владения Rust недостаточно: значение должно иметь одного владельца, а одновременно изменять его из нескольких потоков нельзя.

Может показаться, что для чисел это не проблема — ведь `i32` реализует `Copy`. Попробуем:

```rust
use std::thread;

fn main() {
    let mut counter = 0;

    let handle = thread::spawn(move || {
        counter += 1;
        println!("Внутри потока: {counter}");
    });

    counter += 1;
    println!("В main: {counter}");

    handle.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20counter%20%3D%200%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20counter%20%2B%3D%201%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%D0%92%D0%BD%D1%83%D1%82%D1%80%D0%B8%20%D0%BF%D0%BE%D1%82%D0%BE%D0%BA%D0%B0%3A%20%7Bcounter%7D%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20counter%20%2B%3D%201%3B%0A%20%20%20%20println%21%28%22%D0%92%20main%3A%20%7Bcounter%7D%22%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

Этот код **успешно компилируется** — и в этом всё дело. `counter` имеет тип `i32`, который реализует `Copy`. Когда `move`-замыкание захватывает `Copy`-переменную, происходит **копирование значения**, а не перемещение владения: замыкание получает собственную независимую копию `counter`, а исходная переменная в `main` остаётся полностью действительной и доступной для дальнейшего использования — компилятор не выдаёт здесь никакой ошибки о перемещённом значении.

Вывод программы (порядок первых двух строк не гарантирован):

```text
В main: 1
Внутри потока: 1
```

Обратите внимание: обе строки показывают `1`, а не `2`. Поток изменил *свою собственную копию* `counter` — с 0 до 1. Основной поток независимо изменил *свою собственную* переменную `counter` — тоже с 0 до 1. Эти два значения никак не связаны друг с другом: то, что видит поток, никак не отражает то, что происходит в `main`, и наоборот.

Вот в чём настоящая проблема разделяемого состояния: дело не в том, что компилятор запрещает такой код (для `Copy`-типов он его разрешает), а в том, что **простое копирование не даёт разделения состояния вообще**. Если бы нам действительно было нужно, чтобы десять потоков совместно увеличивали *одно и то же* число, а не каждый — свою независимую копию, `move`-замыкание с `Copy`-типом нам бы не помогло: каждый поток просто работал бы со своим личным значением.

Для типов, которые не реализуют `Copy` (например, `String` или `Vec<T>`), ситуация иная — там `move` действительно перемещает владение, и попытка использовать переменную после этого в `main` уже была бы настоящей ошибкой компиляции:

```rust
use std::thread;

fn main() {
    let data = String::from("shared");

    let handle = thread::spawn(move || {
        println!("{data}");
    });

    // println!("{data}"); // ошибка: data перемещена в замыкание

    handle.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20data%20%3D%20String%3A%3Afrom%28%22shared%22%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bdata%7D%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

Но даже для не-`Copy`-типов перемещение владения — не то же самое, что *разделение* владения: `move` передаёт `data` целиком одному-единственному потоку. Если нам нужно, чтобы **несколько** потоков одновременно получили доступ к **одним и тем же** данным (а не копию и не единолично перемещённое значение), обычного `move` тоже недостаточно.

Итак, для настоящего разделяемого изменяемого состояния нам нужны **два разных механизма**:

1. механизм разделяемого владения — чтобы несколько потоков могли ссылаться на одни и те же данные, а не на копии или на единолично перемещённое значение;
2. механизм синхронизации доступа — чтобы гарантировать, что операции изменения из разных потоков не пересекаются друг с другом непредсказуемым образом.

В Rust обычно это означает комбинацию:

```text
Arc<T>
  │
  │ разделяемое владение
  ▼
Mutex<T>
  │
  │ синхронизация доступа
  ▼
изменяемое T
```

> **Важно:** Rust не просто запрещает data race в конкретном примере. Система типов и модель владения не позволяют построить безопасный код, в котором несколько потоков одновременно получают неконтролируемый изменяемый доступ к одним и тем же данным.

---

## 25.2. Решение: `Arc<Mutex<T>>`

Для безопасного разделения изменяемого состояния используем комбинацию `Arc` (для разделения владения) и `Mutex` (для взаимного исключения):

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let handles: Vec<_> = (0..10)
        .map(|_| {
            let counter = Arc::clone(&counter);
            thread::spawn(move || {
                let mut num = counter.lock().unwrap();
                *num += 1;
            })
        })
        .collect();

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Counter: {}", *counter.lock().unwrap());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3A%7BArc%2C%20Mutex%7D%3B%0Ause%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20counter%20%3D%20Arc%3A%3Anew%28Mutex%3A%3Anew%280%29%29%3B%0A%0A%20%20%20%20let%20handles%3A%20Vec%3C_%3E%20%3D%20%280..10%29%0A%20%20%20%20%20%20%20%20.map%28%7C_%7C%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20let%20counter%20%3D%20Arc%3A%3Aclone%28%26counter%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20let%20mut%20num%20%3D%20counter.lock%28%29.unwrap%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20*num%20%2B%3D%201%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%29%0A%20%20%20%20%20%20%20%20%7D%29%0A%20%20%20%20%20%20%20%20.collect%28%29%3B%0A%0A%20%20%20%20for%20handle%20in%20handles%20%7B%0A%20%20%20%20%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22Counter%3A%20%7B%7D%22%2C%20*counter.lock%28%29.unwrap%28%29%29%3B%0A%7D)

**Вывод:** `Counter: 10`

---

## 25.3. Как работает `Mutex<T>`

`Mutex` (MUTual EXclusion) — это примитив синхронизации, который позволяет только одному потоку одновременно иметь доступ к данным.

```rust
use std::sync::Mutex;

fn main() {
    let data = Mutex::new(42);

    {
        let mut guard = data.lock().unwrap();
        *guard = 100;
        // guard освобождается здесь
    }

    let guard = data.lock().unwrap();
    println!("{}", *guard);
}
```

**Ключевые методы:**
- `lock()` — захватывает мьютекс, возвращает `MutexGuard` (блокирует поток, если мьютекс занят)
- `try_lock()` — пытается захватить мьютекс без блокировки

---

## 25.4. `RwLock` — разделение на читателей и писателей

`RwLock` (Read-Write Lock) предназначен для ситуации, когда данные часто читаются и относительно редко изменяются.

Он предоставляет два вида блокировки:

* `read()` — блокировка для чтения;
* `write()` — блокировка для изменения.

Одновременно может существовать несколько `read`-guard, но `write`-guard является эксклюзивным.

Для использования `RwLock` между несколькими потоками обычно применяется `Arc`:

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(42));

    let reader1 = {
        let data = Arc::clone(&data);

        thread::spawn(move || {
            let value = data.read().unwrap();
            println!("Reader 1: {}", *value);
        })
    };

    let reader2 = {
        let data = Arc::clone(&data);

        thread::spawn(move || {
            let value = data.read().unwrap();
            println!("Reader 2: {}", *value);
        })
    };

    let writer = {
        let data = Arc::clone(&data);

        thread::spawn(move || {
            let mut value = data.write().unwrap();
            *value = 100;
            println!("Writer: {}", *value);
        })
    };

    reader1.join().unwrap();
    reader2.join().unwrap();
    writer.join().unwrap();

    println!("Final value: {}", *data.read().unwrap());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3A%7BArc%2C+RwLock%7D%3B%0Ause+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+Arc%3A%3Anew%28RwLock%3A%3Anew%2842%29%29%3B%0A%0A++++let+reader1+%3D+%7B%0A++++++++let+data+%3D+Arc%3A%3Aclone%28%26data%29%3B%0A%0A++++++++thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++let+value+%3D+data.read%28%29.unwrap%28%29%3B%0A++++++++++++println%21%28%22Reader+1%3A+%7B%7D%22%2C+*value%29%3B%0A++++++++%7D%29%0A++++%7D%3B%0A%0A++++let+reader2+%3D+%7B%0A++++++++let+data+%3D+Arc%3A%3Aclone%28%26data%29%3B%0A%0A++++++++thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++let+value+%3D+data.read%28%29.unwrap%28%29%3B%0A++++++++++++println%21%28%22Reader+2%3A+%7B%7D%22%2C+*value%29%3B%0A++++++++%7D%29%0A++++%7D%3B%0A%0A++++let+writer+%3D+%7B%0A++++++++let+data+%3D+Arc%3A%3Aclone%28%26data%29%3B%0A%0A++++++++thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++let+mut+value+%3D+data.write%28%29.unwrap%28%29%3B%0A++++++++++++*value+%3D+100%3B%0A++++++++++++println%21%28%22Writer%3A+%7B%7D%22%2C+*value%29%3B%0A++++++++%7D%29%0A++++%7D%3B%0A%0A++++reader1.join%28%29.unwrap%28%29%3B%0A++++reader2.join%28%29.unwrap%28%29%3B%0A++++writer.join%28%29.unwrap%28%29%3B%0A%0A++++println%21%28%22Final+value%3A+%7B%7D%22%2C+*data.read%28%29.unwrap%28%29%29%3B%0A%7D)

Здесь:

```rust
let data = Arc::new(RwLock::new(42));
```

означает:

* `Arc` позволяет нескольким потокам владеть одним объектом;
* `RwLock` защищает значение `42`;
* `read()` позволяет одновременно читать значение нескольким потокам;
* `write()` требует эксклюзивного доступа.

Порядок вывода не определён: планировщик ОС может запускать потоки в разной последовательности.

Например, вполне допустим такой результат:

```text
Reader 1: 42
Reader 2: 42
Writer: 100
Final value: 100
```

Но возможен и другой порядок.

**Когда использовать `RwLock`:**

`RwLock` особенно полезен, когда:

* чтений значительно больше, чем записей;
* операции чтения достаточно продолжительные;
* несколько потоков действительно могут выполнять чтение параллельно.

Не следует автоматически считать `RwLock` более быстрым, чем `Mutex`. У `RwLock` есть собственные накладные расходы, а конкретное поведение зависит от реализации и характера нагрузки.

Если чтение и запись происходят примерно одинаково часто, обычный `Mutex` часто оказывается более простым и вполне эффективным решением.



---

## 25.5. Атомарные типы (Atomics)

Для простых операций над числами и логическими значениями можно использовать атомарные типы:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));

    let handles: Vec<_> = (0..10)
        .map(|_| {
            let counter = Arc::clone(&counter);
            thread::spawn(move || {
                counter.fetch_add(1, Ordering::SeqCst);
            })
        })
        .collect();

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Counter: {}", counter.load(Ordering::SeqCst));
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3AArc%3B%0Ause%20std%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicUsize%2C%20Ordering%7D%3B%0Ause%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20counter%20%3D%20Arc%3A%3Anew%28AtomicUsize%3A%3Anew%280%29%29%3B%0A%0A%20%20%20%20let%20handles%3A%20Vec%3C_%3E%20%3D%20%280..10%29%0A%20%20%20%20%20%20%20%20.map%28%7C_%7C%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20let%20counter%20%3D%20Arc%3A%3Aclone%28%26counter%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20counter.fetch_add%281%2C%20Ordering%3A%3ASeqCst%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%29%0A%20%20%20%20%20%20%20%20%7D%29%0A%20%20%20%20%20%20%20%20.collect%28%29%3B%0A%0A%20%20%20%20for%20handle%20in%20handles%20%7B%0A%20%20%20%20%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22Counter%3A%20%7B%7D%22%2C%20counter.load%28Ordering%3A%3ASeqCst%29%29%3B%0A%7D)

**Основные атомарные типы:**
- `AtomicBool` — логическое значение
- `AtomicUsize` / `AtomicIsize` — целые числа
- `AtomicU8` / `AtomicI8` и т.д.
- `AtomicPtr<T>` — указатели

**Основные операции:**
- `load` — чтение
- `store` — запись
- `fetch_add` / `fetch_sub` — атомарное сложение/вычитание
- `compare_exchange` — сравнение и обмен (CAS)

---

## 25.6. `Ordering` — гарантии упорядочивания

Атомарность операции отвечает только на один вопрос:

> Может ли операция быть выполнена безопасно одновременно с другими атомарными операциями?

Но в многопоточном коде есть ещё один вопрос:

> В каком порядке разные потоки должны видеть операции друг друга?

Для этого существует `Ordering`.

Основные варианты:

| Ordering  | Назначение                                                                             |
| --------- | -------------------------------------------------------------------------------------- |
| `Relaxed` | Атомарность без дополнительных гарантий порядка                                        |
| `Acquire` | Устанавливает acquire-синхронизацию при соответствующей release-операции               |
| `Release` | Публикует предшествующие операции для потока, который выполнит соответствующий acquire |
| `AcqRel`  | Комбинация `Acquire` и `Release` для read-modify-write операций                        |
| `SeqCst`  | Самая сильная модель: атомарные операции имеют единый глобальный порядок               |

Важно понимать, что `Acquire` и `Release` сами по себе не означают:

> «все записи во всей программе становятся видимыми всем потокам».

Синхронизация возникает между соответствующими операциями над атомарным объектом.

Например:

```rust
use std::sync::{
    Arc,
    atomic::{AtomicBool, AtomicUsize, Ordering},
};
use std::thread;

fn main() {
    let data = Arc::new(AtomicUsize::new(0));
    let ready = Arc::new(AtomicBool::new(false));

    let writer_data = Arc::clone(&data);
    let writer_ready = Arc::clone(&ready);

    let writer = thread::spawn(move || {
        writer_data.store(42, Ordering::Relaxed);

        // Публикуем факт готовности.
        writer_ready.store(true, Ordering::Release);
    });

    let reader_data = Arc::clone(&data);
    let reader_ready = Arc::clone(&ready);

    let reader = thread::spawn(move || {
        while !reader_ready.load(Ordering::Acquire) {
            thread::yield_now();
        }

        // Acquire синхронизируется с Release выше.
        let value = reader_data.load(Ordering::Relaxed);

        println!("value = {value}");
    });

    writer.join().unwrap();
    reader.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3A%7B%0A++++Arc%2C%0A++++atomic%3A%3A%7BAtomicBool%2C+AtomicUsize%2C+Ordering%7D%2C%0A%7D%3B%0Ause+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+Arc%3A%3Anew%28AtomicUsize%3A%3Anew%280%29%29%3B%0A++++let+ready+%3D+Arc%3A%3Anew%28AtomicBool%3A%3Anew%28false%29%29%3B%0A%0A++++let+writer_data+%3D+Arc%3A%3Aclone%28%26data%29%3B%0A++++let+writer_ready+%3D+Arc%3A%3Aclone%28%26ready%29%3B%0A%0A++++let+writer+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++writer_data.store%2842%2C+Ordering%3A%3ARelaxed%29%3B%0A%0A++++++++%2F%2F+%D0%9F%D1%83%D0%B1%D0%BB%D0%B8%D0%BA%D1%83%D0%B5%D0%BC+%D1%84%D0%B0%D0%BA%D1%82+%D0%B3%D0%BE%D1%82%D0%BE%D0%B2%D0%BD%D0%BE%D1%81%D1%82%D0%B8.%0A++++++++writer_ready.store%28true%2C+Ordering%3A%3ARelease%29%3B%0A++++%7D%29%3B%0A%0A++++let+reader_data+%3D+Arc%3A%3Aclone%28%26data%29%3B%0A++++let+reader_ready+%3D+Arc%3A%3Aclone%28%26ready%29%3B%0A%0A++++let+reader+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++while+%21reader_ready.load%28Ordering%3A%3AAcquire%29+%7B%0A++++++++++++thread%3A%3Ayield_now%28%29%3B%0A++++++++%7D%0A%0A++++++++%2F%2F+Acquire+%D1%81%D0%B8%D0%BD%D1%85%D1%80%D0%BE%D0%BD%D0%B8%D0%B7%D0%B8%D1%80%D1%83%D0%B5%D1%82%D1%81%D1%8F+%D1%81+Release+%D0%B2%D1%8B%D1%88%D0%B5.%0A++++++++let+value+%3D+reader_data.load%28Ordering%3A%3ARelaxed%29%3B%0A%0A++++++++println%21%28%22value+%3D+%7Bvalue%7D%22%29%3B%0A++++%7D%29%3B%0A%0A++++writer.join%28%29.unwrap%28%29%3B%0A++++reader.join%28%29.unwrap%28%29%3B%0A%7D)

Здесь `ready` используется как сигнал публикации:

```text
writer                         reader

data = 42
   │
   ▼
ready.store(true, Release)
   │
   │   synchronization
   └──────────────────────────►
                               ready.load(Acquire)
                                      │
                                      ▼
                               data.load(Relaxed)
                                      │
                                      ▼
                                    42
```

`Release` гарантирует, что предшествующие операции потока публикации становятся частью happens-before отношений, устанавливаемых при успешном соответствующем `Acquire`.

## Какой `Ordering` использовать?

Если вы только изучаете atomics и не уверены в необходимой модели памяти, `SeqCst` часто является хорошей отправной точкой.

Например, для простого счётчика:

```rust
counter.fetch_add(1, Ordering::SeqCst);
```

обычно нет необходимости преждевременно выбирать более слабую модель.

Однако `SeqCst` не является универсальным правилом «для production». Когда требуется высокая производительность или реализуется lock-free алгоритм, выбор `Ordering` должен следовать из конкретного протокола синхронизации.

---

## 25.7. Lock contention — соревнование за блокировку

**Contention** возникает, когда несколько потоков одновременно пытаются получить одну и ту же блокировку.

Например, это плохой вариант:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let handles: Vec<_> = (0..10)
        .map(|_| {
            let counter = Arc::clone(&counter);

            thread::spawn(move || {
                let mut value = counter.lock().unwrap();

                *value += 1;

                // ❌ Плохо: MutexGuard всё ещё удерживается.
                thread::sleep(Duration::from_millis(10));
            })
        })
        .collect();

    for handle in handles {
        handle.join().unwrap();
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3A%7BArc%2C+Mutex%7D%3B%0Ause+std%3A%3Athread%3B%0Ause+std%3A%3Atime%3A%3ADuration%3B%0A%0Afn+main%28%29+%7B%0A++++let+counter+%3D+Arc%3A%3Anew%28Mutex%3A%3Anew%280%29%29%3B%0A%0A++++let+handles%3A+Vec%3C_%3E+%3D+%280..10%29%0A++++++++.map%28%7C_%7C+%7B%0A++++++++++++let+counter+%3D+Arc%3A%3Aclone%28%26counter%29%3B%0A%0A++++++++++++thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++++++let+mut+value+%3D+counter.lock%28%29.unwrap%28%29%3B%0A%0A++++++++++++++++*value+%2B%3D+1%3B%0A%0A++++++++++++++++%2F%2F+%E2%9D%8C+%D0%9F%D0%BB%D0%BE%D1%85%D0%BE%3A+MutexGuard+%D0%B2%D1%81%D1%91+%D0%B5%D1%89%D1%91+%D1%83%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D0%B2%D0%B0%D0%B5%D1%82%D1%81%D1%8F.%0A++++++++++++++++thread%3A%3Asleep%28Duration%3A%3Afrom_millis%2810%29%29%3B%0A++++++++++++%7D%29%0A++++++++%7D%29%0A++++++++.collect%28%29%3B%0A%0A++++for+handle+in+handles+%7B%0A++++++++handle.join%28%29.unwrap%28%29%3B%0A++++%7D%0A%7D%0A)

Пока выполняется `sleep`, поток продолжает удерживать `MutexGuard`. Следовательно, другие потоки не могут получить доступ к данным.

Правильнее минимизировать критическую секцию:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let handles: Vec<_> = (0..10)
        .map(|_| {
            let counter = Arc::clone(&counter);

            thread::spawn(move || {
                {
                    let mut value = counter.lock().unwrap();
                    *value += 1;
                } // MutexGuard освобождён здесь.

                // Долгая работа выполняется без блокировки.
                thread::sleep(Duration::from_millis(10));
            })
        })
        .collect();

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Counter: {}", *counter.lock().unwrap());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3A%7BArc%2C+Mutex%7D%3B%0Ause+std%3A%3Athread%3B%0Ause+std%3A%3Atime%3A%3ADuration%3B%0A%0Afn+main%28%29+%7B%0A++++let+counter+%3D+Arc%3A%3Anew%28Mutex%3A%3Anew%280%29%29%3B%0A%0A++++let+handles%3A+Vec%3C_%3E+%3D+%280..10%29%0A++++++++.map%28%7C_%7C+%7B%0A++++++++++++let+counter+%3D+Arc%3A%3Aclone%28%26counter%29%3B%0A%0A++++++++++++thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++++++%7B%0A++++++++++++++++++++let+mut+value+%3D+counter.lock%28%29.unwrap%28%29%3B%0A++++++++++++++++++++*value+%2B%3D+1%3B%0A++++++++++++++++%7D+%2F%2F+MutexGuard+%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D1%91%D0%BD+%D0%B7%D0%B4%D0%B5%D1%81%D1%8C.%0A%0A++++++++++++++++%2F%2F+%D0%94%D0%BE%D0%BB%D0%B3%D0%B0%D1%8F+%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%B0+%D0%B2%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D1%8F%D0%B5%D1%82%D1%81%D1%8F+%D0%B1%D0%B5%D0%B7+%D0%B1%D0%BB%D0%BE%D0%BA%D0%B8%D1%80%D0%BE%D0%B2%D0%BA%D0%B8.%0A++++++++++++++++thread%3A%3Asleep%28Duration%3A%3Afrom_millis%2810%29%29%3B%0A++++++++++++%7D%29%0A++++++++%7D%29%0A++++++++.collect%28%29%3B%0A%0A++++for+handle+in+handles+%7B%0A++++++++handle.join%28%29.unwrap%28%29%3B%0A++++%7D%0A%0A++++println%21%28%22Counter%3A+%7B%7D%22%2C+*counter.lock%28%29.unwrap%28%29%29%3B%0A%7D%0A)

Главное правило:

> **Не удерживайте блокировку дольше, чем необходимо для доступа к защищаемым данным.**

Способы уменьшить contention:

1. Уменьшать время удержания `MutexGuard`.
2. Не выполнять медленные операции внутри критической секции.
3. Разделять независимые данные между несколькими блокировками, если это действительно оправдано.
4. Использовать атомарные типы для простых состояний и счётчиков.
5. Изменять архитектуру так, чтобы потоки как можно чаще **передавали сообщения**, а не конкурировали за общее состояние.

Последний вариант особенно важен: иногда лучшим решением является вообще отказаться от shared state и использовать модель **message passing**, рассмотренную в предыдущей главе.

---

## 25.8. Deadlock — взаимная блокировка

**Deadlock** возникает, когда потоки блокируют ресурсы и одновременно ожидают ресурсы, удерживаемые друг другом.

Классический пример:

```text
Поток 1:
    lock(A)
    lock(B)

Поток 2:
    lock(B)
    lock(A)
```

Возможная последовательность:

```text
Поток 1                    Поток 2

lock(A)                    lock(B)
   │                           │
   ▼                           ▼
A захвачен                   B захвачен
   │                           │
   │ lock(B)                   │ lock(A)
   │                           │
   └─────── ждёт ◄─────────────┘
                ждёт
```

Ни один поток не может продолжить работу.

Rust **не обнаруживает обычные deadlock на этапе компиляции**. Это логическая проблема программы, возникающая во время выполнения.

Один из наиболее простых способов избежать такого deadlock — договориться о едином порядке захвата блокировок.

Например, всегда:

```text
A → B
```

Тогда оба потока должны использовать одинаковый порядок:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let a = Arc::new(Mutex::new(0));
    let b = Arc::new(Mutex::new(0));

    let a1 = Arc::clone(&a);
    let b1 = Arc::clone(&b);

    let handle1 = thread::spawn(move || {
        let _a = a1.lock().unwrap();
        let _b = b1.lock().unwrap();

        println!("Thread 1");
    });

    let a2 = Arc::clone(&a);
    let b2 = Arc::clone(&b);

    let handle2 = thread::spawn(move || {
        // Тот же порядок: A → B.
        let _a = a2.lock().unwrap();
        let _b = b2.lock().unwrap();

        println!("Thread 2");
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
}
```
[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3A%7BArc%2C+Mutex%7D%3B%0Ause+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+a+%3D+Arc%3A%3Anew%28Mutex%3A%3Anew%280%29%29%3B%0A++++let+b+%3D+Arc%3A%3Anew%28Mutex%3A%3Anew%280%29%29%3B%0A%0A++++let+a1+%3D+Arc%3A%3Aclone%28%26a%29%3B%0A++++let+b1+%3D+Arc%3A%3Aclone%28%26b%29%3B%0A%0A++++let+handle1+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++let+_a+%3D+a1.lock%28%29.unwrap%28%29%3B%0A++++++++let+_b+%3D+b1.lock%28%29.unwrap%28%29%3B%0A%0A++++++++println%21%28%22Thread+1%22%29%3B%0A++++%7D%29%3B%0A%0A++++let+a2+%3D+Arc%3A%3Aclone%28%26a%29%3B%0A++++let+b2+%3D+Arc%3A%3Aclone%28%26b%29%3B%0A%0A++++let+handle2+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++%2F%2F+%D0%A2%D0%BE%D1%82+%D0%B6%D0%B5+%D0%BF%D0%BE%D1%80%D1%8F%D0%B4%D0%BE%D0%BA%3A+A+%E2%86%92+B.%0A++++++++let+_a+%3D+a2.lock%28%29.unwrap%28%29%3B%0A++++++++let+_b+%3D+b2.lock%28%29.unwrap%28%29%3B%0A%0A++++++++println%21%28%22Thread+2%22%29%3B%0A++++%7D%29%3B%0A%0A++++handle1.join%28%29.unwrap%28%29%3B%0A++++handle2.join%28%29.unwrap%28%29%3B%0A%7D%0A)

Другие способы уменьшить риск deadlock:

* избегать вложенных блокировок;
* держать критические секции короткими;
* использовать `try_lock()`, если ожидание блокировки недопустимо;
* централизовать порядок захвата нескольких ресурсов;
* по возможности использовать архитектуру, не требующую одновременного удержания нескольких блокировок.

---

## 25.9. Poisoning — отравление мьютекса

`std::sync::Mutex` использует механизм **poisoning**.

Если поток паникует, удерживая `MutexGuard`, следующий вызов `lock()` обычно получает:

```rust
Err(PoisonError<T>)
```

Это сигнал:

> Поток мог оставить защищаемое состояние в состоянии, которое программа должна проверить перед дальнейшим использованием.

Например:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(42));

    let worker_data = Arc::clone(&data);

    let worker = thread::spawn(move || {
        let mut value = worker_data.lock().unwrap();

        *value = 100;

        panic!("worker failed");
    });

    let _ = worker.join();

    match data.lock() {
        Ok(value) => {
            println!("Data: {}", *value);
        }

        Err(poisoned) => {
            println!("Mutex is poisoned");

            let value = poisoned.into_inner();

            println!("Recovered value: {}", *value);
        }
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3A%7BArc%2C+Mutex%7D%3B%0Ause+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+Arc%3A%3Anew%28Mutex%3A%3Anew%2842%29%29%3B%0A%0A++++let+worker_data+%3D+Arc%3A%3Aclone%28%26data%29%3B%0A%0A++++let+worker+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++let+mut+value+%3D+worker_data.lock%28%29.unwrap%28%29%3B%0A%0A++++++++*value+%3D+100%3B%0A%0A++++++++panic%21%28%22worker+failed%22%29%3B%0A++++%7D%29%3B%0A%0A++++let+_+%3D+worker.join%28%29%3B%0A%0A++++match+data.lock%28%29+%7B%0A++++++++Ok%28value%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Data%3A+%7B%7D%22%2C+*value%29%3B%0A++++++++%7D%0A%0A++++++++Err%28poisoned%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Mutex+is+poisoned%22%29%3B%0A%0A++++++++++++let+value+%3D+poisoned.into_inner%28%29%3B%0A%0A++++++++++++println%21%28%22Recovered+value%3A+%7B%7D%22%2C+*value%29%3B%0A++++++++%7D%0A++++%7D%0A%7D)

В данном примере значение было изменено перед паникой:

```text
42 → 100 → panic
```

Поэтому после получения poisoned mutex нельзя автоматически считать данные корректными.

Если программа знает, что состояние всё ещё валидно, можно получить guard через:

```rust
poisoned.into_inner()
```

Но это **осознанное решение**, а не просто способ избавиться от ошибки.

Важно также понимать, что poisoning — это механизм диагностики, а не абсолютная гарантия того, что «после любой паники mutex обязательно poisoned». В стандартной библиотеке есть ситуации, в которых poisoning может не сработать. Поэтому unsafe-код не должен полагаться на poisoning как на единственную гарантию безопасности.

Для обычного безопасного Rust-кода достаточно помнить:

> `PoisonError` сообщает, что другой поток мог прервать работу с защищаемым состоянием во время panic, и состояние следует рассматривать с осторожностью.

---

## 25.10. Выбор между `Mutex` и атомарными типами

| Критерий                                | `Mutex<T>`                           | Атомарные типы                         |
| --------------------------------------- | ------------------------------------ | -------------------------------------- |
| **Тип данных**                          | Практически любые `T`                | Ограниченный набор атомарных типов     |
| **Сложность операций**                  | Произвольные операции над `T`        | Атомарные операции над одним значением |
| **Блокировка**                          | Да                                   | Нет mutex-блокировки                   |
| **Memory ordering**                     | Скрыт внутри примитива синхронизации | Нужно явно выбирать `Ordering`         |
| **Сложность корректного использования** | Обычно ниже                          | Может быть значительно выше            |
| **Подходит для счётчиков**              | Да                                   | Да                                     |
| **Подходит для `HashMap`/структур**     | Да                                   | Нет, напрямую                          |

Для простого счётчика:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

let counter = AtomicUsize::new(0);

counter.fetch_add(1, Ordering::SeqCst);
```

Для сложной структуры:

```rust
use std::collections::HashMap;
use std::sync::Mutex;

let data = Mutex::new(HashMap::new());

let mut map = data.lock().unwrap();
map.insert("language", "Rust");
```

### Важное уточнение о производительности

Нельзя считать правилом:

> `Atomic` всегда быстрее `Mutex`.

Атомарная операция часто дешевле блокировки, особенно для простых операций над одним значением. Но реальная производительность зависит от архитектуры процессора, количества потоков, уровня contention и используемого алгоритма.

Кроме того, корректный lock-free алгоритм может быть значительно сложнее корректного `Mutex`.

Поэтому практическое правило такое:

> Используйте атомарные типы, когда задача естественно выражается атомарными операциями. Используйте `Mutex`, когда нужно безопасно выполнять составную операцию над состоянием.

---

## 25.11. Lock granularity — размер блокировки

Если приложение использует несколько независимых частей общего состояния, возникает вопрос: сколько блокировок необходимо?

**Грубая блокировка (coarse-grained locking):**

```rust
use std::collections::HashMap;
use std::sync::Mutex;

struct State {
    users: Vec<String>,
    cache: HashMap<String, String>,
    counter: usize,
}

let state = Mutex::new(State {
    users: Vec::new(),
    cache: HashMap::new(),
    counter: 0,
});
```

Здесь одна блокировка защищает всё состояние.

Преимущество — простота:

```text
Mutex<State>
     │
     ├── users
     ├── cache
     └── counter
```

Недостаток: поток, которому нужен только `counter`, конкурирует с потоком, работающим с `cache`.

**Тонкая блокировка (fine-grained locking):**

```rust
use std::collections::HashMap;
use std::sync::{
    Arc,
    Mutex,
    atomic::AtomicUsize,
};

let users = Arc::new(Mutex::new(Vec::<String>::new()));
let cache = Arc::new(Mutex::new(HashMap::<String, String>::new()));
let counter = Arc::new(AtomicUsize::new(0));
```

Теперь разные части состояния могут использоваться независимо.

Но fine-grained locking увеличивает сложность программы. Например, появляется больше потенциальных комбинаций блокировок и, следовательно, больше возможностей получить deadlock.

Поэтому выбор следует делать не по принципу:

> «Чем больше mutex, тем быстрее».

А по принципу:

> **Разделяйте блокировки там, где данные действительно независимы и измерения показывают, что contention становится проблемой.**

Начинать с простой архитектуры обычно безопаснее.

---

## 25.12. Сравнение `Mutex` и `RwLock`

| Характеристика           | `Mutex<T>`          | `RwLock<T>`              |
| ------------------------ | ------------------- | ------------------------ |
| Одновременные читатели   | Нет                 | Да                       |
| Одновременные писатели   | Нет                 | Нет                      |
| Эксклюзивная запись      | Да                  | Да                       |
| Простота                 | Выше                | Ниже                     |
| Накладные расходы        | Обычно ниже         | Обычно выше              |
| Хорош для частого чтения | Не всегда оптимален | Может быть полезен       |
| Хорош для частой записи  | Часто подходит      | Может быть менее выгоден |
| Deadlock возможен        | Да                  | Да                       |

Важно не превращать эту таблицу в правило:

> «Много чтений → всегда `RwLock`».

`RwLock` имеет дополнительные накладные расходы и зависит от конкретной реализации и платформы. При небольших критических секциях обычный `Mutex` может оказаться быстрее даже при большом количестве чтений.

Хорошая практическая стратегия:

```text
Нужно защищать состояние?
        │
        ├── Простая структура / смешанные чтения и записи
        │       │
        │       └── Mutex<T>
        │
        ├── Очень много независимых чтений
        │   и относительно редкие записи
        │       │
        │       └── RwLock<T>
        │
        └── Простое атомарное состояние
                │
                └── Atomic*
```

И окончательный выбор для производительного кода желательно подтверждать измерениями, а не только теоретическими предположениями.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `Arc<Mutex>` vs `Rc<RefCell>`

```rust
// ❌ Не работает между потоками
use std::rc::Rc;
use std::cell::RefCell;

// ✅ Работает между потоками
use std::sync::{Arc, Mutex};
```

### Эксперимент 2: Deadlock с захватом в разном порядке

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let a = Arc::new(Mutex::new(0));
    let b = Arc::new(Mutex::new(0));

    let a_clone = Arc::clone(&a);
    let b_clone = Arc::clone(&b);

    let handle1 = thread::spawn(move || {
        let _x = a_clone.lock().unwrap();
        let _y = b_clone.lock().unwrap(); // Порядок: a → b
    });

    let handle2 = thread::spawn(move || {
        let _y = b.lock().unwrap();
        let _x = a.lock().unwrap(); // Порядок: b → a 🔴 DEADLOCK!
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
}
```

**Исправление:** Используйте одинаковый порядок захвата во всех потоках.

---

## Практика

### Задание 1

Напишите программу, которая создаёт 100 потоков и увеличивает общий счётчик. Используйте `Arc<Mutex<usize>>`.

### Задание 2

Перепишите задание 1, используя `AtomicUsize`. Сравните производительность.

### Задание 3

Создайте структуру `BankAccount` с балансом. Используйте `Arc<Mutex<BankAccount>>` для безопасного изменения баланса из нескольких потоков.

### Задание 4

Напишите пример, демонстрирующий deadlock. Исправьте его.

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если попытаться использовать `Mutex` без `Arc`?

```rust
let data = Mutex::new(42);
thread::spawn(move || {
    *data.lock().unwrap() = 100;
});
```

### Задание 6

🔨 **Эксперимент с компилятором.**

Что произойдёт, если забыть `unwrap()` для `lock()`?

```rust
let data = Arc::new(Mutex::new(42));
let handle = thread::spawn(move || {
    let mut guard = data.lock(); // ❌ Что здесь?
    *guard = 100;
});
```

---

## Главное из этой главы

После этой главы мы понимаем:

- **`Arc<Mutex<T>>`** — стандартный способ разделения изменяемого состояния между потоками.
- **`Mutex`** — взаимное исключение (только один поток за раз).
- **`RwLock`** — разделение на читателей и писателей.
- **Атомарные типы** (`AtomicUsize`, `AtomicBool`) — для простых операций без блокировок.
- **`Ordering`** — уровень упорядочивания атомарных операций.
- **Lock contention** — проблема при высокой конкуренции за блокировку.
- **Deadlock** — взаимная блокировка (захват в разном порядке).
- **Poisoning** — мьютекс становится отравленным после паники.
- **Lock granularity** — выбор между грубой и тонкой блокировкой.

**Самая важная идея:**

> Разделяемое состояние — это сложная задача параллельного программирования. Rust предоставляет мощные инструменты для её решения: `Mutex`, `RwLock` и атомарные типы. Правильный выбор инструмента и понимание его ограничений — ключ к созданию безопасного и эффективного многопоточного кода.


