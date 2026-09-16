# Глава 70. Designing Traits

Трейты — это сердце системы типов Rust. Они определяют, что умеют делать типы, и позволяют писать универсальный код.

Но проектирование трейтов — это не просто перечисление методов. Хороший трейт определяет **минимальный и понятный контракт**, который легко реализовать и удобно использовать. Плохой трейт заставляет реализации поддерживать ненужные возможности, усложняет использование и может надолго связать публичный API с неудачным архитектурным решением.

В этой главе мы разберёмся, как проектировать трейты, которые будут удобными, безопасными и масштабируемыми:

- как выбирать размер и ответственность трейта;
- когда использовать associated types, а когда generic parameters;
- как проектировать default methods;
- когда нужен `dyn Trait`;
- как создавать extension traits;
- как использовать blanket implementations;
- зачем нужны sealed traits;
- как работают coherence и orphan rule;
- как проектировать трейты для реальных библиотек.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 70.1. Принципы проектирования трейтов

Хороший трейт обычно обладает следующими свойствами:

1. **Маленький** — содержит только действительно связанные операции.
2. **Чёткий** — имя и методы имеют однозначную семантику.
3. **Минимальный** — не требует от реализации лишних возможностей.
4. **Предсказуемый** — методы ведут себя в соответствии с очевидным контрактом.
5. **Расширяемый** — дополнительные возможности можно выразить отдельными trait-ами или extension traits.
6. **Совместимый** — публичный trait учитывает последствия будущих изменений.

Например, следующий trait объединяет слишком много различных обязанностей:

```rust
trait DataProcessor {
    fn read(&self) -> Vec<u8>;
    fn process(&mut self, data: &mut [u8]);
    fn write(&self, data: &[u8]);
    fn validate(&self, data: &[u8]) -> bool;
}
```

Здесь смешаны чтение, обработка, запись и валидация.

Гораздо лучше разделить независимые возможности:

```rust
trait Readable {
    fn read(&self) -> Vec<u8>;
}

trait Processable {
    fn process(&mut self);
}

trait Writable {
    fn write(&mut self, data: &[u8]);
}

trait Validatable {
    fn validate(&self) -> bool;
}
```

Теперь конкретный тип может реализовать только те возможности, которые ему действительно нужны.

Например:

```rust
struct Document {
    text: String,
}

impl Readable for Document {
    fn read(&self) -> Vec<u8> {
        self.text.as_bytes().to_vec()
    }
}

impl Validatable for Document {
    fn validate(&self) -> bool {
        !self.text.trim().is_empty()
    }
}
```

`Document` не обязан становиться `Writable` или `Processable` только потому, что существует другой тип, который умеет это делать.

---

## 70.2. Single Responsibility — одна ответственность

Принцип единственной ответственности особенно полезен при проектировании публичных trait-ов.

Плохой вариант:

```rust
struct User {
    id: u64,
    name: String,
}

trait UserManager {
    fn create(&mut self, user: User);
    fn delete(&mut self, id: u64);
    fn find(&self, id: u64) -> Option<User>;
    fn update(&mut self, id: u64, user: User);

    fn authenticate(&self, username: &str, password: &str) -> bool;

    fn send_notification(&self, user: &User, message: &str);
}
```

Здесь объединены три разных области ответственности:

- хранение пользователей;
- аутентификация;
- уведомления.

Лучше разделить их:

```rust
struct User {
    id: u64,
    name: String,
}

trait UserRepository {
    fn create(&mut self, user: User);
    fn delete(&mut self, id: u64);
    fn find(&self, id: u64) -> Option<User>;
    fn update(&mut self, id: u64, user: User);
}

trait Authenticator {
    fn authenticate(&self, username: &str, password: &str) -> bool;
}

trait Notifier {
    fn send_notification(&self, user: &User, message: &str);
}
```

Теперь возможности можно комбинировать:

```rust
struct UserService<R, A, N> {
    repository: R,
    authenticator: A,
    notifier: N,
}

impl<R, A, N> UserService<R, A, N>
where
    R: UserRepository,
    A: Authenticator,
    N: Notifier,
{
    // Методы сервиса могут использовать только необходимые контракты.
}
```

Это один из важных принципов Rust:

> **Не заставляйте тип реализовывать больше поведения, чем требуется потребителю.**

Такой подход особенно полезен для тестирования. Например, вместо большого `UserManager` тесту можно передать небольшой mock, реализующий только `UserRepository`.

---

## 70.3. Associated Types vs Generic Parameters

Это одно из наиболее важных решений при проектировании trait API.

Рассмотрим associated type:

```rust
trait Container {
    type Item;

    fn get(&self) -> Option<&Self::Item>;
    fn insert(&mut self, item: Self::Item);
}

impl Container for Vec<i32> {
    type Item = i32;

    fn get(&self) -> Option<&Self::Item> {
        self.first()
    }

    fn insert(&mut self, item: Self::Item) {
        self.push(item);
    }
}
```

Для `Vec<i32>` существует одна реализация `Container`, и эта реализация определяет:

```rust
type Item = i32;
```

После этого `Container::Item` для данного типа фиксирован.

### Generic parameter

Теперь рассмотрим другой вариант:

```rust
trait ConvertInto<T> {
    fn convert(self) -> T;
}
```

Здесь `T` является частью самой реализации.

Один и тот же тип может иметь разные реализации для разных `T`, если правила coherence это позволяют.

Например, концептуально:

```rust
struct Value(i32);

trait ConvertInto<T> {
    fn convert(self) -> T;
}

impl ConvertInto<String> for Value {
    fn convert(self) -> String {
        self.0.to_string()
    }
}

impl ConvertInto<i64> for Value {
    fn convert(self) -> i64 {
        self.0 as i64
    }
}
```

У `Value` теперь существуют разные реализации `ConvertInto<T>`.

### Главное различие

Associated type:

```rust
trait IteratorLike {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

означает:

> Для данной реализации trait существует **один определяемый реализацией тип `Item`**.

Generic parameter:

```rust
trait ConvertInto<T> {
    fn convert(self) -> T;
}
```

означает:

> Для одного типа может существовать **несколько реализаций trait для разных `T`**.

Поэтому можно использовать следующее практическое правило:

| Ситуация                                                 | Предпочтение      |
| -------------------------------------------------------- | ----------------- |
| У реализации должен быть один логически связанный тип    | Associated type   |
| Один тип должен поддерживать разные варианты параметра   | Generic parameter |
| Тип является частью идентичности реализации              | Generic parameter |
| Тип является результатом/свойством конкретной реализации | Associated type   |

Именно поэтому стандартный `Iterator` использует:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

а не `Iterator<T>`.

---

## 70.4. Default Methods

Trait может содержать реализацию метода по умолчанию.

Это позволяет определить общий алгоритм один раз:

```rust
trait Summary {
    fn title(&self) -> &str;

    fn summary(&self) -> String {
        format!("Title: {}", self.title())
    }
}

struct Article {
    title: String,
}

impl Summary for Article {
    fn title(&self) -> &str {
        &self.title
    }
}

fn main() {
    let article = Article {
        title: "Rust Traits".to_string(),
    };

    println!("{}", article.summary());
}
```

Реализация `Article` обязана предоставить только `title()`. Метод `summary()` автоматически предоставляется trait-ом.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Summary+%7B%0A++++fn+title%28%26self%29+-%3E+%26str%3B%0A%0A++++fn+summary%28%26self%29+-%3E+String+%7B%0A++++++++format%21%28%22Title%3A+%7B%7D%22%2C+self.title%28%29%29%0A++++%7D%0A%7D%0A%0Astruct+Article+%7B%0A++++title%3A+String%2C%0A%7D%0A%0Aimpl+Summary+for+Article+%7B%0A++++fn+title%28%26self%29+-%3E+%26str+%7B%0A++++++++%26self.title%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+article+%3D+Article+%7B%0A++++++++title%3A+%22Rust+Traits%22.to_string%28%29%2C%0A++++%7D%3B%0A%0A++++println%21%28%22%7B%7D%22%2C+article.summary%28%29%29%3B%0A%7D)

Default methods особенно полезны, когда trait содержит несколько операций, из которых некоторые можно выразить через базовые.

Например:

```rust
trait Collection {
    fn len(&self) -> usize;

    fn is_empty(&self) -> bool {
        self.len() == 0
    }
}
```

Реализации должны предоставить только `len()`, а `is_empty()` получают автоматически.

Это также уменьшает дублирование кода между реализациями.

---

## 70.5. Dyn-compatible Traits

Иногда нужно работать не с конкретным типом, а с любым типом, реализующим trait:

```rust
fn print_summary(value: &dyn Summary) {
    println!("{}", value.summary());
}
```

Для этого trait должен быть совместим с использованием через `dyn Trait`.

В старой литературе это обычно называется **object safety**. В современной документации Rust используется термин **dyn compatibility**.

Простейший dyn-compatible trait:

```rust
trait Drawable {
    fn draw(&self);
}

struct Circle;

impl Drawable for Circle {
    fn draw(&self) {
        println!("Drawing circle");
    }
}

fn render(object: &dyn Drawable) {
    object.draw();
}

fn main() {
    let circle = Circle;
    render(&circle);
}
```

### Почему некоторые методы несовместимы с `dyn Trait`?

Рассмотрим generic-метод:

```rust
trait NotDynCompatible {
    fn process<T>(&self, value: T);
}
```

Компилятор не может создать обычный vtable-вызов для произвольного `T`, поэтому такой trait нельзя использовать как:

```rust
// let value: &dyn NotDynCompatible;
```

Однако generic-метод иногда можно оставить в trait, если явно сказать, что он не предназначен для вызова через `dyn Trait`:

```rust
trait Processor {
    fn name(&self) -> &str;

    fn process<T>(&self, value: T)
    where
        Self: Sized,
    {
        let _ = value;
        println!("Processor: {}", self.name());
    }
}
```

Теперь trait всё ещё можно использовать как `dyn Processor`, но метод `process()` через `dyn Processor` вызвать нельзя.

### Практическое правило

Если trait должен использоваться как `dyn Trait`, проверяйте его методы особенно внимательно.

Типичные ограничения:

- generic-методы требуют особого внимания;
- методы, возвращающие `Self`, обычно несовместимы с `dyn Trait`;
- методы, принимающие `Self` по значению, требуют ограничения `Self: Sized`;
- ассоциированные функции без `self` должны быть ограничены `Self: Sized`, если они не могут быть вызваны через trait object.

Например:

```rust
trait Factory {
    fn create() -> Self
    where
        Self: Sized;
}
```

Такой метод не мешает использовать trait как `dyn Factory`, потому что он явно исключён из dyn-вызовов.

---

## 70.6. Extension Traits

**Extension trait** позволяет добавить методы к типу, который мы не можем или не хотим изменять.

Например, можно добавить полезные методы к `str`:

```rust
trait StrExt {
    fn reversed(&self) -> String;
    fn is_palindrome(&self) -> bool;
}

impl StrExt for str {
    fn reversed(&self) -> String {
        self.chars().rev().collect()
    }

    fn is_palindrome(&self) -> bool {
        self == self.reversed()
    }
}

fn main() {
    let text = "racecar";

    println!("Reversed: {}", text.reversed());
    println!("Palindrome: {}", text.is_palindrome());
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+StrExt+%7B%0A++++fn+reversed%28%26self%29+-%3E+String%3B%0A++++fn+is_palindrome%28%26self%29+-%3E+bool%3B%0A%7D%0A%0Aimpl+StrExt+for+str+%7B%0A++++fn+reversed%28%26self%29+-%3E+String+%7B%0A++++++++self.chars%28%29.rev%28%29.collect%28%29%0A++++%7D%0A%0A++++fn+is_palindrome%28%26self%29+-%3E+bool+%7B%0A++++++++self+%3D%3D+self.reversed%28%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+text+%3D+%22racecar%22%3B%0A%0A++++println%21%28%22Reversed%3A+%7B%7D%22%2C+text.reversed%28%29%29%3B%0A++++println%21%28%22Palindrome%3A+%7B%7D%22%2C+text.is_palindrome%28%29%29%3B%0A%7D)

Важная деталь: extension trait должен быть **в области видимости** там, где вызывается его метод.

Например:

```rust
use crate::StrExt;
```

Если trait находится в другом модуле или крейте, его обычно необходимо импортировать.

Extension traits особенно полезны для:

- удобных методов над стандартными типами;
- адаптеров;
- специализированных операций;
- API, которые не хочется добавлять в основной trait.

---

## 70.7. Blanket Implementations

**Blanket implementation** — реализация trait для всех типов, удовлетворяющих определённым ограничениям.

Например:

```rust
trait Double {
    fn double(&self) -> Self;
}

impl<T> Double for T
where
    T: std::ops::Mul<Output = T> + Copy + From<u8>,
{
    fn double(&self) -> Self {
        *self * T::from(2)
    }
}

fn main() {
    let x = 5i32;
    let y = 3.14f64;

    println!("{}", x.double());
    println!("{}", y.double());
}
```

Здесь trait автоматически реализуется для любого `T`, который:

- реализует `Mul<Output = T>`;
- реализует `Copy`;
- может быть создан из `u8`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Double+%7B%0A++++fn+double%28%26self%29+-%3E+Self%3B%0A%7D%0A%0Aimpl%3CT%3E+Double+for+T%0Awhere%0A++++T%3A+std%3A%3Aops%3A%3AMul%3COutput+%3D+T%3E+%2B+Copy+%2B+From%3Cu8%3E%2C%0A%7B%0A++++fn+double%28%26self%29+-%3E+Self+%7B%0A++++++++%2Aself+%2A+T%3A%3Afrom%282%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+x+%3D+5i32%3B%0A++++let+y+%3D+3.14f64%3B%0A%0A++++println%21%28%22%7B%7D%22%2C+x.double%28%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+y.double%28%29%29%3B%0A%7D)

Однако blanket implementations требуют осторожности.

Например, если библиотека объявляет:

```rust
impl<T> MyTrait for T
where
    T: SomeOtherTrait,
{
    // ...
}
```

она фактически резервирует огромный набор потенциальных реализаций `MyTrait`.

Другой crate уже не сможет свободно предоставить конфликтующую реализацию.

Поэтому blanket implementation — мощный инструмент, но его следует рассматривать как **часть дизайна API**, а не просто как способ сократить код.

---

## 70.8. Sealed Traits

Иногда библиотека хочет предоставить trait пользователям, но не хочет разрешать им создавать собственные реализации.

Для этого используется паттерн **sealed trait**.

```rust
mod sealed {
    pub trait Sealed {}
}

pub trait MyTrait: sealed::Sealed {
    fn method(&self);
}

impl sealed::Sealed for i32 {}

impl MyTrait for i32 {
    fn method(&self) {
        println!("i32 implementation");
    }
}
```

Пользователь видит:

```rust
pub trait MyTrait: sealed::Sealed {
    fn method(&self);
}
```

но не может реализовать `sealed::Sealed` для своего типа, потому что trait `Sealed` недоступен для реализации за пределами нашего crate.

### Зачем это нужно?

Например, библиотека может гарантировать:

```rust
pub trait SupportedPlatform: sealed::Sealed {
    fn name(&self) -> &'static str;
}
```

и сама контролировать полный набор поддерживаемых платформ.

Sealed trait особенно полезен, когда:

- библиотека должна контролировать множество реализаций;
- корректность trait зависит от внутренних инвариантов;
- автор хочет иметь возможность добавлять методы в trait без необходимости поддерживать сторонние реализации;
- trait является частью закрытого протокола библиотеки.

При этом sealed trait — не механизм безопасности. Его цель — **контроль API и реализаций**, а не защита от вредоносного кода.

---

## 70.9. Trait Coherence

Rust должен гарантировать, что для конкретной комбинации trait и типа не возникает неоднозначности.

Это обеспечивается правилами **coherence**.

Одно из главных правил — **orphan rule**:

> Реализацию trait можно объявить, если trait или тип, для которого он реализуется, является локальным для текущего crate.

Например:

```rust
trait MyTrait {}

impl MyTrait for Vec<i32> {}
```

Это допустимо:

- `MyTrait` — наш локальный trait;
- `Vec<i32>` — внешний тип.

Обратная ситуация также допустима:

```rust
struct MyType;

impl std::fmt::Display for MyType {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "MyType")
    }
}
```

Здесь:

- `Display` — внешний trait;
- `MyType` — локальный тип.

Но следующее запрещено:

```rust
// ❌ Внешний trait + внешний тип

// impl std::fmt::Display for Vec<i32> {
//     ...
// }
```

Иначе разные crates могли бы независимо попытаться определить разные реализации `Display` для `Vec<i32>`.

### Newtype pattern

Если нам действительно нужна собственная реализация внешнего trait для внешнего типа, можно создать локальный wrapper:

```rust
struct MyVec(Vec<i32>);

impl std::fmt::Display for MyVec {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:?}", self.0)
    }
}

fn main() {
    let value = MyVec(vec![1, 2, 3]);

    println!("{}", value);
}
```

Теперь `MyVec` является нашим типом, поэтому мы можем реализовать для него `Display`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+MyVec%28Vec%3Ci32%3E%29%3B%0A%0Aimpl+std%3A%3Afmt%3A%3ADisplay+for+MyVec+%7B%0A++++fn+fmt%28%26self%2C+f%3A+%26mut+std%3A%3Afmt%3A%3AFormatter%3C%27_%3E%29+-%3E+std%3A%3Afmt%3A%3AResult+%7B%0A++++++++write%21%28f%2C+%22%7B%3A%3F%7D%22%2C+self.0%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+MyVec%28vec%21%5B1%2C+2%2C+3%5D%29%3B%0A++++println%21%28%22%7B%7D%22%2C+value%29%3B%0A%7D)

Newtype — один из фундаментальных приёмов Rust API design.

---

## 70.10. Supertraits и композиция контрактов

Иногда один trait логически зависит от другого.

Например:

```rust
use std::fmt::Display;

trait Identifiable {
    fn id(&self) -> u64;
}

trait Describable: Identifiable + Display {
    fn describe(&self) -> String {
        format!("{} #{}", self, self.id())
    }
}
```

Запись:

```rust
trait Describable: Identifiable + Display
```

означает:

> Любой тип, реализующий `Describable`, обязан также реализовать `Identifiable` и `Display`.

Например:

```rust
struct User {
    id: u64,
    name: String,
}

impl Identifiable for User {
    fn id(&self) -> u64 {
        self.id
    }
}

impl Display for User {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.name)
    }
}

impl Describable for User {}

fn main() {
    let user = User {
        id: 42,
        name: "Alice".to_string(),
    };

    println!("{}", user.describe());
}
```

Supertraits позволяют строить иерархию возможностей, но ими также не следует злоупотреблять.

Если trait требует пять других trait-ов только ради одного метода, API становится тяжёлым для реализации.

---

## 70.11. Практический пример: проектирование трейта для кеша

Рассмотрим реальную задачу: нам нужен универсальный cache API.

Начнём с минимального контракта:

```rust
trait Cache<K, V> {
    fn get(&mut self, key: &K) -> Option<&V>;
    fn set(&mut self, key: K, value: V);
    fn remove(&mut self, key: &K) -> Option<V>;
    fn clear(&mut self);

    fn contains(&mut self, key: &K) -> bool {
        self.get(key).is_some()
    }
}
```

Обратите внимание: здесь `get` принимает `&mut self`. Это позволяет конкретной реализации изменять внутреннюю статистику cache hit/miss.

Теперь создадим расширенный trait:

```rust
trait StatsCache<K, V>: Cache<K, V> {
    fn hits(&self) -> u64;
    fn misses(&self) -> u64;
}
```

Реализация:

```rust
use std::collections::HashMap;
use std::hash::Hash;

struct SimpleCache<K, V> {
    data: HashMap<K, V>,
    hits: u64,
    misses: u64,
}

impl<K, V> SimpleCache<K, V> {
    fn new() -> Self {
        Self {
            data: HashMap::new(),
            hits: 0,
            misses: 0,
        }
    }
}

impl<K, V> Cache<K, V> for SimpleCache<K, V>
where
    K: Hash + Eq,
{
    fn get(&mut self, key: &K) -> Option<&V> {
        if self.data.contains_key(key) {
            self.hits += 1;
        } else {
            self.misses += 1;
        }

        self.data.get(key)
    }

    fn set(&mut self, key: K, value: V) {
        self.data.insert(key, value);
    }

    fn remove(&mut self, key: &K) -> Option<V> {
        self.data.remove(key)
    }

    fn clear(&mut self) {
        self.data.clear();
    }
}

impl<K, V> StatsCache<K, V> for SimpleCache<K, V>
where
    K: Hash + Eq,
{
    fn hits(&self) -> u64 {
        self.hits
    }

    fn misses(&self) -> u64 {
        self.misses
    }
}

fn main() {
    let mut cache = SimpleCache::new();

    cache.set("language", "Rust");

    println!("{:?}", cache.get(&"language"));
    println!("{:?}", cache.get(&"missing"));

    println!("Hits: {}", cache.hits());
    println!("Misses: {}", cache.misses());
}
```

Здесь мы получили два уровня API:

```text
Cache
  │
  └── базовые операции

StatsCache
  │
  └── Cache + статистика
```

При этом конкретная реализация `SimpleCache` может предоставлять оба контракта.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Acollections%3A%3AHashMap%3B%0Ause+std%3A%3Ahash%3A%3AHash%3B%0A%0Atrait+Cache%3CK%2C+V%3E+%7B%0A++++fn+get%28%26mut+self%2C+key%3A+%26K%29+-%3E+Option%3C%26V%3E%3B%0A++++fn+set%28%26mut+self%2C+key%3A+K%2C+value%3A+V%29%3B%0A++++fn+remove%28%26mut+self%2C+key%3A+%26K%29+-%3E+Option%3CV%3E%3B%0A++++fn+clear%28%26mut+self%29%3B%0A%0A++++fn+contains%28%26mut+self%2C+key%3A+%26K%29+-%3E+bool+%7B%0A++++++++self.get%28key%29.is_some%28%29%0A++++%7D%0A%7D%0A%0Atrait+StatsCache%3CK%2C+V%3E%3A+Cache%3CK%2C+V%3E+%7B%0A++++fn+hits%28%26self%29+-%3E+u64%3B%0A++++fn+misses%28%26self%29+-%3E+u64%3B%0A%7D%0A%0Astruct+SimpleCache%3CK%2C+V%3E+%7B%0A++++data%3A+HashMap%3CK%2C+V%3E%2C%0A++++hits%3A+u64%2C%0A++++misses%3A+u64%2C%0A%7D%0A%0Aimpl%3CK%2C+V%3E+SimpleCache%3CK%2C+V%3E+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+data%3A+HashMap%3A%3Anew%28%29%2C+hits%3A+0%2C+misses%3A+0+%7D%0A++++%7D%0A%7D%0A%0Aimpl%3CK%2C+V%3E+Cache%3CK%2C+V%3E+for+SimpleCache%3CK%2C+V%3E%0Awhere%0A++++K%3A+Hash+%2B+Eq%2C%0A%7B%0A++++fn+get%28%26mut+self%2C+key%3A+%26K%29+-%3E+Option%3C%26V%3E+%7B%0A++++++++if+self.data.contains_key%28key%29+%7B+self.hits+%2B%3D+1%3B+%7D+else+%7B+self.misses+%2B%3D+1%3B+%7D%0A++++++++self.data.get%28key%29%0A++++%7D%0A%0A++++fn+set%28%26mut+self%2C+key%3A+K%2C+value%3A+V%29+%7B+self.data.insert%28key%2C+value%29%3B+%7D%0A%0A++++fn+remove%28%26mut+self%2C+key%3A+%26K%29+-%3E+Option%3CV%3E+%7B+self.data.remove%28key%29+%7D%0A%0A++++fn+clear%28%26mut+self%29+%7B+self.data.clear%28%29%3B+%7D%0A%7D%0A%0Aimpl%3CK%2C+V%3E+StatsCache%3CK%2C+V%3E+for+SimpleCache%3CK%2C+V%3E%0Awhere%0A++++K%3A+Hash+%2B+Eq%2C%0A%7B%0A++++fn+hits%28%26self%29+-%3E+u64+%7B+self.hits+%7D%0A++++fn+misses%28%26self%29+-%3E+u64+%7B+self.misses+%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+cache+%3D+SimpleCache%3A%3Anew%28%29%3B%0A++++cache.set%28%22language%22%2C+%22Rust%22%29%3B%0A++++println%21%28%22%7B%3F%7D%22%2C+cache.get%28%26%22language%22%29%29%3B%0A++++println%21%28%22%7B%3F%7D%22%2C+cache.get%28%26%22missing%22%29%29%3B%0A++++println%21%28%22Hits%3A+%7B%7D%22%2C+cache.hits%28%29%29%3B%0A++++println%21%28%22Misses%3A+%7B%7D%22%2C+cache.misses%28%29%29%3B%0A%7D)

---

## 70.12. Практическое правило: generic или `dyn Trait`?

При использовании trait возникает ещё один важный выбор.

Можно написать:

```rust
fn process<T: Processor>(processor: T) {
    processor.process();
}
```

или:

```rust
fn process(processor: &dyn Processor) {
    processor.process();
}
```

Первый вариант использует **статическую диспетчеризацию**. Конкретный тип известен компилятору.

Второй использует **динамическую диспетчеризацию** через `dyn Trait`.

Практически:

```text
T: Trait
    │
    └── конкретный тип известен компилятору
        └── static dispatch

dyn Trait
    │
    └── конкретный тип скрыт
        └── dynamic dispatch
```

Если нет необходимости хранить разные реализации trait через единый интерфейс, generic часто является более простым решением:

```rust
fn save<T: Writable>(value: &T) {
    value.write();
}
```

`dyn Trait` становится особенно полезен, когда необходимо работать с разными конкретными типами через один интерфейс:

```rust
let handlers: Vec<Box<dyn Handler>> = vec![
    Box::new(JsonHandler),
    Box::new(XmlHandler),
];
```

Таким образом, вопрос должен звучать не «что быстрее — generic или dyn Trait?», а:

> **Нужна ли мне абстракция над конкретным типом или мне необходимо скрыть конкретный тип во время выполнения?**

---

## 70.13. Итог: чек-лист дизайна трейта

Перед публикацией trait в API полезно пройти следующий чек-лист.

### Ответственность

1. **Одна ответственность** — trait представляет одну связанную концепцию.
2. **Маленький API** — нет методов «на всякий случай».
3. **Понятное имя** — имя отражает семантику, а не внутреннюю реализацию.

### Типы

4. **Associated type** — если для конкретной реализации существует один логически связанный тип.
5. **Generic parameter** — если один тип должен поддерживать несколько вариантов реализации.
6. **Supertraits** — если существование одного контракта действительно требует другого.

### Методы

7. **Default methods** — общий алгоритм не должен дублироваться во всех реализациях.
8. **Минимальный required API** — обязательными должны быть только фундаментальные операции.
9. **`Self: Sized`** — используйте, если отдельный метод не должен работать через `dyn Trait`.

### Расширение

10. **Extension traits** — для дополнительных удобных методов.
11. **Blanket implementations** — для систематического предоставления поведения типам, удовлетворяющим bounds.
12. **Sealed traits** — когда библиотека должна контролировать множество реализаций.

### Совместимость

13. **Coherence** — убедитесь, что реализация не создаёт конфликтов.
14. **Orphan rule** — помните, что внешний trait нельзя реализовать для внешнего типа.
15. **Публичные bounds** — не добавляйте требования, которые пользователю сложно выполнить без необходимости.

### `dyn Trait`

16. **Dyn compatibility** — если trait должен использоваться как `dyn Trait`, проверьте все его методы.
17. **Generic vs `dyn Trait`** — выбирайте исходя из архитектуры API, а не автоматически.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Нарушение coherence

Попробуйте раскомментировать:

```rust
// Внешний trait + внешний тип.

// impl std::fmt::Display for Vec<i32> {
//     fn fmt(
//         &self,
//         f: &mut std::fmt::Formatter<'_>
//     ) -> std::fmt::Result {
//         write!(f, "custom")
//     }
// }
```

Компилятор сообщит об ошибке, связанной с orphan rule.

Попробуйте исправить программу с помощью newtype:

```rust
struct MyVec(Vec<i32>);
```

и реализуйте `Display` уже для `MyVec`.

---

### Эксперимент 2: Generic-метод и `dyn Trait`

Создайте:

```rust
trait NotDynCompatible {
    fn process<T>(&self, value: T);
}
```

Затем попробуйте:

```rust
fn use_trait(value: &dyn NotDynCompatible) {
    // ...
}
```

Компилятор сообщит, что trait нельзя использовать как `dyn Trait`.

Теперь добавьте:

```rust
where
    Self: Sized
```

к методу:

```rust
trait Processor {
    fn process<T>(&self, value: T)
    where
        Self: Sized;
}
```

После этого сам trait может использоваться через `dyn Processor`, хотя `process()` нельзя вызвать через trait object.

---

### Эксперимент 3: Associated type

Создайте:

```rust
trait Container {
    type Item;

    fn get(&self) -> Self::Item;
}
```

Попробуйте реализовать его дважды для одного и того же типа:

```rust
struct MyContainer;

// Первая реализация
impl Container for MyContainer {
    type Item = i32;

    fn get(&self) -> i32 {
        42
    }
}

// Попытка второй реализации
// impl Container for MyContainer {
//     type Item = String;
//
//     fn get(&self) -> String {
//         "hello".to_string()
//     }
// }
```

Вторая реализация невозможна.

Это хорошо демонстрирует отличие associated type от generic parameter.

---

### Эксперимент 4: Generic parameter

Теперь измените trait:

```rust
trait ConvertInto<T> {
    fn convert(self) -> T;
}
```

Теперь один и тот же тип может иметь разные реализации для разных `T`:

```rust
struct Value(i32);

impl ConvertInto<String> for Value {
    fn convert(self) -> String {
        self.0.to_string()
    }
}

impl ConvertInto<i64> for Value {
    fn convert(self) -> i64 {
        self.0 as i64
    }
}
```

Сравните этот результат с предыдущим экспериментом.

---

## Практика

### Задание 1

Создайте trait `Shape`:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;
}
```

Реализуйте его для:

- `Circle`;
- `Rectangle`.

Создайте функцию:

```rust
fn print_shape(shape: &impl Shape)
```

которая выводит площадь и периметр.

---

### Задание 2

Создайте extension trait:

```rust
trait VecExt {
    // ...
}
```

Добавьте метод:

```rust
even()
```

который возвращает новый `Vec<i32>` только с чётными числами.

Ожидаемое использование:

```rust
let numbers = vec![1, 2, 3, 4, 5, 6];

let even = numbers.even();

assert_eq!(even, vec![2, 4, 6]);
```

---

### Задание 3

Создайте trait с associated type:

```rust
trait Parser {
    type Output;

    fn parse(&self, input: &str) -> Result<Self::Output, String>;
}
```

Создайте две реализации:

- `IntParser`, возвращающий `i32`;
- `BoolParser`, возвращающий `bool`.

---

### Задание 4

Создайте trait с generic parameter:

```rust
trait Converter<T> {
    fn convert(&self) -> T;
}
```

Создайте тип `Temperature` и реализуйте преобразование:

- в `f64`;
- в `String`.

Сравните этот пример с заданием 3.

---

### Задание 5

Создайте dyn-compatible trait:

```rust
trait Logger {
    fn log(&self, message: &str);
}
```

Реализуйте его для:

- `ConsoleLogger`;
- `FileLogger`.

Затем создайте:

```rust
let loggers: Vec<Box<dyn Logger>>;
```

и вызовите `log()` для всех элементов.

---

### Задание 6

Создайте trait:

```rust
trait Processor {
    fn name(&self) -> &str;

    fn process<T>(&self, value: T)
    where
        Self: Sized;
}
```

Проверьте:

1. Можно ли создать `&dyn Processor`?
2. Можно ли вызвать `name()` через `dyn Processor`?
3. Можно ли вызвать `process()` через `dyn Processor`?

Объясните результат.

---

### Задание 7

Создайте blanket implementation для trait:

```rust
trait Describe {
    fn describe(&self) -> String;
}
```

Реализуйте его автоматически для всех типов, которые реализуют `Display`.

Подсказка:

```rust
impl<T> Describe for T
where
    T: std::fmt::Display,
{
    // ...
}
```

---

### Задание 8

Создайте sealed trait `Transport`, который библиотека разрешает реализовать только для заранее определённых типов:

- `Http`;
- `Tcp`.

Попробуйте реализовать `Transport` для пользовательского типа из другой части программы и убедитесь, что это невозможно.

---

### Задание 9

🔨 **Эксперимент с компилятором.**

Попробуйте реализовать внешний trait для внешнего типа:

```rust
impl std::fmt::Display for Vec<String> {
    // ...
}
```

Объясните, почему Rust запрещает такую реализацию.

Затем исправьте программу с помощью newtype.

---

### Задание 10

🔨 **Эксперимент с компилятором.**

Создайте trait с generic-методом:

```rust
trait Handler {
    fn handle<T>(&self, value: T);
}
```

Попробуйте использовать:

```rust
Box<dyn Handler>
```

Затем добавьте:

```rust
where
    Self: Sized
```

и сравните результаты компиляции.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Трейты** — контракты поведения типов.
- **Маленькие трейты** — легче реализовать, тестировать и комбинировать.
- **Associated types** — подходят, когда для конкретной реализации существует один связанный тип.
- **Generic parameters** — позволяют одному типу иметь разные реализации для разных параметров.
- **Default methods** — позволяют реализовать общий алгоритм непосредственно в trait.
- **Supertraits** — позволяют строить составные контракты.
- **Dyn-compatible traits** — могут использоваться через `dyn Trait`.
- **`Self: Sized`** — позволяет исключить отдельные методы из dyn-интерфейса.
- **Extension traits** — позволяют добавлять методы существующим типам.
- **Blanket implementations** — позволяют автоматически реализовывать trait для большого класса типов.
- **Sealed traits** — позволяют библиотеке контролировать множество реализаций.
- **Coherence** — гарантирует однозначность trait implementations.
- **Orphan rule** — ограничивает реализацию внешних trait-ов для внешних типов.
- **Newtype** — позволяет создавать собственный тип поверх существующего и тем самым получать возможность реализовать для него нужные trait-ы.
- **Generic и `dyn Trait`** — это разные способы абстрагирования, предназначенные для разных архитектурных задач.

### Самая важная идея

> **Trait — это контракт, а не просто набор методов.**
>
> Хороший trait определяет минимальный набор возможностей, необходимый для конкретной абстракции. Он не требует от реализации лишнего, не смешивает независимые обязанности и оставляет пространство для расширения.
>
> При проектировании trait API нужно думать не только о том, **что тип умеет делать сейчас**, но и о том, **какие ограничения этот контракт накладывает на будущие реализации и пользователей библиотеки**.
>
> Используйте associated types для свойств конкретной реализации, generic parameters — когда нужны разные реализации для разных параметров, default methods — для общего поведения, extension traits — для расширения API, blanket implementations — для систематического предоставления возможностей, а sealed traits — когда множество допустимых реализаций должно контролироваться библиотекой.
>
> **Хорошо спроектированный trait делает правильное использование API естественным, а неправильное — невозможным или как минимум очевидно ошибочным.**
