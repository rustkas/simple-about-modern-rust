# Глава 38. Cargo Workspaces

## 38.1. Что такое Workspace?

**Workspace** — это набор связанных Cargo-пакетов, которые управляются как единое целое. Каждый member workspace при этом остаётся отдельным package и обычно содержит собственный `Cargo.toml`.

Workspace особенно полезен, когда проект состоит из нескольких библиотек и приложений:

```text
my_workspace/

├── Cargo.toml
├── Cargo.lock
├── target/
│
├── app/
│   ├── Cargo.toml
│   └── src/main.rs
│
├── domain/
│   ├── Cargo.toml
│   └── src/lib.rs
│
└── infrastructure/
    ├── Cargo.toml
    └── src/lib.rs
```

По умолчанию workspace предоставляет:

- единый `Cargo.lock`;
- единую директорию `target/`;
- возможность выполнять Cargo-команды сразу для нескольких packages;
- централизованное управление зависимостями через `workspace.dependencies`;
- централизованное управление метаданными через `workspace.package`;
- единые профили сборки.

Все эти механизмы работают на уровне workspace, но сами packages остаются самостоятельными единицами: каждый имеет собственное имя, версию, зависимости и targets. ([Rust Documentation][1])

```text
                    Workspace
                        │
        ┌───────────────┼───────────────┐
        │               │               │
       app            domain      infrastructure
     binary            lib              lib
        │               │               │
        └───────────────┴───────────────┘
                 общий Cargo.lock
                 общий target/
```

---

## 38.2. Создание Workspace

Для проекта, состоящего только из отдельных members, удобно использовать **virtual workspace**.

### Шаг 1. Создаём каталог

```bash
mkdir my_workspace
cd my_workspace
```

### Шаг 2. Создаём корневой `Cargo.toml`

```toml
[workspace]

resolver = "3"

members = [
    "app",
    "domain",
    "infrastructure",
]
```

Здесь нет секции `[package]`. Поэтому это **virtual workspace**.

Для Edition 2024 следует использовать `resolver = "3"`. В virtual workspace Cargo не может вывести resolver из `edition`, потому что у самого корневого манифеста нет `[package]`. ([Rust Documentation][1])

### Шаг 3. Создаём packages

```bash
cargo new app --bin
cargo new domain --lib
cargo new infrastructure --lib
```

Современный Cargo при создании package внутри workspace может автоматически добавить его в `members`. Но для учебного материала лучше понимать, что фактическая конфигурация workspace находится в корневом `Cargo.toml`.

### Шаг 4. Собираем workspace

```bash
cargo build
```

### Шаг 5. Проверяем workspace

```bash
cargo metadata
```

`cargo metadata` позволяет получить машинно-читаемое описание packages, зависимостей и workspace. Это особенно полезно для IDE, CI и собственных инструментов автоматизации. ([Rust Documentation][3])

---

## 38.3. Структура корневого `Cargo.toml`

Корневой manifest может содержать не только `members`, но и общие настройки всего workspace:

```toml
[workspace]

resolver = "3"

members = [
    "crates/*",
    "tools/*",
]

exclude = [
    "legacy",
]

default-members = [
    "crates/app",
]

[workspace.package]

edition = "2024"
rust-version = "1.85"
license = "MIT"
repository = "https://github.com/example/my-project"

[workspace.dependencies]

serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
anyhow = "1.0"
thiserror = "2.0"

[profile.release]

lto = true
codegen-units = 1
```

### `members`

Определяет packages, входящие в workspace:

```toml
members = [
    "crates/*",
    "tools/*",
]
```

Можно использовать glob-шаблоны:

```toml
members = [
    "crates/*",
]
```

### `exclude`

Позволяет исключить каталог из workspace:

```toml
exclude = [
    "legacy",
]
```

Это особенно полезно, если внутри дерева проекта находится старый package или каталог, который не должен участвовать в основном workspace. ([Rust Documentation][1])

### `default-members`

Определяет, какие packages выбираются по умолчанию, когда команда выполняется из корня workspace:

```toml
default-members = [
    "crates/app",
]
```

Например:

```bash
cargo build
```

будет работать с `crates/app`, тогда как:

```bash
cargo build --workspace
```

явно собирает весь workspace.

Для больших проектов это позволяет сделать обычную разработку быстрой, не отказываясь от возможности собрать весь workspace. ([Rust Documentation][1])

### `workspace.package`

Эта секция позволяет централизованно задавать метаданные:

```toml
[workspace.package]

edition = "2024"
rust-version = "1.85"
license = "MIT"
repository = "https://github.com/example/my-project"
```

Member может наследовать их:

```toml
[package]

name = "domain"

version = "0.1.0"
edition.workspace = true
rust-version.workspace = true
license.workspace = true
repository.workspace = true
```

`workspace.package` поддерживает, среди прочего, `edition`, `version`, `rust-version`, `license`, `repository`, `description`, `keywords`, `categories` и `readme`. ([Rust Documentation][1])

---

## 38.4. Workspace Inheritance — наследование зависимостей

Cargo позволяет определить общие зависимости один раз:

```toml
# Workspace Cargo.toml

[workspace.dependencies]

serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
anyhow = "1.0"
```

После этого member может наследовать зависимость:

```toml
# domain/Cargo.toml

[dependencies]

serde = { workspace = true }
anyhow = { workspace = true }
```

Или в сокращённой форме:

```toml
[dependencies]

anyhow.workspace = true
```

### Дополнительные features

Features из member **добавляются** к features, определённым в workspace:

```toml
# Workspace Cargo.toml

[workspace.dependencies]

serde = { version = "1.0", features = ["derive"] }
```

```toml
# app/Cargo.toml

[dependencies]

serde = { workspace = true, features = ["rc"] }
```

В результате для `app` будут активны:

```text
derive
rc
```

Features не заменяют друг друга — они объединяются. ([Rust Documentation][2])

### Optional dependency

`workspace.dependencies` не может объявить зависимость как `optional`. Это должен сделать конкретный member:

```toml
# Workspace Cargo.toml

[workspace.dependencies]

serde = { version = "1.0", features = ["derive"] }
```

```toml
# app/Cargo.toml

[dependencies]

serde = { workspace = true, optional = true }

[features]

json = ["dep:serde"]
```

Теперь:

```bash
cargo build
```

не включает `serde`, а:

```bash
cargo build --features json
```

включает её.

Это важное правило: workspace централизует **общую конфигурацию зависимости**, но решение о том, является ли она optional, принадлежит конкретному package. ([Rust Documentation][2])

---

## 38.5. Архитектура с Workspace

Workspace удобно использовать для физического разделения архитектурных слоёв.

Например:

```text
workspace/

├── app/
│   └── src/main.rs
│
├── domain/
│   └── src/lib.rs
│
├── infrastructure/
│   └── src/lib.rs
│
└── shared/
    └── src/lib.rs
```

Зависимости могут выглядеть так:

```text
                 app
                /   \
               /     \
              ▼       ▼
          domain   infrastructure
              ▲       │
              │       │
              └───────┘

app ────────────────► shared
domain ─────────────► shared
infrastructure ────► shared
```

То есть:

```text
app → domain
app → infrastructure
infrastructure → domain

app → shared
domain → shared
infrastructure → shared
```

При этом:

```text
domain → app                 ❌
domain → infrastructure     ❌
shared → domain              ❌
shared → infrastructure     ❌
shared → app                 ❌
```

`shared` должен находиться ниже остальных слоёв и поэтому **не должен зависеть от них**.

Но ещё лучше не создавать огромный `shared`-crate без необходимости. Если в нём постепенно появляются бизнес-правила, database types или HTTP-код, это обычно признак того, что границы архитектуры начинают разрушаться.

---

## 38.6. Domain/Infrastructure Separation

Одна из сильных сторон разделения на crates — возможность выразить архитектурные зависимости непосредственно через `Cargo.toml`.

Пусть `domain` определяет бизнес-сущности и интерфейс репозитория.

### `domain/src/lib.rs`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);

impl UserId {
    pub fn new(value: u64) -> Self {
        Self(value)
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct User {
    pub id: UserId,
    pub name: String,
}

pub trait UserRepository {
    type Error;

    fn find_by_id(&self, id: UserId)
        -> Result<Option<User>, Self::Error>;

    fn save(&mut self, user: User)
        -> Result<(), Self::Error>;
}
```

Здесь нет PostgreSQL, SQLx, HTTP или другого инфраструктурного кода.

Domain говорит только:

> «Мне нужен объект, который умеет сохранять и находить `User`».

### `infrastructure/src/lib.rs`

Теперь infrastructure реализует этот интерфейс:

```rust
use domain::{User, UserId, UserRepository};

#[derive(Debug, Default)]
pub struct InMemoryUserRepository {
    users: Vec<User>,
}

impl InMemoryUserRepository {
    pub fn new() -> Self {
        Self::default()
    }
}

impl UserRepository for InMemoryUserRepository {
    type Error = std::convert::Infallible;

    fn find_by_id(
        &self,
        id: UserId,
    ) -> Result<Option<User>, Self::Error> {
        Ok(self.users.iter().find(|user| user.id == id).cloned())
    }

    fn save(
        &mut self,
        user: User,
    ) -> Result<(), Self::Error> {
        self.users.push(user);
        Ok(())
    }
}
```

Это уже полностью рабочая реализация. Позже `InMemoryUserRepository` можно заменить на:

```text
PostgresUserRepository
SqliteUserRepository
DynamoDbUserRepository
FileUserRepository
```

при этом `domain` менять не потребуется.

### `app/src/main.rs`

Приложение использует domain API и конкретную инфраструктурную реализацию:

```rust
use domain::{User, UserId, UserRepository};
use infrastructure::InMemoryUserRepository;

fn main() {
    let mut repository = InMemoryUserRepository::new();

    let user = User {
        id: UserId::new(1),
        name: String::from("Alice"),
    };

    repository.save(user).unwrap();

    let user = repository
        .find_by_id(UserId::new(1))
        .unwrap();

    println!("{user:?}");
}
```

Здесь особенно важно увидеть архитектурную границу:

```text
domain
  │
  │ defines
  ▼
UserRepository trait
  ▲
  │ implements
  │
infrastructure
  │
  │ used by
  ▼
app
```

Таким образом, зависимость инфраструктуры от domain является односторонней:

```text
infrastructure → domain
```

а не наоборот.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%2C+Eq%29%5D%0Apub+struct+UserId%28u64%29%3B%0Aimpl+UserId+%7B+pub+fn+new%28value%3A+u64%29+-%3E+Self+%7B+Self%28value%29+%7D+%7D%0A%23%5Bderive%28Debug%2C+Clone%2C+PartialEq%2C+Eq%29%5D%0Apub+struct+User+%7B+pub+id%3A+UserId%2C+pub+name%3A+String+%7D%0Apub+trait+UserRepository+%7B+type+Error%3B+fn+find_by_id%28%26self%2C+id%3A+UserId%29+-%3E+Result%3COption%3CUser%3E%2C+Self%3A%3AError%3E%3B+fn+save%28%26mut+self%2C+user%3A+User%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E%3B+%7D%0A%23%5Bderive%28Debug%2C+Default%29%5D%0Apub+struct+InMemoryUserRepository+%7B+users%3A+Vec%3CUser%3E+%7D%0Aimpl+InMemoryUserRepository+%7B+pub+fn+new%28%29+-%3E+Self+%7B+Self%3A%3Adefault%28%29+%7D+%7D%0Aimpl+UserRepository+for+InMemoryUserRepository+%7B+type+Error+%3D+std%3A%3Aconvert%3A%3AInfallible%3B+fn+find_by_id%28%26self%2C+id%3A+UserId%29+-%3E+Result%3COption%3CUser%3E%2C+Self%3A%3AError%3E+%7B+Ok%28self.users.iter%28%29.find%28%7Cu%7C+u.id+%3D%3D+id%29.cloned%28%29%29+%7D+fn+save%28%26mut+self%2C+user%3A+User%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E+%7B+self.users.push%28user%29%3B+Ok%28%28%29%29+%7D+%7D%0Afn+main%28%29+%7B+let+mut+repository+%3D+InMemoryUserRepository%3A%3Anew%28%29%3B+repository.save%28User+%7B+id%3A+UserId%3A%3Anew%281%29%2C+name%3A+%22Alice%22.into%28%29+%7D%29.unwrap%28%29%3B+println%21%28%22%7B%3F%7D%22%2C+repository.find_by_id%28UserId%3A%3Anew%281%29%29.unwrap%28%29%29%3B+%7D)

---

## 38.7. Работа с Workspace

Cargo позволяет явно выбирать, с какими members работать.

```bash
# Собрать workspace
cargo build --workspace

# Проверить весь workspace
cargo check --workspace

# Запустить тесты всего workspace
cargo test --workspace

# Собрать конкретный package
cargo build -p app

# Проверить конкретный package
cargo check -p domain

# Запустить binary package
cargo run -p app

# Запустить release-сборку конкретного package
cargo build --release -p app

# Собрать всё, кроме infrastructure
cargo build --workspace --exclude infrastructure
```

Флаг:

```bash
-p package_name
```

означает:

> работать только с указанным package.

Флаг:

```bash
--workspace
```

означает:

> работать со всеми members workspace.

А:

```bash
--exclude package_name
```

позволяет исключить package из операции; он используется вместе с `--workspace`. ([Rust Documentation][1])

---

## 38.8. Особенности Workspace

### Общий `Cargo.lock`

Packages workspace используют общий `Cargo.lock`, расположенный в корне workspace.

Это означает, что зависимости всего проекта разрешаются совместно:

```text
workspace/
│
├── Cargo.lock
│
├── app
├── domain
└── infrastructure
```

Это особенно важно для воспроизводимых сборок.

### Общая `target/`

По умолчанию все packages workspace используют одну директорию:

```text
workspace/
└── target/
```

а не:

```text
app/target/
domain/target/
infrastructure/target/
```

Это позволяет Cargo повторно использовать уже собранные артефакты между packages. ([Rust Documentation][4])

### Независимые версии

Каждый package сохраняет собственную версию:

```toml
# domain/Cargo.toml
[package]
name = "domain"
version = "0.3.0"
```

```toml
# infrastructure/Cargo.toml
[package]
name = "infrastructure"
version = "0.7.0"
```

Поэтому workspace не означает, что все packages обязаны иметь одну версию.

Однако для проектов с синхронным релизом можно централизовать версию:

```toml
# Workspace Cargo.toml

[workspace.package]

version = "1.0.0"
```

и в member:

```toml
[package]

name = "domain"
version.workspace = true
```

---

## 38.9. Публикация из Workspace

Каждый package workspace может публиковаться отдельно:

```bash
cargo publish -p domain
```

```bash
cargo publish -p infrastructure
```

или все packages:

```bash
cargo publish --workspace
```

Но здесь есть важный момент.

### Path dependency и публикация

Такой dependency:

```toml
[dependencies]

domain = { path = "../domain" }
```

подходит для локальной разработки, но **не подходит для публикации на crates.io**.

Если `infrastructure` должен использовать опубликованный `domain`, указываем обе координаты:

```toml
[dependencies]

domain = {
    path = "../domain",
    version = "0.1.0"
}
```

Во время локальной разработки Cargo использует:

```text
../domain
```

При публикации registry dependency использует:

```text
domain = "0.1.0"
```

`path` и `version` вместе — именно тот механизм, который позволяет разрабатывать packages внутри одного workspace и одновременно публиковать их как независимые crates. Cargo проверяет, что локальная версия соответствует указанному version requirement. ([Rust Documentation][2])

Поэтому порядок публикации имеет значение:

```text
domain
   │
   ▼
infrastructure
   │
   ▼
app
```

Если `infrastructure` зависит от опубликованного `domain`, сначала должна существовать соответствующая версия `domain` в registry.

---

## 38.10. Пример: структура реального проекта

Для достаточно крупного проекта разумной отправной точкой может быть следующая структура:

```text
my_project/

├── Cargo.toml
├── Cargo.lock
│
├── crates/
│   ├── app/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── main.rs
│   │
│   ├── domain/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs
│   │
│   ├── infrastructure/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs
│   │
│   └── shared/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs
│
├── tools/
│   └── migration/
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
│
├── examples/
│
└── scripts/
```

Корневой `Cargo.toml`:

```toml
[workspace]

resolver = "3"

members = [
    "crates/*",
    "tools/*",
]

default-members = [
    "crates/app",
]

[workspace.package]

edition = "2024"
license = "MIT"
repository = "https://github.com/example/my-project"

[workspace.dependencies]

anyhow = "1.0"
serde = { version = "1.0", features = ["derive"] }
thiserror = "2.0"
```

Теперь:

```bash
cargo run
```

запускает default member:

```text
crates/app
```

а:

```bash
cargo test --workspace
```

проверяет весь проект.

Такой подход особенно полезен, когда repository содержит не только production crates, но и внутренние инструменты, миграции, генераторы кода и CLI.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Циклическая зависимость

Создайте:

```text
workspace/
├── Cargo.toml
├── app/
└── domain/
```

`app/Cargo.toml`:

```toml
[dependencies]

domain = { path = "../domain" }
```

`domain/Cargo.toml`:

```toml
[dependencies]

app = { path = "../app" }
```

Получается:

```text
app → domain
▲     │
└─────┘
```

Cargo обнаружит цикл и откажется строить граф зависимостей.

Важно понимать: это не ограничение Rust-модулей. Это ограничение **графа package dependencies**.

---

### Эксперимент 2: Package существует, но не является default member

Используем:

```toml
[workspace]

resolver = "3"

members = [
    "app",
    "domain",
]

default-members = [
    "app",
]
```

Теперь:

```bash
cargo build
```

работает с `app`.

Но:

```bash
cargo build --workspace
```

соберёт:

```text
app
domain
```

А конкретный package можно выбрать:

```bash
cargo build -p domain
```

Это хороший способ понять разницу между:

```text
members
```

и:

```text
default-members
```

---

### Эксперимент 3: Две версии одной зависимости

Представим workspace:

```text
app
├── dependency-a
│   └── serde 1.x
│
└── dependency-b
    └── serde 2.x
```

Если требования к версиям несовместимы, Cargo не сможет свести их к одной версии.

Но если существуют две совместимые ветки версий, Cargo может включить **две версии одного package** в dependency graph.

Например, условно:

```text
app
├── library-a
│   └── some-lib 1.x
│
└── library-b
    └── some-lib 2.x
```

это может привести к:

```text
some-lib 1.x
some-lib 2.x
```

одновременно.

Это принципиально отличается от циклической зависимости: несколько версий dependency сами по себе не являются ошибкой. Cargo разрешает dependency graph в соответствии с version requirements. ([Rust Documentation][5])

---

## Практика

### Задание 1

Создайте virtual workspace:

```text
learning/
├── Cargo.toml
├── app/
├── domain/
└── utils/
```

Используйте:

```toml
[workspace]

resolver = "3"

members = [
    "app",
    "domain",
    "utils",
]
```

---

### Задание 2

В `domain` создайте:

```rust
pub struct User {
    pub id: u64,
    pub name: String,
}
```

В `utils` создайте функцию:

```rust
pub fn validate_email(email: &str) -> bool
```

В `app` используйте оба crates.

---

### Задание 3

Добавьте в workspace:

```toml
[workspace.dependencies]

serde = { version = "1.0", features = ["derive"] }
anyhow = "1.0"
```

Затем подключите их в members через:

```toml
[dependencies]

serde.workspace = true
anyhow.workspace = true
```

---

### Задание 4

Добавьте `workspace.package`:

```toml
[workspace.package]

edition = "2024"
license = "MIT"
rust-version = "1.85"
```

Затем заставьте все members наследовать эти значения.

---

### Задание 5

Добавьте:

```toml
default-members = [
    "app",
]
```

Проверьте разницу между:

```bash
cargo build
```

и:

```bash
cargo build --workspace
```

---

### Задание 6

Создайте циклическую зависимость:

```text
app → domain → app
```

Запустите:

```bash
cargo check
```

Изучите сообщение Cargo и объясните, почему такой dependency graph невозможно построить.

---

### Задание 7

🔨 **Эксперимент с публикацией.**

Создайте:

```text
domain
infrastructure
```

где:

```text
infrastructure → domain
```

Сначала используйте:

```toml
domain = { path = "../domain" }
```

Затем измените на:

```toml
domain = {
    path = "../domain",
    version = "0.1.0"
}
```

Проверьте:

```bash
cargo package -p infrastructure
```

и изучите разницу.

Cargo требует, чтобы publishable path dependency имела `version`, если она должна быть заменена registry dependency при публикации. ([Rust Documentation][2])

---

## Главное из этой главы

После этой главы мы понимаем:

- **Workspace** — единая среда управления несколькими packages.
- **`members`** — packages, входящие в workspace.
- **`default-members`** — packages, выбираемые по умолчанию из корня workspace.
- **`--workspace`** — явный выбор всех members.
- **`--exclude`** — исключение packages из workspace-команды.
- **`workspace.dependencies`** — централизованное описание зависимостей.
- **`workspace.package`** — централизованное описание метаданных packages.
- **`workspace = true`** — наследование настроек из workspace.
- **`resolver = "3"`** — актуальный resolver для virtual workspace с Edition 2024.
- **Общий `Cargo.lock`** — единый lockfile workspace.
- **Общий `target/`** — единый каталог артефактов.
- **Path dependencies** — удобный способ связывать crates внутри одного repository.
- **`path + version`** — правильный вариант для publishable workspace dependencies.
- **Архитектурные границы** — workspace позволяет выражать их непосредственно через зависимости между crates.
- **Публикация** — каждый package workspace может быть опубликован независимо.

**Самая важная идея:**

> Workspace — это не просто способ сложить несколько crates в один каталог. Это механизм управления dependency graph, сборкой и архитектурой большого Rust-проекта.
>
> Каждый crate сохраняет собственную ответственность и границы, а workspace предоставляет общий уровень координации. Хороший workspace делает архитектуру видимой прямо в `Cargo.toml`: если `domain` не зависит от `infrastructure`, это ограничение существует не только как договорённость команды — оно закреплено структурой dependency graph.

[1]: https://doc.rust-lang.org/cargo/reference/workspaces.html?highlight=workspace 'Workspaces - The Cargo Book'
[2]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html 'Specifying Dependencies - The Cargo Book'
[3]: https://doc.rust-lang.org/cargo/commands/cargo-metadata.html?highlight=links+manifest+key 'cargo metadata - The Cargo Book'
[4]: https://doc.rust-lang.org/stable/book/ch14-03-cargo-workspaces.html 'Cargo Workspaces - The Rust Programming Language'
[5]: https://doc.rust-lang.org/nightly/cargo/reference/resolver.html 'Dependency Resolution - The Cargo Book'
