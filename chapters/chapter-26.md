# Глава 26. Channels

В предыдущей главе мы научились разделять состояние между потоками с помощью `Mutex` и `Arc`. Но есть и другой подход к организации взаимодействия между потоками — **передача сообщений (message passing)**.

Этот подход основан на принципе: **"Не разделяйте память для общения; общайтесь для разделения памяти"**. Вместо того чтобы разделять данные и синхронизировать доступ к ним, потоки отправляют друг другу сообщения через каналы (channels).

В этой главе мы познакомимся с каналами в Rust, научимся создавать producer-consumer системы, и разберёмся, когда каналы лучше, чем разделяемое состояние.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 26.1. Что такое канал (channel)?

**Канал** — это механизм для передачи сообщений между потоками. Он состоит из двух частей:

- **Отправитель (Sender)** — отправляет сообщения.
- **Получатель (Receiver)** — получает сообщения.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // Создаём канал
    let (tx, rx) = mpsc::channel();

    // Создаём поток-отправитель
    thread::spawn(move || {
        let message = String::from("Hello from thread!");
        tx.send(message).unwrap();
        // message перемещён в канал
    });

    // Получаем сообщение в главном потоке
    let received = rx.recv().unwrap();
    println!("Received: {}", received);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3Ampsc%3B%0Ause%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20%28tx%2C%20rx%29%20%3D%20mpsc%3A%3Achannel%28%29%3B%0A%0A%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20let%20message%20%3D%20String%3A%3Afrom%28%22Hello%20from%20thread%21%22%29%3B%0A%20%20%20%20%20%20%20%20tx.send%28message%29.unwrap%28%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20received%20%3D%20rx.recv%28%29.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22Received%3A%20%7B%7D%22%2C%20received%29%3B%0A%7D)

**Вывод:** `Received: Hello from thread!`

---

## 26.2. mpsc — multiple producer, single consumer

`mpsc` означает **Multiple Producer, Single Consumer** — множество отправителей, один получатель.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    // Клонируем отправитель для нескольких потоков
    let tx1 = tx.clone();
    let tx2 = tx.clone();

    // Первый поток
    thread::spawn(move || {
        tx1.send(String::from("Message from thread 1")).unwrap();
    });

    // Второй поток
    thread::spawn(move || {
        tx2.send(String::from("Message from thread 2")).unwrap();
    });

    // Принимаем сообщения
    for _ in 0..2 {
        let received = rx.recv().unwrap();
        println!("{}", received);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3Ampsc%3B%0Ause%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20%28tx%2C%20rx%29%20%3D%20mpsc%3A%3Achannel%28%29%3B%0A%0A%20%20%20%20let%20tx1%20%3D%20tx.clone%28%29%3B%0A%20%20%20%20let%20tx2%20%3D%20tx.clone%28%29%3B%0A%0A%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20tx1.send%28String%3A%3Afrom%28%22Message%20from%20thread%201%22%29%29.unwrap%28%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20tx2.send%28String%3A%3Afrom%28%22Message%20from%20thread%202%22%29%29.unwrap%28%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20for%20_%20in%200..2%20%7B%0A%20%20%20%20%20%20%20%20let%20received%20%3D%20rx.recv%28%29.unwrap%28%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20received%29%3B%0A%20%20%20%20%7D%0A%7D)

**Вывод (порядок может отличаться):**

```
Message from thread 1
Message from thread 2
```

---

## 26.3. Отправка через канал и владение

Отправка значения через канал **перемещает** владение:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let s = String::from("Hello");
        tx.send(s).unwrap();
        // println!("{}", s); // ❌ Ошибка! s перемещён в канал
    });

    let received = rx.recv().unwrap();
    println!("{}", received);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3Ampsc%3B%0Ause%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20%28tx%2C%20rx%29%20%3D%20mpsc%3A%3Achannel%28%29%3B%0A%0A%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20let%20s%20%3D%20String%3A%3Afrom%28%22Hello%22%29%3B%0A%20%20%20%20%20%20%20%20tx.send%28s%29.unwrap%28%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20received%20%3D%20rx.recv%28%29.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20received%29%3B%0A%7D)

**Важно:** После отправки значение нельзя использовать в потоке-отправителе.

---

## 26.4. Получение сообщений

Основные методы для получения сообщений:

| Метод        | Поведение                                            |
| ------------ | ---------------------------------------------------- |
| `recv()`     | Блокирует поток, пока сообщение не пришло            |
| `try_recv()` | Не блокирует, возвращает `Result` (успех или ошибка) |
| `iter()`     | Итератор по сообщениям (блокирующий)                 |

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        for i in 1..=5 {
            tx.send(i).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    // Использование итератора
    for received in rx {
        println!("Received: {}", received);
    }
    // Итератор завершится, когда все отправители будут закрыты
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3Ampsc%3B%0Ause%20std%3A%3Athread%3B%0Ause%20std%3A%3Atime%3A%3ADuration%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20%28tx%2C%20rx%29%20%3D%20mpsc%3A%3Achannel%28%29%3B%0A%0A%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20for%20i%20in%201..%3D5%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20tx.send%28i%29.unwrap%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20thread%3A%3Asleep%28Duration%3A%3Afrom_millis%28100%29%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20for%20received%20in%20rx%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Received%3A%20%7B%7D%22%2C%20received%29%3B%0A%20%20%20%20%7D%0A%7D)

**Вывод:**

```
Received: 1
Received: 2
Received: 3
Received: 4
Received: 5
```

---

## 26.5. Ограниченные и неограниченные каналы

**Неограниченный канал (unbounded):** сообщения хранятся в буфере без ограничений.

```rust
let (tx, rx) = mpsc::channel(); // Неограниченный
```

**Ограниченный канал (bounded):** буфер имеет ограниченный размер.

```rust
let (tx, rx) = mpsc::sync_channel(10); // Ограниченный (10 сообщений)
```

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // Ограниченный канал на 2 сообщения
    let (tx, rx) = mpsc::sync_channel(2);

    thread::spawn(move || {
        for i in 1..=10 {
            tx.send(i).unwrap();
            println!("Sent: {}", i);
        }
    });

    for received in rx {
        println!("Received: {}", received);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3Ampsc%3B%0Ause%20std%3A%3Athread%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20%28tx%2C%20rx%29%20%3D%20mpsc%3A%3Async_channel%282%29%3B%0A%0A%20%20%20%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20for%20i%20in%201..%3D10%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20tx.send%28i%29.unwrap%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22Sent%3A%20%7B%7D%22%2C%20i%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20for%20received%20in%20rx%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Received%3A%20%7B%7D%22%2C%20received%29%3B%0A%20%20%20%20%7D%0A%7D)

**Разница:**

- **Unbounded:** отправитель никогда не блокируется (но может заполнить память).
- **Bounded:** отправитель блокируется, когда буфер полон (backpressure).

---

## 26.6. Worker pool — пул воркеров

Пул воркеров — один из наиболее полезных практических паттернов для каналов.

Идея проста:

```text
                    ┌─────────────┐
                    │   Worker 1  │
                    └─────────────┘
                           ▲
                           │
┌──────────┐       ┌───────┴───────┐
│ Producer │ ────▶ │  Dispatcher   │
└──────────┘       └───────┬───────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Worker 2  │
                    └─────────────┘
```

Важно: `std::sync::mpsc` означает **multiple producer, single consumer**. Поэтому `Receiver` нельзя просто клонировать для нескольких воркеров:

```rust
let rx2 = rx.clone(); // ❌ Receiver не реализует Clone
```

Для стандартного `mpsc` удобно разделить систему на две части:

- один поток принимает задачи из входного канала;
- dispatcher распределяет задачи между worker threads;
- каждый worker получает задачи через собственный канал.

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    const WORKERS: usize = 4;

    // Канал от producer к dispatcher.
    let (tx, rx) = mpsc::channel::<u32>();

    let mut worker_handles = Vec::new();
    let mut worker_senders = Vec::new();

    // Создаём отдельный канал для каждого worker.
    for id in 0..WORKERS {
        let (worker_tx, worker_rx) = mpsc::channel::<u32>();

        worker_senders.push(worker_tx);

        let handle = thread::spawn(move || {
            for task in worker_rx {
                println!("Worker {id} processing task {task}");

                thread::sleep(Duration::from_millis(100));
            }

            println!("Worker {id} stopped");
        });

        worker_handles.push(handle);
    }

    // Dispatcher получает задачи из общего канала
    // и распределяет их между workers.
    let dispatcher = thread::spawn(move || {
        for (index, task) in rx.into_iter().enumerate() {
            let worker = index % WORKERS;

            if worker_senders[worker].send(task).is_err() {
                eprintln!("Worker {worker} is no longer available");
                break;
            }
        }

        // Закрываем worker channels.
        // После этого workers завершат свои циклы.
        drop(worker_senders);
    });

    // Producer отправляет задачи.
    for task in 0..20 {
        tx.send(task).unwrap();
    }

    // Закрываем входной канал.
    // Dispatcher завершит цикл после обработки всех сообщений.
    drop(tx);

    dispatcher.join().unwrap();

    for handle in worker_handles {
        handle.join().unwrap();
    }

    println!("All tasks processed!");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3Ampsc%3B%0Ause+std%3A%3Athread%3B%0Ause+std%3A%3Atime%3A%3ADuration%3B%0A%0Afn+main%28%29+%7B%0A++++const+WORKERS%3A+usize+%3D+4%3B%0A%0A++++%2F%2F+%D0%9A%D0%B0%D0%BD%D0%B0%D0%BB+%D0%BE%D1%82+producer+%D0%BA+dispatcher.%0A++++let+%28tx%2C+rx%29+%3D+mpsc%3A%3Achannel%3A%3A%3Cu32%3E%28%29%3B%0A%0A++++let+mut+worker_handles+%3D+Vec%3A%3Anew%28%29%3B%0A++++let+mut+worker_senders+%3D+Vec%3A%3Anew%28%29%3B%0A%0A++++%2F%2F+%D0%A1%D0%BE%D0%B7%D0%B4%D0%B0%D1%91%D0%BC+%D0%BE%D1%82%D0%B4%D0%B5%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9+%D0%BA%D0%B0%D0%BD%D0%B0%D0%BB+%D0%B4%D0%BB%D1%8F+%D0%BA%D0%B0%D0%B6%D0%B4%D0%BE%D0%B3%D0%BE+worker.%0A++++for+id+in+0..WORKERS+%7B%0A++++++++let+%28worker_tx%2C+worker_rx%29+%3D+mpsc%3A%3Achannel%3A%3A%3Cu32%3E%28%29%3B%0A%0A++++++++worker_senders.push%28worker_tx%29%3B%0A%0A++++++++let+handle+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++for+task+in+worker_rx+%7B%0A++++++++++++++++println%21%28%22Worker+%7Bid%7D+processing+task+%7Btask%7D%22%29%3B%0A%0A++++++++++++++++thread%3A%3Asleep%28Duration%3A%3Afrom_millis%28100%29%29%3B%0A++++++++++++%7D%0A%0A++++++++++++println%21%28%22Worker+%7Bid%7D+stopped%22%29%3B%0A++++++++%7D%29%3B%0A%0A++++++++worker_handles.push%28handle%29%3B%0A++++%7D%0A%0A++++%2F%2F+Dispatcher+%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B0%D0%B5%D1%82+%D0%B7%D0%B0%D0%B4%D0%B0%D1%87%D0%B8+%D0%B8%D0%B7+%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE+%D0%BA%D0%B0%D0%BD%D0%B0%D0%BB%D0%B0%0A++++%2F%2F+%D0%B8+%D1%80%D0%B0%D1%81%D0%BF%D1%80%D0%B5%D0%B4%D0%B5%D0%BB%D1%8F%D0%B5%D1%82+%D0%B8%D1%85+%D0%BC%D0%B5%D0%B6%D0%B4%D1%83+workers.%0A++++let+dispatcher+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++for+%28index%2C+task%29+in+rx.into_iter%28%29.enumerate%28%29+%7B%0A++++++++++++let+worker+%3D+index+%25+WORKERS%3B%0A%0A++++++++++++if+worker_senders%5Bworker%5D.send%28task%29.is_err%28%29+%7B%0A++++++++++++++++eprintln%21%28%22Worker+%7Bworker%7D+is+no+longer+available%22%29%3B%0A++++++++++++++++break%3B%0A++++++++++++%7D%0A++++++++%7D%0A%0A++++++++%2F%2F+%D0%97%D0%B0%D0%BA%D1%80%D1%8B%D0%B2%D0%B0%D0%B5%D0%BC+worker+channels.%0A++++++++%2F%2F+%D0%9F%D0%BE%D1%81%D0%BB%D0%B5+%D1%8D%D1%82%D0%BE%D0%B3%D0%BE+workers+%D0%B7%D0%B0%D0%B2%D0%B5%D1%80%D1%88%D0%B0%D1%82+%D1%81%D0%B2%D0%BE%D0%B8+%D1%86%D0%B8%D0%BA%D0%BB%D1%8B.%0A++++++++drop%28worker_senders%29%3B%0A++++%7D%29%3B%0A%0A++++%2F%2F+Producer+%D0%BE%D1%82%D0%BF%D1%80%D0%B0%D0%B2%D0%BB%D1%8F%D0%B5%D1%82+%D0%B7%D0%B0%D0%B4%D0%B0%D1%87%D0%B8.%0A++++for+task+in+0..20+%7B%0A++++++++tx.send%28task%29.unwrap%28%29%3B%0A++++%7D%0A%0A++++%2F%2F+%D0%97%D0%B0%D0%BA%D1%80%D1%8B%D0%B2%D0%B0%D0%B5%D0%BC+%D0%B2%D1%85%D0%BE%D0%B4%D0%BD%D0%BE%D0%B9+%D0%BA%D0%B0%D0%BD%D0%B0%D0%BB.%0A++++%2F%2F+Dispatcher+%D0%B7%D0%B0%D0%B2%D0%B5%D1%80%D1%88%D0%B8%D1%82+%D1%86%D0%B8%D0%BA%D0%BB+%D0%BF%D0%BE%D1%81%D0%BB%D0%B5+%D0%BE%D0%B1%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B8+%D0%B2%D1%81%D0%B5%D1%85+%D1%81%D0%BE%D0%BE%D0%B1%D1%89%D0%B5%D0%BD%D0%B8%D0%B9.%0A++++drop%28tx%29%3B%0A%0A++++dispatcher.join%28%29.unwrap%28%29%3B%0A%0A++++for+handle+in+worker_handles+%7B%0A++++++++handle.join%28%29.unwrap%28%29%3B%0A++++%7D%0A%0A++++println%21%28%22All+tasks+processed%21%22%29%3B%0A%7D%0A)

Возможный вывод:

```text
Worker 0 processing task 0
Worker 1 processing task 1
Worker 2 processing task 2
Worker 3 processing task 3
Worker 0 processing task 4
...
Worker 3 processing task 19
Worker 0 stopped
Worker 1 stopped
Worker 2 stopped
Worker 3 stopped
All tasks processed!
```

Порядок вывода может отличаться.

Здесь особенно важно понять жизненный цикл каналов.

```text
Producer
   │
   │ send()
   ▼
input channel
   │
   │ recv()
   ▼
Dispatcher
   │
   ├──▶ worker 0
   ├──▶ worker 1
   ├──▶ worker 2
   └──▶ worker 3
```

Когда producer больше не нужен:

```rust
drop(tx);
```

это сигнал dispatcher:

> новых задач больше не будет.

После завершения dispatcher уничтожаются `worker_senders`. Это, в свою очередь, сообщает каждому worker:

> новых задач больше не будет.

После этого `for task in worker_rx` завершается, и worker может корректно закончить работу.

Это важный паттерн Rust:

```text
drop(Sender)
      ↓
Receiver получает сигнал завершения
      ↓
цикл обработки завершается
      ↓
thread завершается
```

**Важно:** `std::sync::mpsc` — это не многопотребительский канал. Если требуется, чтобы несколько потоков напрямую получали задачи из одного `Receiver`, обычно используют другой channel implementation, например `crossbeam-channel`. В рамках этой главы мы сознательно остаёмся в стандартной библиотеке Rust и поэтому используем dispatcher.

---

## 26.7. Каналы как архитектурная граница

Канал можно рассматривать не просто как контейнер сообщений, а как **границу между компонентами системы**.

```text
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  Producer   │ ───▶  │   Channel   │ ───▶  │  Consumer   │
└─────────────┘       └─────────────┘       └─────────────┘
```

Producer знает, **что нужно отправить**.

Consumer знает, **что нужно сделать с полученным сообщением**.

При этом producer не обязан знать:

- где находится consumer;
- в каком потоке он работает;
- как долго он обрабатывает сообщение;
- сколько внутренних состояний у него есть.

Например:

```rust
enum Job {
    ResizeImage { width: u32, height: u32 },
    DeleteFile(String),
    SendEmail(String),
}
```

Producer может отправлять команды:

```rust
tx.send(Job::ResizeImage {
    width: 1920,
    height: 1080,
})?;
```

А consumer решает, что с ними делать.

Это позволяет строить pipeline:

```text
Input
  │
  ▼
Parser
  │
  ▼
Channel
  │
  ▼
Processor
  │
  ▼
Channel
  │
  ▼
Storage
```

Каждый компонент имеет относительно простую ответственность.

Однако канал **не является универсальной заменой shared state**.

Если нескольким потокам постоянно требуется доступ к одной и той же структуре данных, например к кэшу:

```text
Thread 1 ──┐
Thread 2 ──┼──▶ shared cache
Thread 3 ──┘
```

то `Arc<Mutex<_>>` может оказаться естественнее.

Если же задача выглядит как:

```text
Producer ──▶ command ──▶ Worker
```

то канал часто является более естественным решением.

---

## 26.8. Каналы vs Shared State

| Критерий           | Каналы                                       | Shared State (`Mutex`)                                     |
| ------------------ | -------------------------------------------- | ---------------------------------------------------------- |
| Основная модель    | Передача сообщений                           | Совместный доступ к состоянию                              |
| Владение данными   | Передаётся получателю                        | Остаётся общей собственностью                              |
| Синхронизация      | Встроена в канал                             | Требуется явно организовать                                |
| Блокировки         | Могут быть внутри channel implementation     | Явные `Mutex`/`RwLock`                                     |
| Deadlock           | Возможен, особенно при нескольких каналах    | Возможен при нескольких lock'ах                            |
| Производительность | Зависит от количества сообщений и их размера | Часто эффективнее для небольших частых изменений состояния |
| Backpressure       | `sync_channel`                               | Обычно реализуется отдельно                                |
| Worker pool        | Естественный паттерн                         | Возможен, но менее естественен                             |
| Общий кэш          | Неудобно                                     | Естественно                                                |
| Pipeline           | Очень удобно                                 | Обычно сложнее                                             |
| Передача владения  | Естественна                                  | Обычно требуется совместный доступ                         |
| Когда использовать | Команды, события, задачи, pipeline           | Счётчики, кэши, общие структуры                            |

Не стоит говорить:

> «Channels быстрее Mutex» или «Mutex быстрее Channels».

Это зависит от архитектуры и характера нагрузки.

Также `send()` **не означает автоматического копирования значения**:

```rust
let value = String::from("hello");

tx.send(value).unwrap();
```

Здесь `String` перемещается в канал:

```text
producer
   │
   │ ownership
   ▼
channel
   │
   │ ownership
   ▼
consumer
```

Для `String` не требуется создавать вторую копию строки только потому, что используется канал.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Что происходит после закрытия канала?

Следующий код **не содержит ошибки компиляции**:

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    drop(tx);

    let result = rx.recv();

    println!("{result:?}");
}
```

Результат:

```text
Err(RecvError)
```

Почему?

`recv()` возвращает:

```rust
Result<T, RecvError>
```

Если все `Sender` уничтожены и в канале больше нет сообщений, receiver получает сигнал:

> отправителей больше нет, новых сообщений не будет.

Поэтому закрытие канала — это не ошибка компилятора и не panic. Это нормальный сценарий выполнения.

---

### Эксперимент 2: Перемещение `Sender`

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(42).unwrap();
    });

    println!("Received: {}", rx.recv().unwrap());
}
```

Здесь `tx` перемещается в поток:

```rust
move || {
    tx.send(42).unwrap();
}
```

После этого использовать `tx` в `main` нельзя:

```rust
println!("{:?}", tx); // ❌ borrow of moved value
```

Это обычное правило ownership.

---

### Эксперимент 3: `sync_channel(0)`

Особенно интересен канал:

```rust
let (tx, rx) = mpsc::sync_channel(0);
```

Размер буфера равен нулю.

Это означает, что сообщение не может просто «полежать» в канале.

Отправитель и получатель должны встретиться:

```text
Sender                         Receiver

  send(message)
       │
       │  ожидание
       ├──────────────────────▶ recv()
       │
       │  передача
       ▼
   send() завершён
```

Например:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::sync_channel(0);

    let handle = thread::spawn(move || {
        println!("Before send");

        tx.send("Hello").unwrap();

        println!("After send");
    });

    thread::sleep(Duration::from_millis(500));

    println!("Before receive");

    let message = rx.recv().unwrap();

    println!("Received: {message}");

    handle.join().unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3Ampsc%3B%0Ause%20std%3A%3Athread%3B%0Ause%20std%3A%3Atime%3A%3ADuration%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20%28tx%2C%20rx%29%20%3D%20mpsc%3A%3Async_channel%280%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Before%20send%22%29%3B%0A%0A%20%20%20%20%20%20%20%20tx.send%28%22Hello%22%29.unwrap%28%29%3B%0A%0A%20%20%20%20%20%20%20%20println%21%28%22After%20send%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20thread%3A%3Asleep%28Duration%3A%3Afrom_millis%28500%29%29%3B%0A%0A%20%20%20%20println%21%28%22Before%20receive%22%29%3B%0A%0A%20%20%20%20let%20message%20%3D%20rx.recv%28%29.unwrap%28%29%3B%0A%0A%20%20%20%20println%21%28%22Received%3A%20%7Bmessage%7D%22%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D)

Первые две строки предсказуемы для этой конкретной программы: worker печатает `"Before send"` сразу после старта, а main-поток спит 500 мс, поэтому `"Before receive"` гарантированно печатается позже. Но дальше начинается неопределённость:

```text
Before send
Before receive
Received: Hello    ─┐
After send          ├── порядок этих двух строк НЕ гарантирован
```

`send()` в worker-потоке завершается ровно в тот момент, когда `recv()` в main-потоке успешно получает значение — это и есть точка рандеву. Но после этого момента оба потока продолжают выполнение **параллельно и независимо друг от друга**. Ничто в семантике `sync_channel(0)` не гарантирует, что main напечатает `"Received: Hello"` раньше, чем worker напечатает `"After send"`, или наоборот — реальный порядок этих двух строк может отличаться от запуска к запуску, точно так же, как в примере с несколькими producer'ами в §26.2.

`sync_channel(0)` гарантирует **момент передачи** значения (rendezvous) — то есть что `send()` не завершится, пока кто-то не вызовет `recv()`, и наоборот. Но эта гарантия не распространяется на то, что происходит в каждом потоке *после* этого момента.

Такой канал называется **rendezvous channel**.

---

## Практика

Практика этой главы должна не просто проверить знание API, а показать реальные проблемы многопоточного программирования.

### Задание 1. Общий счётчик

Создайте:

- `4` worker threads;
- общий счётчик;
- каждый worker должен увеличить счётчик `10_000` раз.

Сначала реализуйте решение с:

```rust
Arc<Mutex<u64>>
```

Ожидаемый результат:

```text
Expected: 40000
Actual:   40000
```

Пример:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0_u64));

    let mut handles = Vec::new();

    for _ in 0..4 {
        let counter = Arc::clone(&counter);

        handles.push(thread::spawn(move || {
            for _ in 0..10_000 {
                let mut value = counter.lock().unwrap();
                *value += 1;
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

Обратите внимание на критическую секцию:

```rust
let mut value = counter.lock().unwrap();
*value += 1;
```

Lock удерживается только во время изменения счётчика.

---

### Задание 2. Несколько worker threads

Создайте систему обработки задач:

```text
Producer
   │
   ▼
Channel
   │
   ▼
Dispatcher
   ├──▶ Worker 0
   ├──▶ Worker 1
   ├──▶ Worker 2
   └──▶ Worker 3
```

Producer должен отправить `20` задач:

```rust
0..20
```

Каждый worker должен вывести:

```text
Worker 2 processing task 7
```

После завершения всех задач программа должна вывести:

```text
All tasks processed!
```

Обязательно реализуйте корректное завершение:

```text
drop(tx)
   ↓
dispatcher завершает работу
   ↓
worker Sender уничтожаются
   ↓
workers завершаются
   ↓
join()
```

---

### Задание 3. Найдите race condition

Теперь попробуйте реализовать общий счётчик **без `Mutex`**.

Для этого используйте `AtomicU64`, но намеренно разделите операцию увеличения на две отдельные операции:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));

    let mut handles = Vec::new();

    for _ in 0..4 {
        let counter = Arc::clone(&counter);

        handles.push(thread::spawn(move || {
            for _ in 0..10_000 {
                let current = counter.load(Ordering::Relaxed);

                thread::yield_now();

                counter.store(current + 1, Ordering::Relaxed);
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", counter.load(Ordering::Relaxed));
}
```

Ожидаемый результат:

```text
Expected: 40000
```

Но фактический результат обычно будет меньше:

```text
Result: 13xxx
```

или другим значением.

Это **lost update**.

Два потока могут одновременно увидеть одно и то же значение:

```text
counter = 10

Worker A: load() → 10
Worker B: load() → 10

Worker A: store(11)
Worker B: store(11)

результат: 11
```

Хотя два увеличения должны были дать:

```text
12
```

Важно: `AtomicU64` сам по себе не делает произвольную последовательность операций атомарной.

Последовательность:

```rust
load()
store()
```

не является одной атомарной операцией.

---

### Задание 4. Исправьте race condition

Исправьте предыдущий пример двумя способами.

**Вариант A — `Mutex`:**

```rust
Arc<Mutex<u64>>
```

**Вариант B — атомарная операция:**

```rust
counter.fetch_add(1, Ordering::Relaxed);
```

Второй вариант:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));

    let mut handles = Vec::new();

    for _ in 0..4 {
        let counter = Arc::clone(&counter);

        handles.push(thread::spawn(move || {
            for _ in 0..10_000 {
                counter.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", counter.load(Ordering::Relaxed));
}
```

Теперь результат должен быть:

```text
Result: 40000
```

Это важный урок:

> Atomic-типы защищают отдельные атомарные операции, но не делают автоматически атомарной произвольную последовательность операций.

---

### Задание 5. Устраните общий счётчик с помощью каналов

Теперь решите ту же задачу **без общего mutable state**.

Пусть четыре worker threads не изменяют общий счётчик.

Вместо этого каждый worker отправляет результат своей работы:

```text
Worker 0 ──┐
Worker 1 ──┤
Worker 2 ──┼──▶ Channel ──▶ Main
Worker 3 ──┘
```

Каждый worker отправляет:

```rust
10_000
```

Главный поток суммирует полученные значения:

```rust
let total: u64 = rx.iter().sum();
```

Ожидаемый результат:

```text
40000
```

Таким образом, состояние счётчика вообще не является shared state.

---

### Задание 6. Эксперимент с deadlock

Создайте два `Mutex`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let first = Arc::new(Mutex::new(()));
    let second = Arc::new(Mutex::new(()));

    let first_a = Arc::clone(&first);
    let second_a = Arc::clone(&second);

    let thread_a = thread::spawn(move || {
        let _first = first_a.lock().unwrap();

        thread::sleep(Duration::from_millis(100));

        let _second = second_a.lock().unwrap();

        println!("Thread A finished");
    });

    let first_b = Arc::clone(&first);
    let second_b = Arc::clone(&second);

    let thread_b = thread::spawn(move || {
        let _second = second_b.lock().unwrap();

        thread::sleep(Duration::from_millis(100));

        let _first = first_b.lock().unwrap();

        println!("Thread B finished");
    });

    thread_a.join().unwrap();
    thread_b.join().unwrap();
}
```

Программа может зависнуть:

```text
Thread A:
    lock(first)
       ↓
    waiting for second

Thread B:
    lock(second)
       ↓
    waiting for first
```

Получается цикл:

```text
Thread A ──holds──▶ first
    │
    │ waits for
    ▼
 second ◀──holds── Thread B
    ▲
    │
    │ waits for
    │
Thread B
```

Это **deadlock**.

Простейший способ устранить эту проблему — всегда получать lock'и в одном и том же порядке:

```text
Thread A: first → second
Thread B: first → second
```

Например:

```rust
let _first = first.lock().unwrap();
let _second = second.lock().unwrap();
```

И для второго потока — в том же порядке.

Практическое правило:

> Если нескольким потокам нужны несколько lock'ов, установите единый порядок их получения и соблюдайте его во всей программе.

---

### Задание 7. Эксперимент с каналом и deadlock

Попробуйте создать deadlock уже с каналами.

Например:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::sync_channel(0);

    let handle = thread::spawn(move || {
        println!("Worker: sending...");

        tx.send(42).unwrap();

        println!("Worker: sent");
    });

    println!("Main: waiting for worker");

    handle.join().unwrap();

    println!("Main: receiving...");

    let value = rx.recv().unwrap();

    println!("Received: {value}");
}
```

Что произойдёт?

Главный поток делает:

```rust
handle.join()
```

и ждёт завершения worker.

Worker в это время делает:

```rust
tx.send(42)
```

Но `sync_channel(0)` требует, чтобы другой поток одновременно выполнял `recv()`.

Получается:

```text
Main
 │
 │ join()
 ▼
waiting for Worker
       ▲
       │
       │ Worker
       │
       │ send(42)
       ▼
waiting for recv()
```

Но `recv()` находится **после `join()`** и поэтому никогда не будет вызван.

Получается deadlock.

Исправление:

```rust
let value = rx.recv().unwrap();

handle.join().unwrap();
```

Теперь последовательность правильная:

```text
Worker ──send──▶ Main ──recv──▶ join()
```

Этот пример особенно полезен, потому что показывает:

> Каналы уменьшают количество проблем с shared state, но сами по себе не делают программу свободной от deadlock.

---

## Главное из этой главы

После этой главы мы понимаем:

- **канал (`channel`)** — механизм передачи сообщений между потоками;
- **`mpsc`** — Multiple Producer, Single Consumer;
- **`Sender`** можно клонировать для нескольких producers;
- **`Receiver` в `std::sync::mpsc` нельзя клонировать**;
- **`send()`** передаёт владение значением;
- **`recv()`** блокирует поток до получения сообщения или закрытия канала;
- **`try_recv()`** позволяет получать сообщения без блокировки;
- **`sync_channel()`** позволяет создавать bounded channels;
- **`sync_channel(0)`** создаёт rendezvous channel;
- закрытие всех `Sender` является естественным сигналом завершения для `Receiver`;
- **worker pool** можно построить на каналах;
- channels хорошо подходят для **pipeline, commands, events и task processing**;
- `Mutex` лучше подходит для непосредственного совместного доступа к состоянию;
- channels не гарантируют отсутствие deadlock;
- правильная архитектура каналов может вообще устранить необходимость в shared mutable state;
- `Atomic` гарантирует атомарность конкретных операций, но не произвольных последовательностей `load → modify → store`.

**Самая важная идея:**

> Каналы позволяют передавать владение данными между потоками и строить явные границы между компонентами. Но channels и shared state — не конкурирующие «правильный» и «неправильный» подходы. Это разные инструменты для разных задач.

На практике хороший Rust-код часто использует их вместе:

```text
                 Channel
Producer ─────────────────────▶ Worker
                                  │
                                  │
                              Arc<Mutex<_>>
                                  │
                                  ▼
                              Shared State
```

Главная задача программиста — не просто научиться использовать `channel()` или `Mutex`, а **выбрать модель владения и синхронизации, которая делает состояние программы понятным и корректным**.
