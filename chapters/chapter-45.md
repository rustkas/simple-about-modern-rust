# Глава 45. Raw Pointers

В предыдущей главе мы узнали, что `unsafe` даёт доступ к пяти «сверхспособностям», и первая из них — **разыменование сырых указателей**. Сырые указатели — это основа для взаимодействия с памятью на низком уровне, FFI и реализации эффективных структур данных.

В этой главе мы детально разберёмся с сырыми указателями: что они из себя представляют, как с ними работать, какие у них есть особенности и как их безопасно использовать.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 45.1. Что такое сырой указатель?

**Сырой указатель (raw pointer)** — это значение, содержащее адрес объекта или области памяти без гарантий безопасности, которые Rust предоставляет для обычных ссылок.

В Rust есть два основных типа:

| Тип        | Назначение                                                   |
| ---------- | ------------------------------------------------------------ |
| `*const T` | сырой указатель, через который обычно читают `T`             |
| `*mut T`   | сырой указатель, через который можно читать и записывать `T` |

Важно понимать: `*const T` **не означает, что объект физически неизменяем**. Это ограничение самого указателя: через обычное разыменование `*const T` нельзя записать значение.

Но и обратное неверно: превратить `*const T` в `*mut T` приведением (`as`, `cast_mut()`) — не значит получить право на запись. Право на запись определяется тем, откуда указатель получен. Указатель, созданный из разделяемой ссылки `&T`, права на запись не имеет, и запись через него — UB. Компилятор ловит самые очевидные случаи:

```rust
fn main() {
    let x = 5;
    let p = &x as *const i32 as *mut i32;

    // Компилятор это пропустит, но запись — UB:
    // указатель получен из разделяемой ссылки &x и прав на запись не имеет.
    unsafe {
        *p = 6;
    }
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%205%3B%0A%20%20%20%20let%20p%20%3D%20%26x%20as%20%2Aconst%20i32%20as%20%2Amut%20i32%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9A%D0%BE%D0%BC%D0%BF%D0%B8%D0%BB%D1%8F%D1%82%D0%BE%D1%80%20%D1%8D%D1%82%D0%BE%20%D0%BF%D1%80%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D1%82%2C%20%D0%BD%D0%BE%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8C%20%E2%80%94%20UB%3A%0A%20%20%20%20%2F%2F%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20%D1%80%D0%B0%D0%B7%D0%B4%D0%B5%D0%BB%D1%8F%D0%B5%D0%BC%D0%BE%D0%B9%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8%20%26x%20%D0%B8%20%D0%BF%D1%80%D0%B0%D0%B2%20%D0%BD%D0%B0%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8C%20%D0%BD%D0%B5%20%D0%B8%D0%BC%D0%B5%D0%B5%D1%82.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2Ap%20%3D%206%3B%0A%20%20%20%20%7D%0A%7D%0A)

Результат:

```text
error: assigning to `&T` is undefined behavior, consider using an `UnsafeCell`
 --> src/main.rs:8:9
  |
3 |     let p = &x as *const i32 as *mut i32;
  |             ---------------------------- casting happened here
...
8 |         *p = 6;
  |         ^^^^^^
  |
  = note: `#[deny(invalid_reference_casting)]` on by default
```

Это lint `invalid_reference_casting`, и он заметит только прямолинейный случай, когда приведение и запись находятся рядом. Если спрятать приведение, например, за вызовом функции, код скомпилируется, но останется UB.

Сырые указатели существенно отличаются от ссылок:

- могут быть `null`;
- могут быть невыровненными;
- могут указывать на неинициализированную память;
- могут быть dangling pointers;
- не проверяются borrow checker'ом;
- не имеют lifetime, который компилятор использует для проверки их валидности;
- могут существовать независимо от владения объектом;
- их разыменование требует `unsafe`.

При этом **создание raw pointer само по себе обычно безопасно**:

```rust
fn main() {
    let x = 42;

    let ptr: *const i32 = &x;

    println!("address: {ptr:p}");

    unsafe {
        println!("value: {}", *ptr);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%2042%3B%0A%0A%20%20%20%20let%20ptr%3A%20%2Aconst%20i32%20%3D%20%26x%3B%0A%0A%20%20%20%20println%21%28%22address%3A%20%7Bptr%3Ap%7D%22%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22value%3A%20%7B%7D%22%2C%20%2Aptr%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Главное различие можно сформулировать так:

> **Ссылка — это гарантированно безопасный способ обратиться к объекту. Raw pointer — это способ представить адрес, безопасность которого программист должен обеспечить самостоятельно.**

---

## 45.2. Создание сырых указателей

### Из ссылки

Самый простой способ получить raw pointer — преобразовать ссылку:

```rust
fn main() {
    let x = 42;
    let const_ptr: *const i32 = &x;

    let mut y = 10;
    let mut_ptr: *mut i32 = &mut y;

    println!("const_ptr: {const_ptr:p}");
    println!("mut_ptr:   {mut_ptr:p}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%2042%3B%0A%20%20%20%20let%20const_ptr%3A%20%2Aconst%20i32%20%3D%20%26x%3B%0A%0A%20%20%20%20let%20mut%20y%20%3D%2010%3B%0A%20%20%20%20let%20mut_ptr%3A%20%2Amut%20i32%20%3D%20%26mut%20y%3B%0A%0A%20%20%20%20println%21%28%22const_ptr%3A%20%7Bconst_ptr%3Ap%7D%22%29%3B%0A%20%20%20%20println%21%28%22mut_ptr%3A%20%20%20%7Bmut_ptr%3Ap%7D%22%29%3B%0A%7D%0A)

Адреса в выводе будут разными при каждом запуске.

### Напрямую из места: `&raw const` и `&raw mut`

У записи `&mut y as *mut i32` есть скрытая особенность: сначала создаётся **временная ссылка** `&mut y`, и только потом она приводится к указателю. Создание ссылки накладывает на память требования (выравнивание, уникальность), которых сырой указатель не требует. Иногда эти требования выполнить нельзя.

Операторы `&raw const` и `&raw mut` создают сырой указатель на место **напрямую**, без промежуточной ссылки:

```rust
fn main() {
    let x = 42;
    let mut y = 10;

    // &raw const / &raw mut создают сырой указатель на место напрямую,
    // без промежуточной ссылки.
    let p = &raw const x;
    let q = &raw mut y;

    unsafe {
        *q += *p;
        println!("{}", *q);
    }
}
```

Результат:

```text
52
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%2042%3B%0A%20%20%20%20let%20mut%20y%20%3D%2010%3B%0A%0A%20%20%20%20%2F%2F%20%26raw%20const%20%2F%20%26raw%20mut%20%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D1%8E%D1%82%20%D1%81%D1%8B%D1%80%D0%BE%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D0%B0%20%D0%BC%D0%B5%D1%81%D1%82%D0%BE%20%D0%BD%D0%B0%D0%BF%D1%80%D1%8F%D0%BC%D1%83%D1%8E%2C%0A%20%20%20%20%2F%2F%20%D0%B1%D0%B5%D0%B7%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B9%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8.%0A%20%20%20%20let%20p%20%3D%20%26raw%20const%20x%3B%0A%20%20%20%20let%20q%20%3D%20%26raw%20mut%20y%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2Aq%20%2B%3D%20%2Ap%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aq%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Классический случай, где без `&raw` не обойтись, — поле структуры с `repr(packed)`. Поле `value` лежит по смещению 1, поэтому ссылка на него была бы невыровненной:

```rust
#[repr(packed)]
struct Packed {
    flag: u8,
    value: u32, // лежит по смещению 1 — не выровнен для u32
}

fn main() {
    let packed = Packed { flag: 1, value: 7 };

    let r = &packed.value; // ошибка: ссылка на невыровненное поле
    println!("{} {}", packed.flag, r);
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Brepr%28packed%29%5D%0Astruct%20Packed%20%7B%0A%20%20%20%20flag%3A%20u8%2C%0A%20%20%20%20value%3A%20u32%2C%20%2F%2F%20%D0%BB%D0%B5%D0%B6%D0%B8%D1%82%20%D0%BF%D0%BE%20%D1%81%D0%BC%D0%B5%D1%89%D0%B5%D0%BD%D0%B8%D1%8E%201%20%E2%80%94%20%D0%BD%D0%B5%20%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20u32%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20packed%20%3D%20Packed%20%7B%20flag%3A%201%2C%20value%3A%207%20%7D%3B%0A%0A%20%20%20%20let%20r%20%3D%20%26packed.value%3B%20%2F%2F%20%D0%BE%D1%88%D0%B8%D0%B1%D0%BA%D0%B0%3A%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B0%20%D0%BD%D0%B0%20%D0%BD%D0%B5%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%D0%BD%D0%BE%D0%B5%20%D0%BF%D0%BE%D0%BB%D0%B5%0A%20%20%20%20println%21%28%22%7B%7D%20%7B%7D%22%2C%20packed.flag%2C%20r%29%3B%0A%7D%0A)

Компилятор отвечает ошибкой:

```text
error[E0793]: reference to field of packed struct is unaligned
  = note: creating a misaligned reference is undefined behavior (even if that reference is never dereferenced)
  = help: copy the field contents to a local variable, or replace the reference with a raw pointer and use `read_unaligned`/`write_unaligned`
```

Исправление — взять указатель через `&raw const` и читать через `read_unaligned`:

```rust
#[repr(packed)]
struct Packed {
    flag: u8,
    value: u32,
}

fn main() {
    let packed = Packed { flag: 1, value: 7 };

    let p = &raw const packed.value; // сырой указатель на невыровненное поле
    // SAFETY: p указывает на инициализированное поле живого значения;
    // read_unaligned не требует выравнивания.
    let value = unsafe { p.read_unaligned() };

    println!("{} {}", { packed.flag }, value);
}
```

Результат:

```text
1 7
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Brepr%28packed%29%5D%0Astruct%20Packed%20%7B%0A%20%20%20%20flag%3A%20u8%2C%0A%20%20%20%20value%3A%20u32%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20packed%20%3D%20Packed%20%7B%20flag%3A%201%2C%20value%3A%207%20%7D%3B%0A%0A%20%20%20%20let%20p%20%3D%20%26raw%20const%20packed.value%3B%20%2F%2F%20%D1%81%D1%8B%D1%80%D0%BE%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D0%B0%20%D0%BD%D0%B5%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%D0%BD%D0%BE%D0%B5%20%D0%BF%D0%BE%D0%BB%D0%B5%0A%20%20%20%20%2F%2F%20SAFETY%3A%20p%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BD%D0%B0%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D0%BE%D0%B5%20%D0%BF%D0%BE%D0%BB%D0%B5%20%D0%B6%D0%B8%D0%B2%D0%BE%D0%B3%D0%BE%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8F%3B%0A%20%20%20%20%2F%2F%20read_unaligned%20%D0%BD%D0%B5%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82%20%D0%B2%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20p.read_unaligned%28%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%20%7B%7D%22%2C%20%7B%20packed.flag%20%7D%2C%20value%29%3B%0A%7D%0A)

Обратите внимание на `{ packed.flag }`: фигурные скобки копируют значение поля, и ссылка на него не создаётся.

### Из `Box`

`Box::into_raw()` особенно важен, потому что он передаёт владение памятью вызывающему коду:

```rust
fn main() {
    let value = Box::new(42);

    let ptr = Box::into_raw(value);

    unsafe {
        println!("value: {}", *ptr);

        // Возвращаем владение памятью Box.
        drop(Box::from_raw(ptr));
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20Box%3A%3Anew%2842%29%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20Box%3A%3Ainto_raw%28value%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22value%3A%20%7B%7D%22%2C%20%2Aptr%29%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%92%D0%BE%D0%B7%D0%B2%D1%80%D0%B0%D1%89%D0%B0%D0%B5%D0%BC%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%D1%8E%20Box.%0A%20%20%20%20%20%20%20%20drop%28Box%3A%3Afrom_raw%28ptr%29%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Здесь существует важное правило:

> После `Box::into_raw()` ответственность за освобождение памяти временно переходит от `Box` к программисту.

`Box::from_raw()` следует использовать только с указателем, который был получен из соответствующего выделения и которым вы всё ещё владеете. Для одной и той же allocation нельзя дважды вызвать `Box::from_raw()` с одним и тем же указателем.

### Из адреса

Raw pointer можно получить из целочисленного адреса:

```rust
use std::ptr;

fn main() {
    let address = 0x1000usize;

    // Старая запись: приведение целого к указателю.
    let a = address as *const i32;

    // Явный вариант: указатель БЕЗ provenance — на сам адрес,
    // но не на какой-либо объект.
    let b = ptr::without_provenance::<i32>(address);

    println!("{a:p} {b:p}");
    println!("совпадают: {}", a == b);
}
```

Результат:

```text
0x1000 0x1000
совпадают: true
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aptr%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20address%20%3D%200x1000usize%3B%0A%0A%20%20%20%20%2F%2F%20%D0%A1%D1%82%D0%B0%D1%80%D0%B0%D1%8F%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8C%3A%20%D0%BF%D1%80%D0%B8%D0%B2%D0%B5%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%20%D1%86%D0%B5%D0%BB%D0%BE%D0%B3%D0%BE%20%D0%BA%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8E.%0A%20%20%20%20let%20a%20%3D%20address%20as%20%2Aconst%20i32%3B%0A%0A%20%20%20%20%2F%2F%20%D0%AF%D0%B2%D0%BD%D1%8B%D0%B9%20%D0%B2%D0%B0%D1%80%D0%B8%D0%B0%D0%BD%D1%82%3A%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%91%D0%95%D0%97%20provenance%20%E2%80%94%20%D0%BD%D0%B0%20%D1%81%D0%B0%D0%BC%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%2C%0A%20%20%20%20%2F%2F%20%D0%BD%D0%BE%20%D0%BD%D0%B5%20%D0%BD%D0%B0%20%D0%BA%D0%B0%D0%BA%D0%BE%D0%B9-%D0%BB%D0%B8%D0%B1%D0%BE%20%D0%BE%D0%B1%D1%8A%D0%B5%D0%BA%D1%82.%0A%20%20%20%20let%20b%20%3D%20ptr%3A%3Awithout_provenance%3A%3A%3Ci32%3E%28address%29%3B%0A%0A%20%20%20%20println%21%28%22%7Ba%3Ap%7D%20%7Bb%3Ap%7D%22%29%3B%0A%20%20%20%20println%21%28%22%D1%81%D0%BE%D0%B2%D0%BF%D0%B0%D0%B4%D0%B0%D1%8E%D1%82%3A%20%7B%7D%22%2C%20a%20%3D%3D%20b%29%3B%0A%7D%0A)

Здесь показаны две записи одного и того же. `address as *const i32` — привычное приведение. `ptr::without_provenance` явно говорит, что мы создаём указатель **без provenance**: у него есть адрес, но нет права обращаться к какому-либо объекту (подробнее в разделе 45.8).

Но наличие адреса **не означает наличие объекта** по этому адресу.

Поэтому следующий код не является корректным способом «прочитать адрес»:

```rust
let ptr = 0x1000usize as *const i32;

// Не делайте так:
// unsafe { println!("{}", *ptr); }
```

Число `0x1000` само по себе не доказывает, что по этому адресу находится корректный `i32`.

### Null pointer

Для создания null pointers используются `null()` и `null_mut()`:

```rust
use std::ptr;

fn main() {
    let ptr: *const i32 = ptr::null();
    let mut_ptr: *mut i32 = ptr::null_mut();

    println!("ptr:     {ptr:p}");
    println!("mut_ptr: {mut_ptr:p}");

    println!("ptr is null: {}", ptr.is_null());
    println!("mut_ptr is null: {}", mut_ptr.is_null());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aptr%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20ptr%3A%20%2Aconst%20i32%20%3D%20ptr%3A%3Anull%28%29%3B%0A%20%20%20%20let%20mut_ptr%3A%20%2Amut%20i32%20%3D%20ptr%3A%3Anull_mut%28%29%3B%0A%0A%20%20%20%20println%21%28%22ptr%3A%20%20%20%20%20%7Bptr%3Ap%7D%22%29%3B%0A%20%20%20%20println%21%28%22mut_ptr%3A%20%7Bmut_ptr%3Ap%7D%22%29%3B%0A%0A%20%20%20%20println%21%28%22ptr%20is%20null%3A%20%7B%7D%22%2C%20ptr.is_null%28%29%29%3B%0A%20%20%20%20println%21%28%22mut_ptr%20is%20null%3A%20%7B%7D%22%2C%20mut_ptr.is_null%28%29%29%3B%0A%7D%0A)

---

## 45.3. Разыменовывание (Dereferencing)

Разыменование raw pointer — это `unsafe` операция:

```rust
fn main() {
    let x = 42;
    let ptr = &x as *const i32;

    println!("Pointer: {ptr:p}");

    unsafe {
        let value = *ptr;
        println!("Value: {value}");
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%2042%3B%0A%20%20%20%20let%20ptr%20%3D%20%26x%20as%20%2Aconst%20i32%3B%0A%0A%20%20%20%20println%21%28%22Pointer%3A%20%7Bptr%3Ap%7D%22%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20let%20value%20%3D%20%2Aptr%3B%0A%20%20%20%20%20%20%20%20println%21%28%22Value%3A%20%7Bvalue%7D%22%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Почему Rust требует `unsafe`?

Потому что компилятор не может в общем случае доказать, что указатель:

- не является `null`;
- указывает на существующий объект;
- указывает на инициализированную память;
- правильно выровнен;
- находится в допустимых границах allocation;
- не является dangling pointer;
- используется с соблюдением требований модели памяти Rust.

Например, разыменование нулевого указателя не вызывает panic, которую Rust обязан обнаружить:

```rust
fn main() {
    let ptr: *const i32 = std::ptr::null();

    // Разыменование такого указателя приводит к UB.
    //
    // unsafe {
    //     println!("{}", *ptr);
    // }
}
```

**Undefined behavior не следует путать с panic.** Panic — это определённое, предсказуемое поведение языка. UB — отсутствие какого-либо определённого поведения: программа может аварийно завершиться, может проявить совершенно другое поведение, а может «работать». Полагаться на конкретный результат нельзя. Подробно об этом шла речь в разделе 44.8.

---

## 45.4. Pointer arithmetic — арифметика указателей

Raw pointers можно перемещать относительно элементов allocation.

Для этого используются, в частности:

| Метод             | Назначение                                                                                    |
| ----------------- | --------------------------------------------------------------------------------------------- |
| `add(n)`          | перейти на `n` элементов вперёд                                                               |
| `sub(n)`          | перейти на `n` элементов назад                                                                |
| `offset(n)`       | перейти на `n` элементов вперёд или назад; `n` имеет тип `isize`                              |
| `offset_from(p)`  | узнать расстояние до другого указателя **в элементах**                                        |
| `wrapping_add(n)` | выполнить wrapping-арифметику без требования, чтобы результат оставался в пределах allocation |

Ключевой момент: `add`, `sub` и `offset` **не являются обычной арифметикой адресов**. Они работают в терминах элементов типа `T`.

Например, если `T = i32`, то:

```rust
ptr.add(1)
```

означает переход к следующему `i32`, а не просто увеличение числового адреса на `1`.

Корректный пример:

```rust
fn main() {
    let array = [10, 20, 30, 40, 50];
    let ptr = array.as_ptr();

    unsafe {
        for index in 0..array.len() {
            println!("array[{index}] = {}", *ptr.add(index));
        }
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20array%20%3D%20%5B10%2C%2020%2C%2030%2C%2040%2C%2050%5D%3B%0A%20%20%20%20let%20ptr%20%3D%20array.as_ptr%28%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20for%20index%20in%200..array.len%28%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22array%5B%7Bindex%7D%5D%20%3D%20%7B%7D%22%2C%20%2Aptr.add%28index%29%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A)

Здесь каждый вызов:

```rust
ptr.add(index)
```

должен оставаться в пределах той же allocation либо указывать на допустимую позицию one-past-the-end в случаях, предусмотренных правилами соответствующей операции. **Разыменовывать one-past-the-end нельзя.**

Обратите внимание: это требование относится **к самому вычислению** `add`/`sub`/`offset`, а не только к последующему разыменованию. `ptr.add(100)` для массива из трёх элементов уже является UB, даже если вы результат нигде не используете. Если нужно вычислить «чужой» адрес без этого требования, существует `wrapping_add`.

### `sub`

```rust
fn main() {
    let array = [10, 20, 30, 40, 50];

    let ptr = unsafe { array.as_ptr().add(3) };

    unsafe {
        println!("{}", *ptr);
        println!("{}", *ptr.sub(1));
        println!("{}", *ptr.sub(2));
    }
}
```

Результат:

```text
40
30
20
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20array%20%3D%20%5B10%2C%2020%2C%2030%2C%2040%2C%2050%5D%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20array.as_ptr%28%29.add%283%29%20%7D%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr.sub%281%29%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aptr.sub%282%29%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

### `offset`

`offset` использует `isize`, поэтому позволяет указывать отрицательное смещение:

```rust
fn main() {
    let array = [10, 20, 30, 40, 50];
    let ptr = array.as_ptr();

    unsafe {
        let third = ptr.offset(2);
        let second = third.offset(-1);

        println!("{}", *third);
        println!("{}", *second);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20array%20%3D%20%5B10%2C%2020%2C%2030%2C%2040%2C%2050%5D%3B%0A%20%20%20%20let%20ptr%20%3D%20array.as_ptr%28%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20let%20third%20%3D%20ptr.offset%282%29%3B%0A%20%20%20%20%20%20%20%20let%20second%20%3D%20third.offset%28-1%29%3B%0A%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Athird%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Asecond%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

### `offset_from`

Чтобы узнать, на каком расстоянии друг от друга находятся два указателя, используется `offset_from`. Результат — число **элементов** типа `isize`:

```rust
fn main() {
    let array = [10, 20, 30, 40, 50];
    let base = array.as_ptr();

    // SAFETY: оба указателя получены из одного массива и лежат в его границах.
    unsafe {
        let p = base.add(3);
        let index = p.offset_from(base);

        println!("p указывает на элемент с индексом {index}");
        println!("значение: {}", *p);
    }
}
```

Результат:

```text
p указывает на элемент с индексом 3
значение: 40
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20array%20%3D%20%5B10%2C%2020%2C%2030%2C%2040%2C%2050%5D%3B%0A%20%20%20%20let%20base%20%3D%20array.as_ptr%28%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BE%D0%B1%D0%B0%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8F%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D1%8B%20%D0%B8%D0%B7%20%D0%BE%D0%B4%D0%BD%D0%BE%D0%B3%D0%BE%20%D0%BC%D0%B0%D1%81%D1%81%D0%B8%D0%B2%D0%B0%20%D0%B8%20%D0%BB%D0%B5%D0%B6%D0%B0%D1%82%20%D0%B2%20%D0%B5%D0%B3%D0%BE%20%D0%B3%D1%80%D0%B0%D0%BD%D0%B8%D1%86%D0%B0%D1%85.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20let%20p%20%3D%20base.add%283%29%3B%0A%20%20%20%20%20%20%20%20let%20index%20%3D%20p.offset_from%28base%29%3B%0A%0A%20%20%20%20%20%20%20%20println%21%28%22p%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BD%D0%B0%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%20%D1%81%20%D0%B8%D0%BD%D0%B4%D0%B5%D0%BA%D1%81%D0%BE%D0%BC%20%7Bindex%7D%22%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%3A%20%7B%7D%22%2C%20%2Ap%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Оба указателя обязаны принадлежать одной allocation, иначе результат не определён.

### `wrapping_add`

`wrapping_add` отличается тем, что сама операция не требует результата в пределах allocation:

```rust
fn main() {
    let array = [10, 20, 30];

    let ptr = array.as_ptr();
    let outside = ptr.wrapping_add(100);

    println!("ptr:     {ptr:p}");
    println!("outside: {outside:p}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20array%20%3D%20%5B10%2C%2020%2C%2030%5D%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20array.as_ptr%28%29%3B%0A%20%20%20%20let%20outside%20%3D%20ptr.wrapping_add%28100%29%3B%0A%0A%20%20%20%20println%21%28%22ptr%3A%20%20%20%20%20%7Bptr%3Ap%7D%22%29%3B%0A%20%20%20%20println%21%28%22outside%3A%20%7Boutside%3Ap%7D%22%29%3B%0A%7D%0A)

Но это **не делает указатель валидным для разыменования**.

То есть:

```rust
let outside = ptr.wrapping_add(100);

// Это НЕ становится безопасным:
// unsafe { println!("{}", *outside); }
```

`wrapping_add()` изменяет правила самой арифметической операции, но не создаёт объект в памяти и не расширяет allocation.

---

## 45.5. Null pointers

Сырые указатели могут быть `null`, поэтому `is_null()` часто используется перед операцией, которая допускает null как часть своего контракта:

```rust
fn print_value(ptr: *const i32) {
    if ptr.is_null() {
        println!("Pointer is null");
        return;
    }

    // SAFETY: проверили, что указатель не null; остальные условия
    // (валидность, выравнивание) обеспечивает вызывающий код.
    unsafe {
        println!("Value: {}", *ptr);
    }
}

fn main() {
    let value = 42;
    let valid = &value as *const i32;
    let null = std::ptr::null();

    print_value(valid);
    print_value(null);
}
```

Результат:

```text
Value: 42
Pointer is null
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20print_value%28ptr%3A%20%2Aconst%20i32%29%20%7B%0A%20%20%20%20if%20ptr.is_null%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Pointer%20is%20null%22%29%3B%0A%20%20%20%20%20%20%20%20return%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%B8%D0%BB%D0%B8%2C%20%D1%87%D1%82%D0%BE%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D0%B5%20null%3B%20%D0%BE%D1%81%D1%82%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B5%20%D1%83%D1%81%D0%BB%D0%BE%D0%B2%D0%B8%D1%8F%0A%20%20%20%20%2F%2F%20%28%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D0%BE%D1%81%D1%82%D1%8C%2C%20%D0%B2%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%29%20%D0%BE%D0%B1%D0%B5%D1%81%D0%BF%D0%B5%D1%87%D0%B8%D0%B2%D0%B0%D0%B5%D1%82%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D1%8E%D1%89%D0%B8%D0%B9%20%D0%BA%D0%BE%D0%B4.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Value%3A%20%7B%7D%22%2C%20%2Aptr%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%20%20%20%20let%20valid%20%3D%20%26value%20as%20%2Aconst%20i32%3B%0A%20%20%20%20let%20null%20%3D%20std%3A%3Aptr%3A%3Anull%28%29%3B%0A%0A%20%20%20%20print_value%28valid%29%3B%0A%20%20%20%20print_value%28null%29%3B%0A%7D%0A)

Но проверка `is_null()` **не является универсальной проверкой безопасности**.

Например, указатель может быть ненулевым, но всё равно:

- указывать на освобождённую память;
- быть невыровненным;
- указывать на неинициализированные данные;
- не иметь права на соответствующий доступ.

Поэтому:

> **`!ptr.is_null()` не означает `ptr` valid.**

Для случая, когда `null` — штатное значение, есть удобный метод `as_ref()`: он возвращает `Option<&T>`, причём `null` превращается в `None`:

```rust
fn show(ptr: *const i32) {
    // SAFETY: для ненулевого указателя вызывающий гарантирует валидность;
    // для null as_ref просто вернёт None.
    match unsafe { ptr.as_ref() } {
        Some(value) => println!("значение: {value}"),
        None => println!("указатель нулевой"),
    }
}

fn main() {
    let x = 42;
    show(&x);
    show(std::ptr::null());
}
```

Результат:

```text
значение: 42
указатель нулевой
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20show%28ptr%3A%20%2Aconst%20i32%29%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B4%D0%BB%D1%8F%20%D0%BD%D0%B5%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B3%D0%BE%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8F%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D1%8E%D1%89%D0%B8%D0%B9%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D0%BE%D1%81%D1%82%D1%8C%3B%0A%20%20%20%20%2F%2F%20%D0%B4%D0%BB%D1%8F%20null%20as_ref%20%D0%BF%D1%80%D0%BE%D1%81%D1%82%D0%BE%20%D0%B2%D0%B5%D1%80%D0%BD%D1%91%D1%82%20None.%0A%20%20%20%20match%20unsafe%20%7B%20ptr.as_ref%28%29%20%7D%20%7B%0A%20%20%20%20%20%20%20%20Some%28value%29%20%3D%3E%20println%21%28%22%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%3A%20%7Bvalue%7D%22%29%2C%0A%20%20%20%20%20%20%20%20None%20%3D%3E%20println%21%28%22%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D0%BE%D0%B9%22%29%2C%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%2042%3B%0A%20%20%20%20show%28%26x%29%3B%0A%20%20%20%20show%28std%3A%3Aptr%3A%3Anull%28%29%29%3B%0A%7D%0A)

`as_ref` по-прежнему `unsafe`: он проверяет только `null`, а валидность и выравнивание ненулевого указателя по-прежнему обеспечивает вызывающий код. Метод `as_mut()` работает так же для `*mut T`.

---

## 45.6. Alignment — выравнивание

Каждый тип имеет требование к выравниванию (`alignment`).

Например, `i32` обычно имеет alignment 4, хотя конкретные значения являются свойствами target:

```rust
use std::mem::{align_of, size_of};

fn main() {
    println!("i32 size:  {}", size_of::<i32>());
    println!("i32 align: {}", align_of::<i32>());

    println!("u8 size:   {}", size_of::<u8>());
    println!("u8 align:  {}", align_of::<u8>());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3A%7Balign_of%2C%20size_of%7D%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22i32%20size%3A%20%20%7B%7D%22%2C%20size_of%3A%3A%3Ci32%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22i32%20align%3A%20%7B%7D%22%2C%20align_of%3A%3A%3Ci32%3E%28%29%29%3B%0A%0A%20%20%20%20println%21%28%22u8%20size%3A%20%20%20%7B%7D%22%2C%20size_of%3A%3A%3Cu8%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22u8%20align%3A%20%20%7B%7D%22%2C%20align_of%3A%3A%3Cu8%3E%28%29%29%3B%0A%7D%0A)

Raw pointer может иметь адрес, который не удовлетворяет alignment `T`.

Например:

```rust
fn main() {
    let bytes = [0u8; 8];

    let ptr = unsafe { bytes.as_ptr().add(1) } as *const u32;

    println!("address: {ptr:p}");
    println!("выровнен для u32: {}", ptr.is_aligned());

    // Обычное разыменование такого указателя недопустимо,
    // если адрес не выровнен для u32.
}
```

Результат:

```text
address: 0x7ffd...
выровнен для u32: false
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20bytes%20%3D%20%5B0u8%3B%208%5D%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20bytes.as_ptr%28%29.add%281%29%20%7D%20as%20%2Aconst%20u32%3B%0A%0A%20%20%20%20println%21%28%22address%3A%20%7Bptr%3Ap%7D%22%29%3B%0A%20%20%20%20println%21%28%22%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20u32%3A%20%7B%7D%22%2C%20ptr.is_aligned%28%29%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9E%D0%B1%D1%8B%D1%87%D0%BD%D0%BE%D0%B5%20%D1%80%D0%B0%D0%B7%D1%8B%D0%BC%D0%B5%D0%BD%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20%D1%82%D0%B0%D0%BA%D0%BE%D0%B3%D0%BE%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8F%20%D0%BD%D0%B5%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D0%BC%D0%BE%2C%0A%20%20%20%20%2F%2F%20%D0%B5%D1%81%D0%BB%D0%B8%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%D0%BD%D0%B5%20%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20u32.%0A%7D%0A)

Метод `is_aligned()` позволяет проверить выравнивание явно. Обычное разыменование такого указателя недопустимо, если адрес не выровнен для `u32`.

Для чтения данных из потенциально невыровненной области памяти существуют специальные операции:

```rust
use std::ptr;

fn main() {
    let bytes = [0x78u8, 0x56, 0x34, 0x12];

    let ptr = bytes.as_ptr() as *const u32;

    // SAFETY: в bytes есть 4 инициализированных байта;
    // read_unaligned не требует выравнивания.
    let value = unsafe { ptr::read_unaligned(ptr) };

    println!("value: 0x{value:08x}");
    println!("то же без unsafe: 0x{:08x}", u32::from_ne_bytes(bytes));
}
```

Результат (на little-endian платформе, например x86-64):

```text
value: 0x12345678
то же без unsafe: 0x12345678
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aptr%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20bytes%20%3D%20%5B0x78u8%2C%200x56%2C%200x34%2C%200x12%5D%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20bytes.as_ptr%28%29%20as%20%2Aconst%20u32%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B2%20bytes%20%D0%B5%D1%81%D1%82%D1%8C%204%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D1%85%20%D0%B1%D0%B0%D0%B9%D1%82%D0%B0%3B%0A%20%20%20%20%2F%2F%20read_unaligned%20%D0%BD%D0%B5%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82%20%D0%B2%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20ptr%3A%3Aread_unaligned%28ptr%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22value%3A%200x%7Bvalue%3A08x%7D%22%29%3B%0A%20%20%20%20println%21%28%22%D1%82%D0%BE%20%D0%B6%D0%B5%20%D0%B1%D0%B5%D0%B7%20unsafe%3A%200x%7B%3A08x%7D%22%2C%20u32%3A%3Afrom_ne_bytes%28bytes%29%29%3B%0A%7D%0A)

Порядок байтов зависит от платформы: на big-endian результат `read_unaligned` был бы `0x78563412`. Поэтому для разбора внешних данных безопаснее использовать `from_le_bytes` / `from_be_bytes` с явно заданным порядком, а `from_ne_bytes` соответствует порядку текущей платформы, как и чтение через указатель.

`read_unaligned()` решает **только проблему выравнивания**. Он не делает произвольный адрес валидным. Память всё равно должна удовлетворять остальным требованиям операции.

Аналогично для записи существует `ptr::write_unaligned()`.

---

## 45.7. Validity — валидность указателя

Для обычного разыменования raw pointer недостаточно, чтобы он был ненулевым.

Нужно, чтобы операция удовлетворяла требованиям валидного доступа к `T`.

В частности, необходимо учитывать:

1. **Адрес должен относиться к подходящей allocation.**
2. **Память должна быть достаточно большой для `T`.**
3. **Память должна быть правильно выровнена**, если используется обычное разыменование.
4. **Значение `T` должно быть корректно инициализировано.**
5. **Объект должен всё ещё существовать.**
6. Доступ должен соответствовать правилам **aliasing и mutability** модели Rust.

Корректный пример:

```rust
fn main() {
    let value = 42;
    let ptr = &value as *const i32;

    unsafe {
        println!("{}", *ptr);
    }
}
```

А вот этот указатель не становится валидным только потому, что он имеет тип `*const i32`:

```rust
let ptr = 0x1234usize as *const i32;
```

Тип raw pointer **не содержит гарантии**, что по адресу действительно находится `i32`.

---

## 45.8. Provenance — происхождение указателя

**Provenance** — это информация о том, **к какой памяти указатель имеет право обращаться**. Указатель в Rust семантически состоит из двух частей:

```text
указатель = адрес + provenance
              │         │
              │         └─ какую allocation (и через какую ссылку/выделение)
              │            этот указатель вправе читать и записывать
              └─ числовое значение: «куда смотрит»
```

Это особенно важно при работе с raw pointers. Упрощённо ситуацию можно представить так:

```text
allocation
┌──────────────────────────────┐
│ 10 │ 20 │ 30 │ 40 │ 50       │
└──────────────────────────────┘
  ↑
  │
  ptr  (адрес + право работать с этой allocation)
```

Если `ptr` получен из этой allocation, операции вроде `ptr.add(2)` используются для перехода к третьему элементу той же области памяти: адрес меняется, а provenance сохраняется.

Из этого следует важное практическое правило, которое иллюстрирует документация стандартной библиотеки: два объекта могут лежать в памяти рядом, и указатель на конец первого численно равен адресу начала второго. Но если указатель имеет provenance только первого объекта, то доступ к второму через него — UB, **даже если адреса совпадают**. Равенство адресов не заменяет права доступа.

### Два способа работы с provenance

Стандартная библиотека предлагает два подхода.

**Strict Provenance** — provenance никогда не теряется: вы получаете адрес методом `addr()` и затем строите новый указатель из старого методами `with_addr()` или `map_addr()`. Эти API стабильны, а их семантика чётко определена.

**Exposed Provenance** — оставлен для случаев, когда провести provenance через код невозможно. Приведение указателя к `usize` через `as` (или `expose_provenance()`) «раскрывает» provenance, а приведение целого обратно к указателю (`as *const T` или `with_exposed_provenance()`) «подбирает» ранее раскрытый provenance. Документация прямо предупреждает, что семантика этого подхода менее надёжна, а часть инструментов проверки (в частности, Miri) может его не поддерживать или предупреждать о нём.

### Пример: round-trip через `usize` и `as`

```rust
fn main() {
    let value = 42;

    let ptr = &value as *const i32;
    let address = ptr as usize; // "expose": provenance запоминается

    println!("address: 0x{address:x}");

    // Приведение обратно «подбирает» ранее раскрытый provenance.
    let restored = address as *const i32;

    // Допустимо по правилам Exposed Provenance, но этот путь непрозрачен
    // для оптимизатора и анализаторов (в том числе Miri).
    unsafe {
        println!("value: {}", *restored);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20%26value%20as%20%2Aconst%20i32%3B%0A%20%20%20%20let%20address%20%3D%20ptr%20as%20usize%3B%20%2F%2F%20%22expose%22%3A%20provenance%20%D0%B7%D0%B0%D0%BF%D0%BE%D0%BC%D0%B8%D0%BD%D0%B0%D0%B5%D1%82%D1%81%D1%8F%0A%0A%20%20%20%20println%21%28%22address%3A%200x%7Baddress%3Ax%7D%22%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9F%D1%80%D0%B8%D0%B2%D0%B5%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%BE%D0%B1%D1%80%D0%B0%D1%82%D0%BD%D0%BE%20%C2%AB%D0%BF%D0%BE%D0%B4%D0%B1%D0%B8%D1%80%D0%B0%D0%B5%D1%82%C2%BB%20%D1%80%D0%B0%D0%BD%D0%B5%D0%B5%20%D1%80%D0%B0%D1%81%D0%BA%D1%80%D1%8B%D1%82%D1%8B%D0%B9%20provenance.%0A%20%20%20%20let%20restored%20%3D%20address%20as%20%2Aconst%20i32%3B%0A%0A%20%20%20%20%2F%2F%20%D0%94%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D0%BC%D0%BE%20%D0%BF%D0%BE%20%D0%BF%D1%80%D0%B0%D0%B2%D0%B8%D0%BB%D0%B0%D0%BC%20Exposed%20Provenance%2C%20%D0%BD%D0%BE%20%D1%8D%D1%82%D0%BE%D1%82%20%D0%BF%D1%83%D1%82%D1%8C%20%D0%BD%D0%B5%D0%BF%D1%80%D0%BE%D0%B7%D1%80%D0%B0%D1%87%D0%B5%D0%BD%0A%20%20%20%20%2F%2F%20%D0%B4%D0%BB%D1%8F%20%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D0%B8%D0%B7%D0%B0%D1%82%D0%BE%D1%80%D0%B0%20%D0%B8%20%D0%B0%D0%BD%D0%B0%D0%BB%D0%B8%D0%B7%D0%B0%D1%82%D0%BE%D1%80%D0%BE%D0%B2%20%28%D0%B2%20%D1%82%D0%BE%D0%BC%20%D1%87%D0%B8%D1%81%D0%BB%D0%B5%20Miri%29.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22value%3A%20%7B%7D%22%2C%20%2Arestored%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Этот код допустим по правилам Exposed Provenance: `ptr as usize` раскрыл provenance, а `address as *const i32` подобрал его. Но это самый «непрозрачный» путь: он мешает оптимизатору и анализаторам, поэтому в новом коде его следует избегать.

Критично не само приведение, а наличие ранее раскрытого provenance, которое оправдывает доступ. Если такого provenance нет, приведение целого к указателю даёт указатель, который разыменовывать нельзя:

```rust
// UB при разыменовании: число 0x1000 не раскрывало provenance какого-либо объекта.
let ptr = 0x1000usize as *const i32;
// unsafe { *ptr }

// То же самое для адреса, полученного из addr():
// addr() не раскрывает provenance, и указатель, созданный по такому адресу
// через without_provenance, разыменовывать нельзя.
// let fake = std::ptr::without_provenance::<i32>(value_ptr.addr());
// unsafe { *fake }
```

### Предпочтительный способ: Strict Provenance

```rust
fn main() {
    let value = 42;
    let ptr = &value as *const i32;

    // addr() читает адрес и НЕ раскрывает provenance.
    let address = ptr.addr();

    // with_addr() берёт provenance у ptr и адрес у address.
    let restored = ptr.with_addr(address);

    // SAFETY: restored унаследовал provenance от ptr, адрес тот же.
    unsafe {
        println!("value: {}", *restored);
    }
}
```

Результат:

```text
value: 42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%20%20%20%20let%20ptr%20%3D%20%26value%20as%20%2Aconst%20i32%3B%0A%0A%20%20%20%20%2F%2F%20addr%28%29%20%D1%87%D0%B8%D1%82%D0%B0%D0%B5%D1%82%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%D0%B8%20%D0%9D%D0%95%20%D1%80%D0%B0%D1%81%D0%BA%D1%80%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20provenance.%0A%20%20%20%20let%20address%20%3D%20ptr.addr%28%29%3B%0A%0A%20%20%20%20%2F%2F%20with_addr%28%29%20%D0%B1%D0%B5%D1%80%D1%91%D1%82%20provenance%20%D1%83%20ptr%20%D0%B8%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%D1%83%20address.%0A%20%20%20%20let%20restored%20%3D%20ptr.with_addr%28address%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20restored%20%D1%83%D0%BD%D0%B0%D1%81%D0%BB%D0%B5%D0%B4%D0%BE%D0%B2%D0%B0%D0%BB%20provenance%20%D0%BE%D1%82%20ptr%2C%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%D1%82%D0%BE%D1%82%20%D0%B6%D0%B5.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22value%3A%20%7B%7D%22%2C%20%2Arestored%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Здесь `ptr.addr()` возвращает только число и **не** раскрывает provenance, а `ptr.with_addr(address)` создаёт новый указатель с адресом `address` и provenance, скопированным у `ptr`. Никакой «подборки» не происходит: связь с `ptr` прослеживается прямо в коде.

### Зачем это нужно: pointer tagging

Strict Provenance удобен там, где в указатель кладут дополнительные данные. Классический пример — использовать свободные младшие биты адреса как флаги. Поскольку `u64` выровнен минимум по 2 байтам, младший бит его адреса всегда нулевой:

```rust
fn main() {
    let mut value: u64 = 1234;
    let ptr: *mut u64 = &mut value;

    // u64 выровнен минимум по 2 байтам, значит младший бит адреса всегда 0
    // и его можно использовать как флаг.
    let tagged = ptr.map_addr(|addr| addr | 1);

    let flag = tagged.addr() & 1 == 1;
    let untagged = tagged.map_addr(|addr| addr & !1);

    println!("флаг: {flag}");

    // SAFETY: untagged имеет тот же адрес и provenance, что и ptr.
    unsafe {
        *untagged += 1;
        println!("значение: {}", *untagged);
    }
}
```

Результат:

```text
флаг: true
значение: 1235
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%3A%20u64%20%3D%201234%3B%0A%20%20%20%20let%20ptr%3A%20%2Amut%20u64%20%3D%20%26mut%20value%3B%0A%0A%20%20%20%20%2F%2F%20u64%20%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%20%D0%BC%D0%B8%D0%BD%D0%B8%D0%BC%D1%83%D0%BC%20%D0%BF%D0%BE%202%20%D0%B1%D0%B0%D0%B9%D1%82%D0%B0%D0%BC%2C%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B8%D1%82%20%D0%BC%D0%BB%D0%B0%D0%B4%D1%88%D0%B8%D0%B9%20%D0%B1%D0%B8%D1%82%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%D0%B0%20%D0%B2%D1%81%D0%B5%D0%B3%D0%B4%D0%B0%200%0A%20%20%20%20%2F%2F%20%D0%B8%20%D0%B5%D0%B3%D0%BE%20%D0%BC%D0%BE%D0%B6%D0%BD%D0%BE%20%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D0%BE%D0%B2%D0%B0%D1%82%D1%8C%20%D0%BA%D0%B0%D0%BA%20%D1%84%D0%BB%D0%B0%D0%B3.%0A%20%20%20%20let%20tagged%20%3D%20ptr.map_addr%28%7Caddr%7C%20addr%20%7C%201%29%3B%0A%0A%20%20%20%20let%20flag%20%3D%20tagged.addr%28%29%20%26%201%20%3D%3D%201%3B%0A%20%20%20%20let%20untagged%20%3D%20tagged.map_addr%28%7Caddr%7C%20addr%20%26%20%211%29%3B%0A%0A%20%20%20%20println%21%28%22%D1%84%D0%BB%D0%B0%D0%B3%3A%20%7Bflag%7D%22%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20untagged%20%D0%B8%D0%BC%D0%B5%D0%B5%D1%82%20%D1%82%D0%BE%D1%82%20%D0%B6%D0%B5%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%D0%B8%20provenance%2C%20%D1%87%D1%82%D0%BE%20%D0%B8%20ptr.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2Auntagged%20%2B%3D%201%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%3A%20%7B%7D%22%2C%20%2Auntagged%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Метод `map_addr` преобразует адрес, сохраняя provenance. Именно так делают реализации со «свободными битами»: адрес меняется, но право на доступ остаётся.

`wrapping_add()` также не отменяет provenance и других требований к последующему доступу:

```rust
let next = ptr.wrapping_add(10);

// Получение указателя допустимо,
// но это не означает, что `next` можно разыменовать.
```

Главная практическая мысль:

> **Raw pointer — это не просто число-адрес. При работе с ним нужно учитывать allocation, границы, alignment, validity и provenance. Если можете, не превращайте указатель в число и обратно: сохраняйте provenance методами `addr`, `with_addr`, `map_addr`.**

---

## 45.9. Raw pointers vs References

| Характеристика  | Ссылки (`&T`, `&mut T`)               | Raw pointers (`*const T`, `*mut T`)                            |
| --------------- | ------------------------------------- | -------------------------------------------------------------- |
| Null            | Не могут быть null                    | Могут быть null                                                |
| Lifetime        | Участвуют в lifetime-системе Rust     | Lifetime не проверяется borrow checker'ом                      |
| Borrow checking | Проверяется                           | Не проверяется                                                 |
| Alignment       | Ссылка должна быть корректной для `T` | Raw pointer может быть невыровнен                              |
| Dereference     | Без `unsafe`                          | Требует `unsafe`                                               |
| Validity        | Гарантируется правилами ссылок        | Ответственность лежит на программисте                          |
| Aliasing        | Ограничивается правилами Rust         | Не проверяется компилятором                                    |
| Использование   | Обычный Rust-код                      | FFI, allocators, низкоуровневые структуры, unsafe abstractions |

Важно не делать слишком простой вывод:

> «`*mut T` можно иметь сколько угодно и писать через все указатели».

Само существование нескольких `*mut T` допустимо, но **одновременная запись через несколько указателей может нарушать требования модели памяти Rust**.

Особенно коварен такой код, где указатели создаются через временные `&mut`-ссылки:

```rust
fn main() {
    let mut value = 10;

    let p1 = &mut value as *mut i32; // первая &mut-ссылка (временная)
    let p2 = &mut value as *mut i32; // вторая &mut-ссылка делает p1 недействительным

    unsafe {
        *p1 = 20; // UB по модели aliasing: Miri сообщит об ошибке
        *p2 = 30;
    }

    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%2010%3B%0A%0A%20%20%20%20let%20p1%20%3D%20%26mut%20value%20as%20%2Amut%20i32%3B%20%2F%2F%20%D0%BF%D0%B5%D1%80%D0%B2%D0%B0%D1%8F%20%26mut-%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B0%20%28%D0%B2%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%BD%D0%B0%D1%8F%29%0A%20%20%20%20let%20p2%20%3D%20%26mut%20value%20as%20%2Amut%20i32%3B%20%2F%2F%20%D0%B2%D1%82%D0%BE%D1%80%D0%B0%D1%8F%20%26mut-%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B0%20%D0%B4%D0%B5%D0%BB%D0%B0%D0%B5%D1%82%20p1%20%D0%BD%D0%B5%D0%B4%D0%B5%D0%B9%D1%81%D1%82%D0%B2%D0%B8%D1%82%D0%B5%D0%BB%D1%8C%D0%BD%D1%8B%D0%BC%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2Ap1%20%3D%2020%3B%20%2F%2F%20UB%20%D0%BF%D0%BE%20%D0%BC%D0%BE%D0%B4%D0%B5%D0%BB%D0%B8%20aliasing%3A%20Miri%20%D1%81%D0%BE%D0%BE%D0%B1%D1%89%D0%B8%D1%82%20%D0%BE%D0%B1%20%D0%BE%D1%88%D0%B8%D0%B1%D0%BA%D0%B5%0A%20%20%20%20%20%20%20%20%2Ap2%20%3D%2030%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

Он компилируется и, скорее всего, напечатает `30`, но это **UB**. Причина в том, что каждое выражение `&mut value` создаёт новую уникальную ссылку, а создание второй `&mut` делает недействительной всё, что было получено из первой, включая `p1`. Запись через `p1` после этого нарушает правила aliasing. Miri по умолчанию (модель Stacked Borrows) сообщит здесь об ошибке.

Корректный вариант — получить указатель один раз напрямую и копировать его:

```rust
fn main() {
    let mut value = 10;

    // Один исходный сырой указатель и его копия: оба имеют один provenance.
    let p1 = &raw mut value;
    let p2 = p1;

    // SAFETY: p1 и p2 указывают на живую переменную value, ссылок на неё нет.
    unsafe {
        *p1 = 20;
        *p2 = 30;
    }

    println!("{value}");
}
```

Результат:

```text
30
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20value%20%3D%2010%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9E%D0%B4%D0%B8%D0%BD%20%D0%B8%D1%81%D1%85%D0%BE%D0%B4%D0%BD%D1%8B%D0%B9%20%D1%81%D1%8B%D1%80%D0%BE%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%B8%20%D0%B5%D0%B3%D0%BE%20%D0%BA%D0%BE%D0%BF%D0%B8%D1%8F%3A%20%D0%BE%D0%B1%D0%B0%20%D0%B8%D0%BC%D0%B5%D1%8E%D1%82%20%D0%BE%D0%B4%D0%B8%D0%BD%20provenance.%0A%20%20%20%20let%20p1%20%3D%20%26raw%20mut%20value%3B%0A%20%20%20%20let%20p2%20%3D%20p1%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20p1%20%D0%B8%20p2%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D1%8E%D1%82%20%D0%BD%D0%B0%20%D0%B6%D0%B8%D0%B2%D1%83%D1%8E%20%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%BD%D1%83%D1%8E%20value%2C%20%D1%81%D1%81%D1%8B%D0%BB%D0%BE%D0%BA%20%D0%BD%D0%B0%20%D0%BD%D0%B5%D1%91%20%D0%BD%D0%B5%D1%82.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2Ap1%20%3D%2020%3B%0A%20%20%20%20%20%20%20%20%2Ap2%20%3D%2030%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

Здесь `&raw mut value` не создаёт промежуточных ссылок, а `p2 = p1` — просто копия указателя с тем же provenance. Запись через оба допустима, пока никто не использует в этот момент обычные ссылки на `value`.

Сам факт, что компилятор разрешил создать raw pointers, **не означает автоматически, что любая последующая комбинация операций через них корректна**.

Именно поэтому raw pointers не следует рассматривать как «способ обмануть borrow checker». Их назначение — дать возможность реализовать низкоуровневые механизмы, когда программист способен самостоятельно доказать корректность.

---

## 45.10. Чтение и запись через указатель

Помимо оператора `*`, у сырых указателей есть методы для чтения и записи, которые различаются в тонких, но важных деталях.

| Операция                         | Что делает                                                         | Тонкость                                                         |
| -------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------- |
| `*ptr`                           | читает или записывает значение                                     | при записи **уничтожает старое значение** (вызывает `Drop`)      |
| `ptr.read()`                     | читает значение побитовой копией                                   | для не-`Copy` типов создаёт дубликат владения                    |
| `ptr.write(v)`                   | записывает `v`, **не читая и не уничтожая** старое значение         | подходит для неинициализированной памяти                         |
| `ptr::replace(ptr, v)`           | записывает `v` и возвращает старое значение                        | старое значение не теряется                                      |
| `ptr::swap(a, b)`                | меняет местами значения по двум указателям                         | указатели могут совпадать                                        |
| `ptr::copy_nonoverlapping`       | копирует `n` элементов (аналог `memcpy`)                           | области **не должны** пересекаться                               |
| `ptr::copy`                      | копирует `n` элементов (аналог `memmove`)                          | области могут пересекаться                                       |

Самое важное различие — между `*ptr = v` и `ptr.write(v)`. Если тип владеет ресурсами (`String`, `Vec`), присваивание `*ptr = v` сначала **уничтожит старое значение по этому адресу**. Если там неинициализированная память, это UB. `write` старое значение не трогает:

```rust
use std::mem::MaybeUninit;
use std::ptr;

fn main() {
    // write: записывает значение, не читая и не уничтожая старое.
    let mut slot = MaybeUninit::<String>::uninit();
    let p = slot.as_mut_ptr();

    // SAFETY: p указывает на память, подходящую для String; старого значения нет.
    unsafe {
        p.write(String::from("hello"));
    }
    // SAFETY: значение только что записано.
    let text = unsafe { slot.assume_init() };
    println!("{text}");

    // swap: обмен двух значений.
    let mut a = 1;
    let mut b = 2;
    // SAFETY: оба указателя получены из живых локальных переменных.
    unsafe {
        ptr::swap(&mut a, &mut b);
    }
    println!("{a} {b}");

    // replace: записывает новое значение и возвращает старое.
    let mut c = 10;
    // SAFETY: &mut c валиден.
    let old = unsafe { ptr::replace(&mut c, 20) };
    println!("{old} {c}");
}
```

Результат:

```text
hello
2 1
10 20
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3AMaybeUninit%3B%0Ause%20std%3A%3Aptr%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20%2F%2F%20write%3A%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%2C%20%D0%BD%D0%B5%20%D1%87%D0%B8%D1%82%D0%B0%D1%8F%20%D0%B8%20%D0%BD%D0%B5%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D1%8F%20%D1%81%D1%82%D0%B0%D1%80%D0%BE%D0%B5.%0A%20%20%20%20let%20mut%20slot%20%3D%20MaybeUninit%3A%3A%3CString%3E%3A%3Auninit%28%29%3B%0A%20%20%20%20let%20p%20%3D%20slot.as_mut_ptr%28%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20p%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BD%D0%B0%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%2C%20%D0%BF%D0%BE%D0%B4%D1%85%D0%BE%D0%B4%D1%8F%D1%89%D1%83%D1%8E%20%D0%B4%D0%BB%D1%8F%20String%3B%20%D1%81%D1%82%D0%B0%D1%80%D0%BE%D0%B3%D0%BE%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8F%20%D0%BD%D0%B5%D1%82.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20p.write%28String%3A%3Afrom%28%22hello%22%29%29%3B%0A%20%20%20%20%7D%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D1%87%D1%82%D0%BE%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%BE.%0A%20%20%20%20let%20text%20%3D%20unsafe%20%7B%20slot.assume_init%28%29%20%7D%3B%0A%20%20%20%20println%21%28%22%7Btext%7D%22%29%3B%0A%0A%20%20%20%20%2F%2F%20swap%3A%20%D0%BE%D0%B1%D0%BC%D0%B5%D0%BD%20%D0%B4%D0%B2%D1%83%D1%85%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B9.%0A%20%20%20%20let%20mut%20a%20%3D%201%3B%0A%20%20%20%20let%20mut%20b%20%3D%202%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BE%D0%B1%D0%B0%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8F%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D1%8B%20%D0%B8%D0%B7%20%D0%B6%D0%B8%D0%B2%D1%8B%D1%85%20%D0%BB%D0%BE%D0%BA%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D1%85%20%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%BD%D1%8B%D1%85.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20ptr%3A%3Aswap%28%26mut%20a%2C%20%26mut%20b%29%3B%0A%20%20%20%20%7D%0A%20%20%20%20println%21%28%22%7Ba%7D%20%7Bb%7D%22%29%3B%0A%0A%20%20%20%20%2F%2F%20replace%3A%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BD%D0%BE%D0%B2%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%B8%20%D0%B2%D0%BE%D0%B7%D0%B2%D1%80%D0%B0%D1%89%D0%B0%D0%B5%D1%82%20%D1%81%D1%82%D0%B0%D1%80%D0%BE%D0%B5.%0A%20%20%20%20let%20mut%20c%20%3D%2010%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%26mut%20c%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%B5%D0%BD.%0A%20%20%20%20let%20old%20%3D%20unsafe%20%7B%20ptr%3A%3Areplace%28%26mut%20c%2C%2020%29%20%7D%3B%0A%20%20%20%20println%21%28%22%7Bold%7D%20%7Bc%7D%22%29%3B%0A%7D%0A)

Здесь `MaybeUninit<String>` — безопасная обёртка для неинициализированной памяти. Мы получаем из неё указатель, записываем значение через `write` и только потом объявляем память инициализированной (`assume_init`).

Напротив, `ptr.read()` создаёт **побитовую копию**: если тип не `Copy`, после этого существуют два «владельца» одного значения, и оба будут уничтожены, то есть произойдёт двойное освобождение. Поэтому `read` для не-`Copy` типов нужно сопровождать либо `mem::forget` оригинала, либо заменой его на другое значение (`ptr::replace`), либо считать память после чтения неинициализированной.

Копирование блоков памяти:

```rust
use std::ptr;

fn main() {
    let src = [1, 2, 3, 4, 5];
    let mut dst = [0; 5];

    // SAFETY: src читается, dst пишется, 3 элемента в обоих, области не пересекаются.
    unsafe {
        ptr::copy_nonoverlapping(src.as_ptr(), dst.as_mut_ptr(), 3);
    }
    println!("{dst:?}");

    // Перекрывающиеся области: нужен ptr::copy (аналог memmove).
    let mut data = [1, 2, 3, 4, 5];
    let p = data.as_mut_ptr();
    // SAFETY: обе области внутри data; copy допускает пересечение.
    unsafe {
        ptr::copy(p, p.add(1), 4);
    }
    println!("{data:?}");
}
```

Результат:

```text
[1, 2, 3, 0, 0]
[1, 1, 2, 3, 4]
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aptr%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20src%20%3D%20%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%20%20%20%20let%20mut%20dst%20%3D%20%5B0%3B%205%5D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20src%20%D1%87%D0%B8%D1%82%D0%B0%D0%B5%D1%82%D1%81%D1%8F%2C%20dst%20%D0%BF%D0%B8%D1%88%D0%B5%D1%82%D1%81%D1%8F%2C%203%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%B0%20%D0%B2%20%D0%BE%D0%B1%D0%BE%D0%B8%D1%85%2C%20%D0%BE%D0%B1%D0%BB%D0%B0%D1%81%D1%82%D0%B8%20%D0%BD%D0%B5%20%D0%BF%D0%B5%D1%80%D0%B5%D1%81%D0%B5%D0%BA%D0%B0%D1%8E%D1%82%D1%81%D1%8F.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20ptr%3A%3Acopy_nonoverlapping%28src.as_ptr%28%29%2C%20dst.as_mut_ptr%28%29%2C%203%29%3B%0A%20%20%20%20%7D%0A%20%20%20%20println%21%28%22%7Bdst%3A%3F%7D%22%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9F%D0%B5%D1%80%D0%B5%D0%BA%D1%80%D1%8B%D0%B2%D0%B0%D1%8E%D1%89%D0%B8%D0%B5%D1%81%D1%8F%20%D0%BE%D0%B1%D0%BB%D0%B0%D1%81%D1%82%D0%B8%3A%20%D0%BD%D1%83%D0%B6%D0%B5%D0%BD%20ptr%3A%3Acopy%20%28%D0%B0%D0%BD%D0%B0%D0%BB%D0%BE%D0%B3%20memmove%29.%0A%20%20%20%20let%20mut%20data%20%3D%20%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%20%20%20%20let%20p%20%3D%20data.as_mut_ptr%28%29%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BE%D0%B1%D0%B5%20%D0%BE%D0%B1%D0%BB%D0%B0%D1%81%D1%82%D0%B8%20%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B8%20data%3B%20copy%20%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D0%BA%D0%B0%D0%B5%D1%82%20%D0%BF%D0%B5%D1%80%D0%B5%D1%81%D0%B5%D1%87%D0%B5%D0%BD%D0%B8%D0%B5.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20ptr%3A%3Acopy%28p%2C%20p.add%281%29%2C%204%29%3B%0A%20%20%20%20%7D%0A%20%20%20%20println%21%28%22%7Bdata%3A%3F%7D%22%29%3B%0A%7D%0A)

Второй вызов показывает, зачем нужен отдельный `ptr::copy`: мы сдвигаем элементы внутри одного массива, и области источника и приёмника пересекаются. `copy_nonoverlapping` в такой ситуации давал бы UB.

---

## 45.11. Когда использовать сырые указатели

Raw pointers оправданы прежде всего там, где обычных ссылок и безопасных стандартных абстракций недостаточно.

Типичные случаи:

1. **FFI** — взаимодействие с C/C++ и системными API.
2. **Реализация низкоуровневых структур данных** — например, intrusive collections, деревья, списки и allocators.
3. **Работа с вручную управляемой памятью.**
4. **Низкоуровневый runtime или embedded-код.**
5. **Создание безопасной абстракции**, внутри которой необходимо выполнить небольшое количество `unsafe` операций.
6. **Оптимизация**, когда после измерений доказано, что безопасная реализация действительно недостаточна.

При этом наличие raw pointer само по себе не означает, что код быстрее.

Например, такой код:

```rust
unsafe {
    *ptr.add(i)
}
```

не становится автоматически быстрее:

```rust
array[i]
```

Компилятор Rust часто способен оптимизировать безопасный код очень агрессивно.

Поэтому правильный порядок такой:

> **Сначала безопасная реализация → затем измерение → затем доказательство необходимости `unsafe` → затем минимальная unsafe-оптимизация.**

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `null` pointer

```rust
fn main() {
    let ptr: *const i32 = std::ptr::null();

    unsafe {
        println!("{}", *ptr);
    }
}
```

Не следует ожидать от этого обязательно `panic`.

Разыменование null pointer нарушает требования безопасности и приводит к **undefined behavior**. Наблюдаемое поведение программы не является частью контракта Rust. На rustc 1.99 в debug-сборке программа прерывается с сообщением о нулевой ссылке, а в release завершается аварийно; на другой версии компилятора или платформе результат может быть другим. Подробнее — в разделе 44.8.

### Эксперимент 2: выход за границы массива

```rust
fn main() {
    let array = [10, 20, 30];
    let ptr = array.as_ptr();

    unsafe {
        // add(3) указывает за последним элементом (one-past-the-end).
        let outside = ptr.add(3);

        // Разыменование здесь недопустимо.
        println!("{}", *outside);
    }
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20array%20%3D%20%5B10%2C%2020%2C%2030%5D%3B%0A%20%20%20%20let%20ptr%20%3D%20array.as_ptr%28%29%3B%0A%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20add%283%29%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%B7%D0%B0%20%D0%BF%D0%BE%D1%81%D0%BB%D0%B5%D0%B4%D0%BD%D0%B8%D0%BC%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%BE%D0%BC%20%28one-past-the-end%29.%0A%20%20%20%20%20%20%20%20let%20outside%20%3D%20ptr.add%283%29%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%A0%D0%B0%D0%B7%D1%8B%D0%BC%D0%B5%D0%BD%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20%D0%B7%D0%B4%D0%B5%D1%81%D1%8C%20%D0%BD%D0%B5%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D0%BC%D0%BE.%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20%2Aoutside%29%3B%0A%20%20%20%20%7D%0A%7D%0A)

Здесь особенно важно различать:

```rust
ptr.add(3)
```

и

```rust
*ptr.add(3)
```

Получение позиции one-past-the-end может быть допустимой частью pointer arithmetic, но **разыменовывать такую позицию нельзя**.

Эта программа читает память за массивом, то есть «чужие» байты стека. При запуске мы получили `-608853256` в debug и `1466418584` в release: числа разные и при каждом запуске могут отличаться. Это не ошибка компилятора, а естественный результат UB.

### Эксперимент 3: невыровненный доступ

Сначала посмотрим, что происходит при обычном разыменовании:

```rust
fn main() {
    let bytes = [0u8; 8];

    let ptr = unsafe { bytes.as_ptr().add(1) } as *const u32;

    // UB: адрес не выровнен для u32.
    let value = unsafe { *ptr };

    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground (UB: результат зависит от режима сборки)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20bytes%20%3D%20%5B0u8%3B%208%5D%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20bytes.as_ptr%28%29.add%281%29%20%7D%20as%20%2Aconst%20u32%3B%0A%0A%20%20%20%20%2F%2F%20UB%3A%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%D0%BD%D0%B5%20%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20u32.%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20%2Aptr%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

В debug-сборке современный rustc прерывает программу:

```text
misaligned pointer dereference: address must be a multiple of 0x4 but is 0x7ffd...
```

В release-сборке этой проверки нет, и программа молча печатает `0`, хотя UB остаётся. Исправление — чтение, допускающее невыровненный адрес:

```rust
use std::ptr;

fn main() {
    let bytes = [0u8; 8];

    let ptr = unsafe { bytes.as_ptr().add(1) } as *const u32;

    // Обычное `*ptr` требует корректного alignment.
    //
    // Для потенциально невыровненного чтения используется:
    let value = unsafe { ptr::read_unaligned(ptr) };

    println!("{value}");
}
```

Результат:

```text
0
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aptr%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20bytes%20%3D%20%5B0u8%3B%208%5D%3B%0A%0A%20%20%20%20let%20ptr%20%3D%20unsafe%20%7B%20bytes.as_ptr%28%29.add%281%29%20%7D%20as%20%2Aconst%20u32%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9E%D0%B1%D1%8B%D1%87%D0%BD%D0%BE%D0%B5%20%60%2Aptr%60%20%D1%82%D1%80%D0%B5%D0%B1%D1%83%D0%B5%D1%82%20%D0%BA%D0%BE%D1%80%D1%80%D0%B5%D0%BA%D1%82%D0%BD%D0%BE%D0%B3%D0%BE%20alignment.%0A%20%20%20%20%2F%2F%0A%20%20%20%20%2F%2F%20%D0%94%D0%BB%D1%8F%20%D0%BF%D0%BE%D1%82%D0%B5%D0%BD%D1%86%D0%B8%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%20%D0%BD%D0%B5%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%D0%BD%D0%BE%D0%B3%D0%BE%20%D1%87%D1%82%D0%B5%D0%BD%D0%B8%D1%8F%20%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D1%83%D0%B5%D1%82%D1%81%D1%8F%3A%0A%20%20%20%20let%20value%20%3D%20unsafe%20%7B%20ptr%3A%3Aread_unaligned%28ptr%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A)

### Эксперимент 4: aliasing

Запустите пример `aliasing_bad` из раздела 45.9 в Rust Playground через меню Tools → Miri. Прочитайте сообщение об ошибке и найдите в нём указание на то, какая именно ссылка сделала `p1` недействительным. Затем замените создание указателей на `&raw mut value` и убедитесь, что ошибка исчезла.

### Эксперимент 5: provenance

Возьмите пример `prov_strict` и замените `ptr.with_addr(address)` на `std::ptr::without_provenance::<i32>(address)`. Программа скомпилируется, но разыменование такого указателя — UB: у него есть адрес, но нет права доступа. Проверьте это под Miri.

---

## Практика

### Задание 1

Создайте `*const i32` из ссылки и прочитайте значение через raw pointer.

Убедитесь, что `unsafe` нужен только для разыменования.

### Задание 2

Создайте массив и реализуйте функцию:

```rust
unsafe fn sum(ptr: *const i32, len: usize) -> i32
```

которая вычисляет сумму элементов через `ptr.add(index)`.

Документируйте `# Safety` и сформулируйте все условия, которые должен выполнить вызывающий код. Образец полного решения:

```rust
/// Sums `len` consecutive `i32` values starting at `ptr`.
///
/// # Safety
///
/// - `ptr` must be non-null and properly aligned for `i32`.
/// - `ptr` must be valid for reads of `len` consecutive, initialized `i32` values
///   inside a single allocation.
/// - `len * size_of::<i32>()` must not exceed `isize::MAX`.
/// - The memory must not be mutated through other pointers during the call.
unsafe fn sum(ptr: *const i32, len: usize) -> i32 {
    let mut total = 0;
    for index in 0..len {
        // SAFETY: index < len, остальное гарантирует контракт функции.
        total += unsafe { *ptr.add(index) };
    }
    total
}

fn main() {
    let array = [1, 2, 3, 4];

    // SAFETY: указатель и длина получены из живого массива.
    let result = unsafe { sum(array.as_ptr(), array.len()) };

    println!("{result}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%2F%20Sums%20%60len%60%20consecutive%20%60i32%60%20values%20starting%20at%20%60ptr%60.%0A%2F%2F%2F%0A%2F%2F%2F%20%23%20Safety%0A%2F%2F%2F%0A%2F%2F%2F%20-%20%60ptr%60%20must%20be%20non-null%20and%20properly%20aligned%20for%20%60i32%60.%0A%2F%2F%2F%20-%20%60ptr%60%20must%20be%20valid%20for%20reads%20of%20%60len%60%20consecutive%2C%20initialized%20%60i32%60%20values%0A%2F%2F%2F%20%20%20inside%20a%20single%20allocation.%0A%2F%2F%2F%20-%20%60len%20%2A%20size_of%3A%3A%3Ci32%3E%28%29%60%20must%20not%20exceed%20%60isize%3A%3AMAX%60.%0A%2F%2F%2F%20-%20The%20memory%20must%20not%20be%20mutated%20through%20other%20pointers%20during%20the%20call.%0Aunsafe%20fn%20sum%28ptr%3A%20%2Aconst%20i32%2C%20len%3A%20usize%29%20-%3E%20i32%20%7B%0A%20%20%20%20let%20mut%20total%20%3D%200%3B%0A%20%20%20%20for%20index%20in%200..len%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20index%20%3C%20len%2C%20%D0%BE%D1%81%D1%82%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%D0%B5%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%20%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8.%0A%20%20%20%20%20%20%20%20total%20%2B%3D%20unsafe%20%7B%20%2Aptr.add%28index%29%20%7D%3B%0A%20%20%20%20%7D%0A%20%20%20%20total%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20array%20%3D%20%5B1%2C%202%2C%203%2C%204%5D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%B8%20%D0%B4%D0%BB%D0%B8%D0%BD%D0%B0%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D1%8B%20%D0%B8%D0%B7%20%D0%B6%D0%B8%D0%B2%D0%BE%D0%B3%D0%BE%20%D0%BC%D0%B0%D1%81%D1%81%D0%B8%D0%B2%D0%B0.%0A%20%20%20%20let%20result%20%3D%20unsafe%20%7B%20sum%28array.as_ptr%28%29%2C%20array.len%28%29%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A)

Сравните свой контракт с образцом: не забыты ли условия на выравнивание, на ограничение `isize::MAX` и на отсутствие мутации через другие указатели?

### Задание 3

Напишите функцию:

```rust
unsafe fn write_value(ptr: *mut i32, value: i32)
```

которая записывает значение через raw pointer.

Добавьте документацию `# Safety`. Подумайте, нужно ли требовать, чтобы память по адресу была инициализирована (для записи `i32` — нет, а для `String` через `*ptr = v` — да, потому что старое значение уничтожается). Затем вызовите функцию корректно.

### Задание 4

Создайте `*mut i32`, измените значение через него и затем выведите значение через обычную переменную.

Например:

```rust
fn main() {
    let mut value = 10;
    let ptr = &mut value as *mut i32;

    unsafe {
        *ptr = 42;
    }

    println!("{value}");
}
```

Перепишите пример через `&raw mut` и объясните, чем он отличается.

### Задание 5

🔨 **Эксперимент с компилятором.**

Попробуйте:

```rust
let ptr = std::ptr::null::<i32>();

unsafe {
    println!("{}", *ptr);
}
```

Объясните, почему `unsafe` разрешает компиляцию, но не делает операцию корректной.

### Задание 6

🔨 **Эксперимент с границами.**

Из массива:

```rust
let array = [10, 20, 30];
let ptr = array.as_ptr();
```

получите:

```rust
ptr.add(3)
```

и объясните, почему сам pointer arithmetic отличается от разыменования полученного указателя. Затем вычислите `ptr.add(3).offset_from(ptr)` и объясните результат.

### Задание 7

Используйте `std::ptr::read_unaligned()` для чтения `u32` из массива байтов, начиная со смещения `1`.

Объясните, чем такой код отличается от обычного:

```rust
*ptr
```

Сравните результат с `u32::from_le_bytes` и объясните, при каком порядке байтов они совпадут.

### Задание 8

Напишите функцию `swap_values(a: *mut i32, b: *mut i32)`, которая меняет значения местами без `ptr::swap`, используя `read` и `write`. Что случится, если `a == b`? Сравните с поведением `ptr::swap`.

### Задание 9

Реализуйте pointer tagging для `*mut u64` с помощью `map_addr`: функции `tag(ptr)`, `is_tagged(ptr)`, `untag(ptr)`. Объясните, почему этот код не требует `usize as *mut u64` и каким образом он сохраняет provenance.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`*const T` и `*mut T`** — raw pointers;
- raw pointer может быть `null`, dangling или невыровненным;
- **создание raw pointer обычно безопасно**, а его разыменование — `unsafe`;
- `*const T` не означает, что объект неизменяем; но превращение `*const T` в `*mut T` само по себе не даёт права записи — оно определяется происхождением указателя;
- `*mut T` позволяет выполнять запись только при соблюдении всех требований безопасности;
- **`&raw const` / `&raw mut`** создают указатель на место без промежуточной ссылки и нужны там, где ссылка недопустима (`repr(packed)`, `static mut`);
- **`add`, `sub`, `offset`** работают с элементами типа `T`, а не с байтами, и требуют, чтобы результат оставался в пределах allocation (или в позиции one-past-the-end) уже на момент вычисления;
- `wrapping_add()` не делает полученный указатель автоматически валидным;
- **`null` — только один из возможных вариантов невалидного указателя**;
- `is_null()` не является полной проверкой валидности; `as_ref()` превращает `null` в `None`, но остальные условия остаются на вызывающем коде;
- **alignment** — отдельное требование к адресу;
- для потенциально невыровненных данных существуют `read_unaligned()` и `write_unaligned()`;
- **validity** включает не только ненулевой адрес, но и корректность памяти, размера, инициализации, alignment и других условий;
- **provenance** — часть указателя, определяющая, к какой памяти он вправе обращаться; raw pointer нельзя рассматривать просто как целое число. Используйте Strict Provenance (`addr`, `with_addr`, `map_addr`), а `as`-приведение через `usize` оставляйте для случаев, когда иначе нельзя;
- `*ptr = v`, `ptr.write(v)`, `ptr.read()` и `ptr::replace` ведут себя по-разному; запись в неинициализированную память нужно делать через `write`;
- `ptr::copy_nonoverlapping` требует непересекающихся областей, `ptr::copy` допускает пересечение;
- `unsafe` не отключает правила Rust — он позволяет программисту взять на себя ответственность за те условия, которые компилятор не может доказать.

**Самая важная идея:**

> Raw pointer — это не «адрес памяти без ограничений». Это низкоуровневый инструмент, который позволяет работать с памятью за пределами возможностей обычных ссылок. Но каждое разыменование должно опираться на доказательство того, что указатель действительно можно использовать: allocation существует, память достаточно велика, объект инициализирован, alignment корректен, операция находится в допустимых границах и соблюдены требования модели памяти Rust.
>
> Хороший unsafe-код не просто «работает на моей машине». Он содержит явно сформулированные safety requirements и минимальное количество операций, которые невозможно выразить средствами безопасного Rust.
