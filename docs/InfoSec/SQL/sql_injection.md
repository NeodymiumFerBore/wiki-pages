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

## Resources

[EN - Blackhat Europe 2009 - Advanced SQL injection whitepaper.pdf]({{ assets.path }}/EN - Blackhat Europe 2009 - Advanced SQL injection whitepaper.pdf)

[EN - Blackhat US 2006 _ SQL Injections by truncation.pdf]({{ assets.path }}/EN - Blackhat US 2006 _ SQL Injections by truncation.pdf)

[EN - FAST blind SQL Injection.pdf]({{ assets.path }}/EN - FAST blind SQL Injection.pdf)

[EN - SQL injection in insert, update and delete statements.pdf]({{ assets.path }}/EN - SQL injection in insert, update and delete statements.pdf)
