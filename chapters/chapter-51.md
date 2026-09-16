# Глава 51. Clippy и Rustfmt

До этой главы мы писали код, который работает. Но в Rust есть ещё один важный аспект — **качество кода**. Как сделать код красивым, читаемым и соответствующим лучшим практикам? Как автоматически проверять стиль и находить потенциальные проблемы?

В экосистеме Rust есть два мощных инструмента для этого: **Rustfmt** (автоматическое форматирование) и **Clippy** (линтинг и анализ кода).

В этой главе мы научимся использовать эти инструменты для поддержания высокого качества кода и автоматической проверки в CI.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 51.1. Что такое Rustfmt?

**Rustfmt** — официальный инструмент форматирования Rust-кода. Он автоматически приводит исходный код к единому стилю, чтобы разработчики не тратили время на обсуждение отступов, переносов строк и других механических деталей оформления.

Rustfmt **не исправляет ошибки программы и не является линтером**. Его задача — форматирование.

```bash
# Отформатировать проект
cargo fmt

# Проверить форматирование, ничего не изменяя
cargo fmt --check

# Проверить форматирование конкретного пакета workspace
cargo fmt -p my_project -- --check
```

Например, исходный код:

```rust
fn add  ( a:i32,b:i32 )->i32{a+b}

fn main(){
let result=add(2,3);
println!("{result}");
}
```

после `cargo fmt` будет приведён к нормальному виду:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    let result = add(2, 3);
    println!("{result}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B%0A++++a+%2B+b%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+result+%3D+add%282%2C+3%29%3B%0A++++println%21%28%22%7Bresult%7D%22%29%3B%0A%7D)

**Практическое правило:** обычно не стоит вручную форматировать Rust-код. Лучше позволить Rustfmt сделать это автоматически.

---

## 51.2. Настройка Rustfmt

Настройки Rustfmt можно хранить в `rustfmt.toml` или `.rustfmt.toml`.

Например:

```toml
# rustfmt.toml

edition = "2024"
max_width = 100
hard_tabs = false
tab_spaces = 4
newline_style = "Unix"
```

Наиболее часто используемые параметры:

| Опция           | Назначение                                  |
| --------------- | ------------------------------------------- |
| `edition`       | Edition, используемый при форматировании    |
| `max_width`     | Предпочтительная максимальная ширина строки |
| `hard_tabs`     | Использовать ли символы табуляции           |
| `tab_spaces`    | Количество пробелов для отступа             |
| `newline_style` | Стиль окончания строк                       |

При этом **не следует без необходимости копировать большое количество настроек из чужого проекта**. Чем меньше собственных настроек, тем ближе проект к стандартному стилю Rust.

Edition проекта обычно уже задаётся в `Cargo.toml`:

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2024"
```

Поэтому отдельное указание `edition = "2024"` в `rustfmt.toml` часто не требуется.

---

## 51.3. Игнорирование форматирования

В редких случаях отдельный элемент можно исключить из форматирования с помощью `#[rustfmt::skip]`:

```rust
#[rustfmt::skip]
const LOOKUP_TABLE: &[(&str, i32)] = &[
    ("one", 1),
    ("two", 2),
    ("three", 3),
];
```

То же самое можно сделать для модуля:

```rust
#[rustfmt::skip]
mod generated_code {
    // Код, который не должен форматироваться Rustfmt.
}
```

Однако использовать `#[rustfmt::skip]` следует осторожно.

Если обычный исходный код постоянно требует исключений, это чаще всего сигнал, что лучше изменить структуру кода, а не отключать форматирование.

---

## 51.4. Что такое Clippy?

**Clippy** — набор линтов для Rust, который анализирует код и помогает находить распространённые ошибки, подозрительные конструкции, неоптимальные решения и неидиоматичный Rust.

В отличие от Rustfmt, Clippy **анализирует смысл и структуру кода**.

Запуск:

```bash
cargo clippy
```

Для проверки всех targets и features проекта:

```bash
cargo clippy --all-targets --all-features
```

Для более строгой проверки:

```bash
cargo clippy --all-targets --all-features -- -D warnings
```

Если Clippy отсутствует:

```bash
rustup component add clippy
```

При стандартной установке Rust через `rustup` Clippy обычно уже установлен. ([Rust Documentation][4])

---

## 51.5. Примеры предупреждений Clippy

Clippy может обнаруживать конструкции, которые технически работают, но записаны неоптимально или неидиоматично.

### Ненужное сравнение с `true`

```rust
fn is_valid(value: bool) -> bool {
    if value == true {
        true
    } else {
        false
    }
}
```

Clippy предложит упростить выражение.

Идиоматичный вариант:

```rust
fn is_valid(value: bool) -> bool {
    value
}
```

Ещё более простой вариант — вообще использовать сам `value`, если отдельная функция здесь не нужна.

---

### Ненужный `return`

```rust
fn add(a: i32, b: i32) -> i32 {
    return a + b;
}
```

В Rust последняя expression обычно используется без `return`:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B%0A++++a+%2B+b%0A%7D%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+add%282%2C+3%29%29%3B%0A%7D)

---

### Ненужный `clone`

Рассмотрим:

```rust
fn print_name(name: String) {
    println!("{name}");
}

fn main() {
    let name = String::from("Alice");

    print_name(name.clone());
}
```

После вызова `print_name` исходная строка больше не используется. `clone()` поэтому не нужен:

```rust
fn print_name(name: String) {
    println!("{name}");
}

fn main() {
    let name = String::from("Alice");

    print_name(name);
}
```

Clippy помогает замечать подобные ненужные операции.

---

### Ненужный `Vec`

Например:

```rust
fn sum(values: &[i32]) -> i32 {
    values.iter().sum()
}

fn main() {
    let values = vec![1, 2, 3];

    println!("{}", sum(&values));
}
```

Если значение нужно только для передачи в функцию, `Vec` может быть вообще не нужен:

```rust
fn sum(values: &[i32]) -> i32 {
    values.iter().sum()
}

fn main() {
    println!("{}", sum(&[1, 2, 3]));
}
```

Однако **не каждый `Vec` является `useless_vec`**. Если коллекция действительно нужна как изменяемый динамический массив, `Vec` является правильным выбором.

---

### Ненужный `Option::map`

Следующая конструкция:

```rust
fn process(value: Option<i32>) {
    value.map(|value| {
        println!("{value}");
    });
}
```

использует `map` ради побочного эффекта и игнорирует полученный `Option`.

Если задача — выполнить действие только при наличии значения, гораздо понятнее:

```rust
fn process(value: Option<i32>) {
    if let Some(value) = value {
        println!("{value}");
    }
}
```

Clippy помогает обнаруживать подобные случаи, когда API используется не по назначению.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+process%28value%3A+Option%3Ci32%3E%29+%7B%0A++++if+let+Some%28value%29+%3D+value+%7B%0A++++++++println%21%28%22%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++process%28Some%2842%29%29%3B%0A%7D)

---

## 51.6. Категории линтов Clippy

Clippy группирует линты по назначению:

| Категория             | Назначение                                         | По умолчанию |
| --------------------- | -------------------------------------------------- | ------------ |
| `clippy::correctness` | Код, который является неправильным или бесполезным | `deny`       |
| `clippy::suspicious`  | Подозрительные конструкции                         | `warn`       |
| `clippy::style`       | Более идиоматичный стиль                           | `warn`       |
| `clippy::complexity`  | Ненужная сложность                                 | `warn`       |
| `clippy::perf`        | Возможные проблемы производительности              | `warn`       |
| `clippy::pedantic`    | Более строгие рекомендации                         | `allow`      |
| `clippy::nursery`     | Экспериментальные/новые рекомендации               | `allow`      |

По умолчанию включённые линты объединены группой `clippy::all`. `clippy::pedantic` намеренно не включён по умолчанию, поскольку некоторые его рекомендации могут быть слишком строгими для конкретного проекта. ([Rust Documentation][1])

Например:

```bash
# Стандартная проверка
cargo clippy

# Дополнительно включить pedantic
cargo clippy -- -W clippy::pedantic

# Включить nursery
cargo clippy -- -W clippy::nursery

# Отключить конкретный lint
cargo clippy -- -A clippy::needless_return
```

---

## 51.7. Настройка Clippy

Clippy можно настраивать двумя разными способами:

1. **уровнем lint** — через `allow`, `warn`, `deny`;
2. **параметрами конкретных lint'ов** — через `clippy.toml`.

### Уровни lint

Например:

```rust
#![warn(clippy::pedantic)]

fn main() {
    println!("Hello");
}
```

Для отдельной функции:

```rust
#[allow(clippy::too_many_arguments)]
fn complex_function(
    a: i32,
    b: i32,
    c: i32,
    d: i32,
    e: i32,
    f: i32,
) {
    // ...
}
```

`allow` подавляет предупреждение, `warn` выдаёт предупреждение, а `deny` превращает lint в ошибку. ([Rust Documentation][2])

Например:

```rust
#![deny(clippy::unwrap_used)]

fn get_value() -> i32 {
    let value = Some(42);

    value.unwrap()
}
```

Здесь `unwrap()` становится ошибкой Clippy.

Для тестов проект может разрешить `unwrap()` отдельно через конфигурацию Clippy:

```toml
# clippy.toml

allow-unwrap-in-tests = true
```

Это действительно является параметром Clippy и означает, что `unwrap_used` не будет применяться к тестовым функциям и `#[cfg(test)]`-модулям. По умолчанию значение — `false`. ([Rust Documentation][5])

### Конфигурационный файл

Например:

```toml
# clippy.toml

allow-unwrap-in-tests = true
allow-print-in-tests = true
```

Здесь мы не включаем или выключаем `pedantic`. Уровень `pedantic` задаётся отдельно:

```bash
cargo clippy -- -W clippy::pedantic
```

Это важное различие: **`clippy.toml` не заменяет настройку уровней lint'ов**. Он предназначен для конфигурационных параметров тех lint'ов, которые их поддерживают. ([Rust Documentation][2])

---

## 51.8. Интеграция в CI

Для CI полезно разделить проверки на несколько независимых шагов:

```yaml
name: Rust

on:
  push:
  pull_request:

jobs:
  quality:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Run Clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

      - name: Run tests
        run: cargo test --all-targets --all-features
```

На GitHub-hosted runners Clippy уже доступен, поэтому отдельная установка через сторонний `actions-rs/toolchain` для такого сценария не требуется. Официальная документация Clippy также рекомендует использовать `-D warnings` в CI. ([Rust Documentation][3])

Если проект использует конкретную версию Rust, лучше явно фиксировать toolchain проекта, например через `rust-toolchain.toml`, чтобы локальная разработка и CI использовали один и тот же toolchain.

---

## 51.9. CI как quality gate

Типичный quality gate для Rust-проекта может выглядеть так:

```bash
# 1. Форматирование
cargo fmt --all -- --check

# 2. Clippy
cargo clippy --all-targets --all-features -- -D warnings

# 3. Проверка сборки
cargo check --all-targets --all-features

# 4. Тесты
cargo test --all-targets --all-features
```

`-D warnings` означает: **любое предупреждение становится ошибкой**.

Это особенно полезно в CI:

```bash
cargo clippy --all-targets --all-features -- -D warnings
```

Если разработчик добавил новый Clippy warning, CI завершится с ошибкой и такой код не попадёт в основную ветку.

Clippy рекомендует именно такой подход для CI. ([Rust Documentation][6])

Однако не следует механически запрещать абсолютно всё без понимания последствий. Иногда lint действительно неприменим к конкретному участку кода. В таком случае лучше сделать локальное исключение с объяснением:

```rust
#[allow(clippy::too_many_arguments)]
fn configure(
    host: &str,
    port: u16,
    timeout: u64,
    retries: u32,
    workers: usize,
    cache_size: usize,
) {
    // Здесь большое количество параметров является частью API.
}
```

Так исключение становится **явным и локальным**, а остальные проверки продолжают работать.

---

## 51.10. Лучшие практики

1. **Используйте Rustfmt как стандартный форматтер проекта.**

   ```bash
   cargo fmt
   ```

2. **Проверяйте форматирование в CI.**

   ```bash
   cargo fmt --all -- --check
   ```

3. **Запускайте Clippy регулярно.**

   ```bash
   cargo clippy --all-targets --all-features
   ```

4. **В CI превращайте предупреждения в ошибки.**

   ```bash
   cargo clippy --all-targets --all-features -- -D warnings
   ```

5. **Не включайте `clippy::pedantic` автоматически без необходимости.** Сначала посмотрите, насколько его рекомендации подходят вашему проекту.

6. **Не отключайте lint глобально без причины.** Предпочтительнее локальное исключение:

   ```rust
   #[allow(clippy::some_lint)]
   fn special_case() {
       // Обоснованное исключение.
   }
   ```

7. **Используйте `--all-targets --all-features` в CI**, если хотите проверять не только основной бинарник или библиотеку, но также тесты, examples, benches и дополнительные feature-комбинации.

8. **Используйте один и тот же toolchain локально и в CI.** Clippy должен соответствовать версии Rust, которой собирается проект. ([Rust Documentation][6])

Главная цель Clippy — не заставить разработчика механически исправить каждое предупреждение. Его задача — **обратить внимание разработчика на потенциальную проблему**. Решение всегда принимает разработчик.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Clippy и ненужный `return`

Создайте проект:

```bash
cargo new clippy-experiment
cd clippy-experiment
```

Замените `src/main.rs`:

```rust
fn add(a: i32, b: i32) -> i32 {
    return a + b;
}

fn main() {
    println!("{}", add(2, 3));
}
```

Запустите:

```bash
cargo clippy
```

Изучите предупреждение Clippy и его предложение.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B%0A++++return+a+%2B+b%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+add%282%2C+3%29%29%3B%0A%7D)

### Эксперимент 2: `-D warnings`

Возьмите код с предупреждением:

```rust
fn add(a: i32, b: i32) -> i32 {
    return a + b;
}

fn main() {
    println!("{}", add(2, 3));
}
```

Запустите:

```bash
cargo clippy -- -D warnings
```

Теперь предупреждение рассматривается как ошибка, поэтому Clippy завершает проверку с ненулевым статусом.

Это именно тот механизм, который делает Clippy эффективным **quality gate в CI**.

### Эксперимент 3: локальное отключение lint

```rust
#[allow(clippy::needless_return)]
fn add(a: i32, b: i32) -> i32 {
    return a + b;
}

fn main() {
    println!("{}", add(2, 3));
}
```

Теперь конкретный lint отключён только для этой функции.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Ballow%28clippy%3A%3Aneedless_return%29%5D%0Afn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B%0A++++return+a+%2B+b%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+add%282%2C+3%29%29%3B%0A%7D)

---

## Практика

### Задание 1

Создайте небольшую программу с намеренно неформатированным кодом. Запустите:

```bash
cargo fmt
```

Сравните исходный и отформатированный код.

Затем проверьте:

```bash
cargo fmt --check
```

---

### Задание 2

Создайте проект с несколькими конструкциями, которые может улучшить Clippy.

Запустите:

```bash
cargo clippy
```

Для каждой рекомендации:

1. прочитайте сообщение Clippy;
2. исправьте код;
3. повторно запустите Clippy;
4. убедитесь, что предупреждение исчезло.

---

### Задание 3

Настройте Rustfmt для проекта:

```toml
# rustfmt.toml

max_width = 80
tab_spaces = 2
hard_tabs = false
```

Запустите:

```bash
cargo fmt
```

Проверьте результат.

---

### Задание 4

Создайте `clippy.toml` и настройте разрешение `unwrap()` в тестах:

```toml
allow-unwrap-in-tests = true
```

После этого сравните поведение Clippy для production-кода и `#[cfg(test)]`-кода.

---

### Задание 5

Настройте GitHub Actions, который выполняет:

```bash
cargo fmt --all -- --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-targets --all-features
```

Сделайте так, чтобы Pull Request не проходил CI при наличии предупреждения Clippy.

---

### Задание 6

🔨 **Эксперимент с компилятором.**

Создайте функцию с намеренно неоптимальной конструкцией, которую обнаруживает Clippy.

Запустите:

```bash
cargo clippy
```

Затем:

```bash
cargo clippy -- -D warnings
```

Объясните, почему второй запуск завершает CI с ошибкой.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Rustfmt** — автоматическое форматирование Rust-кода.
- **`cargo fmt`** — форматирование проекта.
- **`cargo fmt --check`** — проверка форматирования без изменения файлов.
- **Clippy** — набор линтов для поиска проблем и улучшения идиоматичности кода.
- **`cargo clippy`** — запуск Clippy.
- **`clippy::all`** — стандартный набор включённых Clippy lint'ов.
- **`clippy::pedantic`** — более строгий набор рекомендаций, не включённый по умолчанию. ([Rust Documentation][1])
- **`allow` / `warn` / `deny`** — способы управления уровнем конкретных lint'ов.
- **`clippy.toml`** — конфигурация параметров поддерживаемых Clippy lint'ов. ([Rust Documentation][5])
- **`-D warnings`** — превращение предупреждений в ошибки.
- **CI quality gate** — автоматическая проверка качества каждого изменения.
- **`--all-targets --all-features`** — более полная проверка проекта.

**Самая важная идея:**

> Rustfmt отвечает за единообразное оформление кода, а Clippy — за его качество и идиоматичность. Вместе они превращают субъективные требования к стилю и многие проверки качества в автоматический процесс. Локально разработчик получает быстрые подсказки, а CI превращает согласованные правила проекта в обязательный quality gate.

[1]: https://doc.rust-lang.org/clippy/ 'Introduction - Clippy Documentation'
[2]: https://doc.rust-lang.org/nightly/clippy/configuration.html 'Configuration - Clippy Documentation'
[3]: https://doc.rust-lang.org/clippy/continuous_integration/github_actions.html 'GitHub Actions - Clippy Documentation'
[4]: https://doc.rust-lang.org/stable/clippy/installation.html 'Installation - Clippy Documentation'
[5]: https://doc.rust-lang.org/clippy/lint_configuration.html 'Lint Configuration - Clippy Documentation'
[6]: https://doc.rust-lang.org/clippy/continuous_integration/index.html 'Continuous Integration - Clippy Documentation'
