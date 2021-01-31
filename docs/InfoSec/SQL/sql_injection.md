# SQL injection

## MySQL

```sql
-- show tables (user's only)

select table_name from information_schema.columns where table_schema not like 'mysql' \
and table_schema not like 'information_schema' and table_schema not like 'performance_schema';
```

---

[pentestmonkey - cheat sheet](http://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet)

---

{{ :it:infosec:en_-_blackhat_europe_2009_-_advanced_sql_injection_whitepaper.pdf |}}

{{ :it:infosec:en_-_blackhat_us_2006_sql_injections_by_truncation.pdf |}}

{{ :it:infosec:en_-_fast_blind_sql_injection.pdf |}}

{{ :it:infosec:en_-_sql_injection_in_insert_update_and_delete_statements.pdf |}}
