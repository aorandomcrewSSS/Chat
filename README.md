# SYS_AUTO_GRANT_SELECT_PRIVELEGES

## Описание

`SYS_AUTO_GRANT_SELECT_PRIVELEGES` — это DAG в Apache Airflow, который автоматически выдает права `SELECT` на новые таблицы и представления в Greenplum пользователям, чьи роли соответствуют шаблону: *_develop_*

---

## Расписание

DAG выполняется два раза в сутки по cron:


То есть ежедневно:

- в 06:00,
- в 13:00.

---

## Подключения

DAG работает с двумя подключениями Airflow:

- `greenplum`
- `greenplum_cloud`

В рамках каждого подключения выполняются свои задачи по поиску таблиц и выдаче прав.

---

## Принцип работы

DAG состоит из двух основных шагов, которые выполняются последовательно для каждого подключения.

---

### 1. Поиск объектов для выдачи прав

Задача `get_table_name_<connection>` выполняет SQL-запрос, который определяет таблицы и представления, на которые ещё не выданы привилегии `SELECT` ролям, оканчивающимся на `_develop_`.

Объекты выбираются из схем:

- `grp_%`
- `dwh_%`
- `grp_dev`

При этом исключаются:

- партиции (`%_prt_%`)
- таблицы, на которые grant уже существует

Пример SQL-запроса:

```sql
select nms.nspname, cl.relname
from pg_class as cl
inner join pg_namespace nms on nms.oid = cl.relnamespace
    and (nms.nspname like 'grp_%'
        or nms.nspname like 'dwh_%'
        or nms.nspname = 'grp_dev')
left join information_schema.role_table_grants rtg
    on cl.relname = rtg.table_name
    and nms.nspname = rtg.table_schema
    and grantee like '%_develop_%'
where cl.relkind in ('r', 'v', 'm')
    and cl.relname not like '%_prt_%'
    and rtg.table_name is null;
```

Результат запроса имеет формат:
(schema_name, table_name)

### 2. Выдача прав
Задача grant_select_<connection> получает список выявленных объектов и для каждого из них выполняет вызов
`select dwh_meta.f_grant_default_privs('<schema>', '<table>');`

Эта функция отвечает за выдачу стандартных прав, включая SELECT, нужным пользователям.

После выполнения каждого SQL-вызова выполняется COMMIT.


# SYS_AUTO_REMOVE_HOSTS_LOGS

## Описание

`SYS_AUTO_REMOVE_HOSTS_LOGS` — это DAG в Apache Airflow, который автоматически очищает директорию системных логов Airflow от устаревших файлов и удаляет пустые директории.  

---

## Расписание

DAG выполняется каждые 3 дня, начиная с:

- 29.07.2025 11:00 UTC

---

## Среды выполнения

В зависимости от имени хоста применяется разный срок хранения логов:

| Hostname           | Среда | Срок хранения |
|--------------------|-------|---------------|
| `msk-dad-air-s1`   | DEV   | 15 дней       |
| `msk-pad-air-s1`   | PROD  | 30 дней       |

---

## Принцип работы

Задача `clear_logs`:

- определяет среду по hostname,
- выбирает срок хранения (`15` дней для DEV, `30` — для PROD),
- удаляет в `/var/log/airflow`:
  - файлы старше установленного срока,
  - пустые директории, оставшиеся после удаления файлов,
- выводит в лог статистику с количеством удалённых элементов.

Пример выполняемого кода:

```bash
if [ $(hostname) = 'msk-dad-air-s1' ]; then
    deleted_days=15
else
    deleted_days=30
fi

echo "INFO: Current host is $(hostname), files older than $deleted_days days will be deleted";

deleted_files=$(find /var/log/airflow -type f -mtime +$deleted_days -print -delete | wc -l)
echo "INFO: Deleted $deleted_files files";

deleted_dirs=$(find /var/log/airflow -type d -empty -print -delete | wc -l)
echo "INFO: Deleted $deleted_dirs empty directories";
```




