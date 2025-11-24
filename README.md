# SYS_AUTO_GRANT_SELECT_PRIVELEGES

## Описание

`SYS_AUTO_GRANT_SELECT_PRIVELEGES` — это DAG в Apache Airflow, который автоматически выдает права `SELECT` на новые таблицы и представления в Greenplum пользователям, чьи роли соответствуют шаблону:


Основная задача — автоматизация выдачи прав на чтение для разработчиков и аналитиков, чтобы исключить необходимость ручного сопровождения и контроля новых объектов в аналитической базе.

---

## Расписание

DAG выполняется два раза в сутки по cron:


То есть ежедневно:

- в 06:00,
- в 13:00.

При этом:

- `catchup = false` — пропущенные запуски не догоняются,
- настроена повторная попытка при ошибках: до 2 раз, с интервалом в 1 минуту.

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
