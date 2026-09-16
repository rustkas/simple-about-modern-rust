# Глава 39. Features и Conditional Compilation

В реальных проектах часто возникает необходимость включать или выключать части кода в зависимости от условий: платформа (Windows/Linux), целевая архитектура, включённые возможности библиотеки или окружение.

Rust предоставляет мощный механизм для условной компиляции — **атрибуты `cfg`** и **Cargo features**. Они позволяют создавать гибкие библиотеки и приложения, которые адаптируются под разные среды и требования.

В этой главе мы разберёмся, как работает условная компиляция, как проектировать features и избегать проблем с их разрастанием.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 39.1. Что такое Conditional Compilation?

**Условная компиляция** — это возможность включать или исключать части кода во время сборки в зависимости от условий.

В Rust это реализуется через:
- **Атрибуты `#[cfg(...)]`** — для элементов кода (функции, модули, структуры).
- **Макрос `cfg!(...)`** — для условного выполнения во время выполнения (но проверяется на этапе компиляции).

```rust
// Код компилируется только на Windows
#[cfg(target_os = "windows")]
fn windows_only_function() {
    println!("This is Windows!");
}

// Код компилируется только на Linux
#[cfg(target_os = "linux")]
fn linux_only_function() {
    println!("This is Linux!");
}

fn main() {
    // Условный код во время выполнения
    if cfg!(target_os = "windows") {
        println!("Running on Windows!");
    } else if cfg!(target_os = "linux") {
        println!("Running on Linux!");
    } else {
        println!("Unknown OS!");
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bcfg%28target_os%20%3D%20%22windows%22%29%5D%0Afn%20windows_only_function%28%29%20%7B%0A%20%20%20%20println%21%28%22This%20is%20Windows%21%22%29%3B%0A%7D%0A%0A%23%5Bcfg%28target_os%20%3D%20%22linux%22%29%5D%0Afn%20linux_only_function%28%29%20%7B%0A%20%20%20%20println%21%28%22This%20is%20Linux%21%22%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20if%20cfg%21%28target_os%20%3D%20%22windows%22%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Running%20on%20Windows%21%22%29%3B%0A%20%20%20%20%7D%20else%20if%20cfg%21%28target_os%20%3D%20%22linux%22%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Running%20on%20Linux%21%22%29%3B%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Unknown%20OS%21%22%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 39.2. Атрибуты `#[cfg]`

`#[cfg]` применяется к элементам кода:

```rust
// Условная компиляция модуля
#[cfg(target_os = "windows")]
mod windows_specific {
    pub fn do_something() {
        println!("Windows specific code");
    }
}

#[cfg(not(target_os = "windows"))]
mod windows_specific {
    pub fn do_something() {
        println!("Not Windows code");
    }
}

// Условная компиляция структуры
#[cfg(feature = "serde")]
#[derive(serde::Serialize, serde::Deserialize)]
struct Config {
    url: String,
    timeout: u32,
}

// Условная компиляция функции
#[cfg(all(feature = "logging", debug_assertions))]
fn debug_log(message: &str) {
    println!("[DEBUG] {}", message);
}
```

---

## 39.3. Атрибуты `cfg_attr`

`cfg_attr` позволяет условно применять атрибуты:

```rust
// Если включена фича "serde", добавляем derive атрибуты
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
struct Config {
    url: String,
    timeout: u32,
}

// Если включена фича "debug", добавляем атрибут для отладки
#[cfg_attr(debug_assertions, allow(dead_code))]
struct InternalData {
    // ...
}
```

---

## 39.4. Условия `cfg`

**Примеры условий:**

| Условие | Что означает |
|---|---|
| `#[cfg(target_os = "windows")]` | Windows |
| `#[cfg(target_os = "linux")]` | Linux |
| `#[cfg(target_arch = "x86_64")]` | 64-битная архитектура |
| `#[cfg(target_arch = "aarch64")]` | ARM 64-бит |
| `#[cfg(debug_assertions)]` | Режим debug (не release) |
| `#[cfg(feature = "serde")]` | Включена фича "serde" |
| `#[cfg(not(feature = "std"))]` | Без стандартной библиотеки |
| `#[cfg(all(feature = "serde", not(debug_assertions)))]` | Комбинация условий |
| `#[cfg(any(feature = "serde", feature = "json"))]` | Хотя бы одно условие |

---

## 39.5. Макрос `cfg!`

`cfg!` — это макрос, который проверяет условие на этапе компиляции и возвращает `bool`:

```rust
fn main() {
    if cfg!(target_os = "windows") {
        println!("Windows-specific behavior");
    } else {
        println!("Unix-like behavior");
    }

    // Для платформозависимого кода
    let path_separator = if cfg!(windows) { '\\' } else { '/' };
    println!("Path separator: {}", path_separator);
}
```

**Отличие от `#[cfg]`:** `cfg!` вычисляется во время компиляции, но код остаётся в бинарнике (условное выполнение). `#[cfg]` исключает код из бинарника.

---

## 39.6. Cargo Features

**Cargo features** — это механизм для включения опциональных возможностей библиотеки.

**Определение features в `Cargo.toml`:**

```toml
[package]
name = "my_lib"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1.0", optional = true }
tokio = { version = "1.0", optional = true }

[features]
default = ["serde"]          # Включается по умолчанию
full = ["serde", "tokio"]    # Включает всё
json = ["serde/derive"]      # Включает конкретную фичу зависимости
no_std = []                  # Для embedded (без std)
```

**Использование в коде:**

```rust
// src/lib.rs

#[cfg(feature = "serde")]
pub mod serialization;

#[cfg(feature = "tokio")]
pub mod async_runtime;

// Условный импорт
#[cfg(feature = "serde")]
use serde::{Serialize, Deserialize};

#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub struct Data {
    pub id: u32,
    pub value: String,
}

// Функция доступна только при включённой фиче
#[cfg(feature = "tokio")]
pub async fn process_data(data: &Data) -> Result<(), Box<dyn std::error::Error>> {
    // Асинхронная обработка
    Ok(())
}
```

---

## 39.7. Опциональные зависимости (Optional dependencies)

Зависимости могут быть опциональными и включаться только через features:

```toml
[features]
default = []
serde = ["dep:serde"]           # Включает зависимость serde
tokio = ["dep:tokio"]           # Включает зависимость tokio
full = ["serde", "tokio"]       # Включает всё
```

**В коде:**

```rust
#[cfg(feature = "serde")]
pub fn to_json<T: serde::Serialize>(value: &T) -> String {
    serde_json::to_string(value).unwrap()
}
```

---

## 39.8. Feature unification (объединение фич)

Если несколько зависимостей вашего проекта транзитивно опираются на **один и тот же** крейт, а требуемая версия позволяет Cargo разрешить их к единственному экземпляру этого крейта, то этот общий крейт будет собран с **объединённым** набором features — то есть со всеми features, которые запросил хотя бы один из его потребителей.

Рассмотрим корректный пример:

```toml
# my_app/Cargo.toml
[dependencies]
lib_a = "1.0"
lib_b = "1.0"
```

Предположим, что и `lib_a`, и `lib_b` внутри себя зависят от одного и того же крейта `common_dep`:

```toml
# lib_a/Cargo.toml
[dependencies]
common_dep = { version = "1.0", features = ["feature1"] }
```

```toml
# lib_b/Cargo.toml
[dependencies]
common_dep = { version = "1.0" }
```

Здесь `lib_b` сам по себе не запрашивает `feature1`. Но поскольку и `lib_a`, и `lib_b` используют одну и ту же (совместимую по версии) зависимость `common_dep`, Cargo разрешит её к единственному экземпляру в дереве зависимостей — и этот единственный экземпляр `common_dep` будет собран с `feature1`, потому что её запросил хотя бы один потребитель. В результате `feature1` окажется активной и для того кода `common_dep`, который использует `lib_b`, даже если сам `lib_b` об этой feature ничего не знает.

Важно: `lib_a` и `lib_b` при этом остаются полностью независимыми крейтами — feature `lib_a` не «передаётся» в `lib_b` напрямую. Объединяются features только **у общей зависимости** `common_dep`, а не между `lib_a` и `lib_b` как таковыми.

```text
my_app
  ├── lib_a ──┐
  │           ├──► common_dep (единственный экземпляр в дереве)
  └── lib_b ──┘         │
                         └── features: {feature1}  ← объединены из всех путей
```

**Последствия:**
- Если feature `common_dep` не является безопасной для включения «просто потому, что кто-то из потребителей её запросил» (например, она заметно меняет поведение по умолчанию), это может неожиданно повлиять на код, который её не запрашивал.
- Именно поэтому features должны быть **аддитивными** — они должны только добавлять функциональность, а не менять уже существующее поведение непредсказуемым для остальных потребителей образом.

---

## 39.9. Feature Resolver

Современный Cargo поддерживает несколько версий feature resolver'а, и версия зависит от edition.

Для workspace нужную версию можно явно указать:

```toml
[workspace]
resolver = "2"

members = [
    "app",
    "library",
]
```

Для пакета соответствующая настройка может находиться в `[package]`:

```toml
[package]
name = "my_library"
version = "0.1.0"
edition = "2024"
resolver = "3"
```

**Важно не путать resolver, ответственный за объединение features (о котором эта глава), с MSRV-aware resolver'ом, о котором мы говорили в главе про Cargo.** Это исторически последовательные, но разные по назначению обновления:

- **Resolver 2** стал стандартом для **Edition 2021** (введён как opt-in начиная с Rust 1.51) — он уменьшает нежелательное объединение features между некоторыми различными контекстами зависимостей (в частности, `build-dependencies`, `dev-dependencies` и target-specific зависимости).
- **Resolver 3** стал стандартом для **Edition 2024** (требует Rust 1.84+) — он расширяет resolver 2 тем же поведением при разрешении features, но дополнительно делает разрешение версий зависимостей чувствительным к `rust-version` (MSRV-aware), о чём подробно шла речь в главе о Cargo.

То есть Edition 2024 не «продолжает использовать resolver 2» — она переходит на resolver 3, который **включает в себя** всё поведение resolver 2 по объединению features (то, что описано в этой главе) и добавляет к нему MSRV-aware разрешение версий. Поэтому всё, что сказано ниже в этом разделе про уменьшение объединения features между `build-dependencies`/`dev-dependencies`/target-specific зависимостями, в равной мере относится и к resolver 2 (Edition 2021), и к resolver 3 (Edition 2024) — это поведение перешло из одного в другой без изменений, а не было заменено.

Однако **feature unification внутри одного конкретного разрешённого экземпляра зависимости сохраняется** в обеих версиях resolver'а.

Поэтому ни resolver 2, ни resolver 3 не превращают features в полностью независимые конфигурации для каждого потребителя одной и той же зависимости.

---

## 39.10. Platform-specific code (платформозависимый код)

```rust
// src/lib.rs

// Модуль только для Unix (Linux, macOS, BSD)
#[cfg(unix)]
mod unix_impl;

// Модуль только для Windows
#[cfg(windows)]
mod windows_impl;

// Модуль для всех, кроме Windows
#[cfg(not(windows))]
mod not_windows_impl;

// Функция с разной реализацией на разных платформах
pub fn get_os_name() -> &'static str {
    if cfg!(target_os = "windows") {
        "Windows"
    } else if cfg!(target_os = "linux") {
        "Linux"
    } else if cfg!(target_os = "macos") {
        "macOS"
    } else {
        "Unknown OS"
    }
}
```

---

## 39.11. Feature design — проектирование фич

**Хорошие практики:**

1. **Минимизируйте количество фич.** Каждая фича — это дополнительная сложность.
2. **Делайте фичи ортогональными.** Фичи не должны зависеть друг от друга.
3. **Документируйте фичи.** В `Cargo.toml` или в документации.

   ```toml
   [package]
   documentation = "https://docs.rs/my_lib"
   ```

4. **Используйте `default = []`** для библиотек (не включайте фичи по умолчанию).
5. **Проверяйте все комбинации фич** в CI.

---

## 39.12. Feature explosion — опасность разрастания

**Feature explosion** — это ситуация, когда количество фич в проекте становится слишком большим.

**Проблемы:**
1. **Сложность тестирования.** Все комбинации фич невозможно протестировать.
2. **Неожиданные взаимодействия.** Фичи могут конфликтовать.
3. **Сложность для пользователей.** Трудно понять, какие фичи нужны.

**Решение:**
1. **Группируйте фичи.** Вместо `feature1`, `feature2`, `feature3` → сделайте `full`.
2. **Используйте фичи только для опциональных возможностей.**
3. **Не делайте фичи для платформ** — используйте `cfg` для этого.
4. **Предпочитайте фичи по умолчанию** для удобства.

---

## 39.13. `cfg` и `feature` в тестах

```rust
#[cfg(test)]
mod tests {
    #[test]
    #[cfg(feature = "serde")]
    fn test_serialization() {
        // Тестируем только с serde
    }

    #[test]
    #[cfg(not(feature = "serde"))]
    fn test_without_serde() {
        // Тестируем без serde
    }
}
```

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Несуществующая фича

```rust
#[cfg(feature = "nonexistent")]
fn impossible_function() {}
```

Само по себе использование `#[cfg(feature = "nonexistent")]` **не обязательно** приводит к ошибке компиляции. Если такая feature не объявлена в `[features]`, современные версии Cargo/rustc обычно выдадут **предупреждение** через lint `unexpected_cfgs` — но не остановят сборку, если этот lint не настроен как `deny`. Функция `impossible_function` при этом просто никогда не попадёт ни в один вариант сборки, потому что условие `feature = "nonexistent"` никогда не станет истинным.

Главная идея эксперимента та же, что и раньше:

> `#[cfg(feature = "...")]` не включает feature. Feature должна быть объявлена в `[features]` и активирована явно — при сборке или через зависимость.

### Эксперимент 2: Конфликт фич

```toml
[features]
feature_a = []
feature_b = []
default = ["feature_a"]
```

Если обе фичи включены, может возникнуть конфликт, если они не ортогональны.

---

## Практика

### Задание 1

Добавьте в библиотеку фичу `logging`, которая включает вывод отладочных сообщений.

### Задание 2

Создайте функцию, которая компилируется по-разному на Windows и Linux.

### Задание 3

Сделайте зависимость `serde` опциональной. Напишите код, который работает только если `serde` включена.

### Задание 4

Добавьте фичу `full`, которая включает все опциональные зависимости.

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если в библиотеке нет фичи, но код пытается её использовать?

### Задание 6

🔨 **Эксперимент с компилятором.**

Что произойдёт, если две фичи конфликтуют? Как это можно решить?

---

## Главное из этой главы

После этой главы мы понимаем:

- **`#[cfg(...)]`** — условная компиляция для кода.
- **`cfg!(...)`** — проверка условий во время выполнения.
- **Cargo features** — опциональные возможности библиотек.
- **`[features]`** в `Cargo.toml` — определение фич.
- **Optional dependencies** — зависимости, включаемые через фичи.
- **Feature unification** — объединение фич из зависимостей.
- **Platform-specific code** — код для разных ОС и архитектур.
- **Feature explosion** — опасность разрастания фич.
- **Проектирование фич** — минимизация, ортогональность, документация.

**Самая важная идея:**

> Features и conditional compilation — это мощные инструменты для создания гибких библиотек и приложений. Они позволяют адаптировать код под разные платформы и окружения, но требуют осторожного проектирования. Чрезмерное количество фич делает код сложным для тестирования и поддержки. Простота и ясность — лучшие принципы при работе с features.