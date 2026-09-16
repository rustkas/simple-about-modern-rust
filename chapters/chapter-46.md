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

    // Теперь мы знаем, что внутри находится инициализированный i32.
    let value = unsafe { value.assume_init() };

    println!("{value}");
}
```

Здесь `unsafe` требуется не для записи значения, а для операции `assume_init()`.

Метод `write()` безопасен, потому что он **создаёт значение непосредственно в предназначенной для него памяти**.

`assume_init()` устроен иначе: он говорит компилятору:

> «Я гарантирую, что эта память уже содержит корректно инициализированный `T`».

Если это утверждение ложно, возникает **undefined behavior (UB)**.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn+main%28%29+%7B%0A++++let+mut+value+%3D+MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%3B%0A%0A++++%2F%2F+%D0%97%D0%B0%D0%BF%D0%B8%D1%81%D1%8B%D0%B2%D0%B0%D0%B5%D0%BC+%D0%BF%D0%BE%D0%BB%D0%BD%D0%BE%D1%86%D0%B5%D0%BD%D0%BD%D0%BE%D0%B5+%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5+i32.%0A++++value.write%2842%29%3B%0A%0A++++%2F%2F+%D0%A2%D0%B5%D0%BF%D0%B5%D1%80%D1%8C+%D0%BC%D1%8B+%D0%B7%D0%BD%D0%B0%D0%B5%D0%BC%2C+%D1%87%D1%82%D0%BE+%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B8+%D0%BD%D0%B0%D1%85%D0%BE%D0%B4%D0%B8%D1%82%D1%81%D1%8F+%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D0%B9+i32.%0A++++let+value+%3D+unsafe+%7B+value.assume_init%28%29+%7D%3B%0A%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

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

Предпочтительный способ:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut value = MaybeUninit::<i32>::uninit();

    value.write(42);

    let value = unsafe { value.assume_init() };

    println!("{value}");
}
```


[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn+main%28%29+%7B%0A++++let+mut+value+%3D+MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%3B%0A%0A++++value.write%2842%29%3B%0A%0A++++let+value+%3D+unsafe+%7B+value.assume_init%28%29+%7D%3B%0A%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D)


`write()` особенно полезен потому, что он не требует предварительно создавать значение `T` и не пытается уничтожить старое значение.

Низкоуровневый вариант через указатель также возможен:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut value = MaybeUninit::<i32>::uninit();

    unsafe {
        value.as_mut_ptr().write(42);
    }

    let value = unsafe { value.assume_init() };

    println!("{value}");
}
```

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

    let value = unsafe {
        value.assume_init()
    };

    println!("{value}");
}
```

Это не «получение случайного значения». Это нарушение контракта `assume_init()` и потенциальное **undefined behavior**.

---

### Частично инициализированный массив

Одна из наиболее полезных возможностей `MaybeUninit` — создание массива, элементы которого инициализируются постепенно.

Например:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut data: [MaybeUninit<i32>; 10] =
        [const { MaybeUninit::uninit() }; 10];

    for (index, slot) in data.iter_mut().enumerate().take(5) {
        slot.write(index as i32 * 10);
    }

    for (index, slot) in data.iter().enumerate().take(5) {
        let value = unsafe { slot.assume_init_ref() };
        println!("data[{index}] = {value}");
    }
}
```

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

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20data%3A%20%5BMaybeUninit%3Ci32%3E%3B%2010%5D%20%3D%20%5Bconst%20%7B%20MaybeUninit%3A%3Auninit%28%29%20%7D%3B%2010%5D%3B%0A%0A%20%20%20%20for%20%28index%2C%20slot%29%20in%20data.iter_mut%28%29.enumerate%28%29.take%285%29%20%7B%0A%20%20%20%20%20%20%20%20slot.write%28index%20as%20i32%20%2A%2010%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20for%20%28index%2C%20slot%29%20in%20data.iter%28%29.enumerate%28%29.take%285%29%20%7B%0A%20%20%20%20%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20slot.assume_init_ref%28%29%20%7D%3B%0A%20%20%20%20%20%20%20%20println%21%28%22data%5B%7Bindex%7D%5D%20%3D%20%7Bvalue%7D%22%29%3B%0A%20%20%20%20%7D%0A%7D)

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
use std::mem::MaybeUninit;
use std::ptr::NonNull;

struct MyVec<T> {
    ptr: NonNull<MaybeUninit<T>>,
    len: usize,
    capacity: usize,
}

impl<T> MyVec<T> {
    fn new() -> Self {
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

        let layout = Layout::array::<MaybeUninit<T>>(capacity)
            .expect("capacity overflow");

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

    fn push(&mut self, value: T) {
        if self.len == self.capacity {
            self.grow();
        }

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

        unsafe {
            Some((&*self.ptr.as_ptr().add(index)).assume_init_ref())
        }
    }

    fn grow(&mut self) {
        let new_capacity = if self.capacity == 0 {
            4
        } else {
            self.capacity
                .checked_mul(2)
                .expect("capacity overflow")
        };

        let mut new_vec = Self::with_capacity(new_capacity);

        for index in 0..self.len {
            unsafe {
                let value = self.ptr.as_ptr().add(index).read().assume_init();
                new_vec.push(value);
            }
        }

        self.deallocate();

        self.ptr = new_vec.ptr;
        self.capacity = new_vec.capacity;
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

        unsafe {
            alloc::dealloc(self.ptr.as_ptr().cast(), layout);
        }

        self.ptr = NonNull::dangling();
        self.capacity = 0;
    }
}

impl<T> Drop for MyVec<T> {
    fn drop(&mut self) {
        unsafe {
            for index in 0..self.len {
                self.ptr
                    .as_ptr()
                    .add(index)
                    .read()
                    .assume_init_drop();
            }
        }

        self.deallocate();
    }
}

fn main() {
    let mut numbers = MyVec::with_capacity(2);

    numbers.push(String::from("one"));
    numbers.push(String::from("two"));
    numbers.push(String::from("three"));

    println!("{}", numbers.get(0).unwrap());
    println!("{}", numbers.get(1).unwrap());
    println!("{}", numbers.get(2).unwrap());
}
```

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

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aalloc%3A%3A%7Bself%2C%20Layout%7D%3B%0Ause%20std%3A%3Amem%3A%3AMaybeUninit%3B%0Ause%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Astruct%20MyVec%3CT%3E%20%7B%0A%20%20%20%20ptr%3A%20NonNull%3CMaybeUninit%3CT%3E%3E%2C%0A%20%20%20%20len%3A%20usize%2C%0A%20%20%20%20capacity%3A%20usize%2C%0A%7D%0A%0Aimpl%3CT%3E%20MyVec%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%20ptr%3A%20NonNull%3A%3Adangling%28%29%2C%20len%3A%200%2C%20capacity%3A%200%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20with_capacity%28capacity%3A%20usize%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20if%20capacity%20%3D%3D%200%20%7B%20return%20Self%3A%3Anew%28%29%3B%20%7D%0A%20%20%20%20%20%20%20%20let%20layout%20%3D%20Layout%3A%3Aarray%3A%3A%3CMaybeUninit%3CT%3E%3E%28capacity%29.expect%28%22capacity%20overflow%22%29%3B%0A%20%20%20%20%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20alloc%3A%3Aalloc%28layout%29%20%7D%3B%0A%20%20%20%20%20%20%20%20let%20ptr%20%3D%20NonNull%3A%3Anew%28ptr%29.unwrap_or_else%28%7C%7C%20alloc%3A%3Ahandle_alloc_error%28layout%29%29.cast%3A%3A%3CMaybeUninit%3CT%3E%3E%28%29%3B%0A%20%20%20%20%20%20%20%20Self%20%7B%20ptr%2C%20len%3A%200%2C%20capacity%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20push%28%26mut%20self%2C%20value%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20if%20self.len%20%3D%3D%20self.capacity%20%7B%20self.grow%28%29%3B%20%7D%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20self.ptr.as_ptr%28%29.add%28self.len%29.write%28MaybeUninit%3A%3Anew%28value%29%29%3B%20%7D%0A%20%20%20%20%20%20%20%20self.len%20%2B%3D%201%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20get%28%26self%2C%20index%3A%20usize%29%20-%3E%20Option%3C%26T%3E%20%7B%0A%20%20%20%20%20%20%20%20if%20index%20%3E%3D%20self.len%20%7B%20return%20None%3B%20%7D%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20Some%28%28%26%2Aself.ptr.as_ptr%28%29.add%28index%29%29.assume_init_ref%28%29%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20grow%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20let%20new_capacity%20%3D%20if%20self.capacity%20%3D%3D%200%20%7B%204%20%7D%20else%20%7B%20self.capacity.checked_mul%282%29.expect%28%22capacity%20overflow%22%29%20%7D%3B%0A%20%20%20%20%20%20%20%20let%20mut%20new_vec%20%3D%20Self%3A%3Awith_capacity%28new_capacity%29%3B%0A%20%20%20%20%20%20%20%20for%20index%20in%200..self.len%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20unsafe%20%7B%20let%20value%20%3D%20self.ptr.as_ptr%28%29.add%28index%29.read%28%29.assume_init%28%29%3B%20new_vec.push%28value%29%3B%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20self.deallocate%28%29%3B%0A%20%20%20%20%20%20%20%20self.ptr%20%3D%20new_vec.ptr%3B%20self.capacity%20%3D%20new_vec.capacity%3B%20new_vec.ptr%20%3D%20NonNull%3A%3Adangling%28%29%3B%20new_vec.capacity%20%3D%200%3B%20new_vec.len%20%3D%200%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20deallocate%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20if%20self.capacity%20%3D%3D%200%20%7B%20return%3B%20%7D%0A%20%20%20%20%20%20%20%20let%20layout%20%3D%20Layout%3A%3Aarray%3A%3A%3CMaybeUninit%3CT%3E%3E%28self.capacity%29.expect%28%22capacity%20overflow%22%29%3B%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20alloc%3A%3Adealloc%28self.ptr.as_ptr%28%29.cast%28%29%2C%20layout%29%3B%20%7D%0A%20%20%20%20%20%20%20%20self.ptr%20%3D%20NonNull%3A%3Adangling%28%29%3B%20self.capacity%20%3D%200%3B%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%3CT%3E%20Drop%20for%20MyVec%3CT%3E%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20for%20index%20in%200..self.len%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20self.ptr.as_ptr%28%29.add%28index%29.read%28%29.assume_init_drop%28%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20self.deallocate%28%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20numbers%20%3D%20MyVec%3A%3Awith_capacity%282%29%3B%0A%20%20%20%20numbers.push%28String%3A%3Afrom%28%22one%22%29%29%3B%0A%20%20%20%20numbers.push%28String%3A%3Afrom%28%22two%22%29%29%3B%0A%20%20%20%20numbers.push%28String%3A%3Afrom%28%22three%22%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20numbers.get%280%29.unwrap%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20numbers.get%281%29.unwrap%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20numbers.get%282%29.unwrap%28%29%29%3B%0A%7D)

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

    unsafe {
        println!("value = {}", *ptr.as_ptr());
    }
}
```

Здесь указатель действительно указывает на живой `i32`, поэтому разыменование корректно.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%20%20%20%20let%20ptr%20%3D%20NonNull%3A%3Afrom%28%26value%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22value%20%3D%20%7B%7D%22%2C%20%2Aptr.as_ptr%28%29%29%3B%0A%20%20%20%20%7D%0A%7D)

### `Option<NonNull<T>>` и null pointer optimization

Одна из важных особенностей `NonNull` — возможность эффективно представлять опциональный указатель.

```rust
use std::mem::size_of;
use std::ptr::NonNull;

fn main() {
    println!(
        "NonNull<i32>: {}",
        size_of::<NonNull<i32>>()
    );

    println!(
        "Option<NonNull<i32>>: {}",
        size_of::<Option<NonNull<i32>>>()
    );

    println!(
        "*mut i32: {}",
        size_of::<*mut i32>()
    );
}
```

Обычно `Option<NonNull<T>>` имеет тот же размер, что и сам указатель: значение `None` можно представить специальным нулевым значением.

Это называется **null pointer optimization (NPO)**.

Именно поэтому `NonNull` очень удобен для структур данных, где нужно представить:

```text
указатель есть
      или
указателя нет
```

без дополнительного поля для флага.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3Asize_of%3B%0Ause%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22NonNull%3Ci32%3E%3A%20%7B%7D%22%2C%20size_of%3A%3A%3CNonNull%3Ci32%3E%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22Option%3CNonNull%3Ci32%3E%3E%3A%20%7B%7D%22%2C%20size_of%3A%3A%3COption%3CNonNull%3Ci32%3E%3E%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22%2Amut%20i32%3A%20%7B%7D%22%2C%20size_of%3A%3A%3C%2Amut%20i32%3E%28%29%29%3B%0A%7D)

### `NonNull::dangling()`

В низкоуровневых структурах данных часто нужно представить «пустой» указатель, который при этом не является `null`.

Для этого существует:

```rust
let ptr = NonNull::<i32>::dangling();
```

Такой указатель **не означает, что по адресу действительно находится `i32`**. Его нельзя разыменовывать.

Он используется как специальное значение, например, когда `capacity == 0`:

```rust
struct Buffer<T> {
    ptr: NonNull<T>,
    len: usize,
    capacity: usize,
}
```

Если `capacity == 0`, `ptr` может содержать `NonNull::dangling()`, потому что память всё равно не используется.

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

означает, что объект нельзя изменять через эту ссылку.

`UnsafeCell<T>` сообщает компилятору:

> «Внутри этого объекта данные могут изменяться даже тогда, когда сам объект доступен через `&self`».

Минимальный пример:

```rust
use std::cell::UnsafeCell;

struct MyCell<T> {
    value: UnsafeCell<T>,
}

impl<T> MyCell<T> {
    fn new(value: T) -> Self {
        Self {
            value: UnsafeCell::new(value),
        }
    }

    fn set(&self, value: T) {
        unsafe {
            self.value.get().write(value);
        }
    }

    fn get(&self) -> T
    where
        T: Copy,
    {
        unsafe {
            *self.value.get()
        }
    }
}

fn main() {
    let cell = MyCell::new(42);

    println!("{}", cell.get());

    cell.set(100);

    println!("{}", cell.get());
}
```

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Acell%3A%3AUnsafeCell%3B%0A%0Astruct%20MyCell%3CT%3E%20%7B%0A%20%20%20%20value%3A%20UnsafeCell%3CT%3E%2C%0A%7D%0A%0Aimpl%3CT%3E%20MyCell%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28value%3A%20T%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%20value%3A%20UnsafeCell%3A%3Anew%28value%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20set%28%26self%2C%20value%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20self.value.get%28%29.write%28value%29%3B%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20get%28%26self%29%20-%3E%20T%0A%20%20%20%20where%20%0A%20%20%20%20%20%20%20%20T%3A%20Copy%2C%0A%20%20%20%20%7B%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20*self.value.get%28%29%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20cell%20%3D%20MyCell%3A%3Anew%2842%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20cell.get%28%29%29%3B%0A%20%20%20%20cell.set%28100%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20cell.get%28%29%29%3B%0A%7D)

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

`Cell<T>` подходит для простых случаев, когда значение можно получать и заменять целиком.

`RefCell<T>` нужен, когда необходимо получать ссылки на внутреннее значение, но проверка правил заимствования переносится с compile time на runtime.

`Mutex<T>` решает уже другую задачу — синхронизацию доступа между потоками.

Таким образом, пользователю библиотеки обычно не нужно работать непосредственно с `UnsafeCell`. Он используется **внутри более высокоуровневых безопасных абстракций**.

---

## 46.8. Сравнение: `MaybeUninit` vs `NonNull` vs `UnsafeCell`

| Тип              | Основная задача                                               | Что гарантирует                                                             | Что не гарантирует                                    |
| ---------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------- |
| `MaybeUninit<T>` | Представить память, где `T` может быть ещё не инициализирован | Сам `MaybeUninit<T>` корректен даже без `T`                                 | Что `T` уже инициализирован                           |
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

```rust
use std::mem::MaybeUninit;

fn main() {
    let value = MaybeUninit::<i32>::uninit();

    let value = unsafe {
        value.assume_init()
    };

    println!("{value}");
}
```

Так делать нельзя.

`assume_init()` требует доказательства, что внутри уже находится корректный `i32`.

Компилятор не может проверить это условие — ответственность лежит на программисте.

---

### Ошибка 2: путать повторную запись с обычным присваиванием

Следующий код допустим:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut value = MaybeUninit::<String>::uninit();

    value.write(String::from("first"));
    value.write(String::from("second"));

    let value = unsafe {
        value.assume_init()
    };

    println!("{value}");
}
```

`write()` записывает новое значение непосредственно в память.

Однако при работе с `T`, который реализует `Drop`, нельзя бездумно воспринимать это как обычное присваивание. В частности, `write()` не вызывает `drop` старого значения перед записью.

Поэтому такой паттерн должен использоваться только тогда, когда программист точно понимает состояние памяти.

---

### Ошибка 3: неправильное выравнивание

`MaybeUninit<T>` гарантирует корректное выравнивание **для `T`**.

Но нельзя произвольно преобразовывать указатель на один тип в указатель на другой:

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut bytes = MaybeUninit::<[u8; 4]>::uninit();

    let ptr = bytes.as_mut_ptr() as *mut i32;

    unsafe {
        ptr.write(42);
    }
}
```

Такой код нельзя считать корректным только потому, что размер `[u8; 4]` равен размеру `i32`.

Для `i32` требуется соответствующее выравнивание, а у `[u8; 4]` оно может быть недостаточным.

Если нужна память под `i32`, следует сразу использовать:

```rust
MaybeUninit::<i32>::uninit()
```

а не пытаться «переиспользовать» память другого типа.

---

## 46.10. `MaybeUninit` и FFI

Рассмотрим типичный сценарий FFI.

Представим C-функцию:

```c
void fill_buffer(int32_t *buffer, size_t count);
```

Она получает указатель и заполняет `count` элементов.

В Rust память можно подготовить следующим образом:

```rust
use std::mem::MaybeUninit;

unsafe extern "C" {
    fn fill_buffer(buffer: *mut i32, count: usize);
}

fn main() {
    let mut buffer = [const { MaybeUninit::<i32>::uninit() }; 4];

    unsafe {
        fill_buffer(buffer.as_mut_ptr().cast(), buffer.len());
    }

    for item in &buffer {
        let value = unsafe { item.assume_init_ref() };
        println!("{value}");
    }
}
```

Но здесь есть важное условие: мы можем вызвать `assume_init_ref()` только после того, как **C-функция действительно записала корректный `i32` в каждый элемент**.

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

`MaybeUninit` позволяет выразить этот контракт на стороне Rust.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Afn%20fill_buffer%28buffer%3A%20%2Amut%20i32%2C%20count%3A%20usize%29%20%7B%0A%20%20%20%20for%20index%20in%200..count%20%7B%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20buffer.add%28index%29.write%28index%20as%20i32%20%2A%2010%29%3B%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20buffer%20%3D%20%5Bconst%20%7B%20MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%20%7D%3B%204%5D%3B%0A%0A%20%20%20%20fill_buffer%28buffer.as_mut_ptr%28%29.cast%28%29%2C%20buffer.len%28%29%29%3B%0A%0A%20%20%20%20for%20item%20in%20%26buffer%20%7B%0A%20%20%20%20%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20item.assume_init_ref%28%29%20%7D%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 46.11. Безопасная абстракция над `MaybeUninit`

Рассмотрим завершённую версию `Lazy<T>`.

Главный инвариант структуры:

```text
initialized == false
    → value не инициализирован

initialized == true
    → value содержит корректный T
```

```rust
use std::mem::MaybeUninit;

struct Lazy<T> {
    value: MaybeUninit<T>,
    initialized: bool,
}

impl<T> Lazy<T> {
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
            Some(unsafe {
                self.value.assume_init_ref()
            })
        } else {
            None
        }
    }
}

impl<T> Drop for Lazy<T> {
    fn drop(&mut self) {
        if self.initialized {
            unsafe {
                self.value.assume_init_drop();
            }
        }
    }
}

fn main() {
    let mut value = Lazy::new();

    assert!(value.get().is_none());

    value.init(String::from("Hello, Rust!"));

    assert_eq!(value.get().unwrap(), "Hello, Rust!");

    println!("{}", value.get().unwrap());
}
```

Обратите внимание на важную деталь: `Drop` проверяет `initialized`.

Если значение никогда не было создано, его нельзя уничтожать.

Если значение было создано, его необходимо уничтожить.

Таким образом, безопасный API скрывает `unsafe` внутри и предоставляет пользователю обычные методы:

```rust
value.init(...);
value.get();
```

Пользователь не должен самостоятельно вызывать `assume_init()`.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0A%0Astruct%20Lazy%3CT%3E%20%7B%0A%20%20%20%20value%3A%20MaybeUninit%3CT%3E%2C%0A%20%20%20%20initialized%3A%20bool%2C%0A%7D%0A%0Aimpl%3CT%3E%20Lazy%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%20value%3A%20MaybeUninit%3A%3Auninit%28%29%2C%20initialized%3A%20false%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20init%28%26mut%20self%2C%20value%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20assert%21%28%21self.initialized%2C%20%22already%20initialized%22%29%3B%0A%20%20%20%20%20%20%20%20self.value.write%28value%29%3B%0A%20%20%20%20%20%20%20%20self.initialized%20%3D%20true%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20get%28%26self%29%20-%3E%20Option%3C%26T%3E%20%7B%0A%20%20%20%20%20%20%20%20if%20self.initialized%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20Some%28unsafe%20%7B%20self.value.assume_init_ref%28%29%20%7D%29%0A%20%20%20%20%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20None%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%3CT%3E%20Drop%20for%20Lazy%3CT%3E%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20if%20self.initialized%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20unsafe%20%7B%20self.value.assume_init_drop%28%29%3B%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%20Lazy%3A%3Anew%28%29%3B%0A%20%20%20%20assert%21%28value.get%28%29.is_none%28%29%29%3B%0A%20%20%20%20value.init%28String%3A%3Afrom%28%22Hello%2C%20Rust%21%22%29%29%3B%0A%20%20%20%20assert_eq%21%28value.get%28%29.unwrap%28%29%2C%20%22Hello%2C%20Rust%21%22%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20value.get%28%29.unwrap%28%29%29%3B%0A%7D)

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `assume_init()` без инициализации

Попробуйте запустить:

```rust
use std::mem::MaybeUninit;

fn main() {
    let value = MaybeUninit::<i32>::uninit();

    let value = unsafe {
        value.assume_init()
    };

    println!("{value}");
}
```

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20MaybeUninit%3A%3A%3Ci32%3E%3A%3Auninit%28%29%3B%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20value.assume_init%28%29%20%7D%3B%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D)

Важно: отсутствие сообщения компилятора **не означает корректность программы**. `unsafe` позволяет написать код, который компилятор не способен доказать безопасным. UB может проявиться непредсказуемо.

### Эксперимент 2: `UnsafeCell` не создаёт потокобезопасность

Сам факт использования `UnsafeCell` не делает структуру безопасной для нескольких потоков.

Например, нельзя сделать вывод:

```text
UnsafeCell<T>
    ↓
можно безопасно менять T из разных потоков
```

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

---

## Практика

### Задание 1

Создайте массив из 10 `MaybeUninit<i32>`.

Инициализируйте только первые пять элементов и выведите их.

Затем ответьте:

> Почему нельзя безопасно превратить этот массив в `[i32; 10]`?

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

Проверьте реализацию не только с `i32`, но и со `String`, чтобы убедиться, что элементы корректно уничтожаются.

---

### Задание 3

Реализуйте собственный `MyCell<T>` на основе `UnsafeCell<T>`.

Добавьте:

```rust
new()
set()
get()
```

Затем объясните, почему такой тип нельзя автоматически считать потокобезопасным.

---

### Задание 4

Создайте `Lazy<T>` на основе:

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

---

## Главное из этой главы

После этой главы важно понимать не только назначение трёх типов, но и **границы их гарантий**.

* **`MaybeUninit<T>`** — позволяет представить память, в которой `T` ещё может быть не инициализирован.
* **`NonNull<T>`** — представляет ненулевой сырой указатель, но не гарантирует, что указатель валиден.
* **`UnsafeCell<T>`** — фундаментальный механизм interior mutability.
* **`assume_init()`** — не проверяет инициализацию, а принимает её как гарантию программиста.
* **`Drop`** при работе с `MaybeUninit` должен уничтожать только действительно инициализированные значения.
* **`NonNull`** удобен для низкоуровневых структур данных и эффективно работает с `Option`.
* **`UnsafeCell`** не делает код автоматически безопасным и не обеспечивает синхронизацию потоков.
* Безопасная абстракция должна скрывать `unsafe` и сама поддерживать необходимые инварианты.

### Самая важная идея

> `MaybeUninit`, `NonNull` и `UnsafeCell` не являются «способами отключить безопасность Rust». Это инструменты, с помощью которых мы можем точно описать низкоуровневые состояния памяти и построить поверх них безопасный API.
>
> При этом `unsafe` означает не «здесь Rust больше не отвечает за безопасность», а «теперь программист сам обязан доказать выполнение определённых инвариантов».
>
> Именно способность локализовать такие `unsafe`-участки внутри небольших, хорошо проверенных абстракций позволяет создавать низкоуровневые структуры данных — и при этом предоставлять остальному коду обычный безопасный Rust API.
