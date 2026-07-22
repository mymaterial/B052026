# PostgreSQL Logical Backup -- Development Activities

## Purpose

Logical backups are used for development, testing, migrations, and
object-level recovery.

-   Table-level backup
-   Schema-level backup
-   Database-level backup

## Table Backup

Export:

``` bash
pg_dump -U kfc_user -d kfc_db -t emp > emp.sql
```

Import:

``` bash
psql -U kfc_user -d kfc_db < emp.sql
```

## Exclude Table

``` bash
pg_dump -U postgres -d postgres -T t1 > t.sql
```

## Backup Formats

### Custom (-Fc)

``` bash
pg_dump -U kfc_user -d kfc_db -t emp -Fc -f emp.bkp
```

-   Binary format
-   Default compression
-   Selective restore
-   Restore using:

``` bash
pg_restore -U kfc_user -d kfc_db emp.bkp
```

### Plain (-Fp)

``` bash
pg_dump -U kfc_user -d kfc_db -t emp -Fp -f emp.sql
```

-   Human readable
-   Editable
-   Restore with psql

### Tar (-Ft)

``` bash
pg_dump -U kfc_user -d kfc_db -t emp -Ft -f emp.tar
```

-   OS tar format
-   Restore using pg_restore

### Directory (-Fd)

``` bash
pg_dump -U kfc_user -d kfc_db -t emp -Fd -f emp
```

-   Directory output
-   Supports parallel dump/restore
-   Compressed

Example restore:

``` bash
pg_restore -j 4 -U kfc_user -d kfc_db emp
```

## Schema Backup

``` bash
pg_dump -U kfc_user -d kfc_db -n sales > sales.sql
```

## Database Backup

``` bash
pg_dump -U kfc_user kfc_db > kfc_db.sql
```

## Editing SQL Dumps

### Small (\< Few MB)

-   Open in editor
-   Find & Replace

### Medium (20--30 GB)

Use sed:

``` bash
sed -i 's/old_schema/new_schema/g' dump.sql
```

### Very Large (\>500 GB)

-   Avoid editing directly
-   Restore to staging database
-   Make changes inside PostgreSQL
-   Re-export if required

## Best Practices

-   Use **-Fc** for most backups.
-   Use **-Fd** for large databases needing parallelism.
-   Use **-Fp** when SQL must be edited.
-   Periodically test restores.

## Format Comparison

  Format            Readable   Compression   Parallel   Restore
  ----------------- ---------- ------------- ---------- ------------
  Plain (-Fp)       Yes        No            No         psql
  Custom (-Fc)      No         Yes           No         pg_restore
  Tar (-Ft)         No         Optional      No         pg_restore
  Directory (-Fd)   No         Yes           Yes        pg_restore
