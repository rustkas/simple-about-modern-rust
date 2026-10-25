# Часть XI. Unsafe Rust

## Глава 44. Что действительно означает `unsafe`

До этой главы мы писали только безопасный код. Rust гарантировал, что мы не сделаем ошибок с памятью, не вызовем неопределённое поведение и не создадим гонок данных. Но иногда безопасных абстракций недостаточно — нужно работать с памятью напрямую, взаимодействовать с C-кодом или реализовывать эффективные структуры данных.

Для таких случаев в Rust есть `unsafe`. Но `unsafe` — это не отключение проверок компилятора. Это **обещание программиста**: «Я знаю, что делаю, и гарантирую безопасность».

В этой главе мы разберёмся, что такое `unsafe`, какие операции он разрешает, что такое undefined behavior и как использовать `unsafe` ответственно.

Все примеры этой главы используют **Rust Edition 2024**. Это важно: именно в этой edition изменились правила для `unsafe fn`, `static mut`, `extern`-блоков и атрибутов, и мы будем на это опираться.

---

## 44.1. Что такое `unsafe`?

`unsafe` — это механизм Rust, который позволяет выполнять несколько операций, безопасность которых компилятор не может доказать автоматически.

Важно понимать принципиальную вещь:

> **`unsafe` не означает «этот код небезопасен» и не означает «этот код безопасен».**
>
> `unsafe` означает: **«компилятор не может доказать безопасность этой операции; программист должен обеспечить необходимые условия»**.

Например, компилятор не может самостоятельно определить, является ли произвольный сырой указатель действительным:

```rust
fn main() {
    let x = 42;
    let ptr = &x as *const i32;

    // Создание и печать указателя не требуют unsafe.
    println!("указатель нулевой? {}", ptr.is_null());

    // Разыменование требует.
    let value = unsafe { *ptr };
    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%2042%3B%0A%20%20%20%20let%20ptr%20%3D%20%26x%20as%20%2Aconst%20i32%3B%0A%0A%20%20%20%20%2F%2F%20%D0%A1%D0%BE%D0%B7%D0%B4%D0%B0%D0%BD%D0%B8%D0%B5%20%D0%B8%20%D0%BF%D0%B5%D1%87%D0%B0%D1%82%D1%8C%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8F%20%D0%BD%D0%B5%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D1%8E%D1%82%20unsafe.%0A%20%20%20%20println%21%28%22%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B9%3F%20%7B%7D%22%2C%20ptr.is_null%28%29%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%A0%D0%B0%D0%B7%D1%8B%D0%BC%D0%B5%D0%BD%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20%2Aptr%20%7D%3B%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

Здесь создание сырого указателя и даже его печать (`is_null()`) безопасны, а **разыменование** требует `unsafe`. Обратите внимание: в блок `unsafe` мы поместили только `*ptr`. Всё остальное компилятор проверяет как обычно, а чем меньше область `unsafe`, тем проще проверить её вручную.

При этом компилятор продолжает выполнять огромное количество проверок:

- проверку типов;
- проверку заимствований в безопасной части программы;
- проверку времени жизни;
- проверку синтаксиса;
- проверку большинства правил языка;
- проверку границ массивов при обычной индексации;
- проверку многих других условий.

`unsafe` **не превращает Rust в C/C++**.

### `unsafe` как граница доверия

Хороший способ мыслить об `unsafe`:

```text
Safe Rust
    │
    │ компилятор может доказать безопасность
    ▼
unsafe boundary
    │
    │ программист обязан доказать дополнительные условия
    ▼
низкоуровневая операция
```

Например:

```rust
unsafe {
    *ptr
}
```

Сам `unsafe`-блок ничего не доказывает. Он лишь говорит компилятору:

> «Я осознаю, что эта операция требует дополнительных гарантий».

Если `ptr` недействителен, программа всё равно может содержать **undefined behavior (UB)**. Что это такое, мы разберём в разделе 44.8.

---

## 44.2. Возможности `unsafe`

У `unsafe` есть несколько конкретных возможностей. В учебном контексте их удобно разделить на пять основных категорий:

| Операция                       | Что позволяет                                                       |
| ------------------------------ | ------------------------------------------------------------------- |
| Разыменование сырого указателя | Читать или записывать через `*const T` / `*mut T`                   |
| Вызов `unsafe fn`              | Вызывать функции с явно заданным safety contract                    |
| Доступ к `static mut`          | Работать с изменяемой глобальной памятью                            |
| Реализация `unsafe trait`      | Вручную подтвердить выполнение контракта трейта                     |
| Доступ к полям `union`         | Читать поле `union`, когда программист знает, что оно действительно |

Кроме того, современный Rust имеет отдельные правила для некоторых других unsafe-конструкций: `unsafe extern` блоков и `unsafe`-атрибутов. В конце этого раздела есть короткий пример, а подробно FFI рассматривается отдельно.

### 1. Разыменование сырого указателя

```rust
fn main() {
    let mut value = 42;
    let ptr = &mut value as *mut i32;

    // SAFETY: ptr получен из живой изменяемой ссылки на value,
    // других обращений к value в этот момент нет.
    unsafe {
        *ptr = 100;
    }

    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%2042%3B%0A%20%20%20%20let%20ptr%20%3D%20%26mut%20value%20as%20%2Amut%20i32%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20%D0%B6%D0%B8%D0%B2%D0%BE%D0%B9%20%D0%B8%D0%B7%D0%BC%D0%B5%D0%BD%D1%8F%D0%B5%D0%BC%D0%BE%D0%B9%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8%20%D0%BD%D0%B0%20value%2C%0A%20%20%20%20%2F%2F%20%D0%B4%D1%80%D1%83%D0%B3%D0%B8%D1%85%20%D0%BE%D0%B1%D1%80%D0%B0%D1%89%D0%B5%D0%BD%D0%B8%D0%B9%20%D0%BA%20value%20%D0%B2%20%D1%8D%D1%82%D0%BE%D1%82%20%D0%BC%D0%BE%D0%BC%D0%B5%D0%BD%D1%82%20%D0%BD%D0%B5%D1%82.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2Aptr%20%3D%20100%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

Комментарий `// SAFETY:` рядом с `unsafe`-блоком — общепринятая практика: он записывает, **почему** условия операции выполнены. К этой привычке мы вернёмся в разделе 44.9.

### 2. Вызов `unsafe fn`

```rust
unsafe fn dangerous() {
    // ...
}

unsafe {
    dangerous();
}
```

### 3. Работа с `static mut`

`static mut` — особенно опасный механизм: любое обращение к нему может оказаться гонкой данных, а ссылки на него легко нарушают правила aliasing. Поэтому в Edition 2024 создание ссылки на `static mut` по умолчанию **запрещено компилятором** (lint `static_mut_refs` имеет уровень `deny`).

Прямые операции вроде `COUNTER += 1` ссылки не создают и допустимы в `unsafe`. Но многое выглядит невинно, а ссылку создаёт — например, `println!`, который неявно берёт ссылку на свой аргумент:

```rust
static mut COUNTER: u32 = 0;

fn main() {
    unsafe {
        COUNTER += 1; // прямая операция: ссылка не создаётся
    }

    unsafe {
        // println! неявно берёт ссылку на COUNTER
        println!("{}", COUNTER);
    }
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=static%20mut%20COUNTER%3A%20u32%20%3D%200%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20COUNTER%20%2B%3D%201%3B%20%2F%2F%20%D0%BF%D1%80%D1%8F%D0%BC%D0%B0%D1%8F%20%D0%BE%D0%BF%D0%B5%D1%80%D0%B0%D1%86%D0%B8%D1%8F%3A%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B0%20%D0%BD%D0%B5%20%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D1%91%D1%82%D1%81%D1%8F%0A%20%20%20%20%7D%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20println%21%20%D0%BD%D0%B5%D1%8F%D0%B2%D0%BD%D0%BE%20%D0%B1%D0%B5%D1%80%D1%91%D1%82%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D1%83%20%D0%BD%D0%B0%20COUNTER%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20COUNTER%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Компилятор откажется собирать этот код:

```text
error: creating a shared reference to mutable static
  --> src/main.rs:10:24
   |
10 |         println!("{}", COUNTER);
   |                        ^^^^^^^ shared reference to mutable static
   |
   = note: shared references to mutable statics are dangerous; it's undefined
     behavior if the static is mutated or if a mutable reference is created for
     it while the shared reference lives
   = note: `#[deny(static_mut_refs)]` (part of `#[deny(rust_2024_compatibility)]`) on by default
```

Исправить это можно, не создавая ссылку: скопировать значение или взять сырой указатель оператором `&raw mut` (он создаёт указатель без промежуточной ссылки):

```rust
static mut COUNTER: u32 = 0;

fn main() {
    // SAFETY: программа однопоточная, других обращений к COUNTER нет.
    unsafe {
        COUNTER += 1;
    }

    // Копируем значение — ссылка на static mut не создаётся.
    let value = unsafe { COUNTER };
    println!("{value}");

    // Или берём сырой указатель напрямую, без промежуточной ссылки.
    let ptr = &raw mut COUNTER;
    unsafe {
        *ptr += 1;
        println!("{}", *ptr);
    }
}
```

Результат:

```text
1
2
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=static%20mut%20COUNTER%3A%20u32%20%3D%200%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B0%20%D0%BE%D0%B4%D0%BD%D0%BE%D0%BF%D0%BE%D1%82%D0%BE%D1%87%D0%BD%D0%B0%D1%8F%2C%20%D0%B4%D1%80%D1%83%D0%B3%D0%B8%D1%85%20%D0%BE%D0%B1%D1%80%D0%B0%D1%89%D0%B5%D0%BD%D0%B8%D0%B9%20%D0%BA%20COUNTER%20%D0%BD%D0%B5%D1%82.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20COUNTER%20%2B%3D%201%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%2F%2F%20%D0%9A%D0%BE%D0%BF%D0%B8%D1%80%D1%83%D0%B5%D0%BC%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%E2%80%94%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B0%20%D0%BD%D0%B0%20static%20mut%20%D0%BD%D0%B5%20%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D1%91%D1%82%D1%81%D1%8F.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20COUNTER%20%7D%3B%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%98%D0%BB%D0%B8%20%D0%B1%D0%B5%D1%80%D1%91%D0%BC%20%D1%81%D1%8B%D1%80%D0%BE%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D0%B0%D0%BF%D1%80%D1%8F%D0%BC%D1%83%D1%8E%2C%20%D0%B1%D0%B5%D0%B7%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B9%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8.%0A%20%20%20%20let%20ptr%20%3D%20%26raw%20mut%20COUNTER%3B%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2Aptr%20%2B%3D%201%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Обратите внимание: даже этот корректный код безопасен только потому, что программа однопоточная. `static mut` не защищает от гонок данных.

Для нового кода обычно следует предпочитать безопасные альтернативы, например атомарные типы:

```rust
use std::sync::atomic::{AtomicU32, Ordering};

static COUNTER: AtomicU32 = AtomicU32::new(0);

fn main() {
    COUNTER.fetch_add(1, Ordering::Relaxed);

    println!("{}", COUNTER.load(Ordering::Relaxed));
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicU32%2C%20Ordering%7D%3B%0A%0Astatic%20COUNTER%3A%20AtomicU32%20%3D%20AtomicU32%3A%3Anew%280%29%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20COUNTER.fetch_add%281%2C%20Ordering%3A%3ARelaxed%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20COUNTER.load%28Ordering%3A%3ARelaxed%29%29%3B%0A%7D%0A)

Таким образом, сам факт существования `static mut` ещё не означает, что его следует использовать.

### 4. Реализация `unsafe trait`

```rust
unsafe trait MyUnsafeTrait {
    // Safety contract определяется автором трейта.
}

unsafe impl MyUnsafeTrait for i32 {}
```

Здесь компилятор доверяет программисту: он не доказывает, что `i32` действительно удовлетворяет safety contract трейта. Реальный пример (`unsafe impl Send`) разобран в разделе 44.7.

### 5. Чтение поля `union`

```rust
union Value {
    integer: u32,
    float: f32,
}

fn main() {
    let value = Value { integer: 42 };

    // SAFETY: последним записано поле integer, читаем его же.
    unsafe {
        println!("{}", value.integer);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=union%20Value%20%7B%0A%20%20%20%20integer%3A%20u32%2C%0A%20%20%20%20float%3A%20f32%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20Value%20%7B%20integer%3A%2042%20%7D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BF%D0%BE%D1%81%D0%BB%D0%B5%D0%B4%D0%BD%D0%B8%D0%BC%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%BE%20%D0%BF%D0%BE%D0%BB%D0%B5%20integer%2C%20%D1%87%D0%B8%D1%82%D0%B0%D0%B5%D0%BC%20%D0%B5%D0%B3%D0%BE%20%D0%B6%D0%B5.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20value.integer%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

`union` не хранит информацию о том, какое поле в данный момент является логически активным. Поэтому чтение поля требует `unsafe`.

Классическое применение — посмотреть на одни и те же биты как на значения разных типов:

```rust
union Value {
    integer: u32,
    float: f32,
}

fn main() {
    let value = Value { float: 1.0 };

    // SAFETY: у u32 и f32 нет недопустимых битовых образов,
    // поэтому чтение чужого поля даёт определённый результат.
    let bits = unsafe { value.integer };

    println!("{bits} = {bits:#x}");
    println!("то же без unsafe: {:#x}", 1.0f32.to_bits());
}
```

Результат:

```text
1065353216 = 0x3f800000
то же без unsafe: 0x3f800000
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=union%20Value%20%7B%0A%20%20%20%20integer%3A%20u32%2C%0A%20%20%20%20float%3A%20f32%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20Value%20%7B%20float%3A%201.0%20%7D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%83%20u32%20%D0%B8%20f32%20%D0%BD%D0%B5%D1%82%20%D0%BD%D0%B5%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D0%BC%D1%8B%D1%85%20%D0%B1%D0%B8%D1%82%D0%BE%D0%B2%D1%8B%D1%85%20%D0%BE%D0%B1%D1%80%D0%B0%D0%B7%D0%BE%D0%B2%2C%0A%20%20%20%20%2F%2F%20%D0%BF%D0%BE%D1%8D%D1%82%D0%BE%D0%BC%D1%83%20%D1%87%D1%82%D0%B5%D0%BD%D0%B8%D0%B5%20%D1%87%D1%83%D0%B6%D0%BE%D0%B3%D0%BE%20%D0%BF%D0%BE%D0%BB%D1%8F%20%D0%B4%D0%B0%D1%91%D1%82%20%D0%BE%D0%BF%D1%80%D0%B5%D0%B4%D0%B5%D0%BB%D1%91%D0%BD%D0%BD%D1%8B%D0%B9%20%D1%80%D0%B5%D0%B7%D1%83%D0%BB%D1%8C%D1%82%D0%B0%D1%82.%0A%20%20%20%20let%20bits%20%3D%20unsafe%20%7B%20value.integer%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bbits%7D%20%3D%20%7Bbits%3A%23x%7D%22%29%3B%0A%20%20%20%20println%21%28%22%D1%82%D0%BE%20%D0%B6%D0%B5%20%D0%B1%D0%B5%D0%B7%20unsafe%3A%20%7B%3A%23x%7D%22%2C%201.0f32.to_bits%28%29%29%3B%0A%7D%0A)

Здесь чтение «чужого» поля корректно, потому что у `u32` и `f32` нет недопустимых битовых образов: любой набор из 32 бит — валидное значение обоих типов. Для типа вроде `bool` или `char` та же операция могла бы породить недопустимое значение, то есть UB. И обратите внимание: для этой конкретной задачи в стандартной библиотеке есть безопасный метод `f32::to_bits`, поэтому `union` здесь — лишь иллюстрация.

### Дополнительно: `unsafe extern` и `unsafe`-атрибуты (Edition 2024)

В Edition 2024 блок `extern` обязан быть записан как `unsafe extern`, а атрибуты, способные нарушить безопасность (например, `no_mangle`), — как `#[unsafe(...)]`. Внутри `unsafe extern` каждую функцию можно пометить как `safe` или `unsafe`: автор объявления берёт на себя ответственность за правильность сигнатуры, а вызывающий код для `safe fn` уже не нуждается в `unsafe`:

```rust
unsafe extern "C" {
    pub safe fn abs(input: i32) -> i32;
    pub unsafe fn strlen(s: *const std::ffi::c_char) -> usize;
}

#[unsafe(no_mangle)]
pub extern "C" fn exported() -> i32 {
    7
}

fn main() {
    println!("{}", abs(-5)); // safe fn: unsafe не нужен

    let s = c"hello";
    // SAFETY: s — валидная C-строка с нулевым терминатором.
    println!("{}", unsafe { strlen(s.as_ptr()) });

    println!("{}", exported());
}
```

Результат:

```text
5
5
7
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=unsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20pub%20safe%20fn%20abs%28input%3A%20i32%29%20-%3E%20i32%3B%0A%20%20%20%20pub%20unsafe%20fn%20strlen%28s%3A%20%2Aconst%20std%3A%3Affi%3A%3Ac_char%29%20-%3E%20usize%3B%0A%7D%0A%0A%23%5Bunsafe%28no_mangle%29%5D%0Apub%20extern%20%22C%22%20fn%20exported%28%29%20-%3E%20i32%20%7B%0A%20%20%20%207%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20abs%28-5%29%29%3B%20%2F%2F%20safe%20fn%3A%20unsafe%20%D0%BD%D0%B5%20%D0%BD%D1%83%D0%B6%D0%B5%D0%BD%0A%0A%20%20%20%20let%20s%20%3D%20c%22hello%22%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20s%20%E2%80%94%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D0%B0%D1%8F%20C-%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B0%20%D1%81%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D1%8B%D0%BC%20%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B0%D1%82%D0%BE%D1%80%D0%BE%D0%BC.%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20unsafe%20%7B%20strlen%28s.as_ptr%28%29%29%20%7D%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20exported%28%29%29%3B%0A%7D%0A)

Сигнатуры функций C компилятор проверить не может, поэтому именно объявление в `unsafe extern` — место, где даётся обещание. Подробно FFI мы разберём в отдельной главе.

---

## 44.3. Сырые указатели (`*const T`, `*mut T`)

Сырые указатели — один из главных инструментов низкоуровневого Rust.

```rust
*const T
*mut T
```

Они существенно отличаются от обычных ссылок `&T` и `&mut T`.

Сырой указатель:

- может быть `null`;
- не обязан указывать на объект, который сейчас существует;
- не имеет обычных гарантий Rust-ссылки;
- может иметь неправильное выравнивание;
- не гарантирует уникальность изменяемого доступа;
- не гарантирует валидность памяти;
- может быть получен из операций низкого уровня;
- может использоваться для FFI.

Создание сырого указателя само по себе обычно безопасно:

```rust
let value = 42;
let ptr = &value as *const i32;
```

Но разыменование требует `unsafe`:

```rust
unsafe {
    println!("{}", *ptr);
}
```

### Сырой указатель может быть `null`

```rust
fn main() {
    let ptr: *const i32 = std::ptr::null();

    println!("is null: {}", ptr.is_null());

    // Это UB и делать так нельзя:
    //
    // unsafe {
    //     println!("{}", *ptr);
    // }
}
```

Важно:

> Разыменование `null` — **не «обычная ошибка» и не гарантированная паника**. Это undefined behavior.

---

### `Box::into_raw` и `Box::from_raw`

Сырые указатели часто используются при ручном управлении владением:

```rust
fn main() {
    let value = Box::new(42);

    let ptr = Box::into_raw(value);

    // SAFETY: ptr получен из Box::into_raw и пока не освобождён.
    unsafe {
        println!("{}", *ptr);

        // Возвращаем владение Box.
        drop(Box::from_raw(ptr));
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20Box%3A%3Anew%2842%29%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20Box%3A%3Ainto_raw%28value%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20Box%3A%3Ainto_raw%20%D0%B8%20%D0%BF%D0%BE%D0%BA%D0%B0%20%D0%BD%D0%B5%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D1%91%D0%BD.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr%29%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%92%D0%BE%D0%B7%D0%B2%D1%80%D0%B0%D1%89%D0%B0%D0%B5%D0%BC%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%20Box.%0A%20%20%20%20%20%20%20%20drop%28Box%3A%3Afrom_raw%28ptr%29%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Здесь есть важный контракт. Указатель, переданный в `Box::from_raw`, должен быть получен из `Box::into_raw` (или выделен тем же аллокатором с подходящим layout), а владение этой аллокацией не должно уже находиться у другого владельца.

Нельзя бездумно делать:

```rust
unsafe {
    let a = Box::from_raw(ptr);
    let b = Box::from_raw(ptr);
}
```

Это приводит к двойному освобождению памяти (double free), то есть к UB.

---

## 44.4. `unsafe fn`

`unsafe fn` сообщает вызывающему коду:

> «Эта функция имеет дополнительные требования безопасности, которые компилятор не может проверить».

Например:

```rust
/// Reads an `i32` from `ptr`.
///
/// # Safety
///
/// `ptr` must be non-null, properly aligned, point to a valid
/// initialized `i32`, and remain valid for the duration of the read.
unsafe fn read_i32(ptr: *const i32) -> i32 {
    // SAFETY: контракт функции требует валидный указатель.
    unsafe { *ptr }
}

fn main() {
    let value = 42;
    let ptr = &value as *const i32;

    // SAFETY: ptr получен из ссылки на живую переменную value.
    let result = unsafe { read_i32(ptr) };

    println!("{result}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%2F%20Reads%20an%20%60i32%60%20from%20%60ptr%60.%0A%2F%2F%2F%0A%2F%2F%2F%20%23%20Safety%0A%2F%2F%2F%0A%2F%2F%2F%20%60ptr%60%20must%20be%20non-null%2C%20properly%20aligned%2C%20point%20to%20a%20valid%0A%2F%2F%2F%20initialized%20%60i32%60%2C%20and%20remain%20valid%20for%20the%20duration%20of%20the%20read.%0Aunsafe%20fn%20read_i32%28ptr%3A%20%2Aconst%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D1%8B%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C.%0A%20%20%20%20unsafe%20%7B%20%2Aptr%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%20%20%20%20let%20ptr%20%3D%20%26value%20as%20%2Aconst%20i32%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8%20%D0%BD%D0%B0%20%D0%B6%D0%B8%D0%B2%D1%83%D1%8E%20%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%BD%D1%83%D1%8E%20value.%0A%20%20%20%20let%20result%20%3D%20unsafe%20%7B%20read_i32%28ptr%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A)

### `unsafe fn` не делает всё её тело автоматически `unsafe`

Это особенно важно в современном Rust. До Edition 2024 тело `unsafe fn` было неявным `unsafe`-контекстом: в нём можно было выполнять небезопасные операции без отдельного блока. Тем самым смешивались две разные вещи:

- `unsafe fn` — **требование к вызывающему** («соблюдайте мой контракт»);
- `unsafe { ... }` — **обещание автора** («я проверил условия этой конкретной операции»).

В Edition 2024 lint `unsafe_op_in_unsafe_fn` включён по умолчанию (уровень `warn`). Такой код:

```rust
unsafe fn read_i32(ptr: *const i32) -> i32 {
    *ptr // операция без unsafe-блока
}

fn main() {
    let value = 42;
    let ptr = &value as *const i32;

    let result = unsafe { read_i32(ptr) };

    println!("{result}");
}
```

приводит к предупреждению:

```text
warning[E0133]: dereference of raw pointer is unsafe and requires unsafe block
 --> src/main.rs:2:5
  |
2 |     *ptr // операция без unsafe-блока
  |     ^^^^ dereference of raw pointer
  |
note: an unsafe function restricts its caller, but its body is safe by default
```

[▶ Открыть пример с предупреждением в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=unsafe%20fn%20read_i32%28ptr%3A%20%2Aconst%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20%2Aptr%20%2F%2F%20%D0%BE%D0%BF%D0%B5%D1%80%D0%B0%D1%86%D0%B8%D1%8F%20%D0%B1%D0%B5%D0%B7%20unsafe-%D0%B1%D0%BB%D0%BE%D0%BA%D0%B0%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%20%20%20%20let%20ptr%20%3D%20%26value%20as%20%2Aconst%20i32%3B%0A%0A%20%20%20%20let%20result%20%3D%20unsafe%20%7B%20read_i32%28ptr%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A)

Исправление — явно ограничить каждую небезопасную операцию блоком `unsafe`, как в примере выше. В старых editions то же поведение включается вручную:

```rust
#![warn(unsafe_op_in_unsafe_fn)]
```

Такой стиль позволяет отличать:

```rust
unsafe fn
```

от:

```rust
unsafe {
    // конкретная unsafe-операция
}
```

Первая конструкция является частью **API и safety contract**, вторая — конкретным местом, где выполняется операция, требующая доверия.

---

## 44.5. Safety Contracts и Invariants

Чтобы писать `unsafe` правильно, необходимо понимать два понятия:

- **safety contract** — условия, которые должны быть выполнены для безопасного использования операции;
- **invariant** — свойство, которое должно оставаться истинным на протяжении существования структуры или абстракции.

Например, рассмотрим структуру, хранящую указатель и длину:

```rust
use std::marker::PhantomData;

struct MySlice<'a> {
    ptr: *const i32,
    len: usize,
    marker: PhantomData<&'a [i32]>,
}
```

У такой структуры есть инварианты:

1. `ptr` должен указывать на начало массива элементов `i32`;
2. первые `len` элементов должны существовать;
3. память должна быть корректно выровнена;
4. элементы должны быть инициализированы;
5. память должна оставаться живой всё время жизни `'a`;
6. пока живёт `MySlice`, эту память нельзя изменять через другие указатели.

Создание такой структуры из произвольного указателя поэтому является `unsafe`. Каждый безопасный метод ниже опирается на эти инварианты, а в `unsafe`-блоке мы записываем, на какой именно пункт контракта он опирается:

```rust
use std::marker::PhantomData;

struct MySlice<'a> {
    ptr: *const i32,
    len: usize,
    marker: PhantomData<&'a [i32]>,
}

impl<'a> MySlice<'a> {
    /// Creates a `MySlice` from a raw pointer and length.
    ///
    /// # Safety
    ///
    /// - `ptr` must be valid for reads of `len` consecutive, initialized `i32` values.
    /// - `ptr` must be properly aligned.
    /// - The memory must remain valid for lifetime `'a` and must not be
    ///   mutated through other pointers during that time.
    /// - `len * size_of::<i32>()` must not exceed `isize::MAX`.
    pub unsafe fn from_raw_parts(ptr: *const i32, len: usize) -> Self {
        Self {
            ptr,
            len,
            marker: PhantomData,
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn get(&self, index: usize) -> Option<i32> {
        if index >= self.len {
            return None;
        }

        // SAFETY: index < len, а контракт from_raw_parts гарантирует,
        // что первые len элементов валидны для чтения.
        unsafe { Some(*self.ptr.add(index)) }
    }
}

fn main() {
    let values = [10, 20, 30, 40];

    // SAFETY: указатель и длина получены из живого массива values.
    let slice = unsafe { MySlice::from_raw_parts(values.as_ptr(), values.len()) };

    println!("length = {}", slice.len());
    println!("item 2 = {:?}", slice.get(2));
    println!("item 9 = {:?}", slice.get(9));
}
```

Результат:

```text
length = 4
item 2 = Some(30)
item 9 = None
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct%20MySlice%3C%27a%3E%20%7B%0A%20%20%20%20ptr%3A%20%2Aconst%20i32%2C%0A%20%20%20%20len%3A%20usize%2C%0A%20%20%20%20marker%3A%20PhantomData%3C%26%27a%20%5Bi32%5D%3E%2C%0A%7D%0A%0Aimpl%3C%27a%3E%20MySlice%3C%27a%3E%20%7B%0A%20%20%20%20%2F%2F%2F%20Creates%20a%20%60MySlice%60%20from%20a%20raw%20pointer%20and%20length.%0A%20%20%20%20%2F%2F%2F%0A%20%20%20%20%2F%2F%2F%20%23%20Safety%0A%20%20%20%20%2F%2F%2F%0A%20%20%20%20%2F%2F%2F%20-%20%60ptr%60%20must%20be%20valid%20for%20reads%20of%20%60len%60%20consecutive%2C%20initialized%20%60i32%60%20values.%0A%20%20%20%20%2F%2F%2F%20-%20%60ptr%60%20must%20be%20properly%20aligned.%0A%20%20%20%20%2F%2F%2F%20-%20The%20memory%20must%20remain%20valid%20for%20lifetime%20%60%27a%60%20and%20must%20not%20be%0A%20%20%20%20%2F%2F%2F%20%20%20mutated%20through%20other%20pointers%20during%20that%20time.%0A%20%20%20%20%2F%2F%2F%20-%20%60len%20%2A%20size_of%3A%3A%3Ci32%3E%28%29%60%20must%20not%20exceed%20%60isize%3A%3AMAX%60.%0A%20%20%20%20pub%20unsafe%20fn%20from_raw_parts%28ptr%3A%20%2Aconst%20i32%2C%20len%3A%20usize%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20ptr%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20len%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20marker%3A%20PhantomData%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20len%28%26self%29%20-%3E%20usize%20%7B%0A%20%20%20%20%20%20%20%20self.len%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20get%28%26self%2C%20index%3A%20usize%29%20-%3E%20Option%3Ci32%3E%20%7B%0A%20%20%20%20%20%20%20%20if%20index%20%3E%3D%20self.len%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20return%20None%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20index%20%3C%20len%2C%20%D0%B0%20%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82%20from_raw_parts%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%2C%0A%20%20%20%20%20%20%20%20%2F%2F%20%D1%87%D1%82%D0%BE%20%D0%BF%D0%B5%D1%80%D0%B2%D1%8B%D0%B5%20len%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%BE%D0%B2%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D1%8B%20%D0%B4%D0%BB%D1%8F%20%D1%87%D1%82%D0%B5%D0%BD%D0%B8%D1%8F.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20Some%28%2Aself.ptr.add%28index%29%29%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20values%20%3D%20%5B10%2C%2020%2C%2030%2C%2040%5D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%B8%20%D0%B4%D0%BB%D0%B8%D0%BD%D0%B0%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D1%8B%20%D0%B8%D0%B7%20%D0%B6%D0%B8%D0%B2%D0%BE%D0%B3%D0%BE%20%D0%BC%D0%B0%D1%81%D1%81%D0%B8%D0%B2%D0%B0%20values.%0A%20%20%20%20let%20slice%20%3D%20unsafe%20%7B%20MySlice%3A%3Afrom_raw_parts%28values.as_ptr%28%29%2C%20values.len%28%29%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22length%20%3D%20%7B%7D%22%2C%20slice.len%28%29%29%3B%0A%20%20%20%20println%21%28%22item%202%20%3D%20%7B%3A%3F%7D%22%2C%20slice.get%282%29%29%3B%0A%20%20%20%20println%21%28%22item%209%20%3D%20%7B%3A%3F%7D%22%2C%20slice.get%289%29%29%3B%0A%7D%0A)

Здесь `PhantomData<&'a [i32]>` важен не для выполнения чтения, а для **моделирования отношения владения/заимствования в типовой системе**: он сообщает компилятору, что `MySlice<'a>` ведёт себя как заимствование среза на время `'a`.

Обратите внимание на метод `get`: он **безопасен**, и поэтому он обязан проверить `index >= self.len` сам. Если убрать эту проверку, любой код без единого `unsafe` смог бы вызвать UB. Именно так инварианты превращают локальный `unsafe` в безопасный API.

> **Замечание.** Это учебный пример. В реальном коде такая структура не нужна: `&'a [i32]` — это ровно то же самое, а построить срез из указателя и длины позволяет `std::slice::from_raw_parts` с тем же самым контрактом.

---

## 44.6. Безопасные абстракции над `unsafe`

Главная идея `unsafe` в хорошо спроектированном Rust-коде выглядит так:

```text
unsafe implementation
        │
        ▼
проверяем invariants
        │
        ▼
безопасная абстракция
        │
        ▼
safe API
```

Пользователь безопасного API не должен каждый раз самостоятельно доказывать те же условия.

### Пример: собственный владеющий указатель

Если нам действительно нужен низкоуровневый указатель, можно инкапсулировать его. Фактически мы напишем упрощённый `Box<T>`:

```rust
use std::{marker::PhantomData, ptr::NonNull};

pub struct SafePtr<T> {
    ptr: NonNull<T>,
    marker: PhantomData<T>,
}

impl<T> SafePtr<T> {
    pub fn new(value: T) -> Self {
        let ptr = Box::into_raw(Box::new(value));

        // SAFETY: Box::into_raw никогда не возвращает null.
        let ptr = unsafe { NonNull::new_unchecked(ptr) };

        Self {
            ptr,
            marker: PhantomData,
        }
    }

    pub fn get(&self) -> &T {
        // SAFETY: ptr валиден, пока жив self, а изменяемых ссылок на него нет,
        // потому что get_mut требует &mut self.
        unsafe { self.ptr.as_ref() }
    }

    pub fn get_mut(&mut self) -> &mut T {
        // SAFETY: &mut self гарантирует уникальный доступ.
        unsafe { self.ptr.as_mut() }
    }
}

impl<T> Drop for SafePtr<T> {
    fn drop(&mut self) {
        // SAFETY: ptr получен из Box::into_raw в new и освобождается ровно один раз.
        unsafe {
            drop(Box::from_raw(self.ptr.as_ptr()));
        }
    }
}

fn main() {
    let mut value = SafePtr::new(42);

    println!("{}", value.get());

    *value.get_mut() = 100;

    println!("{}", value.get());

    let text = SafePtr::new(String::from("hi"));
    println!("{}", text.get());
}
```

Результат:

```text
42
100
hi
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3A%7Bmarker%3A%3APhantomData%2C%20ptr%3A%3ANonNull%7D%3B%0A%0Apub%20struct%20SafePtr%3CT%3E%20%7B%0A%20%20%20%20ptr%3A%20NonNull%3CT%3E%2C%0A%20%20%20%20marker%3A%20PhantomData%3CT%3E%2C%0A%7D%0A%0Aimpl%3CT%3E%20SafePtr%3CT%3E%20%7B%0A%20%20%20%20pub%20fn%20new%28value%3A%20T%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20let%20ptr%20%3D%20Box%3A%3Ainto_raw%28Box%3A%3Anew%28value%29%29%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20Box%3A%3Ainto_raw%20%D0%BD%D0%B8%D0%BA%D0%BE%D0%B3%D0%B4%D0%B0%20%D0%BD%D0%B5%20%D0%B2%D0%BE%D0%B7%D0%B2%D1%80%D0%B0%D1%89%D0%B0%D0%B5%D1%82%20null.%0A%20%20%20%20%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20NonNull%3A%3Anew_unchecked%28ptr%29%20%7D%3B%0A%0A%20%20%20%20%20%20%20%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20ptr%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20marker%3A%20PhantomData%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20get%28%26self%29%20-%3E%20%26T%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%B5%D0%BD%2C%20%D0%BF%D0%BE%D0%BA%D0%B0%20%D0%B6%D0%B8%D0%B2%20self%2C%20%D0%B0%20%D0%B8%D0%B7%D0%BC%D0%B5%D0%BD%D1%8F%D0%B5%D0%BC%D1%8B%D1%85%20%D1%81%D1%81%D1%8B%D0%BB%D0%BE%D0%BA%20%D0%BD%D0%B0%20%D0%BD%D0%B5%D0%B3%D0%BE%20%D0%BD%D0%B5%D1%82%2C%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%BF%D0%BE%D1%82%D0%BE%D0%BC%D1%83%20%D1%87%D1%82%D0%BE%20get_mut%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82%20%26mut%20self.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20self.ptr.as_ref%28%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20get_mut%28%26mut%20self%29%20-%3E%20%26mut%20T%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%26mut%20self%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%20%D1%83%D0%BD%D0%B8%D0%BA%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9%20%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20self.ptr.as_mut%28%29%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%3CT%3E%20Drop%20for%20SafePtr%3CT%3E%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20Box%3A%3Ainto_raw%20%D0%B2%20new%20%D0%B8%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D1%80%D0%BE%D0%B2%D0%BD%D0%BE%20%D0%BE%D0%B4%D0%B8%D0%BD%20%D1%80%D0%B0%D0%B7.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20drop%28Box%3A%3Afrom_raw%28self.ptr.as_ptr%28%29%29%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%20SafePtr%3A%3Anew%2842%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20value.get%28%29%29%3B%0A%0A%20%20%20%20%2Avalue.get_mut%28%29%20%3D%20100%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20value.get%28%29%29%3B%0A%0A%20%20%20%20let%20text%20%3D%20SafePtr%3A%3Anew%28String%3A%3Afrom%28%22hi%22%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20text.get%28%29%29%3B%0A%7D%0A)

Обратите внимание: снаружи нет ни одного `unsafe`.

Это и есть один из главных признаков хорошей unsafe-абстракции:

```rust
let mut value = SafePtr::new(42);

*value.get_mut() = 100;
```

Пользователь работает с обычным безопасным API, а все опасные операции сосредоточены внутри реализации. Инварианты `SafePtr<T>` такие:

1. поле `ptr` всегда указывает на живое значение `T`, выделенное через `Box`;
2. `SafePtr` — единственный владелец этого значения;
3. значение освобождается ровно один раз, в `Drop`;
4. `get_mut` требует `&mut self`, поэтому изменяемая и разделяемая ссылки одновременно существовать не могут — это и есть доказательство того, что `as_ref`/`as_mut` не нарушают правила aliasing.

При этом наличие `unsafe` внутри реализации **само по себе не делает абстракцию безопасной**. Автор должен доказать, что инварианты действительно сохраняются при любых допустимых вызовах публичного API. В реальном коде вместо `SafePtr<T>` просто используйте `Box<T>`: он делает ровно это и уже проверен.

---

## 44.7. `unsafe` Traits

`unsafe trait` используется тогда, когда сам факт реализации трейта должен гарантировать некоторое свойство, которое компилятор не способен проверить автоматически.

```rust
/// Marker trait for types satisfying a custom safety contract.
///
/// # Safety
///
/// Implementors must satisfy the invariants required by the
/// code that relies on this trait.
unsafe trait Sendable {}

unsafe impl Sendable for i32 {}
```

Почему `unsafe impl`?

Потому что компилятор не может проверить произвольный safety contract трейта.

### Важное отличие от обычного трейта

Для обычного трейта:

```rust
trait MyTrait {
    fn method(&self);
}
```

компилятор проверяет, что реализация соответствует требованиям самого языка: методы существуют, типы совпадают и т. д.

Для `unsafe trait` существует ещё один уровень:

```text
компилятор проверяет синтаксис и типы
                    +
программист гарантирует safety contract
```

Поэтому `unsafe impl` следует рассматривать как **утверждение программиста**, а не как способ получить дополнительные возможности.

### Реальный пример: `Send`

Самые известные `unsafe trait` в стандартной библиотеке — `Send` и `Sync`, о которых шла речь в главе про потокобезопасность. Сырые указатели не реализуют `Send`, поэтому структура с `*mut i32` по умолчанию не может быть передана в другой поток:

```rust
use std::thread;

struct Wrapper(*mut i32);

fn main() {
    let w = Wrapper(Box::into_raw(Box::new(41)));

    let handle = thread::spawn(move || {
        let _w = w;
    });

    handle.join().unwrap();
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Astruct%20Wrapper%28%2Amut%20i32%29%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20w%20%3D%20Wrapper%28Box%3A%3Ainto_raw%28Box%3A%3Anew%2841%29%29%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20let%20_w%20%3D%20w%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle.join%28%29.unwrap%28%29%3B%0A%7D%0A)

Компилятор сообщит:

```text
error[E0277]: `*mut i32` cannot be sent between threads safely
```

Если мы действительно знаем, что `Wrapper` — единственный владелец данных и после передачи в поток к ним больше никто не обратится, можно дать компилятору обещание:

```rust
use std::thread;

struct Wrapper(*mut i32);

// SAFETY: Wrapper — единственный владелец данных, на которые указывает
// указатель; после передачи в поток другого доступа к ним нет.
unsafe impl Send for Wrapper {}

fn main() {
    let w = Wrapper(Box::into_raw(Box::new(41)));

    let handle = thread::spawn(move || {
        let w = w;

        // SAFETY: указатель получен из Box::into_raw и больше нигде не используется.
        let boxed = unsafe { Box::from_raw(w.0) };
        *boxed + 1
    });

    println!("{}", handle.join().unwrap());
}
```

Результат:

```text
42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Athread%3B%0A%0Astruct%20Wrapper%28%2Amut%20i32%29%3B%0A%0A%2F%2F%20SAFETY%3A%20Wrapper%20%E2%80%94%20%D0%B5%D0%B4%D0%B8%D0%BD%D1%81%D1%82%D0%B2%D0%B5%D0%BD%D0%BD%D1%8B%D0%B9%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%BB%D0%B5%D1%86%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85%2C%20%D0%BD%D0%B0%20%D0%BA%D0%BE%D1%82%D0%BE%D1%80%D1%8B%D0%B5%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%0A%2F%2F%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%3B%20%D0%BF%D0%BE%D1%81%D0%BB%D0%B5%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%87%D0%B8%20%D0%B2%20%D0%BF%D0%BE%D1%82%D0%BE%D0%BA%20%D0%B4%D1%80%D1%83%D0%B3%D0%BE%D0%B3%D0%BE%20%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF%D0%B0%20%D0%BA%20%D0%BD%D0%B8%D0%BC%20%D0%BD%D0%B5%D1%82.%0Aunsafe%20impl%20Send%20for%20Wrapper%20%7B%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20w%20%3D%20Wrapper%28Box%3A%3Ainto_raw%28Box%3A%3Anew%2841%29%29%29%3B%0A%0A%20%20%20%20let%20handle%20%3D%20thread%3A%3Aspawn%28move%20%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20let%20w%20%3D%20w%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20Box%3A%3Ainto_raw%20%D0%B8%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%BD%D0%B8%D0%B3%D0%B4%D0%B5%20%D0%BD%D0%B5%20%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D1%83%D0%B5%D1%82%D1%81%D1%8F.%0A%20%20%20%20%20%20%20%20let%20boxed%20%3D%20unsafe%20%7B%20Box%3A%3Afrom_raw%28w.0%29%20%7D%3B%0A%20%20%20%20%20%20%20%20%2Aboxed%20%2B%201%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20handle.join%28%29.unwrap%28%29%29%3B%0A%7D%0A)

Теперь компилятор верит нам на слово. Если обещание ложное (например, тот же указатель остался у основного потока), получится гонка данных, то есть UB. Поэтому каждый `unsafe impl Send` должен сопровождаться комментарием, почему он корректен.

---

## 44.8. Undefined behavior: что это и почему «у меня работает» ничего не доказывает

Мы уже много раз говорили «это UB». Пора определить, что это значит.

**Undefined behavior (неопределённое поведение)** — это нарушение правил, на соблюдение которых рассчитывает компилятор. Safe Rust такие нарушения исключает, а `unsafe` перекладывает ответственность на программиста. Компилятор вправе предполагать, что UB никогда не происходит, и оптимизирует программу исходя из этого предположения. Поэтому при UB компилятор **не обязан ничего гарантировать**: ни панику, ни ошибку, ни «просто неправильный результат».

Основные источники UB в Rust:

- **гонка данных** (одновременный доступ из разных потоков, хотя бы один из них — запись, без синхронизации);
- **разыменование висячего, нулевого или невыровненного указателя**;
- **нарушение правил aliasing** (например, существование двух `&mut` на одно значение или изменение данных за `&T`);
- **создание недопустимого значения**: `bool` не 0/1, `char` вне допустимого диапазона, нулевая ссылка, неинициализированная память там, где нужны инициализированные данные, недопустимое значение `enum`;
- **выход за границы объекта** при арифметике указателей и доступе через них;
- **двойное освобождение** и использование после освобождения.

### UB может «работать»

Самая опасная особенность UB — оно не обязано проявляться. Рассмотрим висячий указатель:

```rust
fn main() {
    let ptr = {
        let x = 5;
        &x as *const i32 // x умирает в конце блока
    };

    // UB: ptr — висячий указатель.
    unsafe {
        println!("{}", *ptr);
    }
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20ptr%20%3D%20%7B%0A%20%20%20%20%20%20%20%20let%20x%20%3D%205%3B%0A%20%20%20%20%20%20%20%20%26x%20as%20%2Aconst%20i32%20%2F%2F%20x%20%D1%83%D0%BC%D0%B8%D1%80%D0%B0%D0%B5%D1%82%20%D0%B2%20%D0%BA%D0%BE%D0%BD%D1%86%D0%B5%20%D0%B1%D0%BB%D0%BE%D0%BA%D0%B0%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20%2F%2F%20UB%3A%20ptr%20%E2%80%94%20%D0%B2%D0%B8%D1%81%D1%8F%D1%87%D0%B8%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Переменная `x` уничтожена в конце внутреннего блока, а мы читаем по адресу, который ей принадлежал. Результат зависит от режима сборки (измерено на rustc 1.99, x86-64 Linux):

| Режим                   | Результат       |
| ----------------------- | --------------- |
| debug (`cargo run`)     | печатает `5`    |
| release (`-O`)          | печатает `0`    |

В debug-сборке «всё работало» — как раз потому, что память ещё не успели переиспользовать. В release тот же код печатает другое число. Ни один из этих результатов не «правильный»: программа некорректна в обоих случаях. Если ваши тесты прошли, это не доказательство отсутствия UB.

### Помощь от компилятора — не гарантия

Современные версии rustc в debug-сборках вставляют часть проверок, которые превращают некоторые UB в немедленный сбой. Сравните три примера (измерено на rustc 1.99, x86-64 Linux):

| Пример                          | debug                                                             | release (`-O`)       |
| ------------------------------- | ----------------------------------------------------------------- | -------------------- |
| разыменование `null`            | сбой: программа прерывается с сообщением о нулевой ссылке         | `Illegal instruction` |
| разыменование невыровненного    | паника `misaligned pointer dereference: address must be a multiple of 0x4 ...` | печатает `0` |
| висячий указатель               | печатает `5`                                                      | печатает `0`         |

Эти проверки — отладочная помощь. В release их нет, и программа остаётся с UB. Рассчитывать на них нельзя, да и на другой версии компилятора или платформе результат может быть другим.

### Как искать UB

1. **Miri** — интерпретатор промежуточного представления Rust, который во время выполнения отслеживает многие виды UB: висячие и невыровненные указатели, нарушение aliasing, чтение неинициализированной памяти. Запуск: `rustup +nightly component add miri`, затем `cargo +nightly miri run` (или `miri test`). Miri также доступен в Rust Playground в меню Tools. Он медленный и находит только то UB, которое реально произошло на запущенном пути, но для unsafe-кода это основной инструмент.
2. **Документированные контракты и комментарии `// SAFETY:`** — они позволяют ревьюеру проверить условия вручную (раздел 44.9).
3. **Минимальная область `unsafe`** — чем она меньше, тем меньше кода нужно проверять.

---

## 44.9. Как документировать safety requirements

Для каждой публичной `unsafe fn` необходимо явно документировать её safety contract. Для каждого `unsafe`-блока принято писать комментарий `// SAFETY:` — почему условия контракта вызываемой операции здесь выполнены.

Стандартный формат:

```rust
/// Reads a value from a raw pointer.
///
/// # Safety
///
/// - `ptr` must be non-null and properly aligned for `T`.
/// - `ptr` must point to a valid, initialized `T` and be valid for reads.
/// - The result is a bitwise copy. If `T` is not `Copy`, the caller must
///   ensure the original value is never used or dropped again.
pub unsafe fn read_from_ptr<T>(ptr: *const T) -> T {
    // SAFETY: все условия переданы вызывающему через контракт функции.
    unsafe { ptr.read() }
}

fn main() {
    let x = 7u32;

    // SAFETY: &x — валидная выровненная ссылка на инициализированный u32.
    let y = unsafe { read_from_ptr(&x) };

    println!("{y}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%2F%20Reads%20a%20value%20from%20a%20raw%20pointer.%0A%2F%2F%2F%0A%2F%2F%2F%20%23%20Safety%0A%2F%2F%2F%0A%2F%2F%2F%20-%20%60ptr%60%20must%20be%20non-null%20and%20properly%20aligned%20for%20%60T%60.%0A%2F%2F%2F%20-%20%60ptr%60%20must%20point%20to%20a%20valid%2C%20initialized%20%60T%60%20and%20be%20valid%20for%20reads.%0A%2F%2F%2F%20-%20The%20result%20is%20a%20bitwise%20copy.%20If%20%60T%60%20is%20not%20%60Copy%60%2C%20the%20caller%20must%0A%2F%2F%2F%20%20%20ensure%20the%20original%20value%20is%20never%20used%20or%20dropped%20again.%0Apub%20unsafe%20fn%20read_from_ptr%3CT%3E%28ptr%3A%20%2Aconst%20T%29%20-%3E%20T%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B2%D1%81%D0%B5%20%D1%83%D1%81%D0%BB%D0%BE%D0%B2%D0%B8%D1%8F%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D0%BD%D1%8B%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D1%8E%D1%89%D0%B5%D0%BC%D1%83%20%D1%87%D0%B5%D1%80%D0%B5%D0%B7%20%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8.%0A%20%20%20%20unsafe%20%7B%20ptr.read%28%29%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%207u32%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%26x%20%E2%80%94%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D0%B0%D1%8F%20%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%D0%BD%D0%B0%D1%8F%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B0%20%D0%BD%D0%B0%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D0%B9%20u32.%0A%20%20%20%20let%20y%20%3D%20unsafe%20%7B%20read_from_ptr%28%26x%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7By%7D%22%29%3B%0A%7D%0A)

Здесь важно не просто написать:

```rust
/// # Safety
/// Caller must ensure the pointer is valid.
```

Это слишком расплывчато.

Хороший safety contract должен отвечать на вопрос:

> **Что именно вызывающий обязан гарантировать?**

Например:

```text
ptr != null
ptr корректно выровнен
память существует
значение инициализировано
доступ разрешён
память не освобождена
требования aliasing соблюдены
```

Конкретные требования зависят от операции. Обратите внимание на последний пункт контракта выше: `ptr.read()` создаёт **побитовую копию**. Для `Copy`-типа это безвредно, а для, скажем, `String` дальнейшее использование и копии, и оригинала привело бы к двойному освобождению. Такие требования легко упустить, если не перечитывать документацию вызываемой функции.

### `# Safety` — часть API

Документация `unsafe fn` фактически является частью контракта API:

```rust
unsafe fn operation(...)
```

означает:

> «Перед вызовом прочитайте `# Safety` и докажите, что его условия выполнены».

Поэтому safety-документация должна быть настолько конкретной, чтобы другой разработчик мог проверить её выполнение.

### Инструменты

Clippy умеет проверять эти соглашения:

- `clippy::missing_safety_doc` (включён по умолчанию) — предупреждает, если у публичной `unsafe fn` нет раздела `# Safety`;
- `clippy::undocumented_unsafe_blocks` — предупреждает об `unsafe`-блоке без комментария `// SAFETY:`. Он не включён по умолчанию; включить его можно так: `#![warn(clippy::undocumented_unsafe_blocks)]`.

---

## 44.10. Когда использовать `unsafe`

**Хорошие причины использовать `unsafe`:**

1. **Взаимодействие с FFI (C-кодом).**
2. **Реализация эффективных структур данных** (то, что в стандартной библиотеке сделано за вас: `Vec`, `Box`, `Rc`).
3. **Оптимизация горячих путей** — только после профилирования.
4. **Работа с памятью на низком уровне.**

**Плохие причины использовать `unsafe`:**

1. «Я знаю, что делаю» (без реального понимания).
2. «Так быстрее» (без доказательств).
3. «Мне лень переделывать» (без необходимости).

### Пример: «так быстрее»?

Классическое искушение — убрать проверку границ через `get_unchecked`:

```rust
fn sum_indexed(data: &[u64]) -> u64 {
    let mut total = 0;
    for i in 0..data.len() {
        total += data[i];
    }
    total
}

fn sum_unchecked(data: &[u64]) -> u64 {
    let mut total = 0;
    for i in 0..data.len() {
        // SAFETY: i < data.len() на каждой итерации.
        total += unsafe { *data.get_unchecked(i) };
    }
    total
}

fn sum_iter(data: &[u64]) -> u64 {
    data.iter().sum()
}

fn main() {
    let data: Vec<u64> = (1..=1000).collect();

    println!("{}", sum_indexed(&data));
    println!("{}", sum_unchecked(&data));
    println!("{}", sum_iter(&data));
}
```

Результат:

```text
500500
500500
500500
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20sum_indexed%28data%3A%20%26%5Bu64%5D%29%20-%3E%20u64%20%7B%0A%20%20%20%20let%20mut%20total%20%3D%200%3B%0A%20%20%20%20for%20i%20in%200..data.len%28%29%20%7B%0A%20%20%20%20%20%20%20%20total%20%2B%3D%20data%5Bi%5D%3B%0A%20%20%20%20%7D%0A%20%20%20%20total%0A%7D%0A%0Afn%20sum_unchecked%28data%3A%20%26%5Bu64%5D%29%20-%3E%20u64%20%7B%0A%20%20%20%20let%20mut%20total%20%3D%200%3B%0A%20%20%20%20for%20i%20in%200..data.len%28%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20i%20%3C%20data.len%28%29%20%D0%BD%D0%B0%20%D0%BA%D0%B0%D0%B6%D0%B4%D0%BE%D0%B9%20%D0%B8%D1%82%D0%B5%D1%80%D0%B0%D1%86%D0%B8%D0%B8.%0A%20%20%20%20%20%20%20%20total%20%2B%3D%20unsafe%20%7B%20%2Adata.get_unchecked%28i%29%20%7D%3B%0A%20%20%20%20%7D%0A%20%20%20%20total%0A%7D%0A%0Afn%20sum_iter%28data%3A%20%26%5Bu64%5D%29%20-%3E%20u64%20%7B%0A%20%20%20%20data.iter%28%29.sum%28%29%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20data%3A%20Vec%3Cu64%3E%20%3D%20%281..%3D1000%29.collect%28%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20sum_indexed%28%26data%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20sum_unchecked%28%26data%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20sum_iter%28%26data%29%29%3B%0A%7D%0A)

Три функции возвращают одно и то же, но `sum_unchecked` содержит `unsafe`, а `sum_iter` — нет. Итератор по срезу обычно не уступает варианту с `get_unchecked`: компилятор часто убирает проверки границ в простых циклах, а в итераторах их нет по построению. Поэтому порядок действий такой:

1. написать безопасный вариант;
2. **измерить**, действительно ли он медленный (`cargo bench`, `criterion`, профилировщик);
3. только затем рассматривать `unsafe`, оставив безопасную версию как эталон для тестов;
4. записать `// SAFETY:` с доказательством, что индекс всегда в границах.

Выигрыш от `get_unchecked` без этих шагов — выдуманный, а риск — настоящий.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Разыменование `null`

```rust
fn main() {
    let ptr: *const i32 = std::ptr::null();

    unsafe {
        println!("{}", *ptr);
    }
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20ptr%3A%20%2Aconst%20i32%20%3D%20std%3A%3Aptr%3A%3Anull%28%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Этот код проходит компиляцию, потому что `unsafe` разрешает саму операцию.

Но это **не делает операцию корректной**.

Разыменование `null` приводит к undefined behavior. На rustc 1.99 мы видели, что в debug-сборке программа прерывается с сообщением о нулевой ссылке, а в release завершается аварийно (`Illegal instruction`). Это не повод считать, что «Rust вызовет панику»: другие версии, платформы и уровни оптимизации могут вести себя иначе. При UB компилятор не обязан предоставлять какой-либо конкретный результат.

Если указатель может быть нулевым, проверьте это явно:

```rust
fn main() {
    let ptr: *const i32 = std::ptr::null();

    // SAFETY: as_ref допускает null и вернёт None; для ненулевого указателя
    // он по-прежнему требует, чтобы тот был валидным.
    match unsafe { ptr.as_ref() } {
        Some(value) => println!("значение: {value}"),
        None => println!("указатель нулевой"),
    }
}
```

Результат:

```text
указатель нулевой
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20ptr%3A%20%2Aconst%20i32%20%3D%20std%3A%3Aptr%3A%3Anull%28%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20as_ref%20%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D0%BA%D0%B0%D0%B5%D1%82%20null%20%D0%B8%20%D0%B2%D0%B5%D1%80%D0%BD%D1%91%D1%82%20None%3B%20%D0%B4%D0%BB%D1%8F%20%D0%BD%D0%B5%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B3%D0%BE%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8F%0A%20%20%20%20%2F%2F%20%D0%BE%D0%BD%20%D0%BF%D0%BE-%D0%BF%D1%80%D0%B5%D0%B6%D0%BD%D0%B5%D0%BC%D1%83%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82%2C%20%D1%87%D1%82%D0%BE%D0%B1%D1%8B%20%D1%82%D0%BE%D1%82%20%D0%B1%D1%8B%D0%BB%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D1%8B%D0%BC.%0A%20%20%20%20match%20unsafe%20%7B%20ptr.as_ref%28%29%20%7D%20%7B%0A%20%20%20%20%20%20%20%20Some%28value%29%20%3D%3E%20println%21%28%22%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%3A%20%7Bvalue%7D%22%29%2C%0A%20%20%20%20%20%20%20%20None%20%3D%3E%20println%21%28%22%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B9%22%29%2C%0A%20%20%20%20%7D%0A%7D%0A)

Метод `as_ref` возвращает `Option<&T>`: для `null` он даёт `None`. Он по-прежнему `unsafe`, потому что для ненулевого указателя обязан быть выполнен остальной контракт (валидность, выравнивание).

---

### Эксперимент 2: Неправильное выравнивание

Следующий пример намеренно создаёт неправильно выровненный указатель:

```rust
fn main() {
    let bytes = [0u8; 8];

    let ptr = unsafe { bytes.as_ptr().add(1) as *const u32 };

    unsafe {
        println!("{}", *ptr);
    }
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20bytes%20%3D%20%5B0u8%3B%208%5D%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20bytes.as_ptr%28%29.add%281%29%20as%20%2Aconst%20u32%20%7D%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Адрес `bytes.as_ptr().add(1)` на единицу больше адреса начала массива, а `u32` требует выравнивания по 4 байтам. В debug-сборке rustc прерывает программу с сообщением вида:

```text
misaligned pointer dereference: address must be a multiple of 0x4 but is 0x7ffd...
```

В release-сборке этой проверки нет, и программа молча печатает `0` — но это по-прежнему UB.

Проблема здесь не только в выравнивании. Чтобы безопасно разыменовать `ptr`, необходимо также гарантировать:

- корректное выравнивание;
- достаточный размер доступной памяти;
- валидность указателя;
- инициализированность `u32`;
- соблюдение остальных требований к доступу через указатель.

Поэтому такой код нельзя использовать как способ «прочитать четыре байта». Для работы с байтами следует использовать безопасные операции `u32::from_ne_bytes`, `u32::from_le_bytes` или `u32::from_be_bytes`. А если вам действительно нужен указатель, используйте `read_unaligned`, который допускает невыровненный адрес:

```rust
fn main() {
    let bytes = [1u8, 2, 3, 4, 5, 6, 7, 8];

    // Безопасно: собираем u32 из четырёх байт явно.
    let safe = u32::from_le_bytes([bytes[1], bytes[2], bytes[3], bytes[4]]);

    // Unsafe, но корректно: read_unaligned допускает невыровненный указатель.
    let ptr = unsafe { bytes.as_ptr().add(1) as *const u32 };
    // SAFETY: в bytes есть 4 инициализированных байта начиная со смещения 1.
    let via_ptr = unsafe { ptr.read_unaligned() };

    println!("{safe} {via_ptr} {}", safe == via_ptr);
}
```

Результат:

```text
84148994 84148994 true
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20bytes%20%3D%20%5B1u8%2C%202%2C%203%2C%204%2C%205%2C%206%2C%207%2C%208%5D%3B%0A%0A%20%20%20%20%2F%2F%20%D0%91%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%3A%20%D1%81%D0%BE%D0%B1%D0%B8%D1%80%D0%B0%D0%B5%D0%BC%20u32%20%D0%B8%D0%B7%20%D1%87%D0%B5%D1%82%D1%8B%D1%80%D1%91%D1%85%20%D0%B1%D0%B0%D0%B9%D1%82%20%D1%8F%D0%B2%D0%BD%D0%BE.%0A%20%20%20%20let%20safe%20%3D%20u32%3A%3Afrom_le_bytes%28%5Bbytes%5B1%5D%2C%20bytes%5B2%5D%2C%20bytes%5B3%5D%2C%20bytes%5B4%5D%5D%29%3B%0A%0A%20%20%20%20%2F%2F%20Unsafe%2C%20%D0%BD%D0%BE%20%D0%BA%D0%BE%D1%80%D1%80%D0%B5%D0%BA%D1%82%D0%BD%D0%BE%3A%20read_unaligned%20%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D0%BA%D0%B0%D0%B5%D1%82%20%D0%BD%D0%B5%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%D0%BD%D1%8B%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C.%0A%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20bytes.as_ptr%28%29.add%281%29%20as%20%2Aconst%20u32%20%7D%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B2%20bytes%20%D0%B5%D1%81%D1%82%D1%8C%204%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D1%85%20%D0%B1%D0%B0%D0%B9%D1%82%D0%B0%20%D0%BD%D0%B0%D1%87%D0%B8%D0%BD%D0%B0%D1%8F%20%D1%81%D0%BE%20%D1%81%D0%BC%D0%B5%D1%89%D0%B5%D0%BD%D0%B8%D1%8F%201.%0A%20%20%20%20let%20via_ptr%20%3D%20unsafe%20%7B%20ptr.read_unaligned%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bsafe%7D%20%7Bvia_ptr%7D%20%7B%7D%22%2C%20safe%20%3D%3D%20via_ptr%29%3B%0A%7D%0A)

(Число одинаково, потому что `from_le_bytes` и чтение на little-endian-платформе, например x86-64, интерпретируют байты одинаково. На big-endian-платформе вместо `from_le_bytes` нужно использовать `from_ne_bytes`.)

---

### Эксперимент 3: `static mut` в Edition 2024

Попробуйте скомпилировать код из раздела 44.2 (пример с `println!("{}", COUNTER)`). Прочитайте сообщение компилятора и обратите внимание на строку `#[deny(static_mut_refs)]`. Затем исправьте код тремя способами: скопируйте значение в локальную переменную, возьмите `&raw mut`, замените `static mut` на `AtomicU32`. Какой из способов делает программу корректной и в многопоточном коде?

---

### Эксперимент 4: обещание `Send`

В примере из раздела 44.7 уберите строку `unsafe impl Send for Wrapper {}` и убедитесь, что компилятор отклоняет `thread::spawn`. Затем верните её и объясните своими словами, какое именно утверждение вы тем самым делаете компилятору.

---

## Практика

### Задание 1. Сырой указатель

Создайте переменную:

```rust
let mut value = 42;
```

Получите `*mut i32`, измените значение через сырой указатель и выведите результат.

Обязательно объясните, **почему** разыменование указателя требует `unsafe`, и добавьте комментарий `// SAFETY:`.

---

### Задание 2. `unsafe fn` и safety contract

Напишите:

```rust
unsafe fn read_i32(ptr: *const i32) -> i32
```

и документируйте её через `# Safety`.

Ваш контракт должен явно описывать:

- требования к `null`;
- выравнивание;
- валидность памяти;
- инициализированность значения.

Затем вызовите функцию с корректным указателем. Убедитесь, что в Edition 2024 внутри функции нет предупреждения `unsafe_op_in_unsafe_fn`.

---

### Задание 3. Безопасная абстракция

Создайте структуру, которая внутри хранит сырой указатель, но предоставляет безопасные методы:

```rust
fn get(&self) -> &T
fn get_mut(&mut self) -> &mut T
```

Реализуйте `Drop`. Опишите invariants, которые должны сохраняться.

Цель задания — не просто заставить код работать, а доказать:

> почему пользователь безопасного API не может нарушить safety contract.

---

### Задание 4. Найдите UB

Для каждого из трёх фрагментов ответьте: есть ли здесь UB, и если да, то какое?

```rust
// А
let ptr: *const i32 = std::ptr::null();
unsafe {
    println!("{}", *ptr);
}
```

```rust
// Б
let value = 5;
let ptr = &value as *const i32;
unsafe {
    println!("{}", *ptr);
}
```

```rust
// В
let ptr = {
    let x = 5;
    &x as *const i32
};
unsafe {
    println!("{}", *ptr);
}
```

Затем ответьте:

1. Почему компилятор разрешает все три фрагмента?
2. Почему вывод фрагмента В «5» не означает, что код корректен?
3. Почему результат нельзя описать как «Rust гарантированно вызовет panic»?

---

### Задание 5. `unsafe` без необходимости

Сравните:

```rust
unsafe {
    println!("{}", 42);
}
```

с:

```rust
println!("{}", 42);
```

Объясните, почему первый вариант не даёт никаких преимуществ. Затем сократите область `unsafe` в следующей функции до одной небольшой операции:

```rust
fn sum_first_two(data: &[i32]) -> i32 {
    unsafe {
        let len = data.len();
        let mut total = 0;
        if len >= 2 {
            total += *data.get_unchecked(0);
            total += *data.get_unchecked(1);
        }
        total
    }
}
```

Подумайте, нужен ли здесь `unsafe` вообще.

---

### Задание 6. Safety review

Для следующей функции напишите полный safety contract:

```rust
unsafe fn copy_value<T>(src: *const T, dst: *mut T) {
    unsafe {
        std::ptr::copy_nonoverlapping(src, dst, 1);
    }
}
```

Подумайте, какие условия должны выполняться для:

- `src`;
- `dst`;
- размера `T`;
- инициализированности;
- перекрытия областей памяти;
- допустимости записи в `dst`.

Это уже не упражнение на синтаксис `unsafe`, а упражнение на **анализ safety contract**.

---

### Задание 7. `static mut` в Edition 2024

Следующий код не компилируется в Edition 2024:

```rust
static mut TOTAL: u64 = 0;

fn add(value: u64) {
    unsafe {
        TOTAL += value;
    }
}

fn main() {
    add(1);
    add(2);
    unsafe {
        println!("{}", TOTAL);
    }
}
```

Найдите причину и исправьте код двумя способами: сохранив `static mut` (но не создавая ссылок) и заменив его на `AtomicU64`. Объясните, чем второй вариант лучше.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`unsafe`** — не отключение проверок компилятора, а механизм явного доверия программисту.
- **Raw pointers** — `*const T` и `*mut T`, которые не обладают гарантиями обычных ссылок; создавать их безопасно, а разыменовывать — нет.
- **`unsafe fn`** — функция с дополнительными требованиями безопасности к вызывающему.
- **`unsafe {}`** — локальная граница конкретной unsafe-операции; в Edition 2024 даже внутри `unsafe fn` каждая такая операция должна быть в своём блоке.
- **`unsafe trait`** — трейт с safety contract, который нельзя полностью проверить компилятором; `unsafe impl Send` — обещание, а не разрешение.
- **`union`** — низкоуровневая конструкция, чтение полей которой требует `unsafe`.
- **`static mut`** — в Edition 2024 ссылки на него запрещены; предпочитайте атомарные типы.
- **Safety contract** — явное описание условий, необходимых для безопасного выполнения операции; документируется в `# Safety`, а выполнение условий — в `// SAFETY:`.
- **Invariants** — свойства, которые должны сохраняться внутри безопасной абстракции.
- **`PhantomData`** — инструмент для выражения в типовой системе отношений владения, заимствования и времени жизни, которые не представлены непосредственно хранимыми данными.
- **Безопасная абстракция** — способ локализовать `unsafe` внутри реализации и предоставить безопасный API.
- **Undefined behavior** — ситуация, при которой программа нарушает требования модели Rust; `unsafe` не делает UB допустимым. UB может «работать», может вести себя по-разному в debug и release, и отсутствие сбоя ничего не доказывает; для поиска UB используйте Miri.

**Самая важная идея:**

> `unsafe` не означает: «теперь компилятор мне не мешает».
>
> `unsafe` означает: **«здесь существует свойство безопасности, которое компилятор не способен доказать автоматически, и я беру на себя ответственность за его выполнение»**.
>
> Хороший `unsafe`-код поэтому начинается не с вопроса «как заставить это скомпилироваться?», а с вопросов:
>
> 1. Какой safety contract существует?
> 2. Какие invariants необходимы?
> 3. Где именно находится unsafe-операция?
> 4. Можно ли доказать её корректность?
> 5. Можно ли спрятать её за безопасным API?
>
> Чем меньше область `unsafe`, чем точнее safety contract и чем лучше изолированы invariants, тем надёжнее низкоуровневый Rust-код.
