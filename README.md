# Лабораторная работа №8. Непрерывная интеграция с помощью Github Actions
## Студент
**Gachayev Dmitrii I2302**  
**Выполнено 19.04.2025**  
## Цель работы
В рамках данной работы студенты научатся настраивать непрерывную интеграцию с помощью Github Actions.
## Задание
Создать Web приложение, написать тесты для него и настроить непрерывную интеграцию с помощью Github Actions на базе контейнеров.
## Выполнение
## Создание Web приложения
1. Создаю следующую структуру:

![image](screenshots/Screenshot_1.png)

2. Реализую файл `modules/database.php` отталкиваясь от условий:

Файл `modules/database.php` содержит класс `Database` для работы с базой данных. Для работы с базой данных используйте `SQLite`. Класс должен содержать методы:

- `__construct($path)` - конструктор класса, принимает путь к файлу базы данных SQLite;
- `Execute($sql)` - выполняет SQL запрос;
- `Fetch($sql)` - выполняет SQL запрос и возвращает результат в виде ассоциативного массива.
- `Create($table, $data)` - создает запись в таблице $table с данными из ассоциативного массива $data и возвращает идентификатор созданной записи;
- `Read($table, $id)` - возвращает запись из таблицы $table по идентификатору $id;
- `Update($table, $id, $data)` - обновляет запись в таблице $table по идентификатору $id данными из ассоциативного массива $data;
- `Delete($table, $id)` - удаляет запись из таблицы $table по идентификатору $id.
- `Count($table)` - возвращает количество записей в таблице $table.

Итоговый файл (с документацией `PHPdoc`):
```php
<?php

class Database
{
    private PDO $pdo;

    /**
     * Конструктор класса.
     * @param string $path Путь к файлу базы данных SQLite.
     */
    public function __construct(string $path)
    {
        $this->pdo = new PDO("sqlite:" . $path);
        $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    }

    /**
     * Выполняет SQL-запрос (без получения результата).
     * @param string $sql SQL-запрос.
     */
    public function Execute(string $sql): void
    {
        $this->pdo->exec($sql);
    }

    /**
     * Выполняет SQL-запрос и возвращает результат в виде массива.
     * @param string $sql SQL-запрос.
     * @return array Ассоциативный массив с результатом.
     */
    public function Fetch(string $sql): array
    {
        $stmt = $this->pdo->query($sql);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    /**
     * Создаёт запись в таблице.
     * @param string $table Название таблицы.
     * @param array $data Ассоциативный массив данных.
     * @return int ID новой записи.
     */
    public function Create(string $table, array $data): int
    {
        $columns = implode(', ', array_keys($data));
        $placeholders = ':' . implode(', :', array_keys($data));
        $sql = "INSERT INTO $table ($columns) VALUES ($placeholders)";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($data);
        return (int)$this->pdo->lastInsertId();
    }

    /**
     * Читает запись из таблицы по ID.
     * @param string $table Название таблицы.
     * @param int $id Идентификатор записи.
     * @return array|null Ассоциативный массив данных или null, если не найдено.
     */
    public function Read(string $table, int $id): ?array
    {
        $sql = "SELECT * FROM $table WHERE id = :id";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute(['id' => $id]);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        return $result ?: null;
    }

    /**
     * Обновляет запись в таблице.
     * @param string $table Название таблицы.
     * @param int $id Идентификатор записи.
     * @param array $data Ассоциативный массив новых данных.
     */
    public function Update(string $table, int $id, array $data): void
    {
        $fields = implode(', ', array_map(fn($k) => "$k = :$k", array_keys($data)));
        $data['id'] = $id;
        $sql = "UPDATE $table SET $fields WHERE id = :id";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($data);
    }

    /**
     * Удаляет запись по ID.
     * @param string $table Название таблицы.
     * @param int $id Идентификатор записи.
     */
    public function Delete(string $table, int $id): void
    {
        $sql = "DELETE FROM $table WHERE id = :id";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute(['id' => $id]);
    }

    /**
     * Возвращает количество записей в таблице.
     * @param string $table Название таблицы.
     * @return int Количество записей.
     */
    public function Count(string $table): int
    {
        $sql = "SELECT COUNT(*) FROM $table";
        $stmt = $this->pdo->query($sql);
        return (int)$stmt->fetchColumn();
    }
}
```

3. Реализую файл `modules/page.php` исходя из условий:

Файл `modules/page.php` содержит класс `Page` для работы с страницами. Класс должен содержать методы:

- `__construct($template)` - конструктор класса, принимает путь к шаблону страницы;
- `Render($data)` - отображает страницу, подставляя в шаблон данные из ассоциативного массива $data.

Итоговый файл (c документацией `PHPdoc`):

```php
<?php
class Page
{
    private string $template;

    /**
     * Конструктор класса.
     * @param string $template Путь к файлу шаблона (.tpl).
     */
    public function __construct(string $template)
    {
        $this->template = $template;
    }

    /**
     * Подставляет данные в шаблон и возвращает HTML-строку.
     * @param array $data Ассоциативный массив с данными.
     * @return string HTML-код страницы.
     */
    public function Render(array $data): string
    {
        $template = file_get_contents($this->template);

        foreach ($data as $key => $value) {
            $template = str_replace('{{' . $key . '}}', htmlspecialchars($value), $template);
        }
        return $template;
    }
}
```

4. Реализую файл `templates/index.tpl` содержащий шаблон страницы:
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>{{title}}</title>
    <link rel="stylesheet" href="/styles/style.css">
</head>
<body>
    <h1>{{title}}</h1>
    <div class="content">
        {{content}}
    </div>
</body>
</html>
```

5. Реализую файл `styles/style.css` содержащий стили страницы:
```css
body {
    font-family: Arial, sans-serif;
    margin: 40px;
    background-color: #f9f9f9;
}

h1 {
    color: #333;
}

.content {
    padding: 10px;
    background-color: #fff;
    border: 1px solid #ccc;
}
```

6. Реализую файл `index.php` с кодом для отображения:
```php
<?php

require_once __DIR__ . '/modules/database.php';
require_once __DIR__ . '/modules/page.php';
require_once __DIR__ . '/config.php';

$db = new Database($config["db"]["path"]);

$page = new Page(__DIR__ . '/templates/index.tpl');

$pageId = $_GET['page'];

$data = $db->Read("page", $pageId);
echo $page->Render($data);
```

7. Реализую файл `config.php` с конфигурацией подключения к бд:

```php
<?php

$config = [
    "db" => [
        "path" => __DIR__ . "/../sql/schema.sql"
    ]
];

```

## Подготовка SQL файла для базы данных
1. Создаю в корневом каталоге директорию `/sql`.

2. Создаю файл `schema.sql` со следующим содержимым:
```sql
CREATE TABLE page (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT,
    content TEXT
);

INSERT INTO page (title, content) VALUES ('Page 1', 'Content 1');
INSERT INTO page (title, content) VALUES ('Page 2', 'Content 2');
INSERT INTO page (title, content) VALUES ('Page 3', 'Content 3');
```