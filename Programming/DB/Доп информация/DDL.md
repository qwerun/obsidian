> [!info] **DDL (Data Definition Language)**
> Команды, изменяющие структуру объектов БД — схем, таблиц, индексов, типов и т.д.

## Базовые команды

| Команда | Назначение | Шаблон |
|---------|------------|--------|
| `CREATE`  | Создать объект | `CREATE TABLE ...` |
| `ALTER`   | Изменить объект | `ALTER TABLE ...` |
| `DROP`    | Удалить объект | `DROP TABLE ...` |
| `COMMENT` | Добавить описание | `COMMENT ON TABLE ...` |
| `TRUNCATE`| Быстро очистить таблицу | `TRUNCATE TABLE ...` |

### 1 ▪ CREATE

```sql
CREATE TABLE users (
  id       SERIAL PRIMARY KEY,
  email    TEXT UNIQUE NOT NULL,
  created  TIMESTAMPTZ DEFAULT now()
);
````

### 2 ▪ ALTER

```sql
ALTER TABLE users
ADD COLUMN is_active BOOLEAN DEFAULT true;
```

### 3 ▪ DROP

```sql
DROP TABLE IF EXISTS users CASCADE;
```

### 4 ▪ COMMENT

```sql
COMMENT ON COLUMN users.email IS 'Уникальный адрес пользователя';
```

### 5 ▪ TRUNCATE

```sql
TRUNCATE TABLE users RESTART IDENTITY;
```

> [!tip]  
> Используйте `IF EXISTS / IF NOT EXISTS`, чтобы DDL была идемпотентной.