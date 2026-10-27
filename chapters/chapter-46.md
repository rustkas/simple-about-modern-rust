# Глава 46. `MaybeUninit`, `NonNull` и `UnsafeCell`

В предыдущей главе мы познакомились с сырыми указателями — основным инструментом для низкоуровневой работы с памятью. Но Rust предоставляет ещё несколько типов, которые помогают писать безопасный `unsafe` код: `MaybeUninit`, `NonNull` и `UnsafeCell`. Каждый из них решает свою задачу.

В этой главе мы разберёмся, зачем нужны эти типы, как они работают и как использовать их для создания безопасных абстракций.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 46.1. Что такое `MaybeUninit<T>`?

`MaybeUninit<T>` — это специальный тип для представления **памяти, которая может ещё не содержать корректно инициализированное значение `T`**.

Обычная переменная типа `T` всегда должна содержать корректное значение `T`. Например, Rust не позволяет прочитать переменную до её инициализации.

`MaybeUninit<T>` снимает именно это требование: значение внутри может быть ещё не создано.

При этом сам объект `MaybeUninit<T>` является корректным значением Rust. Опасность возникает только тогда, когда мы утверждаем компилятору, что внутри уже находится полноценный `T`.

Например:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut value = MaybeUninit::<i32>::uninit();

    // Записываем полноценное значение i32.
    value.write(42);

    // SAFETY: значение только что записано.
    let value = unsafe { value.assume_init() };

    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%20MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%97%D0%B0%D0%BF%D0%B8%D1%81%D1%8B%D0%B2%D0%B0%D0%B5%D0%BC%20%D0%BF%D0%BE%D0%BB%D0%BD%D0%BE%D1%86%D0%B5%D0%BD%D0%BD%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20i32.%0A%20%20%20%20value.write%2842%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D1%87%D1%82%D0%BE%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%BE.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20value.assume_init%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

Здесь `unsafe` требуется не для записи значения, а для операции `assume_init()`.

Метод `write()` безопасен, потому что он **создаёт значение непосредственно в предназначенной для него памяти**.

`assume_init()` устроен иначе: он говорит компилятору:

> «Я гарантирую, что эта память уже содержит корректно инициализированный `T`».

Если это утверждение ложно, возникает **undefined behavior (UB)**.

---

## 46.2. Зачем нужен `MaybeUninit`?

Главная причина существования `MaybeUninit` — ситуации, когда **память уже должна существовать, а значение будет создано позднее**.

Типичные случаи:

1. **Отложенная инициализация**

   Память выделяется заранее, а объект создаётся только после получения необходимых данных.

2. **Массивы и буферы**

   Например, нужно создать массив из 1024 элементов, но значения будут получены постепенно.

3. **FFI**

   C-функция может получать указатель на память, которую она сама заполнит:

   ```text
   Rust выделяет память
          ↓
   передаёт указатель C-функции
          ↓
   C заполняет память
          ↓
   Rust рассматривает память как T
   ```

4. **Низкоуровневые структуры данных**

   Например, `Vec<T>` должен уметь иметь выделенную ёмкость, в которой только первые `len` элементов являются инициализированными.

Последний случай особенно важен. Если `Vec<T>` имеет:

```text
capacity = 100
len      = 3
```

то память для 100 элементов может уже существовать, но только первые три позиции содержат реальные значения `T`.

Именно такие ситуации и позволяет моделировать `MaybeUninit<T>`.

---

## 46.3. Работа с `MaybeUninit<T>`

### Создание

Есть два основных варианта:

```rust
use std::mem::MaybeUninit;

let uninit: MaybeUninit<i32> = MaybeUninit::uninit();

let init: MaybeUninit<i32> = MaybeUninit::new(42);
```

`uninit()` создаёт `MaybeUninit<T>`, внутри которого значение `T` ещё не инициализировано.

`new(value)` сразу помещает туда корректное значение.

### Запись значения

Предпочтительный способ показан в первом примере главы: `value.write(42)`.

`write()` особенно полезен потому, что он не требует предварительно создавать значение `T` и не пытается уничтожить старое значение. Подробнее об этом — в разделе 46.9.

Низкоуровневый вариант через указатель также возможен:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut value = MaybeUninit::<i32>::uninit();

    // SAFETY: as_mut_ptr() указывает на память, подходящую для i32.
    unsafe {
        value.as_mut_ptr().write(42);
    }

    // SAFETY: значение только что записано.
    let value = unsafe { value.assume_init() };

    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%20MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20as_mut_ptr%28%29%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BD%D0%B0%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%2C%20%D0%BF%D0%BE%D0%B4%D1%85%D0%BE%D0%B4%D1%8F%D1%89%D1%83%D1%8E%20%D0%B4%D0%BB%D1%8F%20i32.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20value.as_mut_ptr%28%29.write%2842%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D1%87%D1%82%D0%BE%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%BE.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20value.assume_init%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

В обычном коде первый вариант предпочтительнее: он выражает намерение яснее и уменьшает количество `unsafe`.

### Чтение

Если значение уже инициализировано, можно получить его:

```rust
let value = unsafe { value.assume_init() };
```

Если нужно получить ссылку, существует:

```rust
let reference = unsafe { value.assume_init_ref() };
```

Но оба метода требуют от программиста гарантии, что `T` действительно инициализирован.

Нельзя делать так:

```rust
use std::mem::MaybeUninit;

fn main() {
    let value = MaybeUninit::<i32>::uninit();

    // UB: внутри нет инициализированного i32.
    let value = unsafe { value.assume_init() };

    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%3B%0A%0A%20%20%20%20%2F%2F%20UB%3A%20%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B8%20%D0%BD%D0%B5%D1%82%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D0%BE%D0%B3%D0%BE%20i32.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20value.assume_init%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

Это не «получение случайного значения». Это нарушение контракта `assume_init()` и **undefined behavior**. При запуске на rustc 1.99 в debug-сборке мы получили `1970457160`, а в release — `0`. Оба результата одинаково «правильны», потому что у программы с UB нет правильного результата.

---

### Частично инициализированный массив

Одна из наиболее полезных возможностей `MaybeUninit` — создание массива, элементы которого инициализируются постепенно.

Например:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut data: [MaybeUninit<i32>; 10] = [const { MaybeUninit::uninit() }; 10];

    for (index, slot) in data.iter_mut().enumerate().take(5) {
        slot.write(index as i32 * 10);
    }

    for (index, slot) in data.iter().enumerate().take(5) {
        // SAFETY: первые пять элементов инициализированы циклом выше.
        let value = unsafe { slot.assume_init_ref() };
        println!("data[{index}] = {value}");
    }
}
```

Результат:

```text
data[0] = 0
data[1] = 10
data[2] = 20
data[3] = 30
data[4] = 40
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20data%3A%20%5BMaybeUninit%3Ci32%3E%3B%2010%5D%20%3D%20%5Bconst%20%7B%20MaybeUninit%3A%3Auninit%28%29%20%7D%3B%2010%5D%3B%0A%0A%20%20%20%20for%20%28index%2C%20slot%29%20in%20data.iter_mut%28%29.enumerate%28%29.take%285%29%20%7B%0A%20%20%20%20%20%20%20%20slot.write%28index%20as%20i32%20%2A%2010%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20for%20%28index%2C%20slot%29%20in%20data.iter%28%29.enumerate%28%29.take%285%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BF%D0%B5%D1%80%D0%B2%D1%8B%D0%B5%20%D0%BF%D1%8F%D1%82%D1%8C%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%BE%D0%B2%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D1%8B%20%D1%86%D0%B8%D0%BA%D0%BB%D0%BE%D0%BC%20%D0%B2%D1%8B%D1%88%D0%B5.%0A%20%20%20%20%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20slot.assume_init_ref%28%29%20%7D%3B%0A%20%20%20%20%20%20%20%20println%21%28%22data%5B%7Bindex%7D%5D%20%3D%20%7Bvalue%7D%22%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Здесь важно понимать разницу:

```text
[MaybeUninit<i32>; 10]
```

и

```text
[i32; 10]
```

В первом случае каждый элемент **может быть неинициализирован**.

Во втором случае Rust гарантирует, что все десять элементов являются полноценными `i32`.

Поэтому нельзя просто сказать:

```rust
let data: [i32; 10] = unsafe {
    /* частично заполненный массив */
};
```

пока не доказано, что **каждый** элемент был инициализирован.

Когда все элементы инициализированы, массив можно превратить в обычный. Для этого безопасного способа в stable Rust пока нет, поэтому используется `transmute` с доказательством:

```rust
use std::mem::{transmute, MaybeUninit};

fn main() {
    let mut data: [MaybeUninit<i32>; 4] = [const { MaybeUninit::uninit() }; 4];

    for (index, slot) in data.iter_mut().enumerate() {
        slot.write(index as i32 + 1);
    }

    // SAFETY: все 4 элемента инициализированы, а MaybeUninit<i32> и i32
    // имеют одинаковые размер и выравнивание.
    let data: [i32; 4] = unsafe { transmute(data) };

    println!("{data:?}");
}
```

Результат:

```text
[1, 2, 3, 4]
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3A%7Btransmute%2C%20MaybeUninit%7D%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20data%3A%20%5BMaybeUninit%3Ci32%3E%3B%204%5D%20%3D%20%5Bconst%20%7B%20MaybeUninit%3A%3Auninit%28%29%20%7D%3B%204%5D%3B%0A%0A%20%20%20%20for%20%28index%2C%20slot%29%20in%20data.iter_mut%28%29.enumerate%28%29%20%7B%0A%20%20%20%20%20%20%20%20slot.write%28index%20as%20i32%20%2B%201%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B2%D1%81%D0%B5%204%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%B0%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D1%8B%2C%20%D0%B0%20MaybeUninit%3Ci32%3E%20%D0%B8%20i32%0A%20%20%20%20%2F%2F%20%D0%B8%D0%BC%D0%B5%D1%8E%D1%82%20%D0%BE%D0%B4%D0%B8%D0%BD%D0%B0%D0%BA%D0%BE%D0%B2%D1%8B%D0%B5%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%20%D0%B8%20%D0%B2%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5.%0A%20%20%20%20let%20data%3A%20%5Bi32%3B%204%5D%20%3D%20unsafe%20%7B%20transmute%28data%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bdata%3A%3F%7D%22%29%3B%0A%7D%0A)

`transmute` здесь допустим, потому что `MaybeUninit<i32>` и `i32` имеют одинаковый размер и выравнивание, а все элементы инициализированы.

#### Ловушка: `MaybeUninit` никогда не уничтожает содержимое сам

Самая частая ошибка в работе с массивами `MaybeUninit` — забыть, что `MaybeUninit<T>` **не вызывает `Drop` для `T`**. Он не знает, инициализировано ли значение внутри, и поэтому не рискует его уничтожать. Если в массиве лежат значения с деструктором, их придётся уничтожать вручную:

```rust
use std::mem::MaybeUninit;

struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    println!("--- массив из трёх слотов, инициализированы два");
    {
        let mut data: [MaybeUninit<Noisy>; 3] = [const { MaybeUninit::uninit() }; 3];
        data[0].write(Noisy("a"));
        data[1].write(Noisy("b"));
        // MaybeUninit никогда не уничтожает содержимое сам:
        // a и b будут «потеряны» без вызова Drop.
    }

    println!("--- то же, но с ручным уничтожением");
    {
        let mut data: [MaybeUninit<Noisy>; 3] = [const { MaybeUninit::uninit() }; 3];
        data[0].write(Noisy("a"));
        data[1].write(Noisy("b"));

        // SAFETY: слоты 0 и 1 инициализированы и больше не используются.
        unsafe {
            data[0].assume_init_drop();
            data[1].assume_init_drop();
        }
    }
}
```

Результат:

```text
--- массив из трёх слотов, инициализированы два
--- то же, но с ручным уничтожением
drop a
drop b
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Astruct%20Noisy%28%26%27static%20str%29%3B%0A%0Aimpl%20Drop%20for%20Noisy%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22drop%20%7B%7D%22%2C%20self.0%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22---%20%D0%BC%D0%B0%D1%81%D1%81%D0%B8%D0%B2%20%D0%B8%D0%B7%20%D1%82%D1%80%D1%91%D1%85%20%D1%81%D0%BB%D0%BE%D1%82%D0%BE%D0%B2%2C%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D1%8B%20%D0%B4%D0%B2%D0%B0%22%29%3B%0A%20%20%20%20%7B%0A%20%20%20%20%20%20%20%20let%20mut%20data%3A%20%5BMaybeUninit%3CNoisy%3E%3B%203%5D%20%3D%20%5Bconst%20%7B%20MaybeUninit%3A%3Auninit%28%29%20%7D%3B%203%5D%3B%0A%20%20%20%20%20%20%20%20data%5B0%5D.write%28Noisy%28%22a%22%29%29%3B%0A%20%20%20%20%20%20%20%20data%5B1%5D.write%28Noisy%28%22b%22%29%29%3B%0A%20%20%20%20%20%20%20%20%2F%2F%20MaybeUninit%20%D0%BD%D0%B8%D0%BA%D0%BE%D0%B3%D0%B4%D0%B0%20%D0%BD%D0%B5%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D0%B5%D1%82%20%D1%81%D0%BE%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D0%BC%D0%BE%D0%B5%20%D1%81%D0%B0%D0%BC%3A%0A%20%20%20%20%20%20%20%20%2F%2F%20a%20%D0%B8%20b%20%D0%B1%D1%83%D0%B4%D1%83%D1%82%20%C2%AB%D0%BF%D0%BE%D1%82%D0%B5%D1%80%D1%8F%D0%BD%D1%8B%C2%BB%20%D0%B1%D0%B5%D0%B7%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B0%20Drop.%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22---%20%D1%82%D0%BE%20%D0%B6%D0%B5%2C%20%D0%BD%D0%BE%20%D1%81%20%D1%80%D1%83%D1%87%D0%BD%D1%8B%D0%BC%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D0%B5%D0%BC%22%29%3B%0A%20%20%20%20%7B%0A%20%20%20%20%20%20%20%20let%20mut%20data%3A%20%5BMaybeUninit%3CNoisy%3E%3B%203%5D%20%3D%20%5Bconst%20%7B%20MaybeUninit%3A%3Auninit%28%29%20%7D%3B%203%5D%3B%0A%20%20%20%20%20%20%20%20data%5B0%5D.write%28Noisy%28%22a%22%29%29%3B%0A%20%20%20%20%20%20%20%20data%5B1%5D.write%28Noisy%28%22b%22%29%29%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%BB%D0%BE%D1%82%D1%8B%200%20%D0%B8%201%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D1%8B%20%D0%B8%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%BD%D0%B5%20%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D1%83%D1%8E%D1%82%D1%81%D1%8F.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20data%5B0%5D.assume_init_drop%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20data%5B1%5D.assume_init_drop%28%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A)

В первом блоке значения `a` и `b` пропали без вызова `Drop`: это **утечка**. Утечка не является UB (в Rust она считается безопасной), но если `Drop` освобождает ресурс (файл, сокет, память), ресурс останется занятым навсегда. Во втором блоке мы вручную уничтожаем ровно инициализированные слоты. Именно по такой схеме `Drop` работает в `MyVec` в следующем разделе.

---

## 46.4. `MaybeUninit` и безопасные абстракции

Настоящая ценность `MaybeUninit` раскрывается при создании собственных структур данных.

Например, рассмотрим упрощённый `Vec<T>`.

У него есть три важных состояния:

```text
ptr      → начало выделенной памяти
len      → количество инициализированных элементов
capacity → количество доступных элементов
```

Если:

```text
len = 3
capacity = 8
```

то память под восемь элементов существует, но значения `T` находятся только в первых трёх позициях:

```text
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ T  │ T  │ T  │ ?? │ ?? │ ?? │ ?? │ ?? │
└────┴────┴────┴────┴────┴────┴────┴────┘
  ↑              ↑
 len = 3     неинициализировано
```

Это именно тот сценарий, для которого предназначен `MaybeUninit`.

Ниже — законченная минимальная реализация вектора. Она не пытается повторить весь `std::vec::Vec`, но корректно демонстрирует основные инварианты: выделение памяти, `push`, `grow`, доступ к элементам и уничтожение элементов.

```rust
use std::alloc::{self, Layout};
use std::mem::{size_of, MaybeUninit};
use std::ptr::NonNull;

struct MyVec<T> {
    ptr: NonNull<MaybeUninit<T>>,
    len: usize,
    capacity: usize,
}

impl<T> MyVec<T> {
    fn new() -> Self {
        assert!(size_of::<T>() != 0, "типы нулевого размера не поддерживаются");

        Self {
            ptr: NonNull::dangling(),
            len: 0,
            capacity: 0,
        }
    }

    fn with_capacity(capacity: usize) -> Self {
        if capacity == 0 {
            return Self::new();
        }
        assert!(size_of::<T>() != 0, "типы нулевого размера не поддерживаются");

        let layout = Layout::array::<MaybeUninit<T>>(capacity)
            .expect("capacity overflow");

        // SAFETY: layout имеет ненулевой размер (T не нулевого размера, capacity > 0).
        let ptr = unsafe { alloc::alloc(layout) };

        let ptr = NonNull::new(ptr)
            .unwrap_or_else(|| alloc::handle_alloc_error(layout))
            .cast::<MaybeUninit<T>>();

        Self {
            ptr,
            len: 0,
            capacity,
        }
    }

    fn len(&self) -> usize {
        self.len
    }

    fn push(&mut self, value: T) {
        if self.len == self.capacity {
            self.grow();
        }

        // SAFETY: после grow выполняется len < capacity, поэтому слот с номером len
        // лежит внутри выделенной памяти и ещё не содержит значения.
        unsafe {
            self.ptr
                .as_ptr()
                .add(self.len)
                .write(MaybeUninit::new(value));
        }

        self.len += 1;
    }

    fn get(&self, index: usize) -> Option<&T> {
        if index >= self.len {
            return None;
        }

        // SAFETY: index < len, значит слот инициализирован.
        unsafe { Some((*self.ptr.as_ptr().add(index)).assume_init_ref()) }
    }

    fn grow(&mut self) {
        let new_capacity = if self.capacity == 0 {
            4
        } else {
            self.capacity.checked_mul(2).expect("capacity overflow")
        };

        let mut new_vec = Self::with_capacity(new_capacity);

        for index in 0..self.len {
            // SAFETY: слоты 0..len инициализированы; после чтения старая копия
            // считается перемещённой и больше нигде не используется.
            unsafe {
                let value = self.ptr.as_ptr().add(index).read().assume_init();
                new_vec.push(value);
            }
        }

        // Старая память освобождается БЕЗ вызова Drop для элементов:
        // они уже перемещены в new_vec.
        self.deallocate();

        self.ptr = new_vec.ptr;
        self.capacity = new_vec.capacity;

        // new_vec больше не владеет памятью: иначе его Drop освободил бы её второй раз.
        new_vec.ptr = NonNull::dangling();
        new_vec.capacity = 0;
        new_vec.len = 0;
    }

    fn deallocate(&mut self) {
        if self.capacity == 0 {
            return;
        }

        let layout = Layout::array::<MaybeUninit<T>>(self.capacity)
            .expect("capacity overflow");

        // SAFETY: память выделена с таким же layout в with_capacity.
        unsafe {
            alloc::dealloc(self.ptr.as_ptr().cast(), layout);
        }

        self.ptr = NonNull::dangling();
        self.capacity = 0;
    }
}

impl<T> Drop for MyVec<T> {
    fn drop(&mut self) {
        for index in 0..self.len {
            // SAFETY: слоты 0..len инициализированы и ещё не уничтожены.
            unsafe {
                (*self.ptr.as_ptr().add(index)).assume_init_drop();
            }
        }

        self.deallocate();
    }
}

struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let mut numbers = MyVec::with_capacity(2);

    numbers.push(String::from("one"));
    numbers.push(String::from("two"));
    numbers.push(String::from("three")); // вызовет grow

    println!("{}", numbers.get(0).unwrap());
    println!("{}", numbers.get(1).unwrap());
    println!("{}", numbers.get(2).unwrap());
    println!("len = {}, get(3) = {:?}", numbers.len(), numbers.get(3));

    // Проверим, что каждый элемент уничтожается ровно один раз.
    let mut noisy = MyVec::new();
    noisy.push(Noisy("x"));
    noisy.push(Noisy("y"));
    println!("конец main: сейчас должны уничтожиться x и y");
}
```

Результат:

```text
one
two
three
len = 3, get(3) = None
конец main: сейчас должны уничтожиться x и y
drop x
drop y
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aalloc%3A%3A%7Bself%2C%20Layout%7D%3B%0Ause%20std%3A%3Amem%3A%3A%7Bsize_of%2C%20MaybeUninit%7D%3B%0Ause%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Astruct%20MyVec%3CT%3E%20%7B%0A%20%20%20%20ptr%3A%20NonNull%3CMaybeUninit%3CT%3E%3E%2C%0A%20%20%20%20len%3A%20usize%2C%0A%20%20%20%20capacity%3A%20usize%2C%0A%7D%0A%0Aimpl%3CT%3E%20MyVec%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20assert%21%28size_of%3A%3A%3CT%3E%28%29%20%21%3D%200%2C%20%22%D1%82%D0%B8%D0%BF%D1%8B%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B3%D0%BE%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%D0%B0%20%D0%BD%D0%B5%20%D0%BF%D0%BE%D0%B4%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D0%B2%D0%B0%D1%8E%D1%82%D1%81%D1%8F%22%29%3B%0A%0A%20%20%20%20%20%20%20%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20ptr%3A%20NonNull%3A%3Adangling%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20len%3A%200%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20capacity%3A%200%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20with_capacity%28capacity%3A%20usize%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20if%20capacity%20%3D%3D%200%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20return%20Self%3A%3Anew%28%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20assert%21%28size_of%3A%3A%3CT%3E%28%29%20%21%3D%200%2C%20%22%D1%82%D0%B8%D0%BF%D1%8B%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B3%D0%BE%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%D0%B0%20%D0%BD%D0%B5%20%D0%BF%D0%BE%D0%B4%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D0%B2%D0%B0%D1%8E%D1%82%D1%81%D1%8F%22%29%3B%0A%0A%20%20%20%20%20%20%20%20let%20layout%20%3D%20Layout%3A%3Aarray%3A%3A%3CMaybeUninit%3CT%3E%3E%28capacity%29%0A%20%20%20%20%20%20%20%20%20%20%20%20.expect%28%22capacity%20overflow%22%29%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20layout%20%D0%B8%D0%BC%D0%B5%D0%B5%D1%82%20%D0%BD%D0%B5%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B9%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%20%28T%20%D0%BD%D0%B5%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B3%D0%BE%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%D0%B0%2C%20capacity%20%3E%200%29.%0A%20%20%20%20%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20alloc%3A%3Aalloc%28layout%29%20%7D%3B%0A%0A%20%20%20%20%20%20%20%20let%20ptr%20%3D%20NonNull%3A%3Anew%28ptr%29%0A%20%20%20%20%20%20%20%20%20%20%20%20.unwrap_or_else%28%7C%7C%20alloc%3A%3Ahandle_alloc_error%28layout%29%29%0A%20%20%20%20%20%20%20%20%20%20%20%20.cast%3A%3A%3CMaybeUninit%3CT%3E%3E%28%29%3B%0A%0A%20%20%20%20%20%20%20%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20ptr%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20len%3A%200%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20capacity%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20len%28%26self%29%20-%3E%20usize%20%7B%0A%20%20%20%20%20%20%20%20self.len%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20push%28%26mut%20self%2C%20value%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20if%20self.len%20%3D%3D%20self.capacity%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20self.grow%28%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BF%D0%BE%D1%81%D0%BB%D0%B5%20grow%20%D0%B2%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D1%8F%D0%B5%D1%82%D1%81%D1%8F%20len%20%3C%20capacity%2C%20%D0%BF%D0%BE%D1%8D%D1%82%D0%BE%D0%BC%D1%83%20%D1%81%D0%BB%D0%BE%D1%82%20%D1%81%20%D0%BD%D0%BE%D0%BC%D0%B5%D1%80%D0%BE%D0%BC%20len%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%BB%D0%B5%D0%B6%D0%B8%D1%82%20%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B8%20%D0%B2%D1%8B%D0%B4%D0%B5%D0%BB%D0%B5%D0%BD%D0%BD%D0%BE%D0%B9%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D0%B8%20%D0%B8%20%D0%B5%D1%89%D1%91%20%D0%BD%D0%B5%20%D1%81%D0%BE%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D1%82%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8F.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20self.ptr%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20.as_ptr%28%29%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20.add%28self.len%29%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20.write%28MaybeUninit%3A%3Anew%28value%29%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20self.len%20%2B%3D%201%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20get%28%26self%2C%20index%3A%20usize%29%20-%3E%20Option%3C%26T%3E%20%7B%0A%20%20%20%20%20%20%20%20if%20index%20%3E%3D%20self.len%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20return%20None%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20index%20%3C%20len%2C%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B8%D1%82%20%D1%81%D0%BB%D0%BE%D1%82%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20Some%28%28%2Aself.ptr.as_ptr%28%29.add%28index%29%29.assume_init_ref%28%29%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20grow%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20let%20new_capacity%20%3D%20if%20self.capacity%20%3D%3D%200%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%204%0A%20%20%20%20%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20self.capacity.checked_mul%282%29.expect%28%22capacity%20overflow%22%29%0A%20%20%20%20%20%20%20%20%7D%3B%0A%0A%20%20%20%20%20%20%20%20let%20mut%20new_vec%20%3D%20Self%3A%3Awith_capacity%28new_capacity%29%3B%0A%0A%20%20%20%20%20%20%20%20for%20index%20in%200..self.len%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%BB%D0%BE%D1%82%D1%8B%200..len%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D1%8B%3B%20%D0%BF%D0%BE%D1%81%D0%BB%D0%B5%20%D1%87%D1%82%D0%B5%D0%BD%D0%B8%D1%8F%20%D1%81%D1%82%D0%B0%D1%80%D0%B0%D1%8F%20%D0%BA%D0%BE%D0%BF%D0%B8%D1%8F%0A%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20%D1%81%D1%87%D0%B8%D1%82%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D1%89%D1%91%D0%BD%D0%BD%D0%BE%D0%B9%20%D0%B8%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%BD%D0%B8%D0%B3%D0%B4%D0%B5%20%D0%BD%D0%B5%20%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D1%83%D0%B5%D1%82%D1%81%D1%8F.%0A%20%20%20%20%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20let%20value%20%3D%20self.ptr.as_ptr%28%29.add%28index%29.read%28%29.assume_init%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20new_vec.push%28value%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%A1%D1%82%D0%B0%D1%80%D0%B0%D1%8F%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D0%91%D0%95%D0%97%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B0%20Drop%20%D0%B4%D0%BB%D1%8F%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%BE%D0%B2%3A%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%BE%D0%BD%D0%B8%20%D1%83%D0%B6%D0%B5%20%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D1%89%D0%B5%D0%BD%D1%8B%20%D0%B2%20new_vec.%0A%20%20%20%20%20%20%20%20self.deallocate%28%29%3B%0A%0A%20%20%20%20%20%20%20%20self.ptr%20%3D%20new_vec.ptr%3B%0A%20%20%20%20%20%20%20%20self.capacity%20%3D%20new_vec.capacity%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20new_vec%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%BD%D0%B5%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%B5%D1%82%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%D1%8E%3A%20%D0%B8%D0%BD%D0%B0%D1%87%D0%B5%20%D0%B5%D0%B3%D0%BE%20Drop%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B4%D0%B8%D0%BB%20%D0%B1%D1%8B%20%D0%B5%D1%91%20%D0%B2%D1%82%D0%BE%D1%80%D0%BE%D0%B9%20%D1%80%D0%B0%D0%B7.%0A%20%20%20%20%20%20%20%20new_vec.ptr%20%3D%20NonNull%3A%3Adangling%28%29%3B%0A%20%20%20%20%20%20%20%20new_vec.capacity%20%3D%200%3B%0A%20%20%20%20%20%20%20%20new_vec.len%20%3D%200%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20deallocate%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20if%20self.capacity%20%3D%3D%200%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20return%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20let%20layout%20%3D%20Layout%3A%3Aarray%3A%3A%3CMaybeUninit%3CT%3E%3E%28self.capacity%29%0A%20%20%20%20%20%20%20%20%20%20%20%20.expect%28%22capacity%20overflow%22%29%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%20%D0%B2%D1%8B%D0%B4%D0%B5%D0%BB%D0%B5%D0%BD%D0%B0%20%D1%81%20%D1%82%D0%B0%D0%BA%D0%B8%D0%BC%20%D0%B6%D0%B5%20layout%20%D0%B2%20with_capacity.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20alloc%3A%3Adealloc%28self.ptr.as_ptr%28%29.cast%28%29%2C%20layout%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20self.ptr%20%3D%20NonNull%3A%3Adangling%28%29%3B%0A%20%20%20%20%20%20%20%20self.capacity%20%3D%200%3B%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%3CT%3E%20Drop%20for%20MyVec%3CT%3E%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20for%20index%20in%200..self.len%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%BB%D0%BE%D1%82%D1%8B%200..len%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D1%8B%20%D0%B8%20%D0%B5%D1%89%D1%91%20%D0%BD%D0%B5%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B5%D0%BD%D1%8B.%0A%20%20%20%20%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%28%2Aself.ptr.as_ptr%28%29.add%28index%29%29.assume_init_drop%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20self.deallocate%28%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20Noisy%28%26%27static%20str%29%3B%0A%0Aimpl%20Drop%20for%20Noisy%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22drop%20%7B%7D%22%2C%20self.0%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20numbers%20%3D%20MyVec%3A%3Awith_capacity%282%29%3B%0A%0A%20%20%20%20numbers.push%28String%3A%3Afrom%28%22one%22%29%29%3B%0A%20%20%20%20numbers.push%28String%3A%3Afrom%28%22two%22%29%29%3B%0A%20%20%20%20numbers.push%28String%3A%3Afrom%28%22three%22%29%29%3B%20%2F%2F%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B5%D1%82%20grow%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20numbers.get%280%29.unwrap%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20numbers.get%281%29.unwrap%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20numbers.get%282%29.unwrap%28%29%29%3B%0A%20%20%20%20println%21%28%22len%20%3D%20%7B%7D%2C%20get%283%29%20%3D%20%7B%3A%3F%7D%22%2C%20numbers.len%28%29%2C%20numbers.get%283%29%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9F%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%B8%D0%BC%2C%20%D1%87%D1%82%D0%BE%20%D0%BA%D0%B0%D0%B6%D0%B4%D1%8B%D0%B9%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D1%80%D0%BE%D0%B2%D0%BD%D0%BE%20%D0%BE%D0%B4%D0%B8%D0%BD%20%D1%80%D0%B0%D0%B7.%0A%20%20%20%20let%20mut%20noisy%20%3D%20MyVec%3A%3Anew%28%29%3B%0A%20%20%20%20noisy.push%28Noisy%28%22x%22%29%29%3B%0A%20%20%20%20noisy.push%28Noisy%28%22y%22%29%29%3B%0A%20%20%20%20println%21%28%22%D0%BA%D0%BE%D0%BD%D0%B5%D1%86%20main%3A%20%D1%81%D0%B5%D0%B9%D1%87%D0%B0%D1%81%20%D0%B4%D0%BE%D0%BB%D0%B6%D0%BD%D1%8B%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B8%D1%82%D1%8C%D1%81%D1%8F%20x%20%D0%B8%20y%22%29%3B%0A%7D%0A)

Здесь `MaybeUninit<T>` используется не для того, чтобы «обмануть» Rust, а чтобы явно представить реальное состояние памяти:

```text
capacity
   ↓
┌────┬────┬────┬────┬────┐
│ T  │ T  │ T  │ ?? │ ?? │
└────┴────┴────┴────┴────┘
   ←── len ──→
```

`Drop` обязан уничтожить только первые `len` элементов. Неинициализированную память уничтожать нельзя.

`NonNull` здесь используется как удобное представление адреса выделенной памяти, а `MaybeUninit` — для представления ещё не созданных элементов.

### На что обратить внимание в реализации

**1. `grow` перемещает элементы побитовым чтением.** В цикле мы читаем каждый элемент (`read().assume_init()`) и добавляем его в новый вектор. После этого старая память освобождается **без вызова `Drop`** для элементов: они уже живут в новом буфере. Если бы мы уничтожили их ещё раз, получили бы двойное освобождение.

**2. Поля `new_vec` обнуляются в конце `grow`.** Мы забрали у `new_vec` буфер, поэтому его собственный `Drop` не должен ни уничтожать элементы, ни освобождать память — иначе это случилось бы второй раз. Именно для этого `ptr`, `capacity` и `len` сбрасываются.

**3. Типы нулевого размера не поддерживаются.** Для `T = ()` размер `Layout` равен нулю, а вызов `alloc::alloc` с нулевым размером — UB. Мы можем в этом убедиться:

```rust
use std::alloc::Layout;

fn main() {
    // Для типа нулевого размера размер Layout равен 0, а alloc::alloc
    // с нулевым размером — UB. Поэтому MyVec такие типы отвергает.
    let layout = Layout::array::<()>(4).unwrap();
    println!("size = {}", layout.size());
}
```

Результат:

```text
size = 0
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aalloc%3A%3ALayout%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20%2F%2F%20%D0%94%D0%BB%D1%8F%20%D1%82%D0%B8%D0%BF%D0%B0%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B3%D0%BE%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%D0%B0%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%20Layout%20%D1%80%D0%B0%D0%B2%D0%B5%D0%BD%200%2C%20%D0%B0%20alloc%3A%3Aalloc%0A%20%20%20%20%2F%2F%20%D1%81%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D1%8B%D0%BC%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%D0%BE%D0%BC%20%E2%80%94%20UB.%20%D0%9F%D0%BE%D1%8D%D1%82%D0%BE%D0%BC%D1%83%20MyVec%20%D1%82%D0%B0%D0%BA%D0%B8%D0%B5%20%D1%82%D0%B8%D0%BF%D1%8B%20%D0%BE%D1%82%D0%B2%D0%B5%D1%80%D0%B3%D0%B0%D0%B5%D1%82.%0A%20%20%20%20let%20layout%20%3D%20Layout%3A%3Aarray%3A%3A%3C%28%29%3E%284%29.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22size%20%3D%20%7B%7D%22%2C%20layout.size%28%29%29%3B%0A%7D%0A)

В реальном `Vec` такие типы обрабатываются отдельной веткой (память не выделяется вовсе). В нашей учебной реализации мы просто отвергаем их с понятной ошибкой: `MyVec::<()>::new()` вызывает `assert!` с сообщением «типы нулевого размера не поддерживаются».

**4. Чего здесь нет.** Реализация намеренно не содержит `pop`, `insert`, итераторов, защиты от паники внутри `grow` и `Send`/`Sync` (из-за `NonNull` тип по умолчанию не `Send` и не `Sync`). Всё это есть в `std::vec::Vec`, и в реальном коде следует использовать именно его.

---

## 46.5. `NonNull<T>` — ненулевой указатель

`NonNull<T>` — это тип, который представляет **ненулевой сырой указатель**.

Но очень важно не сделать из этого неправильный вывод:

> `NonNull<T>` гарантирует отсутствие `null`, но **не гарантирует, что указатель указывает на живой и корректный объект `T`**.

Например, указатель может быть:

* dangling;
* указывать на уже освобождённую память;
* иметь неправильное выравнивание;
* использоваться после окончания lifetime объекта.

Поэтому `NonNull<T>` не делает операции с памятью автоматически безопасными.

Простейший пример:

```rust
use std::ptr::NonNull;

fn main() {
    let value = 42;

    let ptr = NonNull::from(&value);

    // SAFETY: ptr получен из ссылки на живую переменную value.
    let reference = unsafe { ptr.as_ref() };

    println!("value = {reference}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20NonNull%3A%3Afrom%28%26value%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8%20%D0%BD%D0%B0%20%D0%B6%D0%B8%D0%B2%D1%83%D1%8E%20%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%BD%D1%83%D1%8E%20value.%0A%20%20%20%20let%20reference%20%3D%20unsafe%20%7B%20ptr.as_ref%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22value%20%3D%20%7Breference%7D%22%29%3B%0A%7D%0A)

Здесь указатель действительно указывает на живой `i32`, поэтому получение ссылки корректно. Метод `as_ref()` превращает `NonNull<T>` в `&T` (он `unsafe`, потому что валидность указателя компилятор проверить не может), а `as_ptr()` возвращает обычный `*mut T`.

### `Option<NonNull<T>>` и null pointer optimization

Одна из важных особенностей `NonNull` — возможность эффективно представлять опциональный указатель.

```rust
use std::mem::size_of;
use std::ptr::NonNull;

fn main() {
    println!("NonNull<i32>: {}", size_of::<NonNull<i32>>());
    println!("Option<NonNull<i32>>: {}", size_of::<Option<NonNull<i32>>>());
    println!("*mut i32: {}", size_of::<*mut i32>());
    println!("Option<*mut i32>: {}", size_of::<Option<*mut i32>>());
}
```

Результат (на 64-битной платформе):

```text
NonNull<i32>: 8
Option<NonNull<i32>>: 8
*mut i32: 8
Option<*mut i32>: 16
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3Asize_of%3B%0Ause%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22NonNull%3Ci32%3E%3A%20%7B%7D%22%2C%20size_of%3A%3A%3CNonNull%3Ci32%3E%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22Option%3CNonNull%3Ci32%3E%3E%3A%20%7B%7D%22%2C%20size_of%3A%3A%3COption%3CNonNull%3Ci32%3E%3E%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22%2Amut%20i32%3A%20%7B%7D%22%2C%20size_of%3A%3A%3C%2Amut%20i32%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22Option%3C%2Amut%20i32%3E%3A%20%7B%7D%22%2C%20size_of%3A%3A%3COption%3C%2Amut%20i32%3E%3E%28%29%29%3B%0A%7D%0A)

`Option<NonNull<T>>` имеет тот же размер, что и сам указатель: значение `None` представляется нулевым адресом, который `NonNull` никогда не принимает. Это называется **null pointer optimization (NPO)**, и для `NonNull` она гарантирована документацией. Для обычного `*mut i32`, который может быть нулевым, такой трюк невозможен: нужно дополнительное слово для хранения варианта `Option`, поэтому `Option<*mut i32>` занимает 16 байт.

Именно поэтому `NonNull` очень удобен для структур данных, где нужно представить:

```text
указатель есть
      или
указателя нет
```

без дополнительного поля для флага.

Ещё одно отличие `NonNull<T>` от `*mut T`: `NonNull<T>` **ковариантен** по `T`, а `*mut T` инвариантен. Для абстракций вроде векторов и списков это обычно именно то, что нужно (например, `MyVec<&'static str>` можно использовать там, где ожидается `MyVec<&'a str>`).

### `NonNull::dangling()`

В низкоуровневых структурах данных часто нужно представить «пустой» указатель, который при этом не является `null`.

Для этого существует:

```rust
use std::mem::align_of;
use std::ptr::NonNull;

fn main() {
    let ptr = NonNull::<i32>::dangling();

    println!("адрес: {:?}", ptr.as_ptr());
    println!("align_of::<i32>() = {}", align_of::<i32>());
    println!("не null: {}", !ptr.as_ptr().is_null());
}
```

Результат:

```text
адрес: 0x4
align_of::<i32>() = 4
не null: true
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3Aalign_of%3B%0Ause%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20ptr%20%3D%20NonNull%3A%3A%3Ci32%3E%3A%3Adangling%28%29%3B%0A%0A%20%20%20%20println%21%28%22%D0%B0%D0%B4%D1%80%D0%B5%D1%81%3A%20%7B%3A%3F%7D%22%2C%20ptr.as_ptr%28%29%29%3B%0A%20%20%20%20println%21%28%22align_of%3A%3A%3Ci32%3E%28%29%20%3D%20%7B%7D%22%2C%20align_of%3A%3A%3Ci32%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22%D0%BD%D0%B5%20null%3A%20%7B%7D%22%2C%20%21ptr.as_ptr%28%29.is_null%28%29%29%3B%0A%7D%0A)

Такой указатель **не означает, что по адресу действительно находится `i32`**. Его нельзя разыменовывать. Однако он **выровнен** для `T` (в примере его адрес равен `align_of::<i32>()`), поэтому им можно безопасно заполнять поле `ptr`, пока память не выделена.

Он используется как специальное значение, например, когда `capacity == 0`:

```rust
struct Buffer<T> {
    ptr: NonNull<T>,
    len: usize,
    capacity: usize,
}
```

Если `capacity == 0`, `ptr` может содержать `NonNull::dangling()`, потому что память всё равно не используется. Именно так `ptr` инициализируется в `MyVec::new()`.

Это важная идея:

> **Non-null ≠ valid.**

`NonNull` гарантирует только ненулевое значение указателя. Остальные гарантии должен обеспечить код, который им управляет.

---

## 46.6. `UnsafeCell<T>` — основа внутренней изменяемости

`UnsafeCell<T>` — фундаментальный примитив Rust для **interior mutability**, то есть изменения данных через `&self`.

Обычная ссылка:

```rust
&T
```

означает, что объект нельзя изменять через эту ссылку. Это не просто соглашение: компилятор **вправе предполагать**, что данные за `&T` не меняются, пока ссылка жива, и оптимизирует код исходя из этого. Изменение данных за `&T` в обход `UnsafeCell` — UB.

`UnsafeCell<T>` — единственный разрешённый способ отменить это предположение. Он сообщает компилятору:

> «Внутри этого объекта данные могут изменяться даже тогда, когда сам объект доступен через `&self`».

Рассмотрим учебную ячейку `MyCell<T>`. Начнём с наивной версии, которую легко написать неправильно:

```rust
use std::cell::UnsafeCell;

struct MyCell<T> {
    value: UnsafeCell<T>,
}

impl<T> MyCell<T> {
    fn new(value: T) -> Self {
        Self { value: UnsafeCell::new(value) }
    }

    fn set(&self, value: T) {
        // Записывает значение, но НЕ уничтожает старое.
        unsafe {
            self.value.get().write(value);
        }
    }
}

struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let cell = MyCell::new(Noisy("первое"));
    cell.set(Noisy("второе"));
    println!("конец main");
}
```

Результат:

```text
конец main
drop второе
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Acell%3A%3AUnsafeCell%3B%0A%0Astruct%20MyCell%3CT%3E%20%7B%0A%20%20%20%20value%3A%20UnsafeCell%3CT%3E%2C%0A%7D%0A%0Aimpl%3CT%3E%20MyCell%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28value%3A%20T%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%20value%3A%20UnsafeCell%3A%3Anew%28value%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20set%28%26self%2C%20value%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%97%D0%B0%D0%BF%D0%B8%D1%81%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%2C%20%D0%BD%D0%BE%20%D0%9D%D0%95%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D0%B5%D1%82%20%D1%81%D1%82%D0%B0%D1%80%D0%BE%D0%B5.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20self.value.get%28%29.write%28value%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20Noisy%28%26%27static%20str%29%3B%0A%0Aimpl%20Drop%20for%20Noisy%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22drop%20%7B%7D%22%2C%20self.0%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20cell%20%3D%20MyCell%3A%3Anew%28Noisy%28%22%D0%BF%D0%B5%D1%80%D0%B2%D0%BE%D0%B5%22%29%29%3B%0A%20%20%20%20cell.set%28Noisy%28%22%D0%B2%D1%82%D0%BE%D1%80%D0%BE%D0%B5%22%29%29%3B%0A%20%20%20%20println%21%28%22%D0%BA%D0%BE%D0%BD%D0%B5%D1%86%20main%22%29%3B%0A%7D%0A)

Обратите внимание: значение «первое» **вообще не было уничтожено**. Метод `set` записал новое значение через `write()`, а старое просто перезаписал, не вызвав для него `Drop`. Это не UB, но это утечка, и настоящий `Cell::set` так себя не ведёт. Правильная версия возвращает старое значение через `replace`, которое затем уничтожается:

```rust
use std::cell::UnsafeCell;
use std::mem;

struct MyCell<T> {
    value: UnsafeCell<T>,
}

impl<T> MyCell<T> {
    fn new(value: T) -> Self {
        Self {
            value: UnsafeCell::new(value),
        }
    }

    fn replace(&self, value: T) -> T {
        // SAFETY: MyCell не Sync, ссылки на содержимое наружу не выдаются,
        // поэтому в этот момент к значению больше никто не обращается.
        unsafe { mem::replace(&mut *self.value.get(), value) }
    }

    fn set(&self, value: T) {
        // Старое значение возвращается из replace и тут же уничтожается.
        drop(self.replace(value));
    }

    fn get(&self) -> T
    where
        T: Copy,
    {
        // SAFETY: см. replace; T: Copy, поэтому чтение — это копия.
        unsafe { *self.value.get() }
    }
}

struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let cell = MyCell::new(42);

    println!("{}", cell.get());
    cell.set(100);
    println!("{}", cell.get());

    let noisy = MyCell::new(Noisy("первое"));
    noisy.set(Noisy("второе"));
    println!("конец main");
}
```

Результат:

```text
42
100
drop первое
конец main
drop второе
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Acell%3A%3AUnsafeCell%3B%0Ause%20std%3A%3Amem%3B%0A%0Astruct%20MyCell%3CT%3E%20%7B%0A%20%20%20%20value%3A%20UnsafeCell%3CT%3E%2C%0A%7D%0A%0Aimpl%3CT%3E%20MyCell%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28value%3A%20T%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20value%3A%20UnsafeCell%3A%3Anew%28value%29%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20replace%28%26self%2C%20value%3A%20T%29%20-%3E%20T%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20MyCell%20%D0%BD%D0%B5%20Sync%2C%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8%20%D0%BD%D0%B0%20%D1%81%D0%BE%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D0%BC%D0%BE%D0%B5%20%D0%BD%D0%B0%D1%80%D1%83%D0%B6%D1%83%20%D0%BD%D0%B5%20%D0%B2%D1%8B%D0%B4%D0%B0%D1%8E%D1%82%D1%81%D1%8F%2C%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%BF%D0%BE%D1%8D%D1%82%D0%BE%D0%BC%D1%83%20%D0%B2%20%D1%8D%D1%82%D0%BE%D1%82%20%D0%BC%D0%BE%D0%BC%D0%B5%D0%BD%D1%82%20%D0%BA%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8E%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%BD%D0%B8%D0%BA%D1%82%D0%BE%20%D0%BD%D0%B5%20%D0%BE%D0%B1%D1%80%D0%B0%D1%89%D0%B0%D0%B5%D1%82%D1%81%D1%8F.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20mem%3A%3Areplace%28%26mut%20%2Aself.value.get%28%29%2C%20value%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20set%28%26self%2C%20value%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%A1%D1%82%D0%B0%D1%80%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%B2%D0%BE%D0%B7%D0%B2%D1%80%D0%B0%D1%89%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D0%B8%D0%B7%20replace%20%D0%B8%20%D1%82%D1%83%D1%82%20%D0%B6%D0%B5%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D0%B5%D1%82%D1%81%D1%8F.%0A%20%20%20%20%20%20%20%20drop%28self.replace%28value%29%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20get%28%26self%29%20-%3E%20T%0A%20%20%20%20where%0A%20%20%20%20%20%20%20%20T%3A%20Copy%2C%0A%20%20%20%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%BC.%20replace%3B%20T%3A%20Copy%2C%20%D0%BF%D0%BE%D1%8D%D1%82%D0%BE%D0%BC%D1%83%20%D1%87%D1%82%D0%B5%D0%BD%D0%B8%D0%B5%20%E2%80%94%20%D1%8D%D1%82%D0%BE%20%D0%BA%D0%BE%D0%BF%D0%B8%D1%8F.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20%2Aself.value.get%28%29%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20Noisy%28%26%27static%20str%29%3B%0A%0Aimpl%20Drop%20for%20Noisy%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22drop%20%7B%7D%22%2C%20self.0%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20cell%20%3D%20MyCell%3A%3Anew%2842%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20cell.get%28%29%29%3B%0A%20%20%20%20cell.set%28100%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20cell.get%28%29%29%3B%0A%0A%20%20%20%20let%20noisy%20%3D%20MyCell%3A%3Anew%28Noisy%28%22%D0%BF%D0%B5%D1%80%D0%B2%D0%BE%D0%B5%22%29%29%3B%0A%20%20%20%20noisy.set%28Noisy%28%22%D0%B2%D1%82%D0%BE%D1%80%D0%BE%D0%B5%22%29%29%3B%0A%20%20%20%20println%21%28%22%D0%BA%D0%BE%D0%BD%D0%B5%D1%86%20main%22%29%3B%0A%7D%0A)

Теперь «первое» уничтожается сразу при вызове `set`.

Обратите внимание на комментарий `SAFETY` у `replace`: корректность опирается на два свойства `MyCell`, которые мы должны сохранять. Во-первых, **ссылки на содержимое наружу не выдаются** (методы возвращают значения, а не `&T`), поэтому нигде не может существовать ссылка, которую замена испортит. Во-вторых, `MyCell` **не `Sync`** — об этом ниже.

Но этот пример нужно воспринимать именно как **учебный**.

`UnsafeCell` не проверяет:

* отсутствие одновременных конфликтующих доступов;
* корректность aliasing;
* потокобезопасность;
* время жизни ссылок.

Всё это должен обеспечить автор абстракции.

### `UnsafeCell` не означает «теперь можно делать что угодно»

Например:

```rust
struct BadCell<T> {
    value: T,
}
```

через `&BadCell<T>` нельзя произвольно получить `&mut T`.

Но:

```rust
struct GoodCell<T> {
    value: UnsafeCell<T>,
}
```

позволяет получить сырой указатель:

```rust
let ptr: *mut T = cell.value.get();
```

После этого операции через указатель находятся в `unsafe`-области, и программист отвечает за соблюдение правил Rust.

Именно поэтому `UnsafeCell` — **не готовая безопасная ячейка**, а строительный блок для её реализации.

У `UnsafeCell` есть и безопасные методы, которые не требуют доверия, потому что используют уникальный доступ: `get_mut(&mut self) -> &mut T` и `into_inner(self) -> T`.

### `UnsafeCell` по умолчанию не `Sync`

`UnsafeCell<T>` намеренно **не реализует `Sync`**. Значит, любой тип, содержащий его, **автоматически** тоже не `Sync`, и компилятор не позволит разделить ссылку на него между потоками:

```rust
use std::cell::UnsafeCell;
use std::thread;

struct MyCell<T> {
    value: UnsafeCell<T>,
}

fn main() {
    let cell = MyCell { value: UnsafeCell::new(0) };

    thread::scope(|scope| {
        scope.spawn(|| {
            let _ = &cell;
        });
    });
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Acell%3A%3AUnsafeCell%3B%0Ause%20std%3A%3Athread%3B%0A%0Astruct%20MyCell%3CT%3E%20%7B%0A%20%20%20%20value%3A%20UnsafeCell%3CT%3E%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20cell%20%3D%20MyCell%20%7B%20value%3A%20UnsafeCell%3A%3Anew%280%29%20%7D%3B%0A%0A%20%20%20%20thread%3A%3Ascope%28%7Cscope%7C%20%7B%0A%20%20%20%20%20%20%20%20scope.spawn%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20let%20_%20%3D%20%26cell%3B%0A%20%20%20%20%20%20%20%20%7D%29%3B%0A%20%20%20%20%7D%29%3B%0A%7D%0A)

Компилятор откажется собирать этот код:

```text
error[E0277]: `UnsafeCell<i32>` cannot be shared between threads safely
```

Получить `Sync` для `MyCell` можно только вручную: `unsafe impl<T> Sync for MyCell<T> {}`. Но это было бы ложным обещанием: два потока смогли бы вызывать `set` одновременно, то есть произошла бы гонка данных. Поэтому потокобезопасные ячейки (`Mutex`, `RwLock`, атомарные типы) добавляют синхронизацию к `UnsafeCell`, а не просто объявляют его `Sync`.

---

## 46.7. `UnsafeCell`, `Cell`, `RefCell` и `Mutex`

Эти типы находятся на разных уровнях абстракции:

```text
UnsafeCell<T>
      │
      ├── Cell<T>
      │      └── простая внутренняя изменяемость
      │
      ├── RefCell<T>
      │      └── проверка правил заимствования во время выполнения
      │
      ├── OnceCell<T> / LazyCell<T>
      │      └── однократная инициализация
      │
      └── потокобезопасные примитивы
             ├── Mutex<T>
             └── RwLock<T>
```

Например, `Cell<T>` позволяет менять значение через `&self` без `unsafe` в пользовательском коде:

```rust
use std::cell::Cell;

fn main() {
    let value = Cell::new(10);

    value.set(20);

    println!("{}", value.get());
}
```

Результат:

```text
20
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Acell%3A%3ACell%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20Cell%3A%3Anew%2810%29%3B%0A%0A%20%20%20%20value.set%2820%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20value.get%28%29%29%3B%0A%7D%0A)

`Cell<T>` подходит для простых случаев, когда значение можно получать и заменять целиком.

`RefCell<T>` нужен, когда необходимо получать ссылки на внутреннее значение, но проверка правил заимствования переносится с compile time на runtime.

`Mutex<T>` решает уже другую задачу — синхронизацию доступа между потоками.

Таким образом, пользователю библиотеки обычно не нужно работать непосредственно с `UnsafeCell`. Он используется **внутри более высокоуровневых безопасных абстракций**.

---

## 46.8. Сравнение: `MaybeUninit` vs `NonNull` vs `UnsafeCell`

| Тип              | Основная задача                                               | Что гарантирует                                                             | Что не гарантирует                                    |
| ---------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------- |
| `MaybeUninit<T>` | Представить память, где `T` может быть ещё не инициализирован | Сам `MaybeUninit<T>` корректен даже без `T`                                 | Что `T` инициализирован; что `T` будет уничтожен      |
| `NonNull<T>`     | Представить ненулевой указатель                               | Указатель не равен null                                                     | Что объект существует и указатель можно разыменовать  |
| `UnsafeCell<T>`  | Разрешить interior mutability                                 | Компилятор не применяет обычное правило `&T` → неизменяемость к содержимому | Отсутствие data race, aliasing UB и другие инварианты |

Очень полезно запомнить их как три разных вопроса:

```text
MaybeUninit
    ↓
«Существует ли здесь уже T?»

NonNull
    ↓
«Может ли указатель быть null?»

UnsafeCell
    ↓
«Можно ли изменять содержимое через &T?»
```

Но ни один из этих типов сам по себе не превращает `unsafe` код в безопасный.

---

## 46.9. Типичные ошибки при работе с `MaybeUninit`

### Ошибка 1: `assume_init()` без инициализации

Пример из раздела 46.3 уже показал: `assume_init()` на неинициализированном `MaybeUninit<i32>` — UB, и результат зависит от режима сборки.

`assume_init()` требует доказательства, что внутри уже находится корректный `i32`.

Компилятор не может проверить это условие — ответственность лежит на программисте.

---

### Ошибка 2: повторная запись теряет старое значение

Метод `write()` **не уничтожает** старое значение: он просто записывает новое поверх. Для типов без деструктора (числа) это незаметно. Для типов с `Drop` старое значение пропадает без вызова деструктора:

```rust
use std::mem::MaybeUninit;

struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let mut slot = MaybeUninit::<Noisy>::uninit();

    slot.write(Noisy("first"));
    slot.write(Noisy("second")); // first перезаписан и НЕ уничтожен

    // SAFETY: слот инициализирован значением "second".
    let value = unsafe { slot.assume_init() };

    println!("итоговое значение: {}", value.0);
}
```

Результат:

```text
итоговое значение: second
drop second
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Astruct%20Noisy%28%26%27static%20str%29%3B%0A%0Aimpl%20Drop%20for%20Noisy%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22drop%20%7B%7D%22%2C%20self.0%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20slot%20%3D%20MaybeUninit%3A%3A%3CNoisy%3E%3A%3Auninit%28%29%3B%0A%0A%20%20%20%20slot.write%28Noisy%28%22first%22%29%29%3B%0A%20%20%20%20slot.write%28Noisy%28%22second%22%29%29%3B%20%2F%2F%20first%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%20%D0%B8%20%D0%9D%D0%95%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B5%D0%BD%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%BB%D0%BE%D1%82%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%D0%BC%20%22second%22.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20slot.assume_init%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%D0%B8%D1%82%D0%BE%D0%B3%D0%BE%D0%B2%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%3A%20%7B%7D%22%2C%20value.0%29%3B%0A%7D%0A)

Строки `drop first` нет. Это утечка, а не UB: в Rust утечка памяти считается безопасной. Но если `Drop` освобождает файл или сокет, ресурс останется занятым.

Правильная повторная запись — сначала уничтожить старое значение:

```rust
use std::mem::MaybeUninit;

struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let mut slot = MaybeUninit::<Noisy>::uninit();

    slot.write(Noisy("first"));

    // SAFETY: слот инициализирован, и мы уничтожаем его перед повторной записью.
    unsafe {
        slot.assume_init_drop();
    }

    slot.write(Noisy("second"));

    // SAFETY: слот снова инициализирован.
    let value = unsafe { slot.assume_init() };

    println!("итоговое значение: {}", value.0);
}
```

Результат:

```text
drop first
итоговое значение: second
drop second
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Astruct%20Noisy%28%26%27static%20str%29%3B%0A%0Aimpl%20Drop%20for%20Noisy%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22drop%20%7B%7D%22%2C%20self.0%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20slot%20%3D%20MaybeUninit%3A%3A%3CNoisy%3E%3A%3Auninit%28%29%3B%0A%0A%20%20%20%20slot.write%28Noisy%28%22first%22%29%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%BB%D0%BE%D1%82%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%2C%20%D0%B8%20%D0%BC%D1%8B%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D0%B5%D0%BC%20%D0%B5%D0%B3%D0%BE%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%20%D0%BF%D0%BE%D0%B2%D1%82%D0%BE%D1%80%D0%BD%D0%BE%D0%B9%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8C%D1%8E.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20slot.assume_init_drop%28%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20slot.write%28Noisy%28%22second%22%29%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%BB%D0%BE%D1%82%20%D1%81%D0%BD%D0%BE%D0%B2%D0%B0%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20slot.assume_init%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%D0%B8%D1%82%D0%BE%D0%B3%D0%BE%D0%B2%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%3A%20%7B%7D%22%2C%20value.0%29%3B%0A%7D%0A)

Для этого `assume_init_drop()` требует, чтобы слот действительно был инициализирован: вызов на неинициализированном слоте — снова UB.

---

### Ошибка 3: неправильное выравнивание

`MaybeUninit<T>` гарантирует корректное выравнивание **для `T`**.

Но нельзя произвольно преобразовывать указатель на один тип в указатель на другой:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut bytes = MaybeUninit::<[u8; 4]>::uninit();

    let ptr = bytes.as_mut_ptr() as *mut i32;

    // Допустимо только если адрес случайно оказался кратен 4. Гарантий нет.
    println!("выровнен для i32: {}", ptr.is_aligned());
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20bytes%20%3D%20MaybeUninit%3A%3A%3C%5Bu8%3B%204%5D%3E%3A%3Auninit%28%29%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20bytes.as_mut_ptr%28%29%20as%20%2Amut%20i32%3B%0A%0A%20%20%20%20%2F%2F%20%D0%94%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D0%BC%D0%BE%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D0%B5%D1%81%D0%BB%D0%B8%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%D1%81%D0%BB%D1%83%D1%87%D0%B0%D0%B9%D0%BD%D0%BE%20%D0%BE%D0%BA%D0%B0%D0%B7%D0%B0%D0%BB%D1%81%D1%8F%20%D0%BA%D1%80%D0%B0%D1%82%D0%B5%D0%BD%204.%20%D0%93%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D0%B9%20%D0%BD%D0%B5%D1%82.%0A%20%20%20%20println%21%28%22%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20i32%3A%20%7B%7D%22%2C%20ptr.is_aligned%28%29%29%3B%0A%7D%0A)

У `[u8; 4]` выравнивание равно 1, а у `i32` — 4. Адрес `bytes` может случайно оказаться кратным 4, а может и нет: гарантий нет. Если бы мы после этого вызвали `ptr.write(42)`, программа была бы корректной только при удаче. Такой код нельзя считать корректным только потому, что размер `[u8; 4]` равен размеру `i32`.

Есть три правильных способа:

```rust
use std::mem::MaybeUninit;

#[repr(align(4))]
struct Aligned([u8; 4]);

fn main() {
    // Способ 1: сразу взять память под нужный тип.
    let mut number = MaybeUninit::<i32>::uninit();
    number.write(42);
    // SAFETY: значение только что записано.
    println!("{}", unsafe { number.assume_init() });

    // Способ 2: если байты всё же нужны, задать выравнивание явно.
    let mut bytes = MaybeUninit::<Aligned>::uninit();
    let ptr = bytes.as_mut_ptr() as *mut i32;
    println!("выровнен для i32: {}", ptr.is_aligned());

    // Способ 3: невыровненная запись.
    let mut raw = MaybeUninit::<[u8; 4]>::uninit();
    // SAFETY: в raw помещается 4 байта; write_unaligned не требует выравнивания.
    unsafe {
        (raw.as_mut_ptr() as *mut i32).write_unaligned(42);
    }
    // SAFETY: все 4 байта записаны.
    println!("{:?}", unsafe { raw.assume_init() });
}
```

Результат:

```text
42
выровнен для i32: true
[42, 0, 0, 0]
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0A%23%5Brepr%28align%284%29%29%5D%0Astruct%20Aligned%28%5Bu8%3B%204%5D%29%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20%2F%2F%20%D0%A1%D0%BF%D0%BE%D1%81%D0%BE%D0%B1%201%3A%20%D1%81%D1%80%D0%B0%D0%B7%D1%83%20%D0%B2%D0%B7%D1%8F%D1%82%D1%8C%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%20%D0%BF%D0%BE%D0%B4%20%D0%BD%D1%83%D0%B6%D0%BD%D1%8B%D0%B9%20%D1%82%D0%B8%D0%BF.%0A%20%20%20%20let%20mut%20number%20%3D%20MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%3B%0A%20%20%20%20number.write%2842%29%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D1%87%D1%82%D0%BE%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%BE.%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20unsafe%20%7B%20number.assume_init%28%29%20%7D%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%A1%D0%BF%D0%BE%D1%81%D0%BE%D0%B1%202%3A%20%D0%B5%D1%81%D0%BB%D0%B8%20%D0%B1%D0%B0%D0%B9%D1%82%D1%8B%20%D0%B2%D1%81%D1%91%20%D0%B6%D0%B5%20%D0%BD%D1%83%D0%B6%D0%BD%D1%8B%2C%20%D0%B7%D0%B0%D0%B4%D0%B0%D1%82%D1%8C%20%D0%B2%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20%D1%8F%D0%B2%D0%BD%D0%BE.%0A%20%20%20%20let%20mut%20bytes%20%3D%20MaybeUninit%3A%3A%3CAligned%3E%3A%3Auninit%28%29%3B%0A%20%20%20%20let%20ptr%20%3D%20bytes.as_mut_ptr%28%29%20as%20%2Amut%20i32%3B%0A%20%20%20%20println%21%28%22%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20i32%3A%20%7B%7D%22%2C%20ptr.is_aligned%28%29%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%A1%D0%BF%D0%BE%D1%81%D0%BE%D0%B1%203%3A%20%D0%BD%D0%B5%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%D0%BD%D0%B0%D1%8F%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8C.%0A%20%20%20%20let%20mut%20raw%20%3D%20MaybeUninit%3A%3A%3C%5Bu8%3B%204%5D%3E%3A%3Auninit%28%29%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B2%20raw%20%D0%BF%D0%BE%D0%BC%D0%B5%D1%89%D0%B0%D0%B5%D1%82%D1%81%D1%8F%204%20%D0%B1%D0%B0%D0%B9%D1%82%D0%B0%3B%20write_unaligned%20%D0%BD%D0%B5%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82%20%D0%B2%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%28raw.as_mut_ptr%28%29%20as%20%2Amut%20i32%29.write_unaligned%2842%29%3B%0A%20%20%20%20%7D%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B2%D1%81%D0%B5%204%20%D0%B1%D0%B0%D0%B9%D1%82%D0%B0%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D1%8B.%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20unsafe%20%7B%20raw.assume_init%28%29%20%7D%29%3B%0A%7D%0A)

1. **Сразу брать память под нужный тип**: `MaybeUninit::<i32>::uninit()`.
2. **Задать выравнивание явно** обёрткой с `#[repr(align(4))]`.
3. **Использовать невыровненную запись** `write_unaligned()`, которая не требует кратного 4 адреса.

---

## 46.10. `MaybeUninit` и FFI

Рассмотрим типичный сценарий FFI.

Представим C-функцию:

```c
void fill_buffer(int32_t *buffer, size_t count);
```

Она получает указатель и заполняет `count` элементов.

В Rust память можно подготовить как `[MaybeUninit<i32>; N]` и передать указатель на неё. В нашем примере вместо воображаемой `fill_buffer` мы используем настоящую функцию `memset` из libc, которая заполняет область памяти заданным байтом:

```rust
use std::ffi::{c_int, c_void};
use std::mem::MaybeUninit;

unsafe extern "C" {
    // memset из libc: заполняет n байт значением c.
    fn memset(s: *mut c_void, c: c_int, n: usize) -> *mut c_void;
}

fn main() {
    let mut buffer = [const { MaybeUninit::<i32>::uninit() }; 4];

    // SAFETY: buffer.len() * size_of::<i32>() байт принадлежат нам и доступны для записи.
    // memset записывает каждый байт, значит все 4 элемента будут инициализированы.
    unsafe {
        memset(
            buffer.as_mut_ptr().cast::<c_void>(),
            1,
            std::mem::size_of_val(&buffer),
        );
    }

    for item in &buffer {
        // SAFETY: memset заполнил всю область; любой набор байт — валидный i32.
        let value = unsafe { item.assume_init_ref() };
        println!("{value}");
    }
}
```

Результат:

```text
16843009
16843009
16843009
16843009
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_int%2C%20c_void%7D%3B%0Ause%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20%2F%2F%20memset%20%D0%B8%D0%B7%20libc%3A%20%D0%B7%D0%B0%D0%BF%D0%BE%D0%BB%D0%BD%D1%8F%D0%B5%D1%82%20n%20%D0%B1%D0%B0%D0%B9%D1%82%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%D0%BC%20c.%0A%20%20%20%20fn%20memset%28s%3A%20%2Amut%20c_void%2C%20c%3A%20c_int%2C%20n%3A%20usize%29%20-%3E%20%2Amut%20c_void%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20buffer%20%3D%20%5Bconst%20%7B%20MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%20%7D%3B%204%5D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20buffer.len%28%29%20%2A%20size_of%3A%3A%3Ci32%3E%28%29%20%D0%B1%D0%B0%D0%B9%D1%82%20%D0%BF%D1%80%D0%B8%D0%BD%D0%B0%D0%B4%D0%BB%D0%B5%D0%B6%D0%B0%D1%82%20%D0%BD%D0%B0%D0%BC%20%D0%B8%20%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF%D0%BD%D1%8B%20%D0%B4%D0%BB%D1%8F%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B8.%0A%20%20%20%20%2F%2F%20memset%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BA%D0%B0%D0%B6%D0%B4%D1%8B%D0%B9%20%D0%B1%D0%B0%D0%B9%D1%82%2C%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B8%D1%82%20%D0%B2%D1%81%D0%B5%204%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%B0%20%D0%B1%D1%83%D0%B4%D1%83%D1%82%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D1%8B.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20memset%28%0A%20%20%20%20%20%20%20%20%20%20%20%20buffer.as_mut_ptr%28%29.cast%3A%3A%3Cc_void%3E%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%201%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20std%3A%3Amem%3A%3Asize_of_val%28%26buffer%29%2C%0A%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20for%20item%20in%20%26buffer%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20memset%20%D0%B7%D0%B0%D0%BF%D0%BE%D0%BB%D0%BD%D0%B8%D0%BB%20%D0%B2%D1%81%D1%8E%20%D0%BE%D0%B1%D0%BB%D0%B0%D1%81%D1%82%D1%8C%3B%20%D0%BB%D1%8E%D0%B1%D0%BE%D0%B9%20%D0%BD%D0%B0%D0%B1%D0%BE%D1%80%20%D0%B1%D0%B0%D0%B9%D1%82%20%E2%80%94%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D1%8B%D0%B9%20i32.%0A%20%20%20%20%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20item.assume_init_ref%28%29%20%7D%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Число `16843009` — это `0x01010101`: `memset` записал по байту `1` в каждый из 16 байт, и каждый `i32` получился из четырёх таких байт.

Но здесь есть важное условие: мы можем вызвать `assume_init_ref()` только после того, как **C-функция действительно записала корректный `i32` в каждый элемент**. Если бы `memset` вызвали на меньшем числе байт, часть элементов осталась бы неинициализированной, и вызов `assume_init_ref()` для них был бы UB.

Именно это является типичным контрактом FFI:

```text
Rust:
    выделяет память
        ↓
    передаёт *mut T
        ↓
C:
    записывает T
        ↓
Rust:
    получает инициализированные значения
```

`MaybeUninit` позволяет выразить этот контракт на стороне Rust: пока C не отработал, память честно названа «может быть не инициализирована».

---

## 46.11. Безопасная абстракция над `MaybeUninit`

Рассмотрим завершённую версию ячейки, значение которой появляется позже создания самой ячейки. Назовём её `LateInit<T>` (в стандартной библиотеке аналогичную роль играют `OnceCell` и `LazyCell`).

Главный инвариант структуры:

```text
initialized == false
    → value не инициализирован

initialized == true
    → value содержит корректный T
```

```rust
use std::mem::MaybeUninit;

struct LateInit<T> {
    value: MaybeUninit<T>,
    initialized: bool,
}

impl<T> LateInit<T> {
    fn new() -> Self {
        Self {
            value: MaybeUninit::uninit(),
            initialized: false,
        }
    }

    fn init(&mut self, value: T) {
        assert!(!self.initialized, "already initialized");

        self.value.write(value);
        self.initialized = true;
    }

    fn get(&self) -> Option<&T> {
        if self.initialized {
            // SAFETY: initialized == true означает, что value содержит корректный T.
            Some(unsafe { self.value.assume_init_ref() })
        } else {
            None
        }
    }
}

impl<T> Drop for LateInit<T> {
    fn drop(&mut self) {
        if self.initialized {
            // SAFETY: initialized == true, значение ещё не уничтожено.
            unsafe {
                self.value.assume_init_drop();
            }
        }
    }
}

fn main() {
    let mut value = LateInit::new();

    assert!(value.get().is_none());

    value.init(String::from("Hello, Rust!"));

    assert_eq!(value.get().unwrap(), "Hello, Rust!");

    println!("{}", value.get().unwrap());
}
```

Результат:

```text
Hello, Rust!
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Astruct%20LateInit%3CT%3E%20%7B%0A%20%20%20%20value%3A%20MaybeUninit%3CT%3E%2C%0A%20%20%20%20initialized%3A%20bool%2C%0A%7D%0A%0Aimpl%3CT%3E%20LateInit%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20value%3A%20MaybeUninit%3A%3Auninit%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20initialized%3A%20false%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20init%28%26mut%20self%2C%20value%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20assert%21%28%21self.initialized%2C%20%22already%20initialized%22%29%3B%0A%0A%20%20%20%20%20%20%20%20self.value.write%28value%29%3B%0A%20%20%20%20%20%20%20%20self.initialized%20%3D%20true%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20get%28%26self%29%20-%3E%20Option%3C%26T%3E%20%7B%0A%20%20%20%20%20%20%20%20if%20self.initialized%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20initialized%20%3D%3D%20true%20%D0%BE%D0%B7%D0%BD%D0%B0%D1%87%D0%B0%D0%B5%D1%82%2C%20%D1%87%D1%82%D0%BE%20value%20%D1%81%D0%BE%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D1%82%20%D0%BA%D0%BE%D1%80%D1%80%D0%B5%D0%BA%D1%82%D0%BD%D1%8B%D0%B9%20T.%0A%20%20%20%20%20%20%20%20%20%20%20%20Some%28unsafe%20%7B%20self.value.assume_init_ref%28%29%20%7D%29%0A%20%20%20%20%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20None%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%3CT%3E%20Drop%20for%20LateInit%3CT%3E%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20if%20self.initialized%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20initialized%20%3D%3D%20true%2C%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%B5%D1%89%D1%91%20%D0%BD%D0%B5%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B5%D0%BD%D0%BE.%0A%20%20%20%20%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20self.value.assume_init_drop%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%20LateInit%3A%3Anew%28%29%3B%0A%0A%20%20%20%20assert%21%28value.get%28%29.is_none%28%29%29%3B%0A%0A%20%20%20%20value.init%28String%3A%3Afrom%28%22Hello%2C%20Rust%21%22%29%29%3B%0A%0A%20%20%20%20assert_eq%21%28value.get%28%29.unwrap%28%29%2C%20%22Hello%2C%20Rust%21%22%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20value.get%28%29.unwrap%28%29%29%3B%0A%7D%0A)

Обратите внимание на важную деталь: `Drop` проверяет `initialized`.

Если значение никогда не было создано, его нельзя уничтожать.

Если значение было создано, его необходимо уничтожить.

Таким образом, безопасный API скрывает `unsafe` внутри и предоставляет пользователю обычные методы:

```rust
value.init(...);
value.get();
```

Пользователь не должен самостоятельно вызывать `assume_init()`.

### Как то же самое делает стандартная библиотека

В реальном коде такой тип не пишут вручную. `std::cell::OnceCell` решает ту же задачу и устроен чуть удобнее: `set` принимает `&self`, а не `&mut self`, потому что внутри используется `UnsafeCell`:

```rust
use std::cell::OnceCell;

fn main() {
    let cell = OnceCell::new();

    assert!(cell.get().is_none());

    // set принимает &self, а не &mut self: внутри OnceCell использует UnsafeCell.
    cell.set(String::from("Hello, Rust!")).unwrap();

    // Второй set вернёт значение обратно: ячейка уже заполнена.
    assert!(cell.set(String::from("again")).is_err());

    println!("{}", cell.get().unwrap());
}
```

Результат:

```text
Hello, Rust!
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Acell%3A%3AOnceCell%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20cell%20%3D%20OnceCell%3A%3Anew%28%29%3B%0A%0A%20%20%20%20assert%21%28cell.get%28%29.is_none%28%29%29%3B%0A%0A%20%20%20%20%2F%2F%20set%20%D0%BF%D1%80%D0%B8%D0%BD%D0%B8%D0%BC%D0%B0%D0%B5%D1%82%20%26self%2C%20%D0%B0%20%D0%BD%D0%B5%20%26mut%20self%3A%20%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B8%20OnceCell%20%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D1%83%D0%B5%D1%82%20UnsafeCell.%0A%20%20%20%20cell.set%28String%3A%3Afrom%28%22Hello%2C%20Rust%21%22%29%29.unwrap%28%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%92%D1%82%D0%BE%D1%80%D0%BE%D0%B9%20set%20%D0%B2%D0%B5%D1%80%D0%BD%D1%91%D1%82%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%BE%D0%B1%D1%80%D0%B0%D1%82%D0%BD%D0%BE%3A%20%D1%8F%D1%87%D0%B5%D0%B9%D0%BA%D0%B0%20%D1%83%D0%B6%D0%B5%20%D0%B7%D0%B0%D0%BF%D0%BE%D0%BB%D0%BD%D0%B5%D0%BD%D0%B0.%0A%20%20%20%20assert%21%28cell.set%28String%3A%3Afrom%28%22again%22%29%29.is_err%28%29%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20cell.get%28%29.unwrap%28%29%29%3B%0A%7D%0A)

Для многопоточного варианта существуют `OnceLock` и `LazyLock`, которые добавляют синхронизацию.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `assume_init()` без инициализации

Запустите пример из раздела 46.3 в debug- и release-режимах.

В нашем запуске debug напечатал `1970457160`, release — `0`.

Важно: отсутствие сообщения компилятора **не означает корректность программы**. `unsafe` позволяет написать код, который компилятор не способен доказать безопасным. UB может проявиться непредсказуемо: в одном режиме как мусорное число, в другом как «нормальное» значение `0`.

### Эксперимент 2: `UnsafeCell` не создаёт потокобезопасность

Сам факт использования `UnsafeCell` не делает структуру безопасной для нескольких потоков. Но обратите внимание: компилятор здесь **на вашей стороне**. Запустите пример `mycell_sync` из раздела 46.6 и прочитайте сообщение:

```text
error[E0277]: `UnsafeCell<i32>` cannot be shared between threads safely
```

Теперь добавьте строку `unsafe impl<T> Sync for MyCell<T> {}` и убедитесь, что код компилируется. Объясните, почему это не делает ячейку потокобезопасной, а лишь заставляет компилятор замолчать.

Правильная схема:

```text
UnsafeCell<T>
    ↓
низкоуровневый механизм
    ↓
необходимо обеспечить правила доступа
    ↓
только затем можно создать безопасную абстракцию
```

Для межпоточной изменяемости обычно используются `Mutex<T>`, `RwLock<T>` или атомарные типы, а не непосредственный `UnsafeCell`.

### Эксперимент 3: потерянный `Drop`

Запустите примеры `write_twice` и `mycell_bad`. В обоих случаях старое значение не уничтожается. Исправьте их (через `assume_init_drop()` и `replace` соответственно) и убедитесь, что строка `drop first` / `drop первое` появилась.

### Эксперимент 4: тип нулевого размера в `MyVec`

Создайте `MyVec::<()>::new()` и вызовите `push(())`. Программа завершится с сообщением «типы нулевого размера не поддерживаются». Затем закомментируйте `assert!` и объясните, почему программа без него всё равно некорректна, даже если она «не падает».

---

## Практика

### Задание 1

Создайте массив из 10 `MaybeUninit<i32>`.

Инициализируйте только первые пять элементов и выведите их.

Затем ответьте:

> Почему нельзя безопасно превратить этот массив в `[i32; 10]`?

Повторите упражнение для `MaybeUninit<String>`. Что произойдёт при выходе массива из области видимости, и как уничтожить инициализированные элементы?

---

### Задание 2

Создайте структуру `MyVec<T>` с использованием:

* `NonNull`;
* `MaybeUninit`;
* `len`;
* `capacity`.

Реализуйте:

```rust
new()
with_capacity()
push()
get()
```

После этого добавьте `Drop`.

Проверьте реализацию не только с `i32`, но и со `String`, чтобы убедиться, что элементы корректно уничтожаются. Для проверки используйте тип с «шумным» `Drop`, как `Noisy` в разделе 46.4: каждая строка `drop ...` должна появиться ровно один раз.

Добавьте метод `pop(&mut self) -> Option<T>`. Подсказка: после чтения элемента через `read()` уменьшите `len`, чтобы `Drop` не уничтожил его повторно.

---

### Задание 3

Реализуйте собственный `MyCell<T>` на основе `UnsafeCell<T>`.

Добавьте:

```rust
new()
set()
get()
```

Убедитесь, что `set` не теряет старое значение: проверьте на типе с «шумным» `Drop`. Затем объясните, почему такой тип нельзя автоматически считать потокобезопасным, и что именно компилятор делает по умолчанию.

---

### Задание 4

Создайте `LateInit<T>` на основе:

```rust
MaybeUninit<T>
initialized: bool
```

Реализуйте:

```rust
new()
init()
get()
Drop
```

Главный инвариант:

```text
initialized == false
    → T не существует

initialized == true
    → T существует и его можно безопасно читать
```

Проверьте, что `Drop` не вызывается для неинициализированной ячейки (используйте «шумный» тип).

---

### Задание 5

Исследуйте `NonNull<T>`.

Создайте:

```rust
let ptr = NonNull::<i32>::dangling();
```

и ответьте:

1. Почему этот указатель не равен `null`?
2. Можно ли его разыменовать?
3. Почему `NonNull` не гарантирует валидность объекта?
4. Каким по значению оказывается адрес и почему?

---

### Задание 6

Объясните, почему `Option<NonNull<i32>>` занимает 8 байт, а `Option<*mut i32>` — 16. Проверьте на практике и опишите, какие значения `Option` использует в каждом случае для представления `None`.

---

## Главное из этой главы

После этой главы важно понимать не только назначение трёх типов, но и **границы их гарантий**.

* **`MaybeUninit<T>`** — позволяет представить память, в которой `T` ещё может быть не инициализирован; **никогда не вызывает `Drop` для содержимого**, поэтому инициализированные элементы нужно уничтожать вручную.
* **`NonNull<T>`** — представляет ненулевой сырой указатель, но не гарантирует, что указатель валиден; ковариантен по `T`, а `Option<NonNull<T>>` не занимает лишнего места.
* **`UnsafeCell<T>`** — фундаментальный механизм interior mutability и единственный способ законно изменять данные за `&T`; по умолчанию не `Sync`.
* **`assume_init()`** — не проверяет инициализацию, а принимает её как гарантию программиста.
* **`write()`** не уничтожает старое значение: повторная запись без `assume_init_drop()` приводит к утечке.
* **`Drop`** при работе с `MaybeUninit` должен уничтожать только действительно инициализированные значения.
* **`NonNull`** удобен для низкоуровневых структур данных и эффективно работает с `Option`.
* **`UnsafeCell`** не делает код автоматически безопасным и не обеспечивает синхронизацию потоков.
* Типы нулевого размера требуют отдельной обработки в аллокаторных абстракциях: `alloc::alloc` с нулевым размером — UB.
* Безопасная абстракция должна скрывать `unsafe` и сама поддерживать необходимые инварианты.

### Самая важная идея

> `MaybeUninit`, `NonNull` и `UnsafeCell` не являются «способами отключить безопасность Rust». Это инструменты, с помощью которых мы можем точно описать низкоуровневые состояния памяти и построить поверх них безопасный API.
>
> При этом `unsafe` означает не «здесь Rust больше не отвечает за безопасность», а «теперь программист сам обязан доказать выполнение определённых инвариантов».
>
> Именно способность локализовать такие `unsafe`-участки внутри небольших, хорошо проверенных абстракций позволяет создавать низкоуровневые структуры данных — и при этом предоставлять остальному коду обычный безопасный Rust API.
