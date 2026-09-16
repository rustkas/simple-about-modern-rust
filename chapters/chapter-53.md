# Часть XIII. Rust для реальных приложений

# Глава 53. Работа с файлами

Работа с файловой системой — одна из самых частых задач в реальных приложениях. Чтение конфигураций, запись логов, обработка файлов пользователей — всё это требует понимания того, как Rust взаимодействует с файловой системой.

В этой главе мы научимся читать и записывать файлы, работать с путями, создавать директории и обрабатывать ошибки файловой системы.

Все примеры этой главы используют **Rust Edition 2024** и предполагают наличие файловой системы (не все примеры работают в Rust Playground).

---

## 53.1. `std::fs` — работа с файловой системой

Модуль `std::fs` предоставляет основные операции для работы с файловой системой:

| Функция              | Описание                                     |
| -------------------- | -------------------------------------------- |
| `fs::read`           | Читает файл целиком в `Vec<u8>`              |
| `fs::read_to_string` | Читает файл целиком в `String`               |
| `fs::write`          | Создаёт или полностью перезаписывает файл    |
| `fs::copy`           | Копирует файл                                |
| `fs::rename`         | Переименовывает или перемещает файл          |
| `fs::remove_file`    | Удаляет файл                                 |
| `fs::create_dir`     | Создаёт одну директорию                      |
| `fs::create_dir_all` | Создаёт директорию и отсутствующих родителей |
| `fs::read_dir`       | Читает содержимое директории                 |
| `fs::metadata`       | Получает метаданные файла или директории     |
| `fs::canonicalize`   | Получает канонический абсолютный путь        |

Практически все операции файловой системы возвращают `Result`. Это важно: работа с файловой системой может завершиться ошибкой из-за отсутствующего файла, недостаточных прав, занятого ресурса, неверного пути и множества других причин.

Например:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::write("example.txt", "Hello, Rust!")?;

    let content = fs::read_to_string("example.txt")?;

    println!("{content}");

    fs::remove_file("example.txt")?;

    Ok(())
}
```

Здесь `?` передаёт ошибку вызывающему коду, если любая операция файловой системы завершится неудачно.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++fs%3A%3Awrite%28%22example.txt%22%2C+%22Hello%2C+Rust%21%22%29%3F%3B%0A++++let+content+%3D+fs%3A%3Aread_to_string%28%22example.txt%22%29%3F%3B%0A++++println%21%28%22%7Bcontent%7D%22%29%3B%0A++++fs%3A%3Aremove_file%28%22example.txt%22%29%3F%3B%0A++++Ok%28%28%29%29%0A%7D)

---

## 53.2. Чтение файлов

### Чтение файла целиком

Если файл небольшой и его содержимое удобно держать в памяти, можно использовать `fs::read_to_string`:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::write("example.txt", "Hello, Rust!\nThis is a file.")?;

    let content = fs::read_to_string("example.txt")?;

    println!("File content:\n{content}");

    fs::remove_file("example.txt")?;

    Ok(())
}
```

`read_to_string` возвращает `Result<String, std::io::Error>`.

Важно: метод предполагает, что содержимое является корректным UTF-8. Если файл содержит произвольные байты, используйте `fs::read`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++fs%3A%3Awrite%28%22example.txt%22%2C+%22Hello%2C+Rust%21%5CnThis+is+a+file.%22%29%3F%3B%0A++++let+content+%3D+fs%3A%3Aread_to_string%28%22example.txt%22%29%3F%3B%0A++++println%21%28%22File+content%3A%5Cn%7Bcontent%7D%22%29%3B%0A++++fs%3A%3Aremove_file%28%22example.txt%22%29%3F%3B%0A++++Ok%28%28%29%29%0A%7D)

### Чтение как байтов

Для бинарных данных используется `fs::read`:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::write("data.bin", [0, 1, 2, 3, 255])?;

    let bytes = fs::read("data.bin")?;

    println!("File size: {} bytes", bytes.len());
    println!("Bytes: {bytes:?}");

    fs::remove_file("data.bin")?;

    Ok(())
}
```

`fs::read` возвращает `Vec<u8>`, поэтому его можно использовать для изображений, архивов, бинарных форматов и других данных, которые не являются текстом.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++fs%3A%3Awrite%28%22data.bin%22%2C+%5B0%2C+1%2C+2%2C+3%2C+255%5D%29%3F%3B%0A++++let+bytes+%3D+fs%3A%3Aread%28%22data.bin%22%29%3F%3B%0A++++println%21%28%22File+size%3A+%7B%7D+bytes%22%2C+bytes.len%28%29%29%3B%0A++++println%21%28%22Bytes%3A+%7Bbytes%3A%3F%7D%22%29%3B%0A++++fs%3A%3Aremove_file%28%22data.bin%22%29%3F%3B%0A++++Ok%28%28%29%29%0A%7D)

### Обработка ошибок

Не стоит предполагать, что файл существует:

```rust
use std::fs;
use std::io::ErrorKind;

fn main() {
    match fs::read_to_string("config.toml") {
        Ok(content) => println!("Config:\n{content}"),

        Err(error) if error.kind() == ErrorKind::NotFound => {
            eprintln!("Configuration file was not found");
        }

        Err(error) => {
            eprintln!("Failed to read configuration: {error}");
        }
    }
}
```

Такой подход позволяет различать ожидаемые ошибки и действительно неожиданные ситуации.

---

## 53.3. Запись файлов

### Запись целиком

`fs::write` создаёт файл, если его нет, и **полностью заменяет его содержимое**, если файл уже существует:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::write("output.txt", "Hello, world!")?;

    let content = fs::read_to_string("output.txt")?;
    println!("{content}");

    fs::remove_file("output.txt")?;

    Ok(())
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++fs%3A%3Awrite%28%22output.txt%22%2C+%22Hello%2C+world%21%22%29%3F%3B%0A++++let+content+%3D+fs%3A%3Aread_to_string%28%22output.txt%22%29%3F%3B%0A++++println%21%28%22%7Bcontent%7D%22%29%3B%0A++++fs%3A%3Aremove_file%28%22output.txt%22%29%3F%3B%0A++++Ok%28%28%29%29%0A%7D)

### Запись нескольких строк

Для последовательной записи удобно использовать `File` вместе с `Write`:

```rust
use std::fs::File;
use std::io::Write;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut file = File::create("log.txt")?;

    writeln!(file, "Log entry 1: Started")?;
    writeln!(file, "Log entry 2: Processing")?;
    writeln!(file, "Log entry 3: Done")?;

    Ok(())
}
```

`File::create` создаёт новый файл или **обнуляет существующий**.

### Дозапись в конец файла

Если необходимо сохранить существующее содержимое и добавлять новые данные в конец, используется `OpenOptions`:

```rust
use std::fs::OpenOptions;
use std::io::Write;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut file = OpenOptions::new()
        .append(true)
        .create(true)
        .open("log.txt")?;

    writeln!(file, "New log entry")?;

    Ok(())
}
```

Здесь:

- `append(true)` — запись производится в конец файла;
- `create(true)` — файл создаётся, если его ещё нет.

---

## 53.4. Работа с путями: `Path` и `PathBuf`

Для работы с путями Rust предоставляет два основных типа:

| Тип       | Назначение                |
| --------- | ------------------------- |
| `Path`    | Заимствованный путь       |
| `PathBuf` | Владеющий изменяемый путь |

Обычно `Path` используется, когда функции достаточно прочитать путь, а `PathBuf` — когда путь необходимо построить или изменить.

```rust
use std::path::{Path, PathBuf};

fn main() {
    let path = Path::new("data/input.txt");

    let mut path_buf = PathBuf::from("data");
    path_buf.push("input.txt");

    println!("Path: {}", path.display());
    println!("PathBuf: {}", path_buf.display());

    println!("Equal: {}", path == path_buf);
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apath%3A%3A%7BPath%2C+PathBuf%7D%3B%0A%0Afn+main%28%29+%7B%0A++++let+path+%3D+Path%3A%3Anew%28%22data%2Finput.txt%22%29%3B%0A++++let+mut+path_buf+%3D+PathBuf%3A%3Afrom%28%22data%22%29%3B%0A++++path_buf.push%28%22input.txt%22%29%3B%0A++++println%21%28%22Path%3A+%7B%7D%22%2C+path.display%28%29%29%3B%0A++++println%21%28%22PathBuf%3A+%7B%7D%22%2C+path_buf.display%28%29%29%3B%0A++++println%21%28%22Equal%3A+%7B%7D%22%2C+path+%3D%3D+path_buf%29%3B%0A%7D)

### Основные методы `Path`

```rust
use std::path::Path;

fn main() {
    let path = Path::new("data/docs/readme.txt");

    println!("File name: {:?}", path.file_name());
    println!("Extension: {:?}", path.extension());
    println!("Parent: {:?}", path.parent());

    println!("Is absolute: {}", path.is_absolute());
    println!("Is relative: {}", path.is_relative());

    let new_path = path.join("archive");
    println!("Joined path: {}", new_path.display());
}
```

Важно понимать, что `Path` не обязательно представляет существующий файл. Это всего лишь представление пути.

Методы `exists()`, `is_file()` и `is_dir()` удобны для простых проверок, но они скрывают причину ошибки. Если приложению необходимо различать, например, «файл отсутствует» и «нет прав на доступ», лучше использовать `fs::metadata()` и анализировать `Result`.

---

## 53.5. Буферизированный ввод-вывод

При последовательной работе с большими файлами часто используют `BufReader` и `BufWriter`.

```rust
use std::fs::File;
use std::io::{BufRead, BufReader, BufWriter, Write};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    {
        let file = File::create("input.txt")?;
        let mut writer = BufWriter::new(file);

        for i in 0..1_000 {
            writeln!(writer, "Line {i}")?;
        }
    }

    let file = File::open("input.txt")?;
    let reader = BufReader::new(file);

    let mut output = BufWriter::new(File::create("output.txt")?);

    for line in reader.lines() {
        let line = line?;
        writeln!(output, "{line}")?;
    }

    output.flush()?;

    std::fs::remove_file("input.txt")?;
    std::fs::remove_file("output.txt")?;

    Ok(())
}
```

`BufReader` позволяет читать поток небольшими порциями, а `BufWriter` уменьшает количество операций записи.

Основные преимущества:

- меньше системных вызовов;
- удобная построчная обработка;
- отсутствие необходимости загружать весь файл в память;
- особенно полезно для больших файлов.

При этом буферизация **не является универсальной гарантией ускорения**. Для конкретной задачи производительность следует измерять.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3A%3AFile%3B%0Ause+std%3A%3Aio%3A%3A%7BBufRead%2C+BufReader%2C+BufWriter%2C+Write%7D%3B%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++%7B%0A++++++++let+file+%3D+File%3A%3Acreate%28%22input.txt%22%29%3F%3B%0A++++++++let+mut+writer+%3D+BufWriter%3A%3Anew%28file%29%3B%0A++++++++for+i+in+0..1000+%7B%0A++++++++++++writeln%21%28writer%2C+%22Line+%7Bi%7D%22%29%3F%3B%0A++++++++%7D%0A++++%7D%0A%0A++++let+file+%3D+File%3A%3Aopen%28%22input.txt%22%29%3F%3B%0A++++let+reader+%3D+BufReader%3A%3Anew%28file%29%3B%0A++++let+mut+output+%3D+BufWriter%3A%3Anew%28File%3A%3Acreate%28%22output.txt%22%29%3F%29%3B%0A%0A++++for+line+in+reader.lines%28%29+%7B%0A++++++++writeln%21%28output%2C+%22%7B%7D%22%2C+line%3F%29%3F%3B%0A++++%7D%0A%0A++++output.flush%28%29%3F%3B%0A++++std%3A%3Afs%3A%3Aremove_file%28%22input.txt%22%29%3F%3B%0A++++std%3A%3Afs%3A%3Aremove_file%28%22output.txt%22%29%3F%3B%0A%0A++++Ok%28%28%29%29%0A%7D)

---

## 53.6. Работа с директориями

### Чтение содержимого директории

`fs::read_dir` возвращает итератор по элементам директории:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::create_dir_all("example_dir/subdir")?;
    fs::write("example_dir/file.txt", "Hello")?;

    for entry in fs::read_dir("example_dir")? {
        let entry = entry?;
        let path = entry.path();
        let metadata = entry.metadata()?;

        if metadata.is_dir() {
            println!("DIR  {}", path.display());
        } else if metadata.is_file() {
            println!("FILE {} ({} bytes)", path.display(), metadata.len());
        }
    }

    fs::remove_dir_all("example_dir")?;

    Ok(())
}
```

Порядок элементов, возвращаемых `read_dir`, **не следует считать определённым**. Если приложению нужен конкретный порядок, результаты следует собрать и отсортировать самостоятельно.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++fs%3A%3Acreate_dir_all%28%22example_dir%2Fsubdir%22%29%3F%3B%0A++++fs%3A%3Awrite%28%22example_dir%2Ffile.txt%22%2C+%22Hello%22%29%3F%3B%0A%0A++++for+entry+in+fs%3A%3Aread_dir%28%22example_dir%22%29%3F+%7B%0A++++++++let+entry+%3D+entry%3F%3B%0A++++++++let+path+%3D+entry.path%28%29%3B%0A++++++++let+metadata+%3D+entry.metadata%28%29%3F%3B%0A%0A++++++++if+metadata.is_dir%28%29+%7B%0A++++++++++++println%21%28%22DIR++%7B%7D%22%2C+path.display%28%29%29%3B%0A++++++++%7D+else+if+metadata.is_file%28%29+%7B%0A++++++++++++println%21%28%22FILE+%7B%7D+%28%7B%7D+bytes%29%22%2C+path.display%28%29%2C+metadata.len%28%29%29%3B%0A++++++++%7D%0A++++%7D%0A%0A++++fs%3A%3Aremove_dir_all%28%22example_dir%22%29%3F%3B%0A++++Ok%28%28%29%29%0A%7D)

### Создание директорий

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::create_dir("data")?;
    fs::create_dir_all("data/2026/08")?;

    println!("Directories created");

    fs::remove_dir_all("data")?;

    Ok(())
}
```

`create_dir` создаёт только одну директорию. Если родительских директорий ещё нет, операция завершится ошибкой.

`create_dir_all` создаёт всю необходимую цепочку директорий.

---

## 53.7. Метаданные файлов

Для получения метаданных используется `fs::metadata`:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::write("example.txt", "Hello, Rust!")?;

    let metadata = fs::metadata("example.txt")?;

    println!("File size: {} bytes", metadata.len());
    println!("Is file: {}", metadata.is_file());
    println!("Is dir: {}", metadata.is_dir());
    println!("Readonly: {}", metadata.permissions().readonly());

    if let Ok(modified) = metadata.modified() {
        println!("Modified: {modified:?}");
    }

    fs::remove_file("example.txt")?;

    Ok(())
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++fs%3A%3Awrite%28%22example.txt%22%2C+%22Hello%2C+Rust%21%22%29%3F%3B%0A%0A++++let+metadata+%3D+fs%3A%3Ametadata%28%22example.txt%22%29%3F%3B%0A%0A++++println%21%28%22File+size%3A+%7B%7D+bytes%22%2C+metadata.len%28%29%29%3B%0A++++println%21%28%22Is+file%3A+%7B%7D%22%2C+metadata.is_file%28%29%29%3B%0A++++println%21%28%22Is+dir%3A+%7B%7D%22%2C+metadata.is_dir%28%29%29%3B%0A++++println%21%28%22Readonly%3A+%7B%7D%22%2C+metadata.permissions%28%29.readonly%28%29%29%3B%0A%0A++++if+let+Ok%28modified%29+%3D+metadata.modified%28%29+%7B%0A++++++++println%21%28%22Modified%3A+%7Bmodified%3A%3F%7D%22%29%3B%0A++++%7D%0A%0A++++fs%3A%3Aremove_file%28%22example.txt%22%29%3F%3B%0A++++Ok%28%28%29%29%0A%7D)

Некоторые дополнительные сведения доступны только через платформенные расширения. Например, Unix предоставляет режим доступа, UID и GID через `std::os::unix::fs::MetadataExt`.

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let metadata = fs::metadata("example.txt")?;

    #[cfg(unix)]
    {
        use std::os::unix::fs::MetadataExt;

        println!("Mode: {:o}", metadata.mode());
        println!("UID: {}", metadata.uid());
        println!("GID: {}", metadata.gid());
    }

    Ok(())
}
```

Для переносимого кода не следует рассчитывать на наличие Unix-специфичных полей на Windows.

---

## 53.8. Права доступа (`Permissions`)

Работа с правами доступа зависит от операционной системы.

На Unix-подобных системах можно работать с POSIX mode bits:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;

        fs::write("script.sh", "#!/bin/sh\necho Hello\n")?;

        let metadata = fs::metadata("script.sh")?;
        let mut permissions = metadata.permissions();

        let mode = permissions.mode();
        permissions.set_mode(mode | 0o111);

        fs::set_permissions("script.sh", permissions)?;

        println!("Executable permissions added");
        fs::remove_file("script.sh")?;
    }

    #[cfg(not(unix))]
    {
        println!("This example uses Unix-specific permissions.");
    }

    Ok(())
}
```

Обратите внимание на `#[cfg(unix)]`: это **условная компиляция**. Unix-специфичный код не компилируется на Windows.

Для переносимого кода можно использовать:

```rust
let permissions = metadata.permissions();

if permissions.readonly() {
    println!("File is readonly");
}
```

Однако `readonly()` — это абстрактная проверка, а не универсальное представление всех разрешений операционной системы.

---

## 53.9. Временные файлы

`std::env::temp_dir()` возвращает путь к системной директории для временных файлов, но **не создаёт автоматически временный файл и не удаляет его при завершении программы**.

Например:

```rust
use std::fs;
use std::io::Write;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let temp_dir = std::env::temp_dir();
    let temp_file = temp_dir.join("rust_example_temp.txt");

    let mut file = fs::File::create(&temp_file)?;
    writeln!(file, "Temporary data")?;

    println!("Created: {}", temp_file.display());

    // Удаляем файл явно.
    fs::remove_file(&temp_file)?;

    Ok(())
}
```

Если приложение должно безопасно создавать уникальные временные файлы и автоматически удалять их, лучше использовать специализированный crate, например `tempfile`.

Главная идея:

> `temp_dir()` сообщает, **где** обычно размещаются временные файлы. Управление жизненным циклом самого файла остаётся ответственностью программы.

---

## 53.10. Обработка ошибок файловой системы

Файловые операции могут завершаться разными видами ошибок. Их можно анализировать через `ErrorKind`.

```rust
use std::fs;
use std::io::{Error, ErrorKind};

fn read_config() -> Result<String, Error> {
    match fs::read_to_string("config.toml") {
        Ok(content) => Ok(content),

        Err(error) => match error.kind() {
            ErrorKind::NotFound => {
                Err(Error::new(
                    ErrorKind::NotFound,
                    "Configuration file was not found",
                ))
            }

            ErrorKind::PermissionDenied => {
                Err(Error::new(
                    ErrorKind::PermissionDenied,
                    "Permission denied while reading configuration",
                ))
            }

            _ => Err(error),
        },
    }
}

fn main() {
    match read_config() {
        Ok(content) => println!("Config:\n{content}"),

        Err(error) if error.kind() == ErrorKind::NotFound => {
            println!("Using default configuration");
        }

        Err(error) => {
            eprintln!("Configuration error: {error}");
        }
    }
}
```

На практике не всегда нужно вручную создавать новый `Error`. Если достаточно передать исходную ошибку выше по стеку вызовов, обычно проще использовать `?`:

```rust
use std::fs;
use std::io::Result;

fn read_config() -> Result<String> {
    fs::read_to_string("config.toml")
}
```

Это предпочтительный вариант, если вызывающий код сам знает, как обрабатывать ошибку.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0Ause+std%3A%3Aio%3A%3A%7BErrorKind%2C+Result%7D%3B%0A%0Afn+read_config%28%29+-%3E+Result%3CString%3E+%7B%0A++++fs%3A%3Aread_to_string%28%22config.toml%22%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++match+read_config%28%29+%7B%0A++++++++Ok%28content%29+%3D%3E+println%21%28%22Config%3A%5Cn%7Bcontent%7D%22%29%2C%0A++++++++Err%28error%29+if+error.kind%28%29+%3D%3D+ErrorKind%3A%3ANotFound+%3D%3E+%7B%0A++++++++++++println%21%28%22Using+default+configuration%22%29%3B%0A++++++++%7D%0A++++++++Err%28error%29+%3D%3E+eprintln%21%28%22Configuration+error%3A+%7Berror%7D%22%29%2C%0A++++%7D%0A%7D)

---

## 53.11. Практический пример: обработка CSV-файла

Простейший вариант можно реализовать через `split(',')`, но это **не полноценный CSV-парсер**. Такой подход ломается, например, если значение содержит запятую внутри кавычек:

```text
Alice,30,"Bangkok, Thailand"
```

Поэтому для реального CSV-файла лучше использовать специализированный crate `csv`.

### `Cargo.toml`

```toml
[dependencies]
csv = "1"
```

### Пример программы

```rust
use std::error::Error;

fn process_csv(
    input_path: &str,
    output_path: &str,
) -> Result<(), Box<dyn Error>> {
    let mut reader = csv::Reader::from_path(input_path)?;
    let mut writer = csv::Writer::from_path(output_path)?;

    // Копируем заголовок.
    writer.write_record(reader.headers()?)?;

    for result in reader.records() {
        let record = result?;

        let name = &record[0];
        let age: u32 = record[1].parse()?;
        let city = &record[2];

        // Совершеннолетний: 18 лет и старше.
        if age >= 18 {
            writer.write_record([name, &age.to_string(), city])?;
        }
    }

    writer.flush()?;

    Ok(())
}
```

Здесь библиотека `csv` правильно обрабатывает CSV-синтаксис, включая кавычки и запятые внутри полей.

Для учебного примера с простым CSV можно использовать `split(',')`, но необходимо явно понимать его ограничение:

```text
name,age,city
Alice,30,Bangkok
Bob,17,Pattaya
Charlie,18,Chiang Mai
```

В таком формате ручной разбор допустим, но для production-кода лучше использовать специализированный парсер.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Открытие несуществующего файла

```rust
use std::fs;

fn main() {
    match fs::read_to_string("nonexistent.txt") {
        Ok(content) => println!("{content}"),
        Err(error) => {
            println!("Error: {error}");
            println!("Kind: {:?}", error.kind());
        }
    }
}
```

Ожидаемый результат:

```text
Error: ...
Kind: NotFound
```

Точный текст ошибки зависит от операционной системы.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+main%28%29+%7B%0A++++match+fs%3A%3Aread_to_string%28%22nonexistent.txt%22%29+%7B%0A++++++++Ok%28content%29+%3D%3E+println%21%28%22%7Bcontent%7D%22%29%2C%0A++++++++Err%28error%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Error%3A+%7Berror%7D%22%29%3B%0A++++++++++++println%21%28%22Kind%3A+%7B%3A%3F%7D%22%2C+error.kind%28%29%29%3B%0A++++++++%7D%0A++++%7D%0A%7D)

### Эксперимент 2: Запись в недоступное место

Попытка записать файл в конкретную системную директорию **не является переносимым экспериментом**. Например, `/root` существует не на Windows, а права доступа зависят от среды выполнения.

Вместо этого можно использовать путь, который гарантированно содержит отсутствующего родителя:

```rust
use std::fs;
use std::io::ErrorKind;

fn main() {
    let result = fs::write(
        "directory_that_does_not_exist/file.txt",
        "data",
    );

    match result {
        Ok(()) => println!("File written"),
        Err(error) => {
            println!("Error kind: {:?}", error.kind());

            if error.kind() == ErrorKind::NotFound {
                println!("Parent directory does not exist");
            }
        }
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0Ause+std%3A%3Aio%3A%3AErrorKind%3B%0A%0Afn+main%28%29+%7B%0A++++let+result+%3D+fs%3A%3Awrite%28%22directory_that_does_not_exist%2Ffile.txt%22%2C+%22data%22%29%3B%0A%0A++++match+result+%7B%0A++++++++Ok%28%28%29%29+%3D%3E+println%21%28%22File+written%22%29%2C%0A++++++++Err%28error%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Error+kind%3A+%7B%3A%3F%7D%22%2C+error.kind%28%29%29%3B%0A++++++++++++if+error.kind%28%29+%3D%3D+ErrorKind%3A%3ANotFound+%7B%0A++++++++++++++++println%21%28%22Parent+directory+does+not+exist%22%29%3B%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D)

---

## Практика

### Задание 1

Напишите программу, которая создаёт текстовый файл, читает его построчно с помощью `BufReader` и выводит:

- количество строк;
- количество слов;
- количество символов.

Удалите созданный файл после завершения работы программы.

### Задание 2

Создайте программу, которая копирует большой текстовый файл с использованием `BufReader` и `BufWriter`.

Сравните её с вариантом, который использует `fs::read` и `fs::write`.

### Задание 3

Напишите рекурсивную функцию обхода директории:

```rust
fn visit_dir(path: &Path) -> std::io::Result<()>
```

Она должна:

- заходить во вложенные директории;
- выводить только файлы;
- показывать размер каждого файла.

### Задание 4

🔨 **Эксперимент с ошибками.**

Что произойдёт, если выполнить:

```rust
fs::write("missing/file.txt", "data")
```

Почему возникает `ErrorKind::NotFound`?

Затем исправьте программу с помощью:

```rust
fs::create_dir_all("missing")?;
```

### Задание 5

🔨 **Эксперимент с `Path`.**

Проверьте результаты:

```rust
let path = Path::new("archive.tar.gz");

println!("{:?}", path.file_name());
println!("{:?}", path.extension());
println!("{:?}", path.parent());
```

Обратите внимание, что `extension()` возвращает только последнюю часть расширения — `gz`, а не `tar.gz`.

### Задание 6

Напишите функцию:

```rust
fn copy_if_newer(
    source: &Path,
    destination: &Path,
) -> std::io::Result<()>
```

Она должна копировать файл только в том случае, если исходный файл новее существующего файла назначения.

### Задание 7

Используйте crate `tempfile` и реализуйте временный файл, который автоматически удаляется после выхода из области видимости.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`std::fs`** — основной API стандартной библиотеки для работы с файловой системой.
- **Чтение файлов** — `read_to_string`, `read`, `File`, `BufReader`.
- **Запись файлов** — `write`, `File::create`, `OpenOptions`, `BufWriter`.
- **Пути** — `Path` для заимствованного пути и `PathBuf` для владеющего изменяемого пути.
- **Директории** — `read_dir`, `create_dir`, `create_dir_all`, `remove_dir_all`.
- **Метаданные** — `metadata`, размер, тип, время изменения и права доступа.
- **Права доступа** — частично платформозависимы.
- **Буферизация** — эффективный способ последовательного чтения и записи больших файлов.
- **Ошибки** — файловые операции возвращают `Result`, а причины можно анализировать через `ErrorKind`.
- **Временные файлы** — `temp_dir()` только предоставляет директорию; автоматическое управление временем жизни требует дополнительного решения.
- **CSV** — для реальных CSV-файлов лучше использовать специализированный парсер, а не `split(',')`.

**Самая важная идея:**

> Работа с файловой системой — это работа с внешним состоянием, которое может измениться в любой момент. Файл может отсутствовать, директория может быть недоступна, путь может быть некорректным, а разрешения зависят от операционной системы. Поэтому Rust представляет файловые операции через `Result` и заставляет явно учитывать возможность ошибки. Хороший код не только умеет читать и записывать файлы, но и правильно работает с путями, ресурсами, буферизацией, ошибками и платформенными различиями.
