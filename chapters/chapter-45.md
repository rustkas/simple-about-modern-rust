# Глава 45. Raw Pointers

В предыдущей главе мы узнали, что `unsafe` даёт доступ к пяти superpowers, и первая из них — **разыменовывание сырых указателей**. Сырые указатели — это основа для взаимодействия с памятью на низком уровне, FFI и реализации эффективных структур данных.

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+x+%3D+42%3B%0A++++let+ptr%3A+%2Aconst+i32+%3D+%26x%3B%0A++++println%21%28%22address%3A+%7Bptr%3Ap%7D%22%29%3B%0A++++unsafe+%7B%0A++++++++println%21%28%22value%3A+%7B%7D%22%2C+%2Aptr%29%3B%0A++++%7D%0A%7D)

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+x+%3D+42%3B%0A++++let+const_ptr%3A+%2Aconst+i32+%3D+%26x%3B%0A++++let+mut+y+%3D+10%3B%0A++++let+mut_ptr%3A+%2Amut+i32+%3D+%26mut+y%3B%0A++++println%21%28%22const_ptr%3A+%7Bconst_ptr%3Ap%7D%22%29%3B%0A++++println%21%28%22mut_ptr%3A+++%7Bmut_ptr%3Ap%7D%22%29%3B%0A%7D)

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+value+%3D+Box%3A%3Anew%2842%29%3B%0A++++let+ptr+%3D+Box%3A%3Ainto_raw%28value%29%3B%0A++++unsafe+%7B%0A++++++++println%21%28%22value%3A+%7B%7D%22%2C+%2Aptr%29%3B%0A++++++++drop%28Box%3A%3Afrom_raw%28ptr%29%29%3B%0A++++%7D%0A%7D)

Здесь существует важное правило:

> После `Box::into_raw()` ответственность за освобождение памяти временно переходит от `Box` к программисту.

`Box::from_raw()` следует использовать только с указателем, который был получен из соответствующего выделения и которым вы всё ещё владеете. Для одной и той же allocation нельзя дважды вызвать `Box::from_raw()` с одним и тем же указателем.

### Из адреса

Raw pointer можно получить из целочисленного адреса:

```rust
fn main() {
    let address = 0x1000usize;
    let ptr = address as *const i32;

    println!("pointer: {ptr:p}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+address+%3D+0x1000usize%3B%0A++++let+ptr+%3D+address+as+%2Aconst+i32%3B%0A++++println%21%28%22pointer%3A+%7Bptr%3Ap%7D%22%29%3B%0A%7D)

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Aptr%3B%0A%0Afn+main%28%29+%7B%0A++++let+ptr%3A+%2Aconst+i32+%3D+ptr%3A%3Anull%28%29%3B%0A++++let+mut_ptr%3A+%2Amut+i32+%3D+ptr%3A%3Anull_mut%28%29%3B%0A++++println%21%28%22ptr%3A+++++%7Bptr%3Ap%7D%22%29%3B%0A++++println%21%28%22mut_ptr%3A+%7Bmut_ptr%3Ap%7D%22%29%3B%0A++++println%21%28%22ptr+is+null%3A+%7B%7D%22%2C+ptr.is_null%28%29%29%3B%0A++++println%21%28%22mut_ptr+is+null%3A+%7B%7D%22%2C+mut_ptr.is_null%28%29%29%3B%0A%7D)

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+x+%3D+42%3B%0A++++let+ptr+%3D+%26x+as+%2Aconst+i32%3B%0A++++println%21%28%22Pointer%3A+%7Bptr%3Ap%7D%22%29%3B%0A++++unsafe+%7B%0A++++++++let+value+%3D+%2Aptr%3B%0A++++++++println%21%28%22Value%3A+%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

Почему Rust требует `unsafe`?

Потому что компилятор не может в общем случае доказать, что указатель:

- не является `null`;
- указывает на существующий объект;
- указывает на инициализированную память;
- правильно выровнен;
- находится в допустимых границах allocation;
- не является dangling pointer;
- используется с соблюдением требований модели памяти Rust.

Например:

```rust
fn main() {
    let ptr: *const i32 = std::ptr::null();

    // Это не panic, который Rust обязан обнаружить.
    // Разыменование такого указателя приводит к UB.
    //
    // unsafe {
    //     println!("{}", *ptr);
    // }
}
```

Поэтому комментарий `// ❌ Паника!` здесь неправильный. **Undefined behavior не следует путать с panic.** Программа может аварийно завершиться, а может проявить совершенно другое поведение. Полагаться на конкретный результат нельзя.

---

## 45.4. Pointer arithmetic — арифметика указателей

Raw pointers можно перемещать относительно элементов allocation.

Для этого используются, в частности:

| Метод             | Назначение                                                                                    |
| ----------------- | --------------------------------------------------------------------------------------------- |
| `add(n)`          | перейти на `n` элементов вперёд                                                               |
| `sub(n)`          | перейти на `n` элементов назад                                                                |
| `offset(n)`       | перейти на `n` элементов вперёд или назад; `n` имеет тип `isize`                              |
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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+array+%3D+%5B10%2C+20%2C+30%2C+40%2C+50%5D%3B%0A++++let+ptr+%3D+array.as_ptr%28%29%3B%0A++++unsafe+%7B%0A++++++++for+index+in+0..array.len%28%29+%7B%0A++++++++++++println%21%28%22array%5B%7Bindex%7D%5D+%3D+%7B%7D%22%2C+%2Aptr.add%28index%29%29%3B%0A++++++++%7D%0A++++%7D%0A%7D)

Здесь каждый вызов:

```rust
ptr.add(index)
```

должен оставаться в пределах той же allocation либо указывать на допустимую позицию one-past-the-end в случаях, предусмотренных правилами соответствующей операции. **Разыменовывать one-past-the-end нельзя.**

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+array+%3D+%5B10%2C+20%2C+30%2C+40%2C+50%5D%3B%0A++++let+ptr+%3D+unsafe+%7B+array.as_ptr%28%29.add%283%29+%7D%3B%0A++++unsafe+%7B%0A++++++++println%21%28%22%7B%7D%22%2C+%2Aptr%29%3B%0A++++++++println%21%28%22%7B%7D%22%2C+%2Aptr.sub%281%29%29%3B%0A++++++++println%21%28%22%7B%7D%22%2C+%2Aptr.sub%282%29%29%3B%0A++++%7D%0A%7D)

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+array+%3D+%5B10%2C+20%2C+30%2C+40%2C+50%5D%3B%0A++++let+ptr+%3D+array.as_ptr%28%29%3B%0A++++unsafe+%7B%0A++++++++let+third+%3D+ptr.offset%282%29%3B%0A++++++++let+second+%3D+third.offset%28-1%29%3B%0A++++++++println%21%28%22%7B%7D%22%2C+%2Athird%29%3B%0A++++++++println%21%28%22%7B%7D%22%2C+%2Asecond%29%3B%0A++++%7D%0A%7D)

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+array+%3D+%5B10%2C+20%2C+30%5D%3B%0A++++let+ptr+%3D+array.as_ptr%28%29%3B%0A++++let+outside+%3D+ptr.wrapping_add%28100%29%3B%0A++++println%21%28%22ptr%3A+++++%7Bptr%3Ap%7D%22%29%3B%0A++++println%21%28%22outside%3A+%7Boutside%3Ap%7D%22%29%3B%0A%7D)

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+print_value%28ptr%3A+%2Aconst+i32%29+%7B%0A++++if+ptr.is_null%28%29+%7B%0A++++++++println%21%28%22Pointer+is+null%22%29%3B%0A++++++++return%3B%0A++++%7D%0A++++unsafe+%7B%0A++++++++println%21%28%22Value%3A+%7B%7D%22%2C+%2Aptr%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+42%3B%0A++++let+valid+%3D+%26value+as+%2Aconst+i32%3B%0A++++let+null+%3D+std%3A%3Aptr%3A%3Anull%28%29%3B%0A++++print_value%28valid%29%3B%0A++++print_value%28null%29%3B%0A%7D)

Но проверка `is_null()` **не является универсальной проверкой безопасности**.

Например, указатель может быть ненулевым, но всё равно:

- указывать на освобождённую память;
- быть невыровненным;
- указывать на неинициализированные данные;
- не иметь права на соответствующий доступ.

Поэтому:

> **`!ptr.is_null()` не означает `ptr` valid.**

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amem%3A%3A%7Balign_of%2C+size_of%7D%3B%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22i32+size%3A++%7B%7D%22%2C+size_of%3A%3A%3Ci32%3E%28%29%29%3B%0A++++println%21%28%22i32+align%3A+%7B%7D%22%2C+align_of%3A%3A%3Ci32%3E%28%29%29%3B%0A++++println%21%28%22u8+size%3A+++%7B%7D%22%2C+size_of%3A%3Cu8%3E%28%29%29%3B%0A++++println%21%28%22u8+align%3A++%7B%7D%22%2C+align_of%3A%3Cu8%3E%28%29%29%3B%0A%7D)

Raw pointer может иметь адрес, который не удовлетворяет alignment `T`.

Например:

```rust
use std::ptr;

fn main() {
    let bytes = [0u8; 8];

    let ptr = unsafe { bytes.as_ptr().add(1) } as *const u32;

    println!("address: {ptr:p}");

    // Обычное разыменование такого указателя недопустимо,
    // если адрес не выровнен для u32.
}
```

Для чтения данных из потенциально невыровненной области памяти существуют специальные операции:

```rust
use std::ptr;

fn main() {
    let bytes = [0x78u8, 0x56, 0x34, 0x12];

    let ptr = bytes.as_ptr() as *const u32;

    let value = unsafe {
        ptr::read_unaligned(ptr)
    };

    println!("value: 0x{value:08x}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Aptr%3B%0A%0Afn+main%28%29+%7B%0A++++let+bytes+%3D+%5B0x78u8%2C+0x56%2C+0x34%2C+0x12%5D%3B%0A++++let+ptr+%3D+bytes.as_ptr%28%29+as+%2Aconst+u32%3B%0A++++let+value+%3D+unsafe+%7B%0A++++++++ptr%3A%3Aread_unaligned%28ptr%29%0A++++%7D%3B%0A++++println%21%28%22value%3A+0x%7Bvalue%3A08x%7D%22%29%3B%0A%7D)

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

**Provenance** можно понимать как информацию о том, **какой allocation связан с указателем и какие операции над этой allocation допустимы**.

Это особенно важно при работе с raw pointers.

Упрощённо можно представить ситуацию так:

```text
allocation
┌──────────────────────────────┐
│ 10 │ 20 │ 30 │ 40 │ 50       │
└──────────────────────────────┘
  ↑
  │
  ptr
```

Если `ptr` получен из этой allocation, операции вроде:

```rust
ptr.add(2)
```

используются для перехода к третьему элементу той же области памяти.

Но простое получение числового адреса и последующее превращение этого числа обратно в указатель **не следует рассматривать как способ создать произвольный валидный указатель на объект**.

Например:

```rust
fn main() {
    let value = 42;

    let ptr = &value as *const i32;
    let address = ptr as usize;

    println!("address: 0x{address:x}");

    let restored = address as *const i32;

    unsafe {
        println!("value: {}", *restored);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+value+%3D+42%3B%0A++++let+ptr+%3D+%26value+as+%2Aconst+i32%3B%0A++++let+address+%3D+ptr+as+usize%3B%0A++++println%21%28%22address%3A+0x%7Baddress%3Ax%7D%22%29%3B%0A++++let+restored+%3D+address+as+%2Aconst+i32%3B%0A++++unsafe+%7B%0A++++++++println%21%28%22value%3A+%7B%7D%22%2C+%2Arestored%29%3B%0A++++%7D%0A%7D)

На конкретной платформе такой код может выглядеть совершенно нормально, но **нельзя строить корректность unsafe-кода на предположении, что raw pointer — всего лишь integer с адресом**.

Современная модель Rust рассматривает указатель сложнее, чем просто числовой адрес. В unsafe-коде поэтому следует сохранять происхождение указателей и использовать предусмотренные Rust операции над ними, а не моделировать память как произвольный массив адресов.

`wrapping_add()` также не отменяет provenance и других требований к последующему доступу:

```rust
let next = ptr.wrapping_add(10);

// Получение указателя допустимо,
// но это не означает, что `next` можно разыменовать.
```

Главная практическая мысль:

> **Raw pointer — это не просто число-адрес. При работе с ним нужно учитывать allocation, границы, alignment, validity и provenance.**

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

Например, наличие raw pointers позволяет обойти обычную проверку borrow checker'а:

```rust
fn main() {
    let mut value = 10;

    let p1 = &mut value as *mut i32;
    let p2 = &mut value as *mut i32;

    unsafe {
        *p1 = 20;
        *p2 = 30;
    }

    println!("{value}");
}
```

Сам факт, что компилятор разрешил создать raw pointers, **не означает автоматически, что любая последующая комбинация операций через них корректна**.

Именно поэтому raw pointers не следует рассматривать как «способ обмануть borrow checker». Их назначение — дать возможность реализовать низкоуровневые механизмы, когда программист способен самостоятельно доказать корректность.

---

## 45.10. Когда использовать сырые указатели

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

Разыменование null pointer нарушает требования безопасности и приводит к **undefined behavior**. Наблюдаемое поведение программы не является частью контракта Rust.

### Эксперимент 2: выход за границы массива

```rust
fn main() {
    let array = [10, 20, 30];
    let ptr = array.as_ptr();

    unsafe {
        // `add(3)` указывает за последним элементом.
        let outside = ptr.add(3);

        // Разыменование здесь недопустимо.
        println!("{}", *outside);
    }
}
```

Здесь особенно важно различать:

```rust
ptr.add(3)
```

и

```rust
*ptr.add(3)
```

Получение позиции one-past-the-end может быть допустимой частью pointer arithmetic, но **разыменовывать такую позицию нельзя**.

### Эксперимент 3: невыровненный доступ

```rust
use std::ptr;

fn main() {
    let bytes = [0u8; 8];

    let ptr = unsafe {
        bytes.as_ptr().add(1)
    } as *const u32;

    // Обычное `*ptr` требует корректного alignment.
    //
    // Для потенциально невыровненного чтения используется:
    let value = unsafe {
        ptr::read_unaligned(ptr)
    };

    println!("{value}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Aptr%3B%0A%0Afn+main%28%29+%7B%0A++++let+bytes+%3D+%5B0u8%3B+8%5D%3B%0A++++let+ptr+%3D+unsafe+%7B+bytes.as_ptr%28%29.add%281%29+%7D+as+%2Aconst+u32%3B%0A++++let+value+%3D+unsafe+%7B+ptr%3A%3Aread_unaligned%28ptr%29+%7D%3B%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D)

---

## Практика

### Задание 1

Создайте `*const i32` из ссылки и прочитайте значение через raw pointer.

Убедитесь, что `unsafe` нужен только для разыменования.

### Задание 2

Создайте массив и реализуйте функцию:

```rust
fn sum(ptr: *const i32, len: usize) -> i32
```

которая вычисляет сумму элементов через `ptr.add(index)`.

Документируйте `# Safety` и сформулируйте все условия, которые должен выполнить вызывающий код.

### Задание 3

Напишите функцию:

```rust
unsafe fn write_value(ptr: *mut i32, value: i32)
```

которая записывает значение через raw pointer.

Добавьте документацию:

```rust
/// # Safety
///
/// `ptr` must be non-null, properly aligned, and point to
/// valid writable memory containing an `i32`.
```

Затем вызовите функцию корректно.

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

и объясните, почему сам pointer arithmetic отличается от разыменования полученного указателя.

### Задание 7

Используйте `std::ptr::read_unaligned()` для чтения `u32` из массива байтов, начиная со смещения `1`.

Объясните, чем такой код отличается от обычного:

```rust
*ptr
```

---

## Главное из этой главы

После этой главы мы понимаем:

- **`*const T` и `*mut T`** — raw pointers;
- raw pointer может быть `null`, dangling или невыровненным;
- **создание raw pointer обычно безопасно**, а его разыменование — `unsafe`;
- `*const T` не означает, что объект неизменяем; это лишь pointer type, предназначенный для чтения;
- `*mut T` позволяет выполнять запись только при соблюдении всех требований безопасности;
- **`add`, `sub`, `offset`** работают с элементами типа `T`, а не с байтами;
- `wrapping_add()` не делает полученный указатель автоматически валидным;
- **`null` — только один из возможных вариантов невалидного указателя**;
- `is_null()` не является полной проверкой валидности;
- **alignment** — отдельное требование к адресу;
- для потенциально невыровненных данных существуют `read_unaligned()` и `write_unaligned()`;
- **validity** включает не только ненулевой адрес, но и корректность памяти, размера, инициализации, alignment и других условий;
- **provenance** означает, что raw pointer нельзя концептуально рассматривать просто как произвольное целое число;
- `unsafe` не отключает правила Rust — он позволяет программисту взять на себя ответственность за те условия, которые компилятор не может доказать.

**Самая важная идея:**

> Raw pointer — это не «адрес памяти без ограничений». Это низкоуровневый инструмент, который позволяет работать с памятью за пределами возможностей обычных ссылок. Но каждое разыменование должно опираться на доказательство того, что указатель действительно можно использовать: allocation существует, память достаточно велика, объект инициализирован, alignment корректен, операция находится в допустимых границах и соблюдены требования модели памяти Rust.
>
> Хороший unsafe-код не просто «работает на моей машине». Он содержит явно сформулированные safety requirements и минимальное количество операций, которые невозможно выразить средствами безопасного Rust.
